---
title: 如何使用SAP Cloud Platform Custom Domain Service设置个性化域名
date: 2020-05-18 16:23:09
tags: SAP Cloud Platform; Custom Domain
---
<!-- TOC -->

- [前期准备](#前期准备)
- [执行步骤](#执行步骤)
  - [查看你的域名DNS是否有用](#查看你的域名dns是否有用)
  - [使用CF Custom Domain 服务创建域名，生成私钥和CSR](#使用cf-custom-domain-服务创建域名生成私钥和csr)
  - [去证书机构请求CSR获取签名](#去证书机构请求csr获取签名)
  - [导入并激活证书](#导入并激活证书)
  - [使用自己域名发布应用](#使用自己域名发布应用)
- [参考资料](#参考资料)
- [感谢](#感谢)

<!-- /TOC -->

本文章主旨是为SAP Cloud  Platform Cloud Foundry环境下设置个性化域名，实现在自己的域名下公开自己的应用程序方案。

通过SAP Cloud Platform自定义域服务，可以配置自己的自定义域以公开公开您的SAP Cloud Platform应用程序，而不使用默认子域。

## 前期准备
> 1. cf-cli,下载安装好本机CF Command line。https://docs.cloudfoundry.org/cf-cli/install-go-cli.html
> 2. 安装Custom Domain Self-service插件。https://tools.hana.ondemand.com/#cloud -> SAP Cloud Platform Cloud Foundry CLI Plugin -> Custom Domain Self-service
> 3. 确保自己子账户下面有Custom Domain这一项服务。这项服务具体如如何查看，cf services
> 4. 申请好自己的域名

总概述图如下  
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-02.png)

有概述图可以看出，使用Custom Domain过程中会设计四个不同的机构或平台，看上去复杂执行起来其实不难。请耐心看完以下内容。

以上准备工作都已经完成，接下来我们通过CF command line来完成custom domain的创建和应用的过程。

## 执行步骤

### 查看你的域名DNS是否有用
```bash
nslookup asb.cpgc.cn40.apps.platform.sapcloud.cn
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-06.png)

域名有用说明你申请域名服务器是没有任何问题。

### 使用CF Custom Domain 服务创建域名，生成私钥和CSR
1. 安装Custom Domain Cli
```bash
# 解压custom domain cli
tar -xf custom-domain-cli-1.0.36-darwin-amd64-x86_64.tar.gz
cd darwin-amd64/

# 安装 plugin
cf install-plugin custom-domain-cli
```

2. 登录到CF账户中，创建custom domain服务
```bash
# 登录
cf login -a https://api.cf.cn40.platform.sapcloud.cn

# 创建 domain服务
cf cs INFRA custom_domains pocdomain
cf create-domain "SCP GC CEE_Demo" cpgc.cn40.apps.platform.sapcloud.cn
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-03.png)

>``cpgc.cn40.apps.platform.sapcloud.cn`` 这里域名就是我自己申请的域名

3. 生成私钥和CSR
```bash
cf cdck cpgc-cn40-key "CN=cpgc.cn40.apps.platform.sapcloud.cn" "*.cpgc.cn40.apps.platform.sapcloud.cn"
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-04.png)

4. 获取CSR
```bash
cf custom-domain-get-csr cpgc-cn40-key csr.pem
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-05.png)

保存好你的csr证书，接下来要那这个证书去证书机构进行签发。

>注意，证书内容你只需要保存 「-----BEGIN CERTIFICATE REQUEST----- *** -----END CERTIFICATE REQUEST-----」这里面的内容。

### 去证书机构请求CSR获取签名
拿着上面获得CSR发送给CA证书办法机构去验证，当验证成功之后，CA机构会给你发回的相应的Cert。

我收到有DigiCert发挥的Intermediate Certificates。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-11.png)

最后将三个证书合并成一个Pem文件。
三个证书分别是：你自己的csr, DigiCert Intermediate Cert和[DigiCert Global Root CA.pem](https://github.tools.sap/sgs/SAP-Global-Trust-List/blob/master/Approved/DigiCert%20Global%20Root%20CA.pem)。

保存文件名为<your-domain-name\>-chain.pem, 我的是cpgc-chain.pem。

>如果你不是有机构来颁发的证书，也可以使用第三方开源来颁发，在这里不在多说，有兴趣的请搜索关键字**acme**。作为一个正规机构还是请使用正规渠道。

接下来我们需要导入证书并激活。
### 导入并激活证书
1. 导入证书
```bash
cf cducc cpgc-cn40-key cpgc-chain.pem
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-08.png)

2. 激活证书
```bash
cf cda cpgc-cn40-key  cpgc.cn40.apps.platform.sapcloud.cn *.cpgc.cn40.apps.platform.sapcloud.cn
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-09.png)

3. 查看证书激活状态
```bash
echo -n | openssl s_client -connect cpgc.cn40.apps.platform.sapcloud.cn:443
 -servername *.cpgc.cn40.apps.platform.sapcloud.cn
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-10.png)
好了，你已经成功了。接下来我们就可以使用自己的域名去发布应用了。

### 使用自己域名发布应用
1. 修改你的**manifest.yml**
在yaml里面添加routes,route填写自己的domain。
```yaml
---
applications:
- name: chat
  host: i304185-chat
  routes:
    - route: chat.cpgc.cn40.apps.platform.sapcloud.cn
  path: .
  memory: 128M
  buildpack: nodejs_buildpack
```
2. 部署你的应用
```bash
# 部署
cf push

# 查看apps route地址
cf apps
```
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/03-custom-domain/cd-12.png)

感谢阅读，打完收工。


## 参考资料

1. https://help.sap.com/viewer/74af813c7ee2457cb5eddca0cc70a0c1/Cloud/en-US/4414cc43db2d4229b27b232a5590e253.html

2. SAP内部wiki https://wiki.wdf.sap.corp/wiki/pages/viewpage.action?pageId=2098634698

## 感谢
Bella Wang提供的支持，因为你的支持，我才能尽快完成了custom domain的配置，感恩。