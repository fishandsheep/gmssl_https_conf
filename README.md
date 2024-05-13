# nginx 如何配置国密SSL

### 1. 目的

配置支持国密SSL协议的nginx，并通过支持国密协议的浏览器访问。

### 2. 准备

操作系统: `Ubuntu 20.04.6 LTS (Focal Fossa)`

1. `nginx-1.24.0.tar.gz`：`开源版`

2. `openssl-1.1.1u.tar.gz`: `开源版，生成自签名证书`

3. `gmssl_openssl_1.1_b2024_x64_1.tar.gz`：[GMSSL - 国密SSL实验室](https://www.gmssl.cn/gmssl/index.jsp)

4. `qaxbrowser_1.1.45398.52.exe` : [下载奇安信可信浏览器国密开发者专版 (qianxin.com)](https://www.qianxin.com/ctp/gmbrowser.html)

### 3. 安装步骤

1. 生成国密自签名密钥证书

```shell
# 安装gcc make等
sudo apt install build-essential -y

# 编译安装openssl
./config -fPIC no-gost no-shared no-zlib --prefix=/home/ubuntu/soft/openssl
make && make install

cd /home/ubuntu/soft/openssl/bin
# 生成国密证书
./openssl ecparam -out sm2.key -name SM2 -genkey
./openssl req -config ../ssl/openssl.cnf -key sm2.key -new -out sm2.req
./openssl x509 -req -in sm2.req -signkey sm2.key -out sm2.pem

# 这个证书再签发 server 证书
./openssl ecparam -out sm2_site.key -name SM2 -genkey
./openssl req -config ../ssl/openssl.cnf -key sm2_site.key -new -out sm2_site.req
./openssl x509 -req -in sm2_site.req -CA sm2.pem -CAkey sm2.key  -out sm2_site.pem -CAcreateserial
```

2. 编译支持国密SSL协议的nginx

```shell
# nginx目录下 编辑 auto/lib/openssl/conf，将全部$OPENSSL/.openssl/修改为$OPENSSL/并保存

# 安装PCRE
sudo apt install libpcre3 libpcre3-dev -y

# 编译nginx
./configure  \
--prefix=/home/ubuntu/soft/nginx \
--without-http_gzip_module \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_v2_module \
--with-stream \
--with-file-aio \
--with-openssl="/home/ubuntu/gmssl"

make && make install
```

3. 配置nginx 
   1. conf目录下创建 `ssl.conf`配置文件
   2. `nginx.conf`主配置文件引入`ssl.conf`

```nginx
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-CBC-SM3:ECC-SM4-GCM-SM3;
ssl_verify_client off;

ssl_certificate ca/sm2_site.pem;
ssl_certificate_key ca/sm2_site.key;
```
