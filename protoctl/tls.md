# SSL/TLS使用自签名的证书

在服务器内网中，使用`Grpc`时常用到`SSL/TLS`作为安全传输。因为许多协议都可以默认有安全验证，在线上环境可以使用`OpenCA`和`Dogtag`等开源CA软件，完成内部证书系统。而在开发过程中，常常要使用**自签名的证书**来完成TLS连接的建立。
需要如下步骤：

1. 创建一个私钥。
2. 创建一份证书请求。
3. 签名一个证书。

## 创建私钥

```bash
openssl genrsa -out private.key 2048
```

## 创建证书请求

创建一份配置`mycert.cnf`

```bash
[req]
req_extensions = v3_req
distinguished_name = dn

[dn]
# The main subject name
CN = /services/UserService

[v3_req]
# Subject Alternative Names (SANs)
subjectAltName = DNS:/services/UserService, DNS:/services/UserService, IP:127.0.0.1
```

生成签名请求csr：
```bash
openssl x509 -req -days 365 -in cert.csr -signkey private.key -out cert.crt -config mycert.cnf
```

> 此处使用`sANS`插件作为签名请求，以前使用的`Common Name`

## 签名证书

```bash
openssl x509 -req -days 365 -in cert.csr -signkey private.key -out cert.crt --extensions v3_req -extfile mycert.cnf
```