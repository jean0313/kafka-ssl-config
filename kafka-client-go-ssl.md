# Kafka client go SSL config

## Prepare key/cert

#### 1. Generate client cert
```shell
export PASSWORD=XXXXXX

keytool -exportcert -alias localhost \
    -keystore kafka.server.keystore.jks \
    -rfc \
    -file cert.pem \
    -storepass $PASSWORD
```

#### 2. Generate client key
```shell
keytool -importkeystore \
    -srckeystore kafka.server.keystore.jks \
    -srcalias localhost \
    -destkeystore ck.p12 \
    -deststoretype PKCS12 \
    -storepass $PASSWORD \
    -srcstorepass $PASSWORD

openssl pkcs12 -in ck.p12 \
    -nodes \
    -nocerts \
    -out key.pem \
    -passin pass:$PASSWORD
```

#### 3. Generate RootCA
```shell
keytool -exportcert \
    -alias localhost \
    -keystore kafka.server.keystore.jks \
    -rfc \
    -file RootCA.pem \
    -storepass $PASSWORD
```

## Test
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"os"

	"github.com/IBM/sarama"
)

var (
	broker = "127.0.0.1:9093"
	cert   = "../ssl/cert.pem"
	key    = "../ssl/key.pem"
	rootCA = "../ssl/RootCA.pem"
)

func main() {
	admin, err := sarama.NewClusterAdmin([]string{broker}, createConfig())
	if err != nil {
		panic(err)
	}

	topics, err := admin.ListTopics()
	if err != nil {
		panic(err)
	}
	fmt.Printf("%v\n", topics)
}

func createConfig() *sarama.Config {
	tlsConfig, err := newTLSConfig(cert, key, rootCA)
	if err != nil {
		panic(err)
	}

	config := sarama.NewConfig()
	config.Net.TLS.Enable = true
	config.Net.TLS.Config = tlsConfig
	return config
}

func newTLSConfig(clientCertFile, clientKeyFile, caCertFile string) (*tls.Config, error) {
	tlsConfig := tls.Config{
		InsecureSkipVerify: true,
	}

	// Load client cert
	cert, err := tls.LoadX509KeyPair(clientCertFile, clientKeyFile)
	if err != nil {
		return &tlsConfig, err
	}
	tlsConfig.Certificates = []tls.Certificate{cert}

	// Load CA cert
	caCert, err := os.ReadFile(caCertFile)
	if err != nil {
		return &tlsConfig, err
	}
	caCertPool := x509.NewCertPool()
	caCertPool.AppendCertsFromPEM(caCert)
	tlsConfig.RootCAs = caCertPool
	return &tlsConfig, err
}
```