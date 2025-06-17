
# ⚙️ Requirements
---
- SSH access to ubuntu server.
- Domain pointing to a server's public IP.
- Git and Node.js installed on server.
- Nginx installed.
- SSL certificate (Cloudflare).

# 💡 Guide step by step
---
1. 🔁 Clone repository:
```sh
cd /var/www
git clone https://github.com/your_user/your_repo.git project-name
cd project-name
```

2. ⛏️ Install project dependencies:
```sh
npm install
```

3. 🏃🏻‍♂️ Build project for production:
```sh
npm run build
```
Verify that `dist/` folder is created.

3. 🔑 Adjust permissions for Nginx:
```sh
sudo chown -R $USER:www-data dist
sudo chmod -R 750 dist
```

4. ⚙️ Configure Nginx to serve app:
```sh
sudo nano /etc/nginx/sites-available/my-site.conf
```

.config example:
```sh
server {
    listen 80;
    server_name example.com www.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/ssl/cloudflare.crt;
    ssl_certificate_key /etc/ssl/cloudflare.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        root /home/username/project-name/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

5.  💻  Enable site:
```sh
sudo ln -s /etc/nginx/sites-available/my-site /etc/nginx/sites-enabled/
```

6.  🔃 Test and reload Nginx:
```sh
sudo nginx -t
sudo systemctl reload nginx
```

7. 🌐  Visit site:
```sh
https://example.com
```


# To update your site later
---
```sh
cd /home/username/project-name
git pull origin main
npm install
npm run build
sudo systemctl reload nginx
```


