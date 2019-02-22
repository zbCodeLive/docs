## harbor搭建

### harbor搭建准备
1. 安装docker  
```
yum -y install yum-utils device-mapper-persistent-data lvm2 container-selinux libcgroup pigz libseccomp

若报错如下：
[root@localhost ~]# rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm
警告：/home/soft/docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm: 头V4 RSA/SHA512 Signature, 密钥 ID 621e9f35: NOKEY
错误：依赖检测失败：
	libltdl.so.7()(64bit) 被 docker-ce-18.03.1.ce-1.el7.centos.x86_64 需要
执行：rpm -ivh libltdl7-2.4.2-alt7.x86_64.rpm

rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm
sudo curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://04be47cf.m.daocloud.io
systemctl restart docker
```
2. 安装docker-compose
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
3. 安装python2.7 or higher


4. 安装openssl
```
yum -y install openssl
```

5. 下载harbor离线安装包：[https://github.com/goharbor/harbor/releases](https://note.youdao.com/)
6. 解压harbor：`tar -zxf harbor-offline-installer-v1.7.0.tgz`

### 配置https访问Harbor
1. 获取证书授权
    ```
    openssl genrsa -out ca.key 4096
    
    openssl req -x509 -new -nodes -sha512 -days 3650 \
        -subj "/C=TW/ST=Taipei/L=Taipei/O=example/OU=Personal/CN=yourdomain.com" \
        -key ca.key \
        -out ca.crt
    ```

2. 获取服务器证书  
假设您的注册表的主机名是yourdomain.com，并且其DNS记录指向您正在运行Harbor的主机。在生产环境中，您首先应该从CA获得证书。在测试或开发环境中，您可以使用自己的CA. 证书通常包含.crt文件和.key文件，例如yourdomain.com.crt和yourdomain.com.key。  

    1). 创建自己的私钥：
    ```
    openssl genrsa -out yourdomain.com.key 4096
    ```
    2). 生成证书签名请求：  
    如果您使用像yourdomain.com这样的FQDN 来连接注册表主机，那么您必须使用yourdomain.com作为CN（通用名称）。
    ```
     openssl req -sha512 -new \
        -subj "/C=TW/ST=Taipei/L=Taipei/O=example/OU=Personal/CN=yourdomain.com" \
        -key yourdomain.com.key \
        -out yourdomain.com.csr 
    ```
    3). 生成注册表主机的证书：  
    无论您是使用类似yourdomain.com的 FQDN 还是IP来连接注册表主机，请运行此命令以生成符合主题备用名称（SAN）和x509 v3扩展要求的注册表主机证书：
    ```
    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth 
    subjectAltName = @alt_names
    
    [alt_names]
    DNS.1=yourdomain.com
    DNS.2=yourdomain
    DNS.3=hostname
    EOF
    ```
    ```
    openssl x509 -req -sha512 -days 3650 \
        -extfile v3.ext \
        -CA ca.crt -CAkey ca.key -CAcreateserial \
        -in yourdomain.com.csr \
        -out yourdomain.com.crt
    ```
3. 配置与安装  
    1). 配置服务器证书和harbor密钥  
    获取yourdomain.com.crt和yourdomain.com.key文件后，您可以将它们放入以下目录/root/cert/：  
    ```
    cp yourdomain.com.crt /data/cert/
    cp yourdomain.com.key /data/cert/ 
    ```
    
    2). 为Docker配置服务器证书，密钥和CA.  
    Docker守护程序将.crt文件解释为CA证书，将.cert文件解释为客户端证书。  
    将服务器转换yourdomain.com.crt为yourdomain.com.cert：  
    ```
    openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert
    ```
    Delpoy yourdomain.com.cert，yourdomain.com.key和ca.crtDocker：  
    ```
    mkdir -p /etc/docker/certs.d/yourdomain.com
    cp yourdomain.com.cert /etc/docker/certs.d/yourdomain.com/
    cp yourdomain.com.key /etc/docker/certs.d/yourdomain.com/
    cp ca.crt /etc/docker/certs.d/yourdomain.com/
    ```
    
    3). 配置Harbor
    编辑文件harbor.cfg，更新主机名和协议，并更新属性ssl_cert和ssl_cert_key：
    ```
      #set hostname
      hostname = yourdomain.com
      #set ui_url_protocol
      ui_url_protocol = https
      ......
      #The path of cert and key files for nginx, they are applied only the protocol is set to https 
      ssl_cert = /data/cert/yourdomain.com.crt
      ssl_cert_key = /data/cert/yourdomain.com.key   
    ```
    为Harbor生成配置文件：
    ```
    ./prepare
    ```
    如果Harbor已在运行，请停止并删除现有实例。您的图像数据保留在文件系统中
    ```
    docker-compose down -v
    ```
    最后，重启Harbor
    ```
    docker-compose up -d
    ```
    
    4). Harbor登录(默认端口为443)
    ```
    docker login yourdomain.com
    (默认用户名:admin,默认密码:Harbor12345)
    ```
    
    5). Harbor pull,push镜像(需在docker登录之后)
    ```
    docker tag nginx yourdomain/nginx/nginx
    docker push yourdomain/nginx/nginx
    docker pull yourdomain/nginx/nginx
    ```

4. docker配置登录
```
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["http://04be47cf.m.daocloud.io"],
  "insecure-registries":["https://192.168.12.249:443"]
}

vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.12.249  yourdomain.com

systemctl restart docker
```
## harbor接入ceph

[https://blog.csdn.net/aixiaoyang168/article/details/78909038](https://note.youdao.com/)

