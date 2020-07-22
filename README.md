### 这是做什么的
用于熟悉使用**docker**、**docker-compose**去启服务，熟悉Dockerfile的编写。这里举如下例子：
* PM2 + Koa(2)
* nginx
* mysql
  
#### 项目结构

```
├── docker-compose.yml
├── conf
│   ├── init.sql
│   ├── my.cnf
│   └── nginx.conf
│       └── default.conf
├── koa
│   ├── Dockerfile
│   ├── app.js
│   ├── package.json
│   └── pm2start.yml
└── web
    └── index.html
```

#### Dockerfile
这里先来看一下PM2在docker环境的应用。
首先写个最简单的koa项目：

```sh
... 
├── koa
│   ├── app.js
│   ├── package.json
│   ├── Dockerfile //
│   └── pm2start.yml //PM2的启动配置
...
```

需要注意监听的是什么端口，这里是9000，`app.js`：

```js
const Koa = require('koa');
const app = new Koa();

// response
app.use(ctx => {
  ctx.body = 'Hello Koa';
});

app.listen(9000);
```
因为要用PM2去守护项目，所以可以写一个PM2的启动配置`pm2start.yml`：
```yml
apps:
  - script: ./app.js  #程序启动文件
    name: koa0    #启动服务名（自定义）
    env_production:   #生产环境
      NODE_ENV: production
```
在项目中新建`Dockerfile`：
```docker
FROM keymetrics/pm2:latest-alpine
COPY . /koa
WORKDIR /koa
ENV NPM_CONFIG_LOGLEVEL warn
RUN npm install --production  --registry https://registry.npm.taobao.org 
CMD [ "pm2-runtime", "start", "pm2start.yml" ]
```
使用到的是keymetrics/pm2:latest-alpine，这是PM2官方的image

**这里如果是国外部署可以将淘宝源去掉**

可以用到如下操作，参考[官方](https://github.com/keymetrics/docker-pm2)

  Command | Description
  --------|------------
  ```$ docker exec -it <container-id> pm2 monit``` | Monitoring CPU/Usage of each process
  ```$ docker exec -it <container-id> pm2 list``` | Listing managed processes
  ```$ docker exec -it <container-id> pm2 show``` | Get more information about a process
  ```$ docker exec -it <container-id> pm2 reload all``` | 0sec downtime reload all applications

如果想单独启动这个容器（先构建镜像```docker build -t image-name ./```，然后启动容器```docker run --name demo -p 80:9000 -d image-name:latest```）。

#### docker-compose
同时启动多个docker容器，并且容器间有互相通信，这个时候一个一个构建加启动会很麻烦，所以用到docker-compose。
**使用**：需要写一个`docker-compose.yml`，然后在同级目录使用指令：`docker-compose up` （同理：还有`docker-compose stop`、`docker-compose down`）
对于上述的PM2-Koa项目，`docker-compose.yml`这么写：
```yml
version: "3"
services:
  koa-server:
    image: pm2koa # 
    build: # 如果有build参数，就是要在指定的目录build，并且将生成的镜像命名为参数image的值，下次执行的时候就不再build？
      dockerfile: Dockerfile
      context: ./koa/
    container_name: pm2koa # 指定容器的名称：1、nginx反向代理的时候用到。2、敲命令的时候也可以直接用名字代替容器id
    restart: always
    ports:
      - "9000:9000"
    depends_on: # 起这个容器之前要启动的其他容器
      - web 
      - mysql-db
```

#### nginx
使用最精简的nginx镜像`nginx:alpine` , `docker-compose.yml`添加：

``` yml
version: "3"
services:
  ...
  web:
    restart: always
    image: nginx:alpine
    ports:
      - 80:80
    volumes:
      - ./conf/nginx.conf/:/etc/nginx/conf.d/
      - ./web/:/app/
```
这里挂载了一个静态网页文件夹 和 nginx的配置项文件夹。
* 将宿主机当前目录的`./web`文件夹挂载为容器的`/app`
* `conf/nginx.conf/default.conf`：
```nginx
server {
    listen 80;

    # 静态网页的路径 /app
    location / {
        root   /app; 
        index  index.html index.htm;
        try_files $uri $uri/ /index.html; # 防止重刷新返回404
    }
    
    # 注意 pm2koa 是上述koa服务的容器名，并不是真实url！这里是反向代理localhost/koa到koa服务了
    location /koa {
        proxy_pass http://pm2koa:9000;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
所以宿主机当前目录的`web`文件夹就是静态网页的存放目录，可以实时更改里面的内容。

到这里基本上已经结束，可以正常运行PM2守护的koa项目和静态网页，并且是通过docker容器化后的nginx代理的。

#### mysql
mysql的官方镜像很大，写这个的时候有544M。尝试过用`mysql/mysql-server`，也有三百多兆，而且还有坑，所以就老老实实用`mysql`镜像了。

`docker-compose.yml`添加：

``` yml
version: "3"
services:
  ...
  mysql-db:
    image: mysql # 指定镜像和版本
    container_name: mysql-db # 指定容器的名称
    command:
      --default-authentication-plugin=mysql_native_password # 这行代码解决外网无法访问的问题
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123 # 指定root密码，也可以通过初始sql完成
      MYSQL_DATABASE: table1 # 指定初始创建的数据库，也可以通过初始sql完成
    volumes:
      - "./conf/my.cnf:/etc/my.cnf" # mysql配置
      - "./conf/init.sql:/docker-entrypoint-initdb.d/init.sql" # 要执行的初始sql
```

PS：`mysql/mysql-server`的坑是：挂载的初始init脚本`"./conf/init.sql:/docker-entrypoint-initdb.d/init.sql"`不执行。换成`mysql`镜像就好了。

**`init.sql`中的如下代码，也是用于解决mysql初始化后不能外网链接的问题，将root账号的访问权限改为“%”，即无限制。**
```sql
use mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123';
```


#### 启动
至此，三个容器都已配置完,`docker-compose.yml`：
```yml
version: "3"
services:
  koa-server:
    image: pm2koa # 
    build:
      dockerfile: Dockerfile
      context: ./koa/
    container_name: pm2koa # 指定容器的名称
    restart: always
    ports:
      - "9000:9000"
    depends_on:
      - web
      - mysql-db
  web:
    restart: always
    image: nginx:alpine
    ports:
      - 80:80
    volumes:
      - ./conf/nginx.conf/:/etc/nginx/conf.d/
      - ./web-dist/:/app/
  mysql-db:
    container_name: mysql-db # 指定容器的名称
    image: mysql # 指定镜像和版本
    command:
      --default-authentication-plugin=mysql_native_password #这行代码解决无法访问的问题
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
      MYSQL_DATABASE: table1
    volumes:
      - "./conf/my.cnf:/etc/my.cnf"
      - "./conf/init.sql:/docker-entrypoint-initdb.d/init.sql"
```

```sh
docker-compose up
```
`docker ps`可以查看已经运行了三个容器。
可以访问`localhost`、`localhost/koa`,数据库能连接上。

完
-----
PS:
`volumes`可以挂载文件夹也可以挂载单个文件