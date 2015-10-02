# Stupid OpenSSL Tricks

### Piped AES-256-CBC encrypt & HMAC-SHA256 of a file:

'openssl enc' doesn't support -aes-256-cbc-hmac-sha1. This encrypts an input file and pipes it through tee to 'openssl digest -hmac' to compute an HMAC. Unfortunately, the dgst command can't take a key file as input and needs the actual key value to be passed in as a parameter. I think this is bash-specific syntax as well:

    $ openssl enc -e -aes-256-cbc -in INPUT_FILE -pass file:AES_KEY_FILE | \
        tee >(openssl dgst -sha256 -hmac ACTUAL_HMAC_KEY_VALUE -binary -out OUTPUT_HMAC) > \
        OUTPUT_CIPHERTEXT

### Piped HMAC-SHA256 computation & AES-256-CBC decryption of a file:

This takes a ciphertext, pipes it through tee to compute an HMAC, then pipes through 'openssl enc' to decrypt the file. It's assumed you have the given HMAC (which is OUTPUT_HMAC from above) and can compare it to the computed value. This lets you decrypt and authenticate in one pass, rather than having to save the ciphertext locally first.

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

### Generate and sign data with ECDSA P-384 SHA-256

    $ openssl ecparam -genkey -name secp384r1 -noout -out private.pem
    $ openssl ec -in private.pem -pubout -out public.pem
    read EC key
    writing EC key
    $ echo "Hello world" > input.txt
    $ echo "Some junk" > junk.txt
    $ openssl dgst -sha256 -sign private.pem input.txt > signature.bin
    $ openssl dgst -sha256 -verify public.pem -signature signature.bin input.txt
    Verified OK
    $ openssl dgst -sha256 -verify public.pem -signature signature.bin junk.txt
    Verification Failure
    $ dd if=/dev/zero of=badsig.bin bs=1 count=17
    17+0 records in
    17+0 records out
    17 bytes (17 B) copied, 0.0288165 s, 0.6 kB/s
    $ openssl dgst -sha256 -verify public.pem -signature badsig.bin input.txt
    Error Verifying Data
    140460728022840:error:0D0680A8:asn1 encoding routines:ASN1_CHECK_TLEN:wrong tag:tasn_dec.c:1342:
    140460728022840:error:0D07803A:asn1 encoding routines:ASN1_ITEM_EX_D2I:nested asn1 error:tasn_dec.c:391:Type=ECDSA_SIG
    $ openssl dgst -sha256 -sign private.pem junk.txt > signature2.bin
    $ openssl dgst -sha256 -verify public.pem -signature signature2.bin input.txt
    Verification Failure
