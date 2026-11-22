# Lab: MirrorMaker 2 on Kubernetes with Strimzi (TLS, OpenSSL certs)

Strimzi 1.x (`kafka.strimzi.io/v1`). Namespace `mm2-demo`. Source cluster uses your OpenSSL CA; target cluster uses Strimzi's CA.

## Step 0: Prepare the terminal

```bash
bind 'set enable-bracketed-paste off'
mkdir -p ~/mm2-lab && cd ~/mm2-lab
```

Create files with `vi`: `vi file` → `:set paste` → `i` → paste → `Esc` → `:wq`. To clear a file: `Esc`, then `ggdG`.

## Step 1: Install Strimzi

```bash
kubectl create namespace mm2-demo
kubectl config set-context --current --namespace=mm2-demo
kubectl create -f 'https://strimzi.io/install/latest?namespace=mm2-demo' -n mm2-demo
```

```bash
kubectl wait deployment/strimzi-cluster-operator -n mm2-demo --for=condition=Available --timeout=300s
```

Check the API version (must print `v1`):

```bash
kubectl get crd kafkas.kafka.strimzi.io -o jsonpath='{.spec.versions[*].name}{"\n"}'
```

## Step 2: Create certificates (OpenSSL)

CA:

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/O=MM2 Demo/CN=mm2-demo-root-ca" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -out ca.crt
```

Broker certificate:

```bash
openssl genrsa -out broker.key 2048
openssl req -new -key broker.key -subj "/O=MM2 Demo/CN=source-cluster-kafka-bootstrap" -out broker.csr
```

`vi broker.ext` (the `subjectAltName` line is one line):

```
basicConstraints=CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=DNS:source-cluster-kafka-bootstrap,DNS:source-cluster-kafka-bootstrap.mm2-demo,DNS:source-cluster-kafka-bootstrap.mm2-demo.svc,DNS:source-cluster-kafka-bootstrap.mm2-demo.svc.cluster.local,DNS:*.source-cluster-kafka-brokers.mm2-demo.svc,DNS:*.source-cluster-kafka-brokers.mm2-demo.svc.cluster.local
```

```bash
openssl x509 -req -in broker.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 825 -sha256 -extfile broker.ext -out broker.crt
```

MM2 client certificate:

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/O=MM2 Demo/CN=mm2-user" -out client.csr
```

`vi client.ext`:

```
basicConstraints=CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=clientAuth
```

```bash
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 825 -sha256 -extfile client.ext -out client.crt
```

Convert keys to PKCS#8, build the chain, verify:

```bash
openssl pkcs8 -topk8 -nocrypt -in ca.key -out ca.pkcs8.key
openssl pkcs8 -topk8 -nocrypt -in broker.key -out broker.pkcs8.key
openssl pkcs8 -topk8 -nocrypt -in client.key -out client.pkcs8.key
cat broker.crt ca.crt > broker-chain.crt
openssl verify -CAfile ca.crt broker.crt client.crt
```

## Step 3: Create secrets

```bash
kubectl create secret generic source-broker-tls -n mm2-demo \
  --from-file=tls.crt=broker-chain.crt --from-file=tls.key=broker.pkcs8.key
kubectl create secret generic source-ca -n mm2-demo --from-file=ca.crt=ca.crt
kubectl create secret generic source-user -n mm2-demo \
  --from-file=user.crt=client.crt --from-file=user.key=client.pkcs8.key
```

Use your CA as the source cluster's clients CA:

```bash
kubectl create secret generic source-cluster-clients-ca-cert -n mm2-demo --from-file=ca.crt=ca.crt
kubectl create secret generic source-cluster-clients-ca -n mm2-demo --from-file=ca.key=ca.pkcs8.key
kubectl label secret source-cluster-clients-ca-cert source-cluster-clients-ca -n mm2-demo \
  strimzi.io/kind=Kafka strimzi.io/cluster=source-cluster
kubectl annotate secret source-cluster-clients-ca-cert -n mm2-demo strimzi.io/ca-cert-generation=0
kubectl annotate secret source-cluster-clients-ca -n mm2-demo strimzi.io/ca-key-generation=0
```

```bash
kubectl get secrets -n mm2-demo
```

Expect five: `source-broker-tls`, `source-ca`, `source-user`, `source-cluster-clients-ca-cert`, `source-cluster-clients-ca`.

## Step 4: Source cluster

`vi kafka-source.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  labels:
    strimzi.io/cluster: source-cluster
spec:
  replicas: 1
  roles: [controller, broker]
  storage:
    type: ephemeral
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: source-cluster
spec:
  clientsCa:
    generateCertificateAuthority: false
  kafka:
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
        configuration:
          brokerCertChainAndKey:
            secretName: source-broker-tls
            certificate: tls.crt
            key: tls.key
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
```

```bash
kubectl apply -n mm2-demo -f kafka-source.yaml --dry-run=server
kubectl apply -n mm2-demo -f kafka-source.yaml
```

```bash
kubectl wait kafka/source-cluster -n mm2-demo --for=condition=Ready --timeout=600s
```

## Step 5: Target cluster and MM2 user

`vi kafka-target.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
  labels:
    strimzi.io/cluster: target-cluster
spec:
  replicas: 1
  roles: [controller, broker]
  storage:
    type: ephemeral
---
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: target-cluster
spec:
  kafka:
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1
kind: KafkaUser
metadata:
  name: mm2-target-user
  labels:
    strimzi.io/cluster: target-cluster
spec:
  authentication:
    type: tls
```

```bash
kubectl apply -n mm2-demo -f kafka-target.yaml --dry-run=server
kubectl apply -n mm2-demo -f kafka-target.yaml
```

