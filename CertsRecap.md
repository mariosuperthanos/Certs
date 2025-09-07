### Part 1 - Setting up the Certificate Authority (CA) and the public key 

1. Generate the CA's Private Key

```bash
openssl genrsa -nodes -out ca.key 2048
```
What it does: This command generates a 2048-bit RSA `private key`.
What you get: A file named `ca.key`. This file is the `CA`'s most valuable secret. It is the cryptographic "stamp" that the `CA` will use to sign other people's `certificates`. It must be kept `secure`.
``` key
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC69V5...
... (A whole bunch of random, unreadable characters) ...
... (This is the actual key data, a massive string of ones and zeros) ...
...
w9cM+E6F4N4J3fA4pXpX0z8t8eT2Wj9pG2i8u3j3F6wA4xY4R8c...
-----END PRIVATE KEY-----
```

2. Create the Self-Signed CA Certificate
```bash
openssl req -new -x509 -days 365 -key ca.key -out ca.crt -subj "/CN=My Test CA"
```
What it does: This command creates a certificate for the `CA` itself. The -x509 flag tells openssl to create a `self-signed certificate`, meaning it uses the `CA`'s own private key (`ca.key`) to sign its own `certificate`. The /CN (Common Name) is the identity of this `CA`.

What you get: A file named `ca.crt`. This file contains the `CA's public key` and all the identifying information, signed by the `CA`'s own private key. This is `the file you will give to clients and servers so they can verify signatures made by the CA`.
``` yaml
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            f0:a2:67:84:a5:3b:0c:41:a2:44:81:4e:a3:b4:e5:08
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = My Org, CN = My Test CA
        Validity
            Not Before: Sep  7 14:40:00 2025 GMT
            Not After : Sep  7 14:40:00 2026 GMT
        Subject: C = US, O = My Org, CN = My Test CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c6:f6:ab:12:12:ef:e1:8a:b3:f1:9c:e1:d3:02:
                    ... (hex data) ...
                    ff:01:8b:2d:c2:d3:1c:e1:d0:c4:44:b3:d7:1a:28:
                    f2:c6:8c:13:9e:f2:d8:a4:f1:e7:a1:8f:67:7c:b6:
                    8d:9c:e7:96:e0:5d:0f:70:86:d6:b6:79:d5:b3:b8:
                    1b:13:3b:3d:e0:72:e3:11:c1:0d:f7:7e:b8:e2:39:
                    ... (hex data) ...
                Exponent: 65537 (0x10001)
    X509v3 extensions:
        X509v3 Subject Key Identifier: 
            F5:81:79:D5:49:12:21:C1:2A:43:F6:14:55:0C:C0:CA:E3:6C:54:19
        X509v3 Authority Key Identifier: 
            keyid:F5:81:79:D5:49:12:21:C1:2A:43:F6:14:55:0C:C0:CA:E3:6C:54:19

        X509v3 Basic Constraints: critical
            CA:TRUE
Signature Algorithm: sha256WithRSAEncryption
     1d:2c:5a:33:76:3b:ad:24:94:7b:8a:d1:43:c5:89:7c:9a:bb:
     ... (hex signature data) ...
```

### Part 2 - Creating the Server's Certificate
The server wants to prove its identity to clients. It needs a `certificate` signed by the trusted `CA`.

1. Generate the Server's Private Key
```bash
openssl genrsa -out server.key 2048
```

What it does: Just like the `CA`, the server needs its own unique, secret private key. This key will be used to prove the server's identity during the handshake.

What you get: A file named `server.key`. This is the server's secret, and it must never be shared.

2. Create a Certificate Signing Request (CSR)
```bash
openssl req -new -key server.key -out server.csr -subj "/CN=myserver.com"
```
What it does: The server doesn't have the authority to create its own valid certificate. So, it creates a request for a trusted party (the `CA`) to sign one for it. This command packages the `server's public key` and its identity information (myserver.com) into a request file.

What you get: A file named `server.csr`. This is a formal request. It's sent to the `CA`. It does not contain the server's private key.

3. Have the `CA` Sign the Server's CSR
```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

What it does: This is the most crucial step. You, as the `CA`, take the server's request (`server.csr`). You use the `CA's private key` (`ca.key`) to cryptographically sign the request. This signature says, "I, the `CA`, officially vouch that this `public key` belongs to myserver.com."

What you get: A file named server.crt. This is the finished, official certificate that the server will present to clients. It contains the server's public key and the CA's trusted signature.

### Part 3: Creating the Client's Certificate (for Mutual TLS)
In a mutual TLS setup, the client (e.g., a microservice or a kubelet) also needs to prove its identity. The process is identical to the server's.

1. Generate the Client's Private Key
```bash
openssl genrsa -out client.key 2048
```
What you get: A file named `client.key`. The client's unique secret.

2.Create a Certificate Signing Request (CSR)
```bash
openssl req -new -key client.key -out client.csr -subj "/CN=myclient-user"
```

What you get: A file named `client.csr`. The client's request to the CA.

3. Have the CA Sign the Client's CSR
```bash
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
```
What you get: A file named `client.crt`. The client's finished, official certificate with the CA's trusted signature.

### Part 4: Verifying the Certificates
This is the final check, and it's what happens during a handshake.

1. Verify the Server's Certificate
```bash
openssl verify -CAfile ca.crt server.crt
```
What it does: This command simulates what a client does. It takes the CA's certificate (`ca.crt`) and uses the public key inside it to check the digital signature on the server's certificate (server.crt).

The outcome: If the signatures match, it returns OK. This proves the certificate is authentic and has not been tampered with.

2. Verify the Client's Certificate
```bash
openssl verify -CAfile ca.crt client.crt
```
What it does: The same process, but this is what the server does to verify the client's certificate during a mutual TLS handshake. It ensures the client is legitimate.