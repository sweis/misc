# Stupid OpenSSL Tricks

### Piped encrypt & HMAC of a file:
   
   openssl enc -e -aes-256-cbc -in INPUT_FILE -pass file:AES_KEY_FILE | tee >(openssl dgst -sha256 -hmac ACTUAL_HMAC_KEY_VALUE -binary -out OUTPUT_HMAC) > OUTPUT_CIPHERTEXT

### Piped HMAC computation & decryption of a file:
   
   cat OUTPUT_CIPHERTEXT | tee >(openssl dgst -sha256 -hmac ACTUAL_HMAC_KEY_VALUE -binary > COMPUTED_HMAC) | openssl enc -d -aes-256-cbc -pass file:AES_KEY_FILE -out DECRYPTED_FILE
