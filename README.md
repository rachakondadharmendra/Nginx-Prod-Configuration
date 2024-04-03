# Optimizing Server Performance: Nginx's Reverse Proxy Configuration

This documentation walks you through setting up NGINX to serve as a reverse proxy for a Jenkins server. Beginning with prerequisites like having Ubuntu OS 20.04 or 22.04 installed and a registered domain name with DNS records configured, it then provides step-by-step instructions for installing NGINX and Certbot, configuring NGINX, obtaining SSL certificates, and optimizing performance with optional caching.

## Prerequisites
Before proceeding with the NGINX setup, ensure the following prerequisites are met:
- Ubuntu OS 20.04 or 22.04 is installed on your server.
- A domain name is registered and DNS records are configured, pointing to your server's IP address.

## Installation
### Step 1: Install NGINX and Certbot
```bash
sudo apt-get update -y && sudo apt-get dist-upgrade -y
sudo apt install nginx -y
sudo snap install core; sudo snap refresh core; 
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
### Step 2: Set up NGINX Configuration
```bash

sudo mv /etc/nginx/ /etc/nginx-previous
sudo git clone https://github.com/h5bp/server-configs-nginx.git /etc/nginx/
sudo systemctl restart nginx
cd /etc/nginx/conf.d
sudo mv no-ssl.default.conf no-ssl.default.conf.bkp
```

### Step 3: Run the server on a domain name
After configuring NGINX to serve as a reverse proxy for Jenkins, you need to create a configuration file for your domain and then reload NGINX.

```bash
sudo touch jenkins.rachakondadharmendra.info.conf && sudo vi jenkins.rachakondadharmendra.info.conf
sudo nginx -t
sudo service nginx reload
```
```nginx
#/etc/nginx/conf.d/jenkins.rachakondadharmendra.info.conf
# NGINX configuration for Jenkins reverse proxy

upstream jenkins_server {
    server localhost:8080;
}

server {
    listen 80;
    listen [::]:80;
    server_name www.jenkins.rachakondadharmendra.info jenkins.rachakondadharmendra.info;

    access_log /var/log/nginx/jenkins.access.log;
    error_log /var/log/nginx/jenkins.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass http://jenkins_server;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## SSL Certificate Setup
### Step 4: Obtain SSL Certificates
```bash
sudo mkdir -p /var/www/certbot

# Obtain certificates using webroot plugin
sudo certbot certonly --webroot -w /var/www/certbot \
    -d jenkins.rachakondadharmendra.info \
    -d www.jenkins.rachakondadharmendra.info \
    --post-hook "sudo service nginx reload" \
    --non-interactive --agree-tos \
    --email rachakondadharmendrainfo@gmail.com \
    --force-renewal

# Alternatively, obtain certificates using nginx plugin
sudo certbot certonly --nginx \
    -d jenkins.rachakondadharmendra.info \
    -d www.jenkins.rachakondadharmendra.info \
    --post-hook "sudo service nginx reload" \
    --non-interactive --agree-tos \
    --email rachakondadharmendrainfo@gmail.com \
    --force-renewal
```

## NGINX Configuration for Jenkins
### Step 5: Configure NGINX for Jenkins
After SSL is done we need a configuration file for Jenkins that will able to use https, so you need to edit it and configure NGINX as a reverse proxy.

```nginx
#/etc/nginx/conf.d/jenkins.rachakondadharmendra.info.conf
# NGINX configuration for Jenkins reverse proxy

upstream jenkins_server {
    server localhost:8080;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name www.jenkins.rachakondadharmendra.info jenkins.rachakondadharmendra.info;

    # Access and error logs
    access_log /var/log/nginx/jenkins.access.log;
    error_log /var/log/nginx/jenkins.error.log;

    # Proxy settings
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass http://jenkins_server;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;

        # Proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SSL configuration
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    include h5bp/tls/ssl_engine.conf;
    ssl_certificate /etc/letsencrypt/live/jenkins.rachakondadharmendra.info/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.rachakondadharmendra.info/privkey.pem;
    include h5bp/tls/policy_balanced.conf;
}

server {
    listen 80;
    listen [::]:80;
    server_name www.jenkins.rachakondadharmendra.info jenkins.rachakondadharmendra.info;
    return 301 https://jenkins.rachakondadharmendra.info$request_uri;
}
```

### Step 6: Reload NGINX Configuration
Once the NGINX configuration file is updated, you need

 to verify the configuration and reload NGINX.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Adding Cache to NGINX
### Step 7: Configure NGINX Cache (Optional)
To add caching to NGINX for better performance, you need to create a cache directory, adjust permissions, and edit the NGINX configuration file.

```bash
sudo mkdir -p /var/cache/nginx
sudo chown -R www-data:www-data /var/cache/nginx
sudo chmod -R 770 /var/cache/nginx
sudo vi /etc/nginx/conf.d/jenkins.rachakondadharmendra.info.conf
```

```nginx
#/etc/nginx/conf.d/jenkins.rachakondadharmendra.info.conf
# NGINX configuration with caching for Jenkins reverse proxy

upstream jenkins_server {
    server localhost:8080;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=jenkins_cache:10m max_size=1g inactive=60m use_temp_path=off;
proxy_cache_key "$scheme$request_method$host$request_uri";

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name www.jenkins.rachakondadharmendra.info jenkins.rachakondadharmendra.info;

    access_log /var/log/nginx/jenkins.access.log;
    error_log /var/log/nginx/jenkins.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass http://jenkins_server;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache jenkins_cache;
        proxy_cache_valid 200 302 1h;
        proxy_cache_valid 404 1m;
    }

    include h5bp/tls/ssl_engine.conf;
    ssl_certificate /etc/letsencrypt/live/jenkins.rachakondadharmendra.info/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.rachakondadharmendra.info/privkey.pem;
    include h5bp/tls/policy_balanced.conf;
}

server {
    listen 80;
    listen [::]:80;
    server_name www.jenkins.rachakondadharmendra.info jenkins.rachakondadharmendra.info;
    return 301 https://jenkins.rachakondadharmendra.info$request_uri;
}
```

## Conclusion
After completing all the steps, your NGINX server should be properly configured with SSL integration and optional caching for your Jenkins server. Make sure to test your setup thoroughly to ensure everything is working as expected.

## Resources
- [SSL Labs](https://www.ssllabs.com/ssltest/analyze.html?d=rachakondadharmendra.info)
- [Best NGINX Setup Resource](https://www.linode.com/docs/guides/getting-started-with-nginx-part-1-installation-and-basic-setup/)
- [YouTube Video](https://www.youtube.com/watch?v=RbJLplHRKCM&t=209s)
- [GitHub Example](https://github.com/koddr/example-static-website-docker-nginx-certbot/)
- [SSL Configuration](https://www.redswitches.com/blog/certbot-nginx/)
