# outline-deployment-guide
very complete deployment guide for ouline wiki

可实时协作的在线团队知识库，docker部署，可通过浏览器随时随地访问。

* markdown编辑器 / 调用 
* 可导入docx、txt、markdown、html
* 可导出PDF、markdown 、JSON(用于outline自身还原)
* 可插入表格，图片，附件，音频，视频文件

  界面如下：
  <img width="1342" alt="image" src="https://github.com/user-attachments/assets/5d9bbf8b-cd50-4983-9d2b-380077eaf470" />


**开源链接**


:::info
<https://github.com/outline/outline>

:::

## 参考资料

本文档参考了以下作者的资料：

**#Bi0x的一键部署脚本中没有nginx代理，keycloak版本仅支持到23.0.7。**


:::tip
<https://github.com/Bi0x/outline-keycloak-installer> 

:::

**#jiang-taibai提供了Nginx的配置参考、SSL证书申请、以及整体架构的知识框架**


:::tip
<https://gitee.com/jiang-taibai/deploy-outline-via-nginx/tree/main/docs> 

:::

**#id404提供了解决logout问题的配置参考**


:::tip
<https://www.cnblogs.com/id404/p/18003307> 

:::

**#[vicalloy](https://github.com/vicalloy) 的部署存在用户登录退出时仍然不会注销的问题**


:::tip
<https://id404.github.io/2024/01/26/outline-deploy/>

<https://github.com/vicalloy/outline-docker-compose>

:::


\#这个链接没怎么看懂里面的backup部分配置，但配置参数比较详细，可以参考研究

<https://github.com/huketo/outline-keycloak-docker-compose>


## 架构图
参考图： 该手册没有对图中的端口做了一下修改。
<img width="1104" alt="image" src="https://github.com/user-attachments/assets/95cb1340-94b8-4eeb-acd1-7f7b379357b7" />





## **前提条件**


1. 威联通QNAP NAS 及安装了Container Station容器工作站, 确保系统中无其他程序占用80，443端口
2. 其他系统linux操作系统，安装docker 和 docker compose；
3. 在腾讯云申请自己的域名，需要四个域名：kb.yourdomain.com、sso.yourdomain.com、minio.yourdomain.com、

   minio-admin.yourdomain.com
4. 在腾讯云申请SSL证书，腾讯云有3个月免费SSL证书申请使用，申请证书后下载nginx使用的格式证书，4个域名共8个证书文件。

   kb.yourdomain.com_bundle.crt 、kb.yourdomain.com.key

   sso.yourdomain.com_bundle.crt、sso.yourdomain.com.key

   minio.yourdomain.com_bundle.crt、minio.yourdomain.com.key

   minio-admin.yourdomain.com_bundle.crt、minio-admin.yourdomain.com.key
5. 在云解析中添加对应DNS解析, 四个域名都解析到你的docker宿主机ip地址
6. 创建目录/opt/outline/

## 一、准备部署文件


1. 创建`nginx.conf`放在`/opt/outline/nginx/conf/目录`下
2. 创建`default.conf、sso.conf、kb.conf、minio.conf、minio-admin.conf`五个配置文件，放在`/opt/outline/nginx/conf/conf.d/`目录下
3. 数据库初始化文件`postgresql-init.sql` 放在`/opt/outline/`目录下
4. outline主程序docker compose 文件`outine-all-in-one-compose.yaml`放在`/opt/outline/`目录下
5. outline环境参数变量文件 `outline.env` 文件放在`/opt/outline/`目录下以上文件的内容分别如下：

### nginx.conf

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

\
events {
    worker_connections  1024;
}  

\
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

### **default.conf**

```nginx
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

### sso.conf 

==注意替换文件中的域名==

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name sso.yourdomain.com;

    ssl_certificate ssl/sso.yourdomain.com_bundle.crt; 
    ssl_certificate_key ssl/sso.yourdomain.com.key; 
    
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3; 
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
    
    location / {
        proxy_pass http://outline-data-keycloak:8080/;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;proxy_set_header Host $host;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

\
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

}
```

### kb.conf

==注意替换文件中的域名==

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name kb.yourdomain.com;

    ssl_certificate ssl/kb.yourdomain.com_bundle.crt; 
    ssl_certificate_key ssl/kb.yourdomain.com.key; 
    
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3; 
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
    
    # Allow special characters in headers
    ignore_invalid_headers off;

    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 1000m;

    # Disable buffering
    proxy_buffering off;
    proxy_request_buffering off; 

    location / {
        proxy_pass http://outline-data-wiki:3000/;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;proxy_set_header Host $host;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

}
```

### minio.conf

==注意替换文件中的域名==

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name minio.yourdomain.com;

    ssl_certificate ssl/minio.yourdomain.com_bundle.crt; 
    ssl_certificate_key ssl/minio.yourdomain.com.key; 
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3; 
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
   
    # Allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 1000m;
    # Disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://outline-data-minio:9000;        
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 300;
        # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;

    }

}
```

