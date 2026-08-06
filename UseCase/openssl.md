```sh
#Basic

#Encrypt a file
##common cipher name: -aes-256-cbc (cbc is mode), see more in man
#openssl enc <ciphername> [<key-derivation-function> -salt] -in <infile> -out <outfile>
openssl enc -aes-256-cbc -pbkdf2 -salt -in <infile> -out <outfile>

#Decrypt file: add `-d` after cipher

#Check digest between encrypted and origin files if they are identical
openssl dgst -sha256 <file>

---
#Pub-priv key

#Gen private key
#openssl genpkey -algorithm <algorithm> -out <outfile> -pkeyopt <>
openssl genpkey -algorithm rsa -out private.pem -pkeyopt rsa_keygen_bits:2048

#Gen public key from private key
openssl rsa -pubout -in private.pem -out public.pem

#Encrypt message with public key:
openssl pkeyutl -encrypt -inkey public.pem -pubin -in message.txt -out message.enc

#Decrypt with private key:
openssl pkeyutl -decrypt -inkey private.pem -in message.enc -out message_decrypted.txt

---
#Signing

#Sign the message with private key:
openssl dgst -sha256 -sign private.pem -out message.sig message.txt

#Verify signature with public key:
openssl dgst -sha256 -verify public.pem -signature message.sig message.txt

---
#Creating and Inspecting X.509 Certificates with a Chain

#Step 1: Create Root CA (self-signed)
openssl genpkey -algorithm RSA -out rootCA.key -pkeyopt rsa_keygen_bits:4096

openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem -subj "/C=AU/ST=NSW/L=Wollongong/O=ExampleRootCA/OU=RootCA/CN=Example Root CA"

#Step 2: Create Intermediate CA (signed by Root CA)
openssl genpkey -algorithm RSA -out intermediate.key -pkeyopt rsa_keygen_bits:4096

openssl req -new -key intermediate.key -out intermediate.csr \
-subj "/C=AU/ST=NSW/L=Wollongong/O=ExampleIntermediateCA/OU=Intermediate/CN=Example
Intermediate CA"

openssl x509 -req -in intermediate.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial \
-out intermediate.pem -days 1825 -sha256 -extfile <(printf
"basicConstraints=CA:TRUE\nkeyUsage=critical,keyCertSign,cRLSign")

#Step 3: Create Server Certificate (signed by Intermediate CA)
openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048

openssl req -new -key server.key -out server.csr \
-subj "/C=AU/ST=NSW/L=Wollongong/O=ExampleServer/OU=IT/CN=www.example.com"

openssl x509 -req -in server.csr -CA intermediate.pem -CAkey intermediate.key -CAcreateserial \
-out server.pem -days 825 -sha256 \
-extfile <(printf
"basicConstraints=CA:FALSE\nkeyUsage=digitalSignature,keyEncipherment\nsubjectAltName=DNS:www.ex
ample.com")

# Step 4: Verify the Certificate Chain
openssl verify -CAfile <(cat rootCA.pem intermediate.pem) server.pem

#Step 5: Inspect Certificates
openssl x509 -in server.pem -noout -text
openssl x509 -in intermediate.pem -noout -text
openssl x509 -in rootCA.pem -noout -text

```