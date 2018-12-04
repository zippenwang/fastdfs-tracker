# fastdfs-tracker
基于[ygqygq2/fastdfs-nginx](https://hub.docker.com/r/ygqygq2/fastdfs-nginx/)，修复部分bug，并将其拆分成fastdfs-storage和fastdfs-tracker两个镜像，此篇文章是介绍fastdfs-tracker。

### 特点
- 为tracker节点整合了nginx，作为各个storage节点中nginx的反向代理；
- 将nginx的端口号作为环境变量，创建容器时指定。

### 环境变量
- TRACKER_SERVER：tracker的ip及port，格式为ip:port，用于设置client.conf中的tracker_server，方便监控和调试（可选）；
- NGINX_PORT：nginx的端口号，默认为80（可选）；

### tracker中的重要目录
- /var/fdfs：日志、元信息的存储路径，包含data，logs两个子目录，需要将该目录挂载出来；
- /usr/local/nginx/conf/conf.d：nginx配置文件所在目录，可将该目录挂载出来，在其中创建配置文件，添加反向代理功能；

### Demo
```
docker run -dti --network=host --name tracker -e TRACKER_SERVER=ip:port -e NGINX_PORT=20000 -v $PWD/tracker:/var/fdfs zippenwang/fastdfs-tracker
docker run -dti --network=host --name tracker -e TRACKER_SERVER=ip:port -v $PWD/tracker:/var/fdfs -v $PWD/tracker_nginx:/usr/local/nginx/conf/conf.d zippenwang/fastdfs-tracker
```

### tracker中的nginx
#### 作用
以统一的路口访问fastdfs集群下不同group中的文件资源，无需为不同的group切换不同的storage节点ip。

#### 方法一：进入容器中修改
创建tracker容器时，指定NGINX_PORT参数，如：
```
docker run -dti --network=host --name tracker -e TRACKER_SERVER=ip:port -e NGINX_PORT=20000 -v $PWD/tracker:/var/fdfs zippenwang/fastdfs-tracker
```
容器启动后，进入容器中（docker attach -it tracker /bin/bash），修改nginx的配置文件/usr/local/nginx/conf/conf.d/tracker.conf，为tracker中的nginx设置反向代理。

#### 方法二：将nginx的配置文件挂载出来
事先准备好nginx的配置文件tracker.conf，假设存储在xxx/tracker_nginx路径下，再创建tracker容器，此时无需指定NGINX_PORT参数，因为nginx的端口由tracker.conf文件中的配置决定，如：
```
docker run -dti --network=host --name tracker -e TRACKER_SERVER=ip:port -v $PWD/tracker:/var/fdfs -v $PWD/tracker_nginx:/usr/local/nginx/conf/conf.d zippenwang/fastdfs-tracker
```

#### nginx反向代理配置文件可参考：
```
# tracker.conf
upstream fdfs_group1 {
    server ip11:port11;
}

upstream fdfs_group2 {
    server ip21:port21;
}

server {
    listen    20000 ;
    # server_name  _ ;

    location / {
        root   html;
        index  index.html index.htm;
    }

    location /group1 {
        proxy_pass http://fdfs_group1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /group2 {
        proxy_pass http://fdfs_group2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

### 注意点
- 此容器只适用于host网络，即--network=host，和宿主机共用一套网络配置；
- 默认情况下，创建并启动tracker容器，nginx会启动失败，因为nginx的配置文件不准确；
