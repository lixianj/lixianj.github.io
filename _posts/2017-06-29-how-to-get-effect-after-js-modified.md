---
layout: post
title: "SpringBoot环境下如何使修改后的JS，CSS文件生效"
description: "JS,CSS"
date: 2017-06-29
tags: [SpringBoot,JS]
comments: true
share: true
---

SpringBoot环境下如何让修改后的JS，CSS文件生效，方法如下。

1.生产环境下,利用SpringBoot特性，配置如下。

@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {


    @Bean
    public ResourceUrlEncodingFilter resourceUrlEncodingFilter() {
        return new ResourceUrlEncodingFilter();
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {

       registry.addResourceHandler("/**/*.js", "/**/*.css", "/**/*.less")
                .addResourceLocations("classpath:/static/")
                .setCachePeriod(ONE_YEAR_SECONDS)
                .resourceChain(true) // 生产环境为true，开发环境为false
                .addResolver(new VersionResourceResolver()
                        .addContentVersionStrategy("/**/*.js", "/**/*.css"));
    }
}

重点是以下两行。
		.resourceChain(true) // 生产环境为true，开发环境为false
                .addResolver(new VersionResourceResolver()
                        .addContentVersionStrategy("/**/*.js", "/**/*.css"));
注意点：
Service需要重新启动。


2.没有1的配置的情况下,
	Chrome环境，安装cache killer,然后将该插件开启（ON）。
	IE环境，在进入修改的JS或者CSS页面之前，按F12（Debug模式），即可。

---