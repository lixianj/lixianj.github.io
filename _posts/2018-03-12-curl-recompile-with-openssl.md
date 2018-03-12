---
layout: post
title: "在CURL里面用OPENSSL替换已有的NSS模块"
description: "Linux, curl, openssl"
date: 2018-03-12
tags: [curl,recompile, NSS, OPENSSL]
comments: true
share: true
---

如题，方法如下。

1. https://curl.haxx.se/download/ 
   下载合适的版本。
2. tar zxf curl-7.29.0.tar.gz
   Linux环境下解压。
3. ./configure --without-nss --with-ssl=/usr/include/openssl --disable-shared
   选项[--without-nss] 重要。

4. make

5. make install

6. reboot
   重新启动Server

7. echo "/usr/local/lib" >> /etc/ld.so.conf && ldconfig

8. curl --verion
   验证结果
---