---
layout: post
title: one way authentication by with pkcs12.pfx file
description: "Linux, curl, openssl, httpclient4.5"
date: 2018-03-14
tags: [curl, openssl, httpclient,pkcs12]
comments: true
share: true
---

如何在只有pkcs12.pfx证书的情况下，进行单方认证（Server验证client, Client侧不验证Server(因为没有CA证书）)  

#为了确保Server侧的Service确实能够正常连接，有必要用curl进行check  

用curl 进行连接前，确保 curl 使用 openssl编译过。  

[root]# curl --version  

curl 7.29.0 (x86_64-unknown-linux-gnu) libcurl/7.29.0 OpenSSL/1.0.1e zlib/1.2.  
Protocols: dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp smtp smtps telnet tftp  
Features: IPv6 Largefile NTLM NTLM_WB SSL libz  

准备工作  
1. 客户端证书  
[root]#openssl pkcs12 -in client_certificate.pfx -nodes -out client.pem  

2. 客户端私钥  
[root]#openssl rsa -in client.pem -out client_key.pem  
或者  
[root]#openssl pkcs12 -in client_certificate.pfx -nocerts -nodes | openssl rsa > client_key.pem  

3. 验证Servcie Server连接  
如果需要代理，执行前先设置代理<br>
[root]#export https_proxy=https://web-proxy.your.com:XXXX/  
[root]#curl -k -v --cert "./cert.pem" --key "./keynp.pem" -H "Content-type: application/json" -X POST -d '{"name":"01","Id":"001"}' "https://services.your.com/service/v1/api"  

这里 option[-k]就表示client端不验证Server.  


#用HttpClient4.5来实现单方认证连接Server的Service  

1.<br>
RequestConfig config = RequestConfig.custom()  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.setConnectTimeout(this.connTimeout * 1000)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.setSocketTimeout(this.socketTimeout * 1000)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.build();  
2.<br>
try {  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;KeyStore keyStore = KeyStore.getInstance("PKCS12");  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//PKCS12证书  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;InputStream instream =This.class.getClassLoader().getResourceAsStream(this.vNextCrtFile);  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//从Server端导出PKCS12证书时的password  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;keyStore.load(instream, this.vNextPassword.toCharArray());  
  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SSLContext sslcontext = SSLContexts.custom()  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.loadKeyMaterial(keyStore, this.vNextPassword.toCharArray())  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; //忽略Client端对Server的验证。类似 curl命令的[-k]。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; //否则会提示[unable to find valid certification path to requested target]  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.loadTrustMaterial(new TrustSelfSignedStrategy())  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.build();  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sslsf = new SSLConnectionSocketFactory(sslcontext, new String[] { "TLSv1.2" }, null,  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SSLConnectionSocketFactory.getDefaultHostnameVerifier());  
&nbsp;&nbsp;&nbsp;&nbsp;} catch (Exception e) {  
&nbsp;&nbsp;&nbsp;&nbsp;//TODO  
&nbsp;&nbsp;&nbsp;&nbsp;}  

3.<br>
&nbsp;try (CloseableHttpClient httpclient = HttpClientBuilder.create().useSystemProperties()  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.setDefaultRequestConfig(config)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.setSSLSocketFactory(sslsf).build()) {  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpPost httpPost = new HttpPost(this.url2VNext);  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;StringEntity entity = new StringEntity(json);  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;httpPost.setEntity(entity);  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;// Create a custom response handler  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ResponseHandler<Integer> responseHandler = response -> {<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;int status = response.getStatusLine().getStatusCode();<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;return status;<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;};<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;statusCode = httpclient.execute(httpPost, responseHandler);<br>
&nbsp;&nbsp;} catch (Exception e) {<br>
&nbsp;&nbsp;&nbsp;&nbsp;//TODO <br>
&nbsp;&nbsp;}<br>

 注意用下面的JVM参数设置代理，需要使用HttpClientBuilder.create().useSystemProperties()<br>
 ※JVM参数<br>
 ※-Djava.net.useSystemProxies=true -Dhttps.proxyHost=web-proxy.your.com -Dhttps.proxyPort=8080<br>
---