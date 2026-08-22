# Asymmetric 
```sh
#Create private key 
openssl genpkey -algorithm RSA -out private-key.pem -pkeyopt rsa_keygen_bits:4096

#Create public key
openssl rsa -in private-key.pem -pubout -out public-key.pem

#Read property of key
openssl rsa -in private-key.pem -text -noout
```
