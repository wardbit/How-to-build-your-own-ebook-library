^[该文借鉴[【Calibre-Web】Calibre-Web服务器安装详细步骤（个人搭建自用的电子书网站，docker-compose安装）-CSDN博客](https://blog.csdn.net/mudarn/article/details/144267291)的部分操作写成，nginx反代域名访问部分有我写成,略有区别，本人使用腾讯云docker ce系统镜像完成全过程]
# 一、Calibre-Web和Calibre的区别是什么？使用场景分别是什么？
## Calibre:

功能完整的桌面应用程序
重点在于电子书的管理和处理
独立运行的本地软件
## Calibre-Web:

基于Web的在线图书馆系统
重点在于图书的展示和阅读
需要服务器部署的网页应用
主要功能对比

| 功能    | Calibre  | Calibre-Web  |
| ----- | -------- | ------------ |
| 图书管理  | ✅ 完整强大   | ⭕️ 基础管理      |
| 元数据编辑 | ✅ 专业完整   | ⭕️ 基础编辑      |
| 格式转换  | ✅ 支持多种格式 | ❌ 需依赖Calibre |
| 在线阅读  | ❌ 不支持    | ✅ 支持         |
| 多用户支持 | ❌ 单用户    | ✅ 多用户系统      |
| 远程访问  | ❌ 本地使用   | ✅ 随处访问       |

Calibre 专注于管理和处理
Calibre-Web 专注于展示和阅读
最理想的方案是：

用 Calibre 做后台管理
用 Calibre-Web 做前台展示
两者结合获得最佳体验
# 二、服务器安装docker和docker-compose[^2]

[^2]: 由于使用腾讯云docker ce来完成，所以其实我只安装了docker-compose即可。
## 1.ubuntu 安装教程
### 1.更新包

```bash
sudo su -
sudo apt update && sudo apt upgrade -y
```

### 2.安装必要的包

```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```
### 3.添加docker官方GPG密钥
```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
###  （4）添加 Docker 的 APT 源
```shell
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```
## （5）更新 APT 包索引
```shell
sudo apt update
```
### （6）安装 Docker 和 Docker Compose
```shell
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

```
### （7）验证安装
```shell
docker version 
docker compose version
```
# 三、服务器安装Calibre-Web步骤

## 1、安装完成后的目录结构```
```安装完成后结构
/data/calibre/
├── docker-compose.yml    # 配置文件
├── config/               # Calibre-Web配置目录
│   ├── app.db           # 应用数据库
│   ├── calibre-web.log  # 日志文件
│   └── config.json      # 配置文件
├── books/               # 图书库目录
│   ├── metadata.db    # 数据库文件，这个文件必须存在
│   └── [作者名]/[书名]  # 图书文件
└── upload/              # 上传目录

```
## 2、安装步骤

1. **创建必要的目录**
```shell
# 创建主目录
mkdir -p /data/calibre

# 创建子目录
mkdir -p /data/calibre/{config,books,upload}

# 进入calibre目录
cd /data/calibre

# 使用普通用户权限
sudo chown -R 1000:1000 /data/calibre/{config,books,upload}
chmod -R 755 config books upload

```
2. **创建 docker-compose.yml 文件**
```bash
vim docker-compose.yml
```
3. **粘贴以下内容**
```yaml
version: '3'
services:
  calibre-web:
    # 官方下载失败可以使用可用的镜像dockerpull.org
    # dockerpull.org/linuxserver/calibre-web:latest
    # 24年11月整理了20来个可用的镜像网站且用且珍惜
    #	https://download.csdn.net/download/mudarn/90051682
    # 官方镜像
    image: linuxserver/calibre-web:latest
    container_name: calibre-web
    environment:
      # 使用普通用户权限，避免安全问题
      - PUID=1000
      - PGID=1000
      # 设置时区为上海
      - TZ=Asia/Shanghai
      # 安装完整的Calibre，支持格式转换等功能
      # dockerpull.org/linuxserver/mods:universal-calibre
      # 使用本地 Calibre 管理 -> 可以不需要 DOCKER_MODS
      #- DOCKER_MODS=linuxserver/mods:universal-calibre
    ports:
      # Web访问端口
      - "8083:8083"
    volumes:
      # 配置文件目录
      - ./config:/config
      # 图书库目录，存放所有图书和数据库
      - ./books:/books
      # 上传目录，用于本地Calibre同步上传
      - ./upload:/upload
    # 容器重启策略
    restart: always
    # 使用bridge网络，保持网络隔离
    networks:
      - calibre-net
networks:
  calibre-net:
    driver: bridge

```
4. 提前准备数据库文件
```bash
# 下载初始数据库文件
wget https://raw.githubusercontent.com/janeczku/calibre-web/master/library/metadata.db -O books/metadata.db

# 设置权限，所有者和所属组更改为 UID 和 GID 为 1000 的用户和组。
sudo chown 1000:1000 books/metadata.db

# 设置权限 644，即文件所有者可以读取和写入，所属组和其他用户只能读取。
sudo chmod 644 books/metadata.db

```
5. 启动
```shell
# 启动
docker-compose up -d

# 查看日志
docker-compose logs -f

#日后维护常用命令
# 查看容器状态
docker-compose ps

# 重启服务
docker-compose restart

# 更新镜像
docker-compose pull && docker-compose up -d

# 查看资源使用
docker stats calibre-web

# 清理并重建
docker-compose down --rmi all && docker-compose up -d

```
## 3.初始访问
- 访问地址：http://服务器IP:8083
- 默认账户：admin
- 默认密码：admin123

![[b1e9018186058ba84b1366ba2b00a718_MD5.jpg]]


- 首次登录配置：

设置图书库路径为：`/books`
![[images.assess/f8da5024073921abcb6b04aff2b75c7c_MD5.jpg]]

![[images.assess/46bb64834c7ecca125c6267f95780be4_MD5.jpg]]
更改中文和修改默认密码
[[images.assess/a3390bbbc5e365466b27283be909274e_MD5.jpg|Open: Pasted image 20250914143204.png]]
![[images.assess/a3390bbbc5e365466b27283be909274e_MD5.jpg]]
#### 4、启动上传

管理权限–编辑基本配置–功能配置–启动上传

![[images.assess/cf853ad769006973eb68346d93789258_MD5.jpg]]
![[images.assess/ecf080cc34245f71b7218460db5bf184_MD5.jpg]]

![[images.assess/1b321a375950ebe1c5b2ebc0a81f9f93_MD5.jpg]]
# 四. nginx反向代理绑定域名
每个人申请域名和ssl的证书的过程都不尽相同（阿里云，腾讯云，cloudflare等），不一一赘述
## 1.下载nginx
```shell
# 1. 更新仓库信息 
sudo apt-get update # 2. 安装nginx 
sudo apt-get install nginx # 3. 验证安装 
sudo nginx -
```
## 2.修改nginx配置文件
```shell
nginx -V 2>&1 | grep -o '\-\-conf-path=[^ ]*' | cut -d= -f2
```
1. 找到图中内容
![[images.assess/e1441f21f2ef64deee82b16a58d653b0_MD5.jpg]]
2. 修改配置文件
```bash
sudo vim /etc/nginx/nginx.conf
```

```vim
http{
	#前面的内容
	#允许文件大小
	client_max_body_size 2g;

	upstream bookbackend{
		server 127.0.0.1:8083
	}
	server{
	    listen 80; 
	    server_name #你的域名;
	    return 301 https://$server_name$request_uri;
	    location /{
	        proxy_pass http://bookbackend;
	        proxy_connect_timeout 300s;   # 连接上游服务器超时
	        proxy_send_timeout     300s;  # 发送请求到上游超时
	        proxy_read_timeout     300s;  # ⭐  读取响应超时（最重要）
	        send_timeout           300s;  # 客户端发送响应超时
                     
			proxy_http_version 1.1;
			proxy_set_header Connection ""; #设置keepalive
			 # === 启用缓冲 ===
	        proxy_buffering on; 
		    proxy_buffer_size 128k;
		    proxy_buffers 4 256k;
		    proxy_busy_buffers_size 256k;
	        proxy_set_header Host $host;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_set_header X-Forwarded-Proto $scheme;
	        # 访问日志（可选）
		    access_log /var/log/nginx/wubooks.access.log;
		    error_log /var/log/nginx/wubooks.error.log;
    }
    server{
    #SSL 默认访问端口号为 443
     listen 443 ssl;
     #请填写绑定证书的域名
     server_name #你的域名;
     #请填写证书文件的相对路径或绝对路径
     ssl_certificate #你的.crt;
     #请填写私钥文件的相对路径或绝对路径
     ssl_certificate_key #你的.key;
     ssl_session_timeout 5m;
     #请按照以下协议配置
     ssl_protocols TLSv1.2 TLSv1.3;
     #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
     ssl_prefer_server_ciphers on;
     location /{
        proxy_pass http://bookbackend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        # 超时设置
        proxy_connect_timeout 300s;   # 连接上游服务器超时
		proxy_send_timeout     300s;  # 发送请求到上游超时
        proxy_read_timeout     300s;  # ⭐   读取响应超时（最重要）
	    send_timeout           300s;  # 客户端发送响应超时
        # === 启用缓冲 ===
	    proxy_http_version 1.1;
		proxy_set_header Connection "";
	    proxy_buffering on;
	    proxy_buffer_size 128k;
	    proxy_buffers 4 256k;
	    proxy_busy_buffers_size 256k;
    }
}
```
## 3.测试配置文件并运行
```bash
sudo nginx -t
sudo nginx -s reload
```
