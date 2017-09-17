# How to configure FileBeat and Logstash with SSL mutual authentication.

How to configure SSL for FileBeat and Logstash step by step with OpenSSL (Create CA, CSRs, Certificates, etc).

The Elasticsearch documentation "[Securing Communication With Logstash by Using SSL](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-ssl-logstash.html)" does not show how to create with openssl the necessary keys and certificates to have the mutual authentication between FileBeat (output) and Logstash (input). It is not a difficult task but it can be very tedious if you are not familiar with the use of openssl.

These are some errors that can be found in the FileBeat and Logstash logs when SSL is not properly configured.
```
# FileBeat.

ERR Failed to publish events caused by: EOF
ERR Connecting error publishing events (retrying): remote error: tls: handshake failure
ERR Failed to publish events caused by: read tcp X.X.X.X:XXXXX->X.X.X.X:XXXX: i/o timeout


# Logstash.

Exception: javax.net.ssl.SSLHandshakeException: error:100000b8:SSL routines:OPENSSL_internal:NO_SHARED_CIPHER
[ERROR][logstash.inputs.beats    ] Looks like you either have an invalid key or your private key was not in PKCS8 format. {:exception=>java.lang.IllegalArgumentException: File does not contain valid private key: /XXXX/XXXX.key}
[INFO ][org.logstash.beats.BeatsHandler] Exception: javax.net.ssl.SSLHandshakeException: error:10000418:SSL routines:OPENSSL_internal:TLSV1_ALERT_UNKNOWN_CA
[INFO ][org.logstash.beats.BeatsHandler] Exception: javax.net.ssl.SSLHandshakeException: General OpenSslEngine problem
```

These are the steps to configure Filebeat with Logstash using SSL, mutual authentication and TLS 2.0 encryption.
**Tested in Logstash / Filebeat version**: 5.6

### Logstash input beat configuration (files ca.crt,server.crt, and server.key are needed).
```
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate_authorities => ["/etc/ca.crt"]
    ssl_certificate => "/etc/server.crt"
    ssl_key => "/etc/server.key"
    ssl_verify_mode => "force_peer"
  }
}
```

### Filebeat output (SSL) configuration (files ca.crt, client.crt and client.key are needed).
```
output.logstash:
  hosts: ["logs.mycompany.com:5044"]
  ssl.certificate_authorities: ["/etc/ca.crt"]
  ssl.certificate: "/etc/client.crt"
  ssl.key: "/etc/client.key"
# ssl.key_passphrase: "PASSWORD"
  ssl.supported_protocols: "TLSv1.2"
```

### CA (create files ca.key and ca.crt).
```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt
```

### Logstasg server (create server.key and server.crt).

File server.conf
```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName                     = XX
stateOrProvinceName             = XXXXXX
localityName                    = XXXXXX
postalCode                      = XXXXXX
organizationName                = XXXXXX
organizationalUnitName          = XXXXXX
commonName                      = XXXXXX
emailAddress                    = XXXXXX

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = DOMAIN_1
DNS.2 = DOMAIN_2
DNS.3 = DOMAIN_3
DNS.4 = DOMAIN_4
```

```bash
openssl genrsa -out server.key 2048
openssl req -sha512 -new -key server.key -out server.csr -config server.conf
echo "C2E9862A0DA8E970" > serial
openssl x509 -days 3650 -req -sha512 -in server.csr -CAserial serial -CA ca.crt -CAkey ca.key -out server.crt -extensions v3_req -extfile server.conf
mv server.key server.key.pem && openssl pkcs8 -in server.key.pem -topk8 -nocrypt -out server.key
```

### FileBeat shipper (create files client.key and client.crt).

File client.conf.
```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
 
[req_distinguished_name]
countryName                     = XX
stateOrProvinceName             = XXXXXX
localityName                    = XXXXXX
postalCode                      = XXXXXX
organizationName                = XXXXXX
organizationalUnitName          = XXXXXX
commonName                      = XXXXXX
emailAddress                    = XXXXXX

[ usr_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, server
nsComment = "OpenSSL FileBeat Server / Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment, keyAgreement, nonRepudiation
extendedKeyUsage = serverAuth, clientAuth

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
```
```
openssl genrsa -out client.key 2048
openssl req -sha512 -new -key client.key -out client.csr -config client.conf
openssl x509 -days 3650 -req -sha512 -in client.csr -CAserial serial -CA ca.crt -CAkey ca.key -out client.crt -extensions v3_req -extensions usr_cert  -extfile client.conf
```
```
# If the client key is not encrypted by passphrase, it can always be added later (filebeat "ssl.key_passphrase").
# openssl rsa -des -in client.key -out client4.key
```
