Creating a PKI with Picnic Signatures and Keys
----------------------------------------------
These instructions explain how to create a small public key infrastructure
(PKI) for OpenVPN server(s) and client(s) using the Picnic post-quantum
signature algorithm.  We create a CA that issues one or more server
certificates, and optionally client certificates.  If clients use password
authentication they do not require certificates. Servers must use a certificate
and the CA certificate is required by all clients in order to validate the
server certificate.

For ease of setting up a PKI in an experimental environment, these instructions generate
all credentials on a single system, and then the user is responsible for transporting them
to the client and server systems. In a production scenario, private keys should be generated
on the systems where they will be used and never transported between systems. Instead,
certificate signing requests (CSRs) and signed certificates should be sent between clients,
servers, and the CA. These instructions are not meant to reflect best practices for creating
a production PKI.

The `gencerts.sh` script in this directory is provided as a way of quickly creating credentials.
Open it in an editor and customize the variables before attempting to use.

## Setup and create CA
The Linux build package contains the OQS version of the OpenSSL command line tool (`oqs-openssl-output/openssl/bin/openssl`)
that we can use to create Picnic X509 certificates. 
Update it's search path so that it points to the OQS fork of OpenSSL.
```
    chrpath -r /usr/local/oqssl/lib ./openssl
```
(this assumes you've moved the OQS-OpenSSL libraries to `/usr/local/oqssl/lib`).

If this doesn't work, you can try the patchelf tool:
```
    patchelf --set-rpath ./openssl
```
Now to create a Picnic private key for the CA:
```
    ./openssl genpkey -algorithm picnicl1fs -out ca.key 
```
Then create a self-signed root certificate:
```
    ./openssl req -new -x509 -days 365 -key ca.key -out ca.crt -sha512 -subj /CN=PQ-OpenVPN-Demo-CA -config /etc/ssl/openssl.cnf
```
Note we have to specify the location of the openssl config file since our build
was done with a nonstandard prefix.  We don't need anything specific from it,
so you can also use the default `/etc/ssl/openssl.cnf` on most systems, or the
one included in `pq-openvpn-linux.tgz`. If none is specified, openssl will
output a warning, but everything should proceed. Also, currently Picnic certificates must use SHA-512. 

## Create a server certificate

Create a Picnic private key and CSR for the server:
```
    ./openssl req -new -newkey picnicl1fs -keyout server.key -out server.csr -sha512 -batch -subj /CN=PQ-OpenVPN-Demo-Server -config /etc/ssl/openssl.cnf
```

Sign the server certificate from the CSR with the CA's key:
```
   ./openssl x509 -req -in server.csr -out server.crt -CA ca.crt -days 365 -CAkey ca.key -CAcreateserial -sha512 -extensions server -extfile cert-exts 
```
The file `openvpn/config/keys/picnic/cert-exts` must be in the current (or edit the above command to include the correct path).
The file should contain:
```
    [ server ]
    keyUsage=digitalSignature
    extendedKeyUsage=serverAuth
    
    [ client ]
    keyUsage=digitalSignature
    extendedKeyUsage=clientAuth
```
Note that with some versions of OQS this command may crash while exiting, but the server certificate is created correctly.  

View the server cert
```
    ./openssl x509 -in server.crt -noout -text|head -n40
```

Check that the server cert is valid:
```
    ./openssl verify -CAfile ca.crt -verbose server.crt
```

## (Optional) Create a client certificate

Create a Picnic private key and CSR for the client :
```
    ./openssl req -new -newkey picnicl1fs -keyout client.key -out client.csr -sha512 -batch -subj /CN=PQ-OpenVPN-Demo-client
```
Sign the client certificate from the CSR with the CA's key:
```
   ./openssl x509 -req -in client.csr -out client.crt -CA ca.crt -days 365 -CAkey ca.key -CAcreateserial -sha512 -extensions client -extfile cert-exts 
```
View and verify the client cert
```
    ./openssl x509 -in client.crt -noout -text|head -n40
    ./openssl verify -CAfile ca.crt -verbose client.crt
```