### **minio-admin.conf**

==注意替换文件中的域名==

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name minio-admin.yourdomain.com;

    #certificate
    ssl_certificate ssl/minio-admin.yourdomain.com_bundle.crt; 
    #key
    ssl_certificate_key ssl/minio-admin.yourdomain.com.key; 
    ssl_session_timeout 5m;
    #protocols
    ssl_protocols TLSv1.2 TLSv1.3; 
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
    
    # Allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 100m;
    # Disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
     
        proxy_pass http://outline-data-minio:9001;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy true;

        # This is necessary to pass the correct IP to be hashed
        real_ip_header X-Real-IP;

        proxy_connect_timeout 300;

        # To support websockets in MinIO versions released after January 2023
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        chunked_transfer_encoding off;
        
    }
}
```

### postgresql-init.sql

```sql
CREATE DATABASE outline;
CREATE DATABASE outline_test;
```

### outline-all-in-one-compose.yaml

```yaml
services:
  keycloak:
    image: docker.1ms.run/keycloak/keycloak:25.0.6
    container_name: outline-data-keycloak
    restart: unless-stopped
    ports:
      - "6001:8080" # HTTP Port
      - "6002:8443" # HTTPs Port (Only available with SSL FILE)
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=password
    # password 可以等安装完后进系统再修改
    
    ## below used in version 26.0.0 and later 
    # - KC_BOOTSTRAP_ADMIN_USERNAME=admin
    # - KC_BOOTSTRAP_ADMIN_PASSWORD=password

      - KC_HTTPS_CERTIFICATE_FILE=
      - KC_HTTPS_CERTIFICATE_KEY_FILE=
      - PROXY_ADDRESS_FORWARDING=true
      
    # volumes:
    #   - $PWD/certs/domain.crt:/opt/keycloak/conf/server.crt.pem
    #   - $PWD/certs/domain.key:/opt/keycloak/conf/server.key.pem
    
    ## below used in version 23.0.7 to 25.0.6, this version can redirect to login page after logout
    command: start-dev --spi-login-protocol-openid-connect-legacy-logout-redirect-uri=true --proxy edge 
    
    ## below used in version 26.0.0+, this version canot redirect to login page after logout
    #command: start-dev --proxy-headers xforwarded 

  postgresql: 
      image: docker.1ms.run/postgres:17.4
      container_name: outline-data-postgresql
      restart: unless-stopped
      healthcheck:
        test: ["CMD-SHELL", "pg_isready"]
        start_period: 20s
        interval: 30s
        retries: 5
        timeout: 5s
      ports:
        - "6432:5432"
      volumes:
        - ./postgresql-init.sql:/docker-entrypoint-initdb.d/init.sql
        - ./postgres:/var/lib/postgresql/data
      environment:
        POSTGRES_USER: postgre_user
        POSTGRES_PASSWORD: postgre_pass_3dwsnatnfp3s

  minio:
    image: docker.1ms.run/minio/minio:RELEASE.2025-03-12T18-04-18Z
    container_name: outline-data-minio
    restart: unless-stopped
    ports:
      - 9000:9000
      - 9001:9001
    environment:
      MINIO_ROOT_USER: minio_user
      MINIO_ROOT_PASSWORD: 7dwb67ees2nak4b5vsbafxapws8jph2w
    volumes:
      - ./minio/data:/data
      - ./minio/config:/root/.minio/
    command: server --console-address ':9001' /data

  outline:
    image: docker.1ms.run/outlinewiki/outline:0.82.0
    container_name: outline-data-wiki
    restart: unless-stopped
    env_file: ./outline.env
    ports:
      - "6003:3000"
    depends_on:
      - redis

  redis:
    image: docker.1ms.run/redis
    container_name: outline-data-redis
    restart: unless-stopped
    env_file: ./outline.env
    command: ["redis-server", "/redis.conf"]
    volumes:
      - ./redis.conf:/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3

  nginx:
    image: docker.1ms.run/nginx
    container_name: outline-data-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf/conf.d:/etc/nginx/conf.d
      - ./nginx/conf/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf/ssl:/etc/nginx/ssl
      - ./nginx/logs:/var/log/nginx
      - ./nginx/www:/usr/share/nginx/html
```

/以上conf配置文件中的证书名称与ssl目录下的文件名对应

将ssl证书放在此目录下/opt/outline/nginx/conf/ssl/

### outline.env

```none
#Outline settings
NODE_ENV=production
SECRET_KEY=vbxdjuvzsc4jyy4tcyme88nvzxz5nckp5zp87armefxypar26jcad6edwkxws8yn
UTILS_SECRET=m88pts8sz5ncdmu836juz3yejbpwzm4tdveu4uupk27v5wuy2fxcv2h62vbs4h38
URL=https://kb.yourdomain.com
PORT=3000
FORCE_HTTPS=false
ENABLE_UPDATES=false
WEB_CONCURRENCY=1
FILE_STORAGE_IMPORT_SIZE=1024000000000
LOG_LEVEL=warn
RATE_LIMITER_ENABLED=true
RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60
DEFAULT_LANGUAGE=zh_CN

