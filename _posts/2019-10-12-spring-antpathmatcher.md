---
layout: post
title: "SpringBoot AntPathMatcher"
subtitle: 'SpringBoot AntPathMatcher'
author: "WongCU"
header-style: text
tags:
  - SpringBoot
---


可以做URLs匹配，规则如下

1. ？匹配一个字符
2. *匹配0个或多个字符
3. **匹配0个或多个目录

用例如下

- /trip/api/*x    匹配 /trip/api/x，/trip/api/ax，/trip/api/abx ；但不匹配 /trip/abc/x；
- /trip/a/a?x    匹配 /trip/a/abx；但不匹配 /trip/a/ax，/trip/a/abcx
- /**/api/alie    匹配 /trip/api/alie，/trip/dax/api/alie；但不匹配 /trip/a/api
- /**/*.htmlm   匹配所有以.htmlm结尾的路径



样例

```java
public static void main(String[] args) {
        String pattern = "/{id}/{uuid}/{ext}/";
        String path = "/xxxx/12/145/";
        AntPathMatcher matcher = new AntPathMatcher();
        System.out.println(matcher.match(pattern, path)); //输出 true
        Map<String, String> result = matcher.extractUriTemplateVariables(pattern, path);
        System.out.println(result);//输出 {id=xxxx, uuid=12, ext=145}
    }
```

