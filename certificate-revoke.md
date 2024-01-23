
## Certificate Revoke

```shell
# init
mkdir demoCA
touch ./demoCA/index.txt
echo 00 > ./demoCA/crlnumber

export PASSWORD=changeit
# create root ca
openssl req -new -x509 \
       -keyout ca-key \
       -out ca-cert \
       -days 365 \
       -passin pass:$PASSWORD \
       -passout pass:$PASSWORD \
       -subj "/C=CN/ST=CN/L=CN/O=CN/CN=localhost"

# create client key and csr
openssl req \
       -newkey rsa:2048 -nodes -keyout domain.key \
       -out domain.csr \
       -subj "/C=CN/ST=CN/L=CN/O=CN/CN=localhost" \
       -passin pass:$PASSWORD -passout pass:$PASSWORD

# sign csr with ca
openssl x509 -req \
       -in domain.csr \
       -CA ca-cert \
       -CAkey ca-key \
       -CAcreateserial \
       -out domain.crt \
       -passin pass:$PASSWORD

# perform revoke client crt
openssl ca -revoke domain.crt -keyfile ca-key -cert ca-cert

# generate crl
openssl ca -gencrl -keyfile ca-key -cert ca-cert -out pulp_crl.pem

# test revoked crt
cat ca-cert pulp_crl.pem > test.pem
openssl verify -extended_crl -verbose -CAfile test.pem -crl_check domain.crt
```