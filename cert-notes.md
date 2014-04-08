Create a self-signed certificate:
---------------------------------
    openssl genrsa -out private.key 2048
    openssl req -new -key private.key -out self-sign-request.csr
    openssl x509 -req -days 365 -in self-sign-request.csr -signkey private.key -out self-signed-cert.pem

Sign request using the self-signed certificate as a CA:
---------------------------------------------------------------
    openssl x509 -req -days 365 -in another-cert-request.csr -CA self-signed-cert.pem -CAkey private.key  -CAcreateserial -out another-cert.pem

Add a self-signed cert to Java's keystore
-----------------------------------------
    
If prompted, the default Java keystore password is "changeit"

    keytool -list -keystore /etc/ssl/certs/java/cacerts
    sudo keytool -keystore /etc/ssl/certs/java/cacerts -importcert -alias MyCert -file self-signed-cert.pem


Get a PEM certificate from a server
-------------------------------

    openssl x509 -in <(openssl s_client -showcerts -connect example.com:443 -prexit 2>/dev/null)
    
Get a fingerprint from a certificate
------------------------------------

    openssl x509 -hash -fingerprint -noout -in certificate.pem
