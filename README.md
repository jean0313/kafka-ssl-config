# Kafka SSL config

## Prepare truststore and keystore

#### 1.CA certificate
```shell
export PASSWORD=XXXXXX

openssl req \
    -new -x509 \
    -keyout ca-key \
    -out ca-cert \
    -days 365 \
    -passin pass:$PASSWORD \
    -passout pass:$PASSWORD \
    -subj "/C=CN/ST=CN/L=CN/O=CN/CN=RootCA"
```

#### 2.Truststore
```shell
keytool -keystore kafka.server.truststore.jks \
    -alias CARoot \
    -import \
    -file ca-cert \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 3.Keystore for kafka broker
```shell
keytool -keystore kafka.server.keystore.jks \
    -alias localhost \
    -validity 365 \
    -genkey \
    -keyalg RSA \
    -dname "CN=127.0.0.1, OU=CN, O=CN, L=CN, S=CN, C=CN" \
    -storepass $PASSWORD \
    -keypass $PASSWORD
```

#### 4.CSR
```shell
keytool -keystore kafka.server.keystore.jks \
    -alias localhost \
    -certreq \
    -file cert-file \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 5.SSL certificate
```shell
openssl x509 -req \
    -CA ca-cert \
    -CAkey ca-key \
    -in cert-file \
    -out cert-signed \
    -days 365 \
    -CAcreateserial \
    -passin pass:$PASSWORD
```

#### 6.Import ca-cert into keystore
```shell
keytool -keystore kafka.server.keystore.jks \
    -alias CARoot \
    -import \
    -file ca-cert \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 7.Import ssl certificate into keystore
```shell
keytool -keystore kafka.server.keystore.jks \
    -alias localhost \
    -import \
    -file cert-signed \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 8.Truststore for client
```shell
keytool -keystore kafka.client.truststore.jks \
    -alias CARoot \
    -import \
    -file ca-cert \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 9.Keystore for client
```shell
keytool -keystore kafka.client.keystore.jks \
    -alias localhost \
    -validity 365 \
    -genkey \
    -keyalg RSA \
    -dname "CN=127.0.0.1, OU=CN, O=CN, L=CN, S=CN, C=CN" \
    -storepass $PASSWORD \
    -keypass $PASSWORD
```

#### 10.Client CSR
```shell
keytool -keystore kafka.client.keystore.jks \
    -alias localhost \
    -certreq \
    -file cert-file.client \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 11.Client SSL certificate
```shell
openssl x509 -req \
    -CA ca-cert \
    -CAkey ca-key \
    -in cert-file.client \
    -out cert-signed.client \
    -days 365 \
    -CAcreateserial \
    -passin pass:$PASSWORD
```

#### 12.Import ca-cert into client keystore
```shell
keytool -keystore kafka.client.keystore.jks \
    -alias CARoot \
    -import \
    -file ca-cert \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```

#### 13.Import ssl certificate into client keystore
```shell
keytool -keystore kafka.client.keystore.jks \
    -alias localhost \
    -import \
    -file cert-signed.client \
    -storepass "$PASSWORD" \
    -keypass "$PASSWORD" \
    -noprompt
```



## Config

#### zookeeper: zookeeper-ssl.properties
```properties
secureClientPort=3183
secureClientPortAddress=127.0.0.1
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory

ssl.trustStore.location=/ssl/kafka.client.truststore.jks
ssl.trustStore.password=changeit
ssl.keyStore.location=/ssl/kafka.client.keystore.jks
ssl.keyStore.password=changeit
ssl.endpoint.identification.algorithm=
```

#### kafka: server-ssl.properties
```properties
listeners=SSL://127.0.0.1:9093

ssl.truststore.location=/ssl/kafka.server.truststore.jks
ssl.truststore.password=changeit
ssl.keystore.location=/ssl/kafka.server.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.type=JKS
ssl.keystore.type=JKS
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.endpoint.identification.algorithm=
security.inter.broker.protocol=SSL

advertised.listeners=SSL://127.0.0.1:9093


# for zookeeper
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
zookeeper.ssl.client.enable=true
zookeeper.client.secure=true
zookeeper.ssl.hostnameVerification=false

zookeeper.ssl.truststore.location=/ssl/kafka.client.truststore.jks$$
zookeeper.ssl.truststore.password=changeit
zookeeper.ssl.keystore.location=/ssl/kafka.client.keystore.jks
zookeeper.ssl.keystore.password=changeit
zookeeper.ssl.key.password=changeit
zookeeper.ssl.truststore.type=JKS
zookeeper.ssl.keystore.type=JKS
```

#### kafka console client: client-ssl.properties
```properties
bootstrap.servers=127.0.0.1:9093
security.protocol=SSL
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1

ssl.truststore.location=/ssl/kafka.client.truststore.jks
ssl.truststore.password=changeit
ssl.keystore.location=/ssl/kafka.client.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.endpoint.identification.algorithm=
```

## Run
```shell
# start zookeeper
bin/zookeeper-server-start.sh config/zookeeper-ssl.properties

# start kafka
bin/kafka-server-start.sh config/server-ssl.properties

# start client
bin/kafka-topics.sh --bootstrap-server localhost:9093 --command-config config/client-ssl.properties --list
```
