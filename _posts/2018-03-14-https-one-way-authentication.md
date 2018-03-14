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
如果需要代理，执行前先设置代理。 
[root]#export https_proxy=https://web-proxy.your.com:XXXX/  
[root]#curl -k -v --cert "./cert.pem" --key "./keynp.pem" -H "Content-type: application/json" -X POST -d '{"name":"01","Id":"001"}' "https://services.your.com/service/v1/api"  

这里 option[-k]就表示client端不验证Server.  


#用HttpClient4.5来实现单方认证连接Server的Service  

1.  
RequestConfig config = RequestConfig.custom()  
\             	.setConnectTimeout(this.connTimeout * 1000)  
\		.setSocketTimeout(this.socketTimeout * 1000)  
\		.build();  
2.  
\try {  
\	KeyStore keyStore = KeyStore.getInstance("PKCS12");  
\	//PKCS12证书  
\	InputStream instream =This.class.getClassLoader().getResourceAsStream(this.vNextCrtFile);  
\	//从Server端导出PKCS12证书时的password  
\	keyStore.load(instream, this.vNextPassword.toCharArray());  
\ 
\	SSLContext sslcontext = SSLContexts.custom()  
\			.loadKeyMaterial(keyStore, this.vNextPassword.toCharArray())  
\			 //忽略Client端对Server的验证。类似 curl命令的[-k]。 
\			 //否则会提示[unable to find valid certification path to requested target]  
\			.loadTrustMaterial(new TrustSelfSignedStrategy())  
\			.build();  
\	sslsf = new SSLConnectionSocketFactory(sslcontext, new String[] { "TLSv1.2" }, null,  
\				SSLConnectionSocketFactory.getDefaultHostnameVerifier());  
\   } catch (Exception e) {  
\       //TODO  
\   }  
\
3.  
\  try (CloseableHttpClient httpclient = HttpClientBuilder.create().useSystemProperties()  
\				.setDefaultRequestConfig(config)  
\				.setSSLSocketFactory(sslsf).build()) {  
\           HttpPost httpPost = new HttpPost(this.url2VNext);  
\           StringEntity entity = new StringEntity(json);  
\           httpPost.setEntity(entity);  
\           // Create a custom response handler  
\           ResponseHandler<Integer> responseHandler = response -> {  
\                int status = response.getStatusLine().getStatusCode();  
\                return status;  
\           };  
\	statusCode = httpclient.execute(httpPost, responseHandler);  
\   } catch (Exception e) {   
\    //TODO  
\   }   

 注意用下面的JVM参数设置代理，需要使用HttpClientBuilder.create().useSystemProperties()  
 ※JVM参数  
 ※-Djava.net.useSystemProxies=true -Dhttps.proxyHost=web-proxy.your.com -Dhttps.proxyPort=8080  
---