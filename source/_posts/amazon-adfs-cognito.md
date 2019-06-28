---
title: 使用企业AD接入Amazon服务
date:  2019-06-30
tags:
    - 架构
    - 部署
---


# Overview

使用Microsoft AD 目录，通过AWS Cognito认证接入Web Service的认证流程如下图所示，我们的安装步骤将按照这个顺序依次进行：

![Figure: overview](amazon-adfs-cognito/mainDiagram.png)



# Install && Config

## 1. 安装AD和ADFS

在AWS EC2中启动Windows_Server-2012-R2实例。之后使用远程桌面登录该实例并安装AD/ADFS服务。

详细安装过程请参考[视屏](https://www.youtube.com/watch?v=yagCgNvl3vs)

## 2. 配置ADFS并导出SAML文件

1. 

2. 

3. 导出SAML文件 [](#23)

## 3. 创建Amazon Cognito User Pool

1. 登录 [Amazon Cognito console](https://console.aws.amazon.com/cognito/home)

2. 选择创建User Pool，出入pool名字(ad-test)，点击逐步介绍设置

    ![Figure: overview](amazon-adfs-cognito/user-pool-1.png)

3. 属性

    ![Figure: overview](amazon-adfs-cognito/user-pool-2.png)

4. 策略

    ![Figure: overview](amazon-adfs-cognito/user-pool-3.png)

5. 设备

    ![Figure: overview](amazon-adfs-cognito/user-pool-4.png)

6. 添加应用客户端

    ![Figure: overview](amazon-adfs-cognito/user-pool-5.png)

7. 审核并创建Pool

    ![Figure: overview](amazon-adfs-cognito/user-pool-6.png)

8. 在*联合身份验证->身份提供商*上传步骤2-10导出的ADFS的SAML文件(adfs.xml)

    ![Figure: overview](amazon-adfs-cognito/user-pool-7.png)

9. 在*联合身份验证->属性映射*添加
    * SAML属性: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress
    * User Pool 属性: email

    ![Figure: overview](amazon-adfs-cognito/user-pool-8.png)

10. 在*应用程序继承->域名*设置Cognito域名

    ![Figure: overview](amazon-adfs-cognito/user-pool-9.png)

## 4. 发布web app

1. 下载测试使用的APP项目: git clone git@github.com:leodrak/ADFSCognito.git