```bash
kubectl wait kafka/target-cluster -n mm2-demo --for=condition=Ready --timeout=600s
```

```bash
kubectl wait kafkauser/mm2-target-user -n mm2-demo --for=condition=Ready --timeout=120s
```

## Step 6: MirrorMaker 2

In `v1`, MM2 uses `spec.target` and `mirrors[].source` (no `clusters` or `connectCluster`), and `target` needs the group ID and three storage topics.

`vi mm2.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1
kind: KafkaMirrorMaker2
metadata:
  name: demo-mm2
spec:
  replicas: 1
  target:
    alias: target
    bootstrapServers: target-cluster-kafka-bootstrap:9093
    groupId: mirrormaker2-cluster
    configStorageTopic: mirrormaker2-cluster-configs
    offsetStorageTopic: mirrormaker2-cluster-offsets
    statusStorageTopic: mirrormaker2-cluster-status
    tls:
      trustedCertificates:
        - secretName: target-cluster-cluster-ca-cert
          certificate: ca.crt
    authentication:
      type: tls
      certificateAndKey:
        secretName: mm2-target-user
        certificate: user.crt
        key: user.key
    config:
      config.storage.replication.factor: 1
      offset.storage.replication.factor: 1
      status.storage.replication.factor: 1
  mirrors:
    - source:
        alias: source
        bootstrapServers: source-cluster-kafka-bootstrap:9093
        tls:
          trustedCertificates:
            - secretName: source-ca
              certificate: ca.crt
        authentication:
          type: tls
          certificateAndKey:
            secretName: source-user
            certificate: user.crt
            key: user.key
      topicsPattern: ".*"
      groupsPattern: ".*"
      sourceConnector:
        tasksMax: 2
        config:
          replication.factor: 1
          offset-syncs.topic.replication.factor: 1
          refresh.topics.interval.seconds: 10
          sync.topic.configs.enabled: "true"
      checkpointConnector:
        tasksMax: 1
        config:
          checkpoints.topic.replication.factor: 1
          sync.group.offsets.enabled: "true"
          sync.group.offsets.interval.seconds: 10
          emit.checkpoints.interval.seconds: 10
          refresh.groups.interval.seconds: 10
```

```bash
kubectl apply -n mm2-demo -f mm2.yaml --dry-run=server
kubectl apply -n mm2-demo -f mm2.yaml
```

```bash
kubectl wait kmm2/demo-mm2 -n mm2-demo --for=condition=Ready --timeout=300s
```

## Step 7: Verify TLS

```bash
kubectl logs demo-mm2-mirrormaker2-0 -n mm2-demo --tail=60 | grep -iE "ssl|handshake|error" || echo "no TLS errors"
```

```bash
kubectl get kmm2 demo-mm2 -n mm2-demo -o jsonpath='{.status.connectors}' | python3 -m json.tool
```

Connectors should be `RUNNING`.

## Step 8: Produce data on the source

```bash
SRC="kubectl exec -i -n mm2-demo source-cluster-dual-role-0 --"
TGT="kubectl exec -i -n mm2-demo target-cluster-dual-role-0 --"
```

```bash
$SRC bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic orders --partitions 3
```

Run the next command as one line by itself:

```bash
seq 1 10 | sed 's/^/order-/' | $SRC bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
```

```bash
$SRC bin/kafka-get-offsets.sh --bootstrap-server localhost:9092 --topic orders
```

The three offsets add up to 10.

## Step 9: Check replication on the target

```bash
sleep 20
$TGT bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

Expect `source.orders`.

```bash
$TGT bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic source.orders --from-beginning --timeout-ms 15000
```

All 10 messages print. The `TimeoutException` at the end is normal.

## Step 10: Consumer group offset sync

```bash
$SRC bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --group billing --from-beginning --max-messages 6 --timeout-ms 15000 > /dev/null
```

```bash
sleep 40
$TGT bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group billing
```

Group `billing` appears on `source.orders` with offsets adding up to 6. If empty, wait 60 seconds and repeat.

## Step 11: Live replication

```bash
seq 11 15 | sed 's/^/order-/' | $SRC bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
```

```bash
sleep 10
$TGT bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic source.orders --from-beginning --timeout-ms 15000 2>/dev/null | wc -l
```

Expected: `15`.

## Step 12: Cleanup

```bash
kubectl delete kmm2 demo-mm2 -n mm2-demo
kubectl delete kafkauser mm2-target-user -n mm2-demo
kubectl delete kafka source-cluster target-cluster -n mm2-demo
kubectl delete kafkanodepool --all -n mm2-demo
```

```bash
kubectl delete -f 'https://strimzi.io/install/latest?namespace=mm2-demo' -n mm2-demo
kubectl delete namespace mm2-demo
```

Optional, removes Strimzi CRDs cluster-wide:

```bash
kubectl get crd -o name | grep strimzi.io | xargs -r kubectl delete
```

```bash
cd ~ && rm -rf ~/mm2-lab
```

## Common errors

| Error | Fix |
|---|---|
| `no matches for kind ... v1beta2` | Use `apiVersion: kafka.strimzi.io/v1` |
| `no matches for kind ... CRDs` | Operator not installed: redo Step 1 |
| `unknown field` on dry-run | Run `kubectl explain <kind>.spec --api-version=kafka.strimzi.io/v1` |
| Pasted YAML corrupted | Step 0: paste mode off, use `vi` with `:set paste` |
| `No subject alternative names matching` | Add the hostname to `broker.ext`, redo Steps 2 and 3 |
| `source.orders` missing | Check Step 7 connectors and MM2 logs |