# Redis connection
REDIS_URL=redis://redis:6379

# SQL connection
DATABASE_URL=postgres://postgre_user:postgre_pass_3dwsnatnfp3s@outline-data-postgresql:5432/outline
DATABASE_URL_TEST=postgres://postgre_user:postgre_pass_3dwsnatnfp3s@outline-data-postgresql:5432/outline_test
PGSSLMODE=disable

# MinIO
AWS_ACCESS_KEY_ID=t2287yhfxrunswx5
AWS_SECRET_ACCESS_KEY=yt8t787ta6hnmzfbdcwd6h2xms2ddfdp
AWS_REGION=us-east-1
AWS_S3_UPLOAD_BUCKET_URL=https://minio.yourdomain.com
AWS_S3_UPLOAD_BUCKET_NAME=outline-bucket-bsrkavzp
FILE_STORAGE_UPLOAD_MAX_SIZE=1024000000000
AWS_S3_FORCE_PATH_STYLE=true
AWS_S3_ACL=private

# Keycloak SSO OIDC
OIDC_CLIENT_ID=outline_realm_client_jkz8cb
OIDC_CLIENT_SECRET=cd2ej7rrzee7mze2cmzxbcrzd43xjs2x
OIDC_AUTH_URI=https://sso.yourdomain.com/realms/outline_realm_pex2tf/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://sso.yourdomain.com/realms/outline_realm_pex2tf/protocol/openid-connect/token
OIDC_USERINFO_URI=https://sso.yourdomain.com/realms/outline_realm_pex2tf/protocol/openid-connect/userinfo
OIDC_LOGOUT_URI=https://sso.yourdomain.com/realms/outline_realm_pex2tf/protocol/openid-connect/logout?redirect_uri=https%3A%2F%2Fkb.yourdomain.com%2F
OIDC_USERNAME_CLAIM=preferred_username
OIDC_DISPLAY_NAME=Keycloak
OIDC_SCOPES=openid email profile
```

## 二、开始部署

将outline-all-in-one-compose.yaml 和 outline.env文件放在同/opt/outline/目录下，执行以下docker命令安装

```none
docker compose -f outline-all-in-one-compose.yaml up -d
```

## **三、配置Keycloak**


:::info
参考配置步骤：  <https://bra.live/using-keycloak-as-oidc-provider-on-outline-wiki/>  

:::

     登陆https://sso.yourdomain.com


1. 创建新的realm: ==outline_realm_pex2tf==
2. 新创建的realm下创建client：==outline_realm_client_jkz8cb==

   **==Settings==主要配置参数如下:**

   **Client ID:** ==outline_realm_client_jkz8cb==

   **Access settings**

          **Root URL：***https://kb.yourdomain.com/*

          **Home URL:**  *https://kb.yourdomain.com*

          **Valid redirect URIs*:*** *https://kb.yourdomain.com/\**

          **Web origins:** *https://kb.yourdomain.com/*

          **Admin URL:** *https://kb.yourdomain.com*

   **Capability config**

         Client authentication: **On**

        Authentication flow: **Standard flow**

   配置好后**save**
3. 在**==Credentials==**中复制client secret , 替换`outline.env`文件中的 `OIDC_CLIENT_SECRET=`  secret

   ![](/api/attachments.redirect?id=6a21d356-2254-485b-b6e2-d68cac1ac179 " =1453x760")
4. 更新配置，再次执行以下命令

   ```none
   docker compose -f outline-all-in-one-compose.yaml up -d
   ```

   
5. 新建登陆outline的user

## **四、配置minio**

访问<https://minio-admin.yourdomain.com>

 ![](/api/attachments.redirect?id=4ce58445-cd97-40a4-bb50-400f54e1e5a6 " =1456x727")用户名密码位于outline-all-in-one.yaml文件中

```none
MINIO_ROOT_USER: 
MINIO_ROOT_PASSWORD:
```

根据outline-all-in-one.yaml文件中以下字段后面的值，在minio中创建对应配置

```none
AWS_S3_UPLOAD_BUCKET_NAME：
AWS_ACCESS_KEY_ID：
AWS_SECRET_ACCESS_KEY：
```

**到此，已经完成所有部署和配置，然后可以正式使用outline了，浏览器访问**

https://kb.yourdomain.com

## 五、一些运维命令 

```none
docker ps 容器id                      #查看运行中的容器
docker stop 容器id                    #停止容器
docker rm 容器id                      #删除容器
docker exec -it 容器id /bin/bash      #进入容器
nginx -s reload                       #在nginx容器内更新nginx配置
```


