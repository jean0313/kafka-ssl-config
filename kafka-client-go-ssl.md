# Kafka client go SSL config

## Prepare key/cert

### Option 1. Create key/cert from keystore
```shell
export PASSWORD=XXXXXX

# Generate client cert
keytool -exportcert -alias localhost \
    -keystore kafka.server.keystore.jks \
    -rfc \
    -file cert.pem \
    -storepass $PASSWORD

# Generate client key
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

# Generate RootCA
keytool -exportcert \
    -alias localhost \
    -keystore kafka.server.keystore.jks \
    -rfc \
    -file RootCA.pem \
    -storepass $PASSWORD
```

### Option 2. Create key/cert from scratch
```shell

# Create new key, csr
openssl req \
       -newkey rsa:2048 -nodes -keyout domain.key \
       -out domain.csr \
       -subj "/C=CN/ST=CN/L=CN/O=CN/CN=localhost" \
       -passin pass:$PASSWORD -passout pass:$PASSWORD

# Sign csr with ca
openssl x509 -req -in domain.csr -CA ca-cert -CAkey ca-key -CAcreateserial -out domain.crt -passin pass:$PASSWORD
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
	cert   = "../ssl/domain.crt"
	key    = "../ssl/domain.key"
	rootCA = "../ssl/ca-cert"
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