MULTIDOMAIN REVERSE-PROXY NGINX SETUP WITH CERTBOT AND CLASSIC CERT
-

This sample project describes setup of multi-domain but single-site web application with nginx reverse-proxy.
It consists if docker-compose.yml, which describes this project from docker perspective.
There are necessary ports, directories and finally, containers.

Nginx container from jonasal/nginx-certbot. Thanks to this - container will automatically generates a cert
and continuously update it if necessary. Automation! Thnx to jonasal.

This setup will pass correct certificate for two different domains, but all requests will be passed to one final web
application.


_____________________
docker-compose description
_____________________
```
version: '3.1'

services:
  site: 
    container_name: sample_web_app
    networks:
      - site_net
    build:
      context: ./dock
    command: java -jar sample.jar
    restart: always
  frontend:
    container_name: nginx-certbot
    restart: unless-stopped
    networks:
      - site_net
    image: jonasal/nginx-certbot
    ports:
      - 80:80/tcp
      - 443:443/tcp
    environment:
      CERTBOT_EMAIL: admin@sampledomain.com
    volumes:
      - ./conf.d:/etc/nginx/user_conf.d:ro
      - ./letsencrypt:/etc/letsencrypt
      - ./certs:/etc/certs
networks:
  site_net:
    ipam:
      config:
        - subnet: "172.16.32.0/24"
```
As we can see - there are two descripted services:
- site
- frontend

Site describes sample_web-Application made on java with build context for Docker.

You're unable to run sample project, until you'll replace this with working one. Probably, I will replace it with
another sample.
There are no any published ports, only network - it is necessary to join nginx and web later.


Frontend describes nginx reverse-proxy jonasal/nginx-certbot.

- "site-net" in networks connects this container to the same network, as site does.
- ports - we're publishing 80 andd 443. 80 is necsessary to force redirect to an ssl port - redirect is described in nginx.conf
- environment - certbot container will parse it and use domain after "@" as a target to get and install certificate. For more details please visit origin docs for jonasal/nginx-certbot
- volumes:
  - ./conf.d - This folder is necessary for nginx configuration. It has ngninx.conf inside with the descriptions of domains and other options.
    
  - ./letsencrypt - This folder is done for permanent storage of letsencrypt certificates. After starting container you're able to find yours letsencrypt cert here.
    
  - ./certs - This folder is for storing your own certificate for your second domain.


_________________

 NGINX configuration:

```
server {
    listen 80;
    server_name sampledomain.com;
    return 301 https://sampledomain.com;
}

server {
    listen              443 ssl default_server;
    listen              [::]:443 ssl default_server;

    server_name         sampledomain.com;

    ssl_certificate /etc/letsencrypt/live/sampledomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sampledomain.com/privkey.pem;


    location / {
        client_max_body_size 20M;
        proxy_pass http://site:80/;
        proxy_set_header Origin '';
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
server {
    listen              443 ssl reuseport;
    listen              [::]:443 ssl reuseport;

    server_name         xn--d1abbjucifekd.xn--p1ai;

    ssl_certificate /etc/certs/xn--d1abbjucifekd.xn--p1ai/primer_rf.ca-bundle;
    ssl_certificate_key /etc/certs/xn--d1abbjucifekd.xn--p1ai/primer_rf_key.pem;


    location / {
        client_max_body_size 20M;
        proxy_pass http://site:80/;
        proxy_set_header Origin '';
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}



```
Describes proxying requests for backend domain. Replies for incoming SSL with necessary requests depending on the domain
in headers.

It has 3 server statements:
- Listen on 80 and force 301 redirect to SSL(443) port.
- Listen on 443 for sampledomain.com requests and passes sampledomain.com certificate. SSL certificate is described in this
section, and this server statement uses letsencrypt cert. Finally, it is proxying requests to tcp/80 "site" container.
- Listen on 443 for xn--d1abbjucifekd.xn--p1ai requests and passes xn--d1abbjucifekd.xn--p1ai certificate. SSL certificate is described in this
section, and this server statement uses your own certificate. Finally, it is proxying requests to tcp/80 "site"
  container.
  
Also, there are some additional options for nginx that were convenient for my Setup.