---
title: 创建私有 CA 与签发证书
date: 2017-09-29 09:22:59
category: Linux
tags:
	- CA
	- openssl
	- private ca
---



环境：

* 系统 CentOS 7.3
* openssl 版本 openssl-1.0.1e-60.el7.x86_64

`openssl` 配置文件 `/etc/pki/tls/openssl.cnf`

信息匹配策略：

* match：要求申请填写的信息与 CA 设置信息必须一致
* supplied：必须填写的信息
* optional：可以填也可以不填 

创建根 CA 自签证书

创建必要文件

```sh
;创建数据库文件
touch /etc/pki/CA/index.txt
;创建证书列表数据库并定义起始序号
echo 01 /etc/pki/CA/serial
;创建证书吊销列表数据库并定义起始序号
echo 01 /etc/pki/CA/crlnumber
```

生成私钥文件

```sh
 ;进入 CA 配置目录
 cd /etc/pki/CA/
 ;将改命令括起来并使用 umask 066，表示该设置仅在当前命令中生效；
 ;genrsa: 使用支持数字签名与加密的 rsa 算法
 ;-out: 保存位置(在 CA 配置文件 /etc/pki/tls/openssl.cnf 定义)
 ;-des3: 是否加密(可选，使用改选项后每次使用该证书时需要输入密码)
 ;2048: 密钥长度(可用 4096、8192 等)
 (umask 066 ; openssl genrsa -out private/cakey.pem -des3 2048)
 Generating RSA private key, 2048 bit long modulus
.................+++
...................................+++
e is 65537 (0x10001)
Enter pass phrase for private/cakey.pem: ;输入密码
Verifying - Enter pass phrase for private/cakey.pem: ;确认密码
```

生成自签名证书

```sh
;-new: 生成新证书签署请求
;-x509: 专用于 CA 生成自签证书
;-key: 生成请求时用到的私钥文件
;-days n: 证书有效期限
;-out: 保存位置
openssl req -new -x509 -key private/cakey.pem -out private/cacert.pem
Enter pass phrase for private/cakey.pem:	;私钥文件的密码
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
;以下信息规则在 openssl.cnf 定义，如果为 match 那么这里填写的是什么，证书申请者也必须填什么
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Sichuan
Locality Name (eg, city) [Default City]:Chengdu
Organization Name (eg, company) [Default Company Ltd]:New world Ltd.
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:weidong.io
Email Address []:1993plus@gmail.com
```

`/etc/pki/CA`文件结构

```sh
tree
.
├── cacert.pem
├── certs
├── crl
├── crlnumber
├── index.txt
├── newcerts
├── private
│   └── cakey.pem
└── serial
```



客户端申请证书

给应用服务生成私钥文件

```sh
cd /etc/pki/tls/
(umask 066 ; openssl genrsa -out private/app.key 2048)
Generating RSA private key, 2048 bit long modulus
................+++
..............................+++
e is 65537 (0x10001)
```

生成证书申请文件

```sh
openssl req -new -key private/app.key -out app.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Sichuan
Locality Name (eg, city) [Default City]:Chengdu
Organization Name (eg, company) [Default Company Ltd]:New world Ltd.
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:test.io   
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

将证书申请文件传到根 `CA` 服务器上颁发证书

```sh
openssl ca -in ~/app.csr -out /etc/pki/CA/certs/app.crt -days 365
Using configuration from /etc/pki/tls/openssl.cnf
;输入 CA 私钥密码
Enter pass phrase for /etc/pki/CA/private/cakey.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Sep 29 04:45:06 2017 GMT
            Not After : Sep 29 04:45:06 2018 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = Sichuan
            organizationName          = New world Ltd.
            commonName                = test.io
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                46:3A:C7:6D:A2:4C:B9:F0:E5:44:B3:2C:49:B5:06:6D:06:63:E7:55
            X509v3 Authority Key Identifier: 
                keyid:7E:D6:7F:B6:EA:06:AF:61:BE:D2:FA:D7:6A:DC:BE:0C:47:6A:CC:AD

Certificate is to be certified until Sep 29 04:45:06 2018 GMT (365 days)
;确认签发证书
Sign the certificate? [y/n]:y

;提交请求
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

此时数据库中会增加一条数据

```sh
cat index.txt
V       180929044506Z           01      unknown /C=CN/ST=Sichuan/O=New world Ltd./CN=test.io
```

现在把证书传给客户端就可以使用了

查看证书信息

```sh
;-text:  查看所有信息
openssl x509 -in app.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Sichuan, L=Chengdu, O=New world Ltd., CN=weidong.io/emailAddress=1993plus@gmail.com
        Validity
            Not Before: Sep 29 04:45:06 2017 GMT
            Not After : Sep 29 04:45:06 2018 GMT
        Subject: C=CN, ST=Sichuan, O=New world Ltd., CN=test.io
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                ...略...
;-subject: 查看证书所有者信息
openssl x509 -in app.crt -noout -subject
subject= /C=CN/ST=Sichuan/O=New world Ltd./CN=test.io
;-issuer: 查看证书颁发者信息
openssl x509 -in app.crt -noout -issuer
issuer= /C=CN/ST=Sichuan/L=Chengdu/O=New world Ltd./CN=weidong.io/emailAddress=1993plus@gmail.com
;-serial: 查看证书序列号
openssl x509 -in app.crt -noout -serial
serial=01
;-dates: 查看证书有效期
 openssl x509 -in app.crt -noout -dates
notBefore=Sep 29 04:45:06 2017 GMT
notAfter=Sep 29 04:45:06 2018 GMT
```

`CA` 吊销证书

