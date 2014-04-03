1. Generate a self-signed certificate.
Inputs:
	Target directory (/etc/apache2/ssl)
	Domain name (example.com)

# mkdir /etc/apache2/ssl
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -subj "/CN=example.com" -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt

2. Enable the Apache SSL module and generate the default-ssl file:
# a2enmod ssl
# service apache2 restart

3. Configure SSL settings

Inputs:
	Target directory (/etc/apache2/ssl)
	Domain name (example.com)

Edit "/etc/apache2/sites-available/default-ssl"

Add this line after ServerAdmin:
ServerName example.com:443

Add or edit these lines:
SSLEngine on
SSLProtocol ALL -SSLv2
SSLHonorCipherOrder On
SSLCipherSuite "ECDH+AESGCM DH+AESGCM ECDH+AES256 DH+AES256 ECDH+AES128 DH+AES ECDH+3DES DH+3DES RSA+AESGCM RSA+AES !aNULL !eNULL !LOW !3DES !MD5 !PSK !SRP !DSS"
SSLCompression Off
SSLCertificateFile /etc/apache2/ssl/apache.crt
SSLCertificateKeyFile /etc/apache2/ssl/apache.key

4. Enable the default-ssl site:
# a2ensite default-ssl

5. Redirect HTTP to HTTPS.
Inputs:
  Domain name (example.com)
	
Replace contents of "/etc/apache2/sites-available/default"

<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        ServerName example.com
        RedirectPermanent / https://example.com/
</VirtualHost>

6. Restart Apache:
# service apache2 reload
