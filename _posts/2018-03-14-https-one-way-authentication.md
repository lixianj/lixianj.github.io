---
layout: post
title: one way authentication by one way with pkcs12.pfx file
description: "Linux, curl, openssl, httpclient4.5"
date: 2018-03-14
tags: [curl, openssl, httpclient,pkcs12]
comments: true
share: true
---

如何在只有pkcs12.pfx证书的情况下，进行单方认证（Server验证client, Client侧不验证Server(因为没有CA证书）)

#为了确保Server侧的Service确实能够正常连接，有必要用curl进行check

用curl 进行连接前，确保 curl 使用 openssl编译过。
[root]# curl --version__
curl 7.29.0 (x86_64-unknown-linux-gnu) libcurl/7.29.0 OpenSSL/1.0.1e zlib/1.2.__
Protocols: dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp smtp smtps telnet tftp__
Features: IPv6 Largefile NTLM NTLM_WB SSL libz__

准备工作__
1. 客户端证书__
#openssl pkcs12 -in client_certificate.pfx -nodes -out client.pem__

2. 客户端私钥__
#openssl rsa -in client.pem -out client_key.pem__
或者__
#openssl pkcs12 -in client_certificate.pfx -nocerts -nodes | openssl rsa > client_key.pem__

3. 验证Servcie Server连接__
如果需要代理，执行前先设置代理。__
[root]#export https_proxy=https://web-proxy.your.com:XXXX/__
[root]#curl -k -v --cert "./cert.pem" --key "./keynp.pem" -H "Content-type: application/json" -X POST -d '{"name":"01","Id":"001"}' "https://services.your.com/service/v1/api"__

这里 option[-k]就表示client端不验证Server.__


#用HttpClient4.5来实现单方认证连接Server的Service__

1.__
RequestConfig config = RequestConfig.custom()__
              	.setConnectTimeout(this.connTimeout * 1000)__
		.setSocketTimeout(this.socketTimeout * 1000)__
		.build();__
2.__
try {__
	KeyStore keyStore = KeyStore.getInstance("PKCS12");__
	//PKCS12证书__
	InputStream instream =This.class.getClassLoader().getResourceAsStream(this.vNextCrtFile);__
	//从Server端导出PKCS12证书时的password__
	keyStore.load(instream, this.vNextPassword.toCharArray());__
__
	SSLContext sslcontext = SSLContexts.custom()__
			.loadKeyMaterial(keyStore, this.vNextPassword.toCharArray())__
			 //忽略Client端对Server的验证。类似 curl命令的[-k]。__
			 //否则会提示[unable to find valid certification path to requested target]__
			.loadTrustMaterial(new TrustSelfSignedStrategy())__
			.build();__
	sslsf = new SSLConnectionSocketFactory(sslcontext, new String[] { "TLSv1.2" }, null,__
				SSLConnectionSocketFactory.getDefaultHostnameVerifier());__
	} catch (Exception e) {__
}__

3.__
try (CloseableHttpClient httpclient = HttpClientBuilder.create().useSystemProperties()__
				.setDefaultRequestConfig(config)__
				.setSSLSocketFactory(sslsf).build()) {__
           HttpPost httpPost = new HttpPost(this.url2VNext);__
           StringEntity entity = new StringEntity(json);__
           httpPost.setEntity(entity);__
           // Create a custom response handler__
           ResponseHandler<Integer> responseHandler = response -> {__
                int status = response.getStatusLine().getStatusCode();__
                return status;__
           };__
	statusCode = httpclient.execute(httpPost, responseHandler);__
} catch (Exception e) {__
__
}__

 注意用下面的JVM参数设置代理，需要使用HttpClientBuilder.create().useSystemProperties()__
 ※JVM参数__
 ※-Djava.net.useSystemProxies=true -Dhttps.proxyHost=web-proxy.your.com -Dhttps.proxyPort=8080__
---