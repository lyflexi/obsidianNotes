
```Shell
server{
    listen 80;
    server_name localhost;
    proxy_connect_timeout 12000;
    proxy_send_timeout 12000;
    proxy_read_timeout 12000;
    location / {
        proxy_pass http://distributedLock;    
    }
}
```