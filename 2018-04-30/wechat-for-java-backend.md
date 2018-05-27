---
title: 微信公众号后台在SpringBoot2.0中的实现（上）
description: 微信公众号后台在SpringBoot2.0中的实现（上）
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/wechat-logo.png'
category: JAVA
abbrlink: 637facd6
date: 2018-05-01 18:13:02
---
现在微信在中国作为最重要的社交软件，相信很多人都使用微信公众号。
而且微信公众号也成为一些企业传播资讯最好的平台。
那么今天就来讲下微信公众号后台具体如何来实现。
一、申请公众号
-------
首先我们根据自己活着企业的所需的东西，选定好一下几种类别的账号。
![wechat-1](https://weaf.oss-cn-beijing.aliyuncs.com/wechat-1.png)
不同的账户拥有的权限不尽相同。
这个我们可以根据权限文档进行查看！

[接口权限：https://mp.weixin.qq.com/advanced/advanced?action=table&token=1776791094&lang=zh_CN](https://mp.weixin.qq.com/advanced/advanced?action=table&token=1776791094&lang=zh_CN)
然后就是根据自己的接口的要求，去审批。
二、设置基本配置
---------
![wechat-2](https://weaf.oss-cn-beijing.aliyuncs.com/wechat-2.png)
这里需要我们先开启自己的开发者密码。（这个是我们在开发过程中需要使用的）

服务器地址（URL）:这里需要设置公网可访问的接口

令牌（Token）：这块我们可以设置为一个32为UUID的字符串。

更多细节请参考官方文档：[https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421135319](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421135319)

三、代码的实现
其他的也就不多说了，直接上代码。

我们首先创建一个介入指南中，微信需要验证的接口。
``` java

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;

/**---------*/

@RequestMapping("verification")
public String weChatVerification(String signature, String timestamp, String nonce, String echostr, HttpServletRequest request) throws AesException {
    log.info("微信校验参数：signature={}，timestamp={}，nonce={}，echostr={}",signature,timestamp,nonce,echostr);
    if(signature.equals(getSHA1(token,timestamp,nonce)))
            return echostr;
    else
    {
        return "";
    }

}

private String getSHA1(String token, String timestamp, String nonce) throws AesException
    {
        try {
            String[] array = new String[] { token, timestamp, nonce };
            StringBuffer sb = new StringBuffer();
            // 字符串排序
            Arrays.sort(array);
            for (int i = 0; i < 3; i++) {
                sb.append(array[i]);
            }
            String str = sb.toString();
            // SHA1签名生成
            MessageDigest md = MessageDigest.getInstance("SHA-1");
            md.update(str.getBytes());
            byte[] digest = md.digest();

            StringBuffer hexstr = new StringBuffer();
            String shaHex = "";
            for (int i = 0; i < digest.length; i++) {
                shaHex = Integer.toHexString(digest[i] & 0xFF);
                if (shaHex.length() < 2) {
                    hexstr.append(0);
                }
                hexstr.append(shaHex);
            }
            return hexstr.toString();
        } catch (Exception e) {
            e.printStackTrace();
            throw new AesException(AesException.ComputeSignatureError);
        }
    }
```
现在就已经接入了微信公众平台。





