#### [反向代理Web Service](https://blog.csdn.net/mn960mn/article/details/50716768)

```json
server {  
    listen  6633;  
    location / {  
        proxy_set_header Host $host:$server_port;  
        proxy_pass http://192.168.100.95:6633;  
    }  
} 
```

Linux下编译安装

./configure --with-stream --prefix=/home/yebk/yebk/nginx;make;make install;