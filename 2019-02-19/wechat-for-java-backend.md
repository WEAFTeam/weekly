---
title: 微信公众号后台在SpringBoot2.0中的实现（中）
description: 微信公众号后台在SpringBoot2.0中的实现（中）
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/wechat-logo.png'
category: JAVA
date: 2019-02-19 18:13:02
---
继之前的微信公众号实现，现在第二篇来袭，写了一些最基本的使用和配置。

一、申请公众号
-------
此次我们开发使用的是服务号，大部分微信接口都是拥有权限的。
![wechat-1](https://weaf.oss-cn-beijing.aliyuncs.com/wechat-1.png)
不同的账户拥有的权限不尽相同。
这个我们可以根据权限文档进行查看！

[接口权限：https://mp.weixin.qq.com/advanced/advanced?action=table&token=1776791094&lang=zh_CN](https://mp.weixin.qq.com/advanced/advanced?action=table&token=1776791094&lang=zh_CN)

二、设置基本配置
---------
这里最基本的**基本配置**在上一节中我们已经说到了，这里我们需要配置一些其他的东西。
首先我们需要绑定开发者，开发者绑定位置可已在**开发者工具**->**web开发者工具**里边进行绑定，这样我们就可以使用web开发工具，并调试自己的界面
具体文档和下载位置：[https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1455784140](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1455784140)

另外我们需要配置**公众号设置**->**功能设置**，这里边的*业务域名*、*JS接口安全域名*、*网页授权域名*，这样我们的工作好就可以使用js-sdk，网页授权等功能。

还要记得在**基本配置**里边配置好**IP白名单**哦。

三、个性化菜单
------

因为打开开发者中心，
导致自定义菜单不可用，这样我们需要通过接口定义菜单，
具体我们可以通过[微信公众平台接口调试工具：https://mp.weixin.qq.com/debug?token=1185341662&lang=zh_CN](https://mp.weixin.qq.com/debug?token=1185341662&lang=zh_CN)进行设置就可以了。

菜单定义可以通过定义好的json数据进行配置
```json
{
     "button":[
     {    
        "type":"click",
        "name":"今日歌曲",
         "key":"V1001_TODAY_MUSIC" },
    {     "name":"菜单",
        "sub_button":[
        {            
            "type":"view",
            "name":"搜索",
            "url":"http://www.soso.com/"},
            {
                         "type":"miniprogram",
                         "name":"wxa",
                         "url":"http://mp.weixin.qq.com",
                         "appid":"wx286b93c14bbf93aa",
                         "pagepath":"pages/lunar/index"
            },
             {
        "type":"click",
        "name":"赞一下我们",
        "key":"V1001_GOOD"
           }]
 }],
"matchrule":{
  "tag_id":"2",
  "sex":"1",
  "country":"中国",
  "province":"广东",
  "city":"广州",
  "client_platform_type":"2",
  "language":"zh_CN"
  }
}
```
![wechat-2](https://weaf.oss-cn-beijing.aliyuncs.com/we-chat-1.png)

更多参数请看文档：[https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1455782296](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1455782296)

四、代码的实现
-------
之前代码实现，只是实现了最近本的接入，其他相关的都没有涉及。
这次我们涉及的比较多哦。

### 1.请求触发响应

通过第三步，我们设置了个性化菜单，有部分我们添加的按钮，是需要触发事件的，那么我们如何响应，并处理呢？

上代码,这里我们配置成功后就可以修改响应接受微信得到消息了。具体修改如下↓。
``` java
@RequestMapping("verification")
public String weChatVerification(String signature, String timestamp, String nonce, String echostr, HttpServletRequest request) throws AesException {
    log.info("微信校验参数：signature={}，timestamp={}，nonce={}，echostr={}", signature, timestamp, nonce, echostr);
    String res = weChatService.processRequest(request);
    log.info("微信消息回复体：{}", res);
    return res;
}
```

处理请求代码
``` java
@Override
public String processRequest(HttpServletRequest request) {
    String respMessage = null;
    try {
        // 默认返回的文本消息内容
        String respContent = "请求处理异常，请稍候尝试！";

        // xml请求解析
        Map<String, String> requestMap = MessageUtil.getInstance().parseXml(request);
        log.info("微信消息体：{}", requestMap);
        // 发送方帐号（open_id）
        String fromUserName = requestMap.get("FromUserName");
        // 公众帐号
        String toUserName = requestMap.get("ToUserName");
        // 消息类型
        String msgType = requestMap.get("MsgType");

        // 回复文本消息
        TextMessage textMessage = new TextMessage();
        textMessage.setToUserName(fromUserName);
        textMessage.setFromUserName(toUserName);
        textMessage.setCreateTime(System.currentTimeMillis());
        textMessage.setMsgType(MessageUtil.RESP_MESSAGE_TYPE_TEXT);

        // 文本消息
        if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_TEXT)) {
            respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
        }
        // 图片消息
        else if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_IMAGE)) {
            respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
        }
        // 地理位置消息
        else if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_LOCATION)) {
            respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
        }
        // 链接消息
        else if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_LINK)) {
            respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
        }
        // 音频消息
        else if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_VOICE)) {
            respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
        }
        // 事件推送
        else if (msgType.equals(MessageUtil.REQ_MESSAGE_TYPE_EVENT)) {
            // 事件类型
            log.info("微信公众平台接收到事件：{}", msgType);
            String eventType = requestMap.get("Event");
            // 订阅
            if (eventType.equals(MessageUtil.EVENT_TYPE_SUBSCRIBE)) {
                respContent = "谢谢您的关注，您可以通过点击下方“扫码入群”来加入我们的社群了解更多资讯和新闻哦(●'◡'●)！";
            }
            // 取消订阅
            else if (eventType.equals(MessageUtil.EVENT_TYPE_UNSUBSCRIBE)) {
                // TODO 取消订阅后用户再收不到公众号发送的消息，因此不需要回复消息
            }
            // 自定义菜单点击事件
            else if (eventType.equals(MessageUtil.EVENT_TYPE_CLICK)) {
                // 事件KEY值，与创建自定义菜单时指定的KEY值对应
                String eventKey = requestMap.get("EventKey");
                log.info("微信公众平台接收到自定义菜单事件：eventType={},key={}", eventType, eventKey);
                if (eventKey.equals("JOIN_US")) {
                    PicMessage picMessage = new PicMessage();
                    picMessage.setToUserName(fromUserName);
                    picMessage.setFromUserName(toUserName);
                    picMessage.setCreateTime(new Date().getTime());
                    picMessage.setMsgType(MessageUtil.RESP_MESSAGE_TYPE_IMAGE);
                    picMessage.setMediaId("6X_RduJsEvCbkeXlbtbMS61PHRQtMBRMf52oTpzBj8k");
                    return picMessage.toXMLMessag();
                } else {
                    respContent = "点击了按钮";
                }
            }
        }
        textMessage.setContent(respContent);
        respMessage = MessageUtil.getInstance().textMessageToXml(textMessage);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return respMessage;
}
```

具体实现的代码整理好后我会通过github上传，届时请关注仓库[https://github.com/songyaxu/wechat-service](https://github.com/songyaxu/wechat-service)

### 2.access_token刷新实现

我们使用公众号过程中会通过access_token获取相关信息，而access_token还具有实效性，那么我们该如何实现实时获取，且正确呢?

我这里实现的方式是最简单的方式，就是使用redis缓存起来，获取不到时重新获取。因为业务小的原因，我这里选用内存存储缓存GuavaCache进行缓存。

具体实现↓：
``` java
@Value("${wechat.appId}")
private String appId;

@Value("${wechat.appSecret}")
private String appSecret;

@Autowired
private GuavaCacheService<String,String> guavaCacheService;

private final String ACCESS_KEY = "ACCESS_KEY_YUNCAIYUAN";

private final String GET_ACCESS_TOKEN_URL="https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential";
   
	
		...


public String getAccessToken() {
    String value = guavaCacheService.getIfPresent(ACCESS_KEY);
    if (StringUtils.isNotBlank(value)){
        return value;
    }
    String accessToken = null;
    String getUrl = GET_ACCESS_TOKEN_URL + "&appid=" + appId + "&secret=" + appSecret;
    try {
        String result = HttpUtil.get(getUrl);
        JSONObject jsonObject = JSONObject.parseObject(result);
        if (null != jsonObject) {
            try {
                accessToken = jsonObject.getString("access_token");
                //Long expiresIn = jsonObject.getLong("expires_in");
                guavaCacheService.put(ACCESS_KEY,accessToken);
            } catch (JSONException e) {
                accessToken = StringUtils.EMPTY;
                log.error("获取token失败 errcode:{} errmsg:{}", jsonObject.getIntValue("errcode"), jsonObject.getString("errmsg"));
            }
        }
        log.info("获取了微信AccessToken：{}", accessToken);
        return accessToken;
    } catch (Exception e) {
        log.error("请求失败" + e.getMessage());
    }
    return StringUtils.EMPTY;
}
```

具体GuavaCache实现内存缓存机制可以查看:[https://github.com/songyaxu/guava-cache](https://github.com/songyaxu/guava-cache)

五、参考链接
------

1.微信工作平台技术文档：[https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1445241432](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1445241432)













