# Stupid OpenSSL Tricks

### Piped AES-256-CBC encrypt & HMAC-SHA256 of a file:

    $ openssl enc -e -aes-256-cbc -in INPUT_FILE -pass file:AES_KEY_FILE | \
        tee >(openssl dgst -sha256 -hmac ACTUAL_HMAC_KEY_VALUE -binary -out OUTPUT_HMAC) > \
        OUTPUT_CIPHERTEXT

### Piped HMAC-SHA256 computation & AES-256-CBC decryption of a file:

    $ cat OUTPUT_CIPHERTEXT | \
        tee >(openssl dgst -sha256 -hmac ACTUAL_HMAC_KEY_VALUE -binary > COMPUTED_HMAC) |
        openssl enc -d -aes-256-cbc -pass file:AES_KEY_FILE -out DECRYPTED_FILE

### Create a self-signed RSA-2048 certificate:

    $ openssl genrsa -out private.key 2048
    $ openssl req -new -key private.key -out self-sign-request.csr
    $ openssl x509 -req -days 365 -in self-sign-request.csr -signkey private.key -out self-signed-cert.pem

### Sign request using the self-signed certificate as a CA:

    $ openssl x509 -req -days 365 -in another-cert-request.csr \
        -CA self-signed-cert.pem -CAkey private.key  \
        -CAcreateserial -out another-cert.pem

### Get a PEM certificate from a server

    openssl x509 -in <(openssl s_client -showcerts -connect example.com:443 -prexit 2>/dev/null)
    
### Get a fingerprint from a certificate

    openssl x509 -hash -fingerprint -noout -in certificate.pem
