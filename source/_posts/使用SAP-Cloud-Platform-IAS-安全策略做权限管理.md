---
title: 使用SAP Cloud Platform IAS 安全策略做权限管理
date: 2020-05-14 14:49:48
tags:
---
<!-- TOC -->

- [使用SAP Cloud Platform IAS 安全策略做权限管理](#使用sap-cloud-platform-ias-安全策略做权限管理)
  - [1 SAP Cloud Platform和IAS IDP互信](#1-sap-cloud-platform和ias-idp互信)
  - [2 SAP Cloud Platform Service与IAS Group信息相互映射](#2-sap-cloud-platform-service与ias-group信息相互映射)
  - [3 总结](#3-总结)
- [参考](#参考)

<!-- /TOC -->
## 使用SAP Cloud Platform IAS 安全策略做权限管理
如果你希望通过SAP IAS的服务完成对登录用户的认证和授权功能，而不是使用SAP默认的IDP来授权，那这边文章就是教你如何使用SAP的IAS来管理跟SAP Cloud Platform用户认证和授权问题。

最终达到的目的使用IAS来管理用户，自己来管理用户的认证和授权问题。

这样做的好处有以下几点：
- 一套密码，登录所有系统，实现真正SSO。
- 集团历史用户一键迁移，不需要为每一个用户去创建S-User，就可以使用SCP云平台服务
- 使用IAS管理所有通过SAML2.0认证系统。不仅对于SAP云台，对于所有所有使用SAML2.0单点登录有了集中的平台管理
- 不仅可以集成SAP系统，同样也就集成非SAP第三方应用。IAS的强大的地方不仅可以集成SAP系统，同样也可以集成非SAP，包括Microsoft, Amazon, Google的系统。

> IAS(Identity Authentication service ) 身份认证是任何SAP云应用程序认证的战略中心。 许多SAP云解决方案已预先与身份认证集成在一起。 在这些情况下，使用身份验证不会产生任何额外费用–因此，如果您使用的是SAP S/4HANA公共云，SAP SuccessFactors，SAP集成业务规划（IBP）或SAP Cloud Platform，则您已经有一个身份验证租户，并且您可能在不知道的情况下使用它。

下面以一个案例来说明IAS如何来保护SAP云平台系统。
### 1 SAP Cloud Platform和IAS IDP互信
使用SAML2.0协议实现两个系统IDP互信。
1. 登录Cloud Platform Cockpit,进入Security一览。讲Local Service Provider Type改成Custom。保存好，点击下方的**Get Metadata**。保存这个文件下面需要用到。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-01.png)

2. 登录到IAS系统，新建用于SAP Cloud Platform的应用。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-02.png)

3. 将第一步获取的到的metadata导入。将SAML2.0的Metadata导入到
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-03.png)
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-04.png)

4. 获取IAS平台IDP的SAML2.0 metadata。用于SCP平台互信证书使用。
选择Tenant Setting,从SAML2.0 Configuration获取。Download Metadata,现在已经轻车熟路。注意这个文件有用，别跟上面下载的metadata搞混了。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-05.png)
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-06.png)

5. 回到SAP Cloud Platform云平台 Cockpit,导入IAS SAML2.0 Metadata。
Security -> Trust -> Application Identity Provider -> Add Trusted Identity Provider,导入Metadata。同时选择导入IDP。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-07.png)
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-08.png)

经过以下五步我们完成了系统间互信。接下来可以打开WebIDE，试试是不是你的SSO登录页面是不是指向到你新配置的IAS了。

但是问题来了，系统告诉我没有权限。不要急接来下就需要为你的在IAS上面的用户配置相应的访问权限。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-09.png)



### 2 SAP Cloud Platform Service与IAS Group信息相互映射
1. 在IAS创建WebIDE Group，点击User Group -> add,新建Group。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-10.png)

2. 将新建的Group分配给具体的User, User Management -> User -> User Groups -> Assgin Group。现在你就拥有了WebIDE权限啦。但是还不够，接下来需要在Cockpit端多Mapping。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-11.png)

3. 回到Cockpit, Security-> Authorization -> Groups,新建一个Groups CP-WebIDE, Assign Roles, 我给的了一个DiAdministrator的权限，你当然也可以给Developer的权限。按需分配。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-13.png)

4. 将IAS定义的Group和SCP定义的Group关系Mapping起来。
Trust-> Application Identify Provider -> 点击 IDP连接，会跳到另外的配置地方。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-12.png)
Group -> Add Assertion-Based Group -> 将两个Group Mapping关系填好，保存。快成功啦。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-14.png)

5. 最后一步，回头IAS，将Group Assert配置上，这样在返回的SAMLAssertion里面才会有Group这个字段。
Application -> apj-neo -> Assertion Attribute
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-15.png)

Add,将Group这个重要字段加上。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-16.png)

经过以上五步完成了所有工作，接下来我看再次打开webIDE这个服务，注意要将Cache清一下，或者使用无痕模式打开。
是不是成功啦，可以看见我的User已经变成了我在IAS上面的P User了。
![1](https://blog-1252828110.cos.ap-shanghai.myqcloud.com/img/02/ias-17.png)

### 3 总结
通过以上配置完成IAS和SCP的互信认证。关于SCP的其余服务，比如CPI， Portal, Workflow等等，都可以按照以上方法去配置。

当然我这边只是冰山一角，IAS的还有更高阶应用，待您慢慢去体验和使用。

加油朋友们。如果疑问可以Email我，sam.sun02@sap.com。

## 参考
1. Setup a Platform Identity Provider for SAP Cloud Platform https://blogs.sap.com/2018/07/18/setup-a-platform-identity-provider-for-sap-cloud-platform/

2. Enterprise Security Services – Cloud Identity Services https://blogs.sap.com/2020/02/19/the-cloud-enterprise-security-suite-cloud-identity-services/