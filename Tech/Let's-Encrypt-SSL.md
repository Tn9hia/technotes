```bash
sudo apt-get install letsencrypt

certbot certonly --manual --preferred-challenges=dns --email webmaster@avanet.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d avanet.com -d *.avanet.com
```

```
sudo certbot certonly --webroot -w /var/www/html -d diastudio.com.vn -d www.diastudio.com.vn

```
