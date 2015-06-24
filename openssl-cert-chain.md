# OpenSSL Certificate Verification

This is a short walkthrough in how certificate verification works in Openssl.

### Fetch a certificate chain

Fetch some example TPM certificates in DER format:
      $ curl http://www.infineon.com/dgdl/IFX+TPM+EK+Root+CA.cer?fileId=db3a304412b407950112b4166b1a207b > IFX_TPM_EK_Root_CA.der
      $ curl http://www.infineon.com/dgdl/IFX_TPM_EK_Intermediate_CA_21.crt?fileId=5546d461464245d301466c6a9acf6393 > IFX_TPM_EK_Intermediate_CA_21.der

??? Get the Verizon TPM root CA (Can't find source)
VRSN_TPM_Root_CA.pem

Convert to PEM since that's what 'c_rehash' expects:
      $ openssl x509 -inform DER -in IFX_TPM_EK_Root_CA.der -outform PEM -out IFX_TPM_EK_Root_CA.pem
      $ openssl x509 -inform DER -in IFX_TPM_EK_Intermediate_CA_21.der -outform PEM -out IFX_TPM_EK_Intermediate_CA_21.pem

### Generating a CA directory

Run 'c_rehash' will create symbolic links using derived hash values as names:
      $ c_rehash .
      Doing .
      IFX_TPM_EK_Root_CA.pem => 8ea82df6.0
      VRSN_TPM_Root_CA.pem => 589f27ba.0
      IFX_TPM_EK_Intermediate_CA_21.pem => 4bb67ba6.0

Those filenames are derived from a hash of the certificate's subject name:
    $ openssl x509 -subject -noout -in IFX_TPM_EK_Root_CA.pem
    subject= /C=DE/ST=Bavaria/O=Infineon Technologies AG/OU=AIM/CN=IFX TPM EK Root CA
    $ openssl x509 -hash -noout -in IFX_TPM_EK_Root_CA.pem
    8ea82df6
    $ openssl x509 -hash -noout -in IFX_TPM_EK_Intermediate_CA_21.pem
    4bb67ba6

These hashes are of the use a canonicalized subject name. I haven't been able to reproduce that value from the command line with sha1sum. 
Here's the source that generates them: https://github.com/openssl/openssl/blob/OpenSSL_1_0_2-stable/crypto/x509/x509_cmp.c#L229

### Verifying certificates

Now you can verify the intermediate certificate:
    $ openssl verify -CApath . IFX_TPM_EK_Intermediate_CA_21.pem
    IFX_TPM_EK_Intermediate_CA_21.pem: OK


### How OpenSSL finds the next certificate in the CApath

OpenSSL verify starts with the certificate provided and looks in the CApath for a [hash].0 symlink pointing to another certificate.
This is derived from the "issuer" field in the given certificate:
    $ openssl x509 -noout -in IFX_TPM_EK_Intermediate_CA_21.pem -issuer
    issuer= /C=DE/ST=Bavaria/O=Infineon Technologies AG/OU=AIM/CN=IFX TPM EK Root CA

That issuer field is hashed to give the expected filename: 
    $ openssl x509 -noout -in IFX_TPM_EK_Intermediate_CA_21.pem -issuer_hash 
    8ea82df6

### How OpenSSL actually verifies the next certificate

The certificate to verify also has a longer fingerprint of the Authority, which is the same as the Issuer:
    $ openssl x509 -noout -in IFX_TPM_EK_Intermediate_CA_21.pem -text | grep -A 1 "Authority Key Identifier"
        X509v3 Authority Key Identifier:
            keyid:56:EB:91:44:85:63:D6:72:B3:AE:D4:45:96:0B:F7:94:0E:54:42:A6

That value will be the same as the [hash].0 file's key identifier:
    $ openssl x509 -noout -in 8ea82df6.0 -text | grep -A 1 "Subject Key Identifier"
       X509v3 Subject Key Identifier:
            56:EB:91:44:85:63:D6:72:B3:AE:D4:45:96:0B:F7:94:0E:54:42:A6

That needs to be cryptographically verified. It's actually derived from a hash of ASN.1 encoded public key bits in the certificate.
You can see this as follows:
    $ openssl x509 -noout -in 8ea82df6.0 -pubkey | openssl asn1parse
        0:d=0  hl=4 l= 290 cons: SEQUENCE
        4:d=1  hl=2 l=  13 cons: SEQUENCE
        6:d=2  hl=2 l=   9 prim: OBJECT            :rsaEncryption
       17:d=2  hl=2 l=   0 prim: NULL
       19:d=1  hl=4 l= 271 prim: BIT STRING

asn1parse can grab the "BIT STRING" portion and output it as a separate file. Note the offset '19':
    $ openssl x509 -noout -in 8ea82df6.0 -pubkey | openssl asn1parse -strparse 19 -out pub.der

This value matches the Subject Key Identifier from above:
    $ openssl dgst -c -sha1 pub.der
SHA1(pub.der)= 56:eb:91:44:85:63:d6:72:b3:ae:d4:45:96:0b:f7:94:0e:54:42:a6
