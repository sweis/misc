Create a self-signed certificate:
---------------------------------
    openssl genrsa -out private.key 2048
    openssl req -new -key private.key -out self-sign-request.csr
    openssl x509 -req -days 365 -in self-sign-request.csr -signkey private.key -out self-signed-cert.pem

Sign request using the self-signed certificate as a CA:
---------------------------------------------------------------
    openssl x509 -req -days 365 -in another-cert-request.csr -CA self-signed-cert.pem -CAkey private.key  -CAcreateserial -out another-cert.pem