```
;确认证书序列号
openssl x509 -in app.crt -noout -serial
serial=01
;确认序列号后吊销证书
openssl ca -revoke /etc/pki/CA/newcerts/01.pem 
Using configuration from /etc/pki/tls/openssl.cnf
; 输入 CA 私钥密码
Enter pass phrase for /etc/pki/CA/private/cakey.pem:
Revoking Certificate 01.
Data Base Updated
;更新证书吊销列表
ca -gencrl -out /etc/pki/CA/crl/crl.pem
Using configuration from /etc/pki/tls/openssl.cnf
Enter pass phrase for /etc/pki/CA/private/cakey.pem:
;查看 crl  文件
openssl crl -in /etc/pki/CA/crl/crl.pem -noout -text
Certificate Revocation List (CRL):
        Version 2 (0x1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: /C=CN/ST=Sichuan/L=Chengdu/O=New world Ltd./CN=weidong.io/emailAddress=1993plus@gmail.com
        Last Update: Sep 29 05:32:08 2017 GMT
        Next Update: Oct 29 05:32:08 2017 GMT
        CRL extensions:
            X509v3 CRL Number: 
                1
Revoked Certificates:
    Serial Number: 01
        Revocation Date: Sep 29 05:30:29 2017 GMT
```

申请中级 `CA`

创建必要文件

```sh
touch index.txt
echo 01 > serial
echo 01 > crlnumber
```

生成私钥文件 

```sh
openssl genrsa -out private/cakey.pem 2048)  
Generating RSA private key, 2048 bit long modulus
.........................................+++
...............................................................................................+++
e is 65537 (0x10001)
```

生成证书申请文件

```sh
[root@com-domain CA]# openssl req -new -key private/cakey.pem -out intermediate.csr 
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Sichuan
Locality Name (eg, city) [Default City]:Chengdu
Organization Name (eg, company) [Default Company Ltd]:Intermediate CA Ltd.
Organizational Unit Name (eg, section) []:CA 
Common Name (eg, your name or your server's hostname) []:www.intermediate.net
Email Address []:245860762@qq.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:Intermediate Ca Ltd.
```

`CA` 签发证书

```sh
;-policy: 使用其它规则
openssl ca -extensions v3_ca -in intermediate.csr -out /etc/pki/CA/certs/intermediate.crt -policy policy_anything -days 3650
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4 (0x4)
        Validity
            Not Before: Sep 29 08:18:24 2017 GMT
            Not After : Sep 27 08:18:24 2027 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = Sichuan
            localityName              = Chengdu
            organizationName          = Intermediate CA Ltd.
            organizationalUnitName    = CA
            commonName                = www.intermediate.net
            emailAddress              = 245860762@qq.com
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                BB:A0:46:6B:58:EC:AB:A8:86:C2:EB:B2:DE:96:D5:FB:27:57:31:D1
            X509v3 Authority Key Identifier: 
                keyid:FE:DA:06:54:99:24:5E:10:A4:DB:20:46:FA:D3:11:02:BE:BA:96:0F

            X509v3 Basic Constraints: 
                CA:TRUE
Certificate is to be certified until Sep 27 08:18:24 2027 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

现在把证书传给中级 `CA` 服务器到 `/etc/pki/CA` 目录下并改名为 `cacert.pem` 

```sh
mv intermediate.crt cacert.pem
```

现在中 `CA` 也可以为申请者颁发证书了

为应用申请证书

生成私钥文件

```sh
(umask 066 ; openssl genrsa -out private/app.key 2048)
```

生成证书申请文件

```sh
openssl req -new -key private/app.key -out www.weidong.io.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Sichuan
Locality Name (eg, city) [Default City]:Chengdu
Organization Name (eg, company) [Default Company Ltd]:Intermediate CA Ltd.
Organizational Unit Name (eg, section) []:Opt
Common Name (eg, your name or your server's hostname) []:www.weidong.io
Email Address []:1993plus@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

使用中级 `CA` 为应用签发证书

```sh
openssl ca -in ~/www.weidong.io.csr -out certs/www.weidong.io.crt -days 365
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 2 (0x2)
        Validity
            Not Before: Sep 29 08:39:46 2017 GMT
            Not After : Sep 29 08:39:46 2018 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = Sichuan
            organizationName          = Intermediate CA Ltd.
            organizationalUnitName    = Opt
            commonName                = www.weidong.io
            emailAddress              = 1993plus@gmail.com
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                D6:E7:42:F6:19:DB:65:14:F0:47:94:62:13:54:F7:EE:13:09:5C:BC
            X509v3 Authority Key Identifier: 
                keyid:BB:A0:46:6B:58:EC:AB:A8:86:C2:EB:B2:DE:96:D5:FB:27:57:31:D1

Certificate is to be certified until Sep 29 08:39:46 2018 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

现在把证书传给申请者就可以使用了

你也可以在 `Windows` 或者 MAC OS 上查看证书文件

根 `CA`

![Root CA](http://ov2iiuul1.bkt.clouddn.com/root_ca.png)

中级 `CA`

![intermediate CA](http://ov2iiuul1.bkt.clouddn.com/intermediate_ca.png)

应用证书文件

![app certificate](http://ov2iiuul1.bkt.clouddn.com/app_ca.png)



**FAQ**

* 如果在签发证书的时候没有创建 `index.txt` 数据库文件会提示以下错误信息

```sh
/etc/pki/CA/index.txt: No such file or directory
unable to open '/etc/pki/CA/index.txt'
139854476003232:error:02001002:system library:fopen:No such file or directory:bss_file.c:398:fopen('/etc/pki/CA/index.txt','r')
139854476003232:error:20074002:BIO routines:FILE_CTRL:system lib:bss_file.c:400:
```



END!