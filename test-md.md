# 快速集成指南


## API调用时序图

在了解下面创建应用以及安装SDK过程之前，我们先从一张时序图了解一下整个会议中API的调用顺序，如果开发过程中遇到问题不如我们回到这张图中查找答案。  


```
sequenceDiagram
    participant clientA
　　participant RTCMeetKit
　　participant clientB
　　
　　clientA-->clientB: init
　　
　　clientA-->>RTCMeetKit: setLocalVideoCapturer()
　　RTCMeetKit-->>clientA: onSetLocalVideoCapturerResult == 0
　　Note over clientB: 省略clientA重复操作
　　clientA-->>RTCMeetKit: joinRTC()
　　RTCMeetKit-->>clientA: onRTCJoinMeetOK
　　
　　loop Notify
　　　　RTCMeetKit->RTCMeetKit: 将其他参会人员通知给clientA
　　end
　　
　　RTCMeetKit-->>clientA: onRTCOpenVideoRender
　　Note over clientA: 创建参会人员的视图
　　RTCMeetKit-->>clientA: onRTCRemoteStream
　　
　　clientA-->clientB: switch Device(切换设备)
　　clientA-->>RTCMeetKit: switchDevice()
　　RTCMeetKit-->>clientA: onLocalStreamUpdated
　　RTCMeetKit-->>clientB: onRTCCloseVideoRender
　　RTCMeetKit-->>clientB: onRTCOpenVideoRender
　　RTCMeetKit-->>clientB: onRTCRemoteStream
　　
　　clientA-->clientB: share Scrren(屏幕共享)
　　clientB-->>RTCMeetKit: setUserShareEnable()
　　RTCMeetKit-->>clientB: onRTCSetUserShareEnableResult == OK
　　clientB-->>RTCMeetKit: setUserShareInfo()
　　clientB-->>RTCMeetKit: startScreenCap(stream)
　　RTCMeetKit-->>clientA: onRTCUserShareOpen
　　RTCMeetKit-->>clientA: onRTCOpenVideoRender
　　RTCMeetKit-->>clientA: onRTCRemoteStream
　　
　　clientA-->clientB: IM
　　clientB-->>RTCMeetKit: sendUserMessage()
　　RTCMeetKit-->>clientA: onRTCUserMessage
　　Note over clientA: IM不仅仅用来发送实时消息，<br/>同时还可以用来做场控，<br/>譬如：禁言、关闭远方摄像头<br/>传输、踢人，在互动直播SDK<br/>中，还能用来反向邀请<br/>等等等功能
　　
　　clientA-->clientB: leaveRTC
　　clientB-->>RTCMeetKit: leaveRTC()
　　RTCMeetKit-->>clientA: onRTCCloseVideoRender
　　Note over clientA: 移除该退会参会人员的图像
　　
　　#RTCMeetKit-->>clientB: onSetLocalVideoCapturerResult == 0
　　#clientB-->>RTCMeetKit: joinRTC()
　　
```  
  

如上图所示，集成十分的简单，我们只需做两件事，告诉服务端我们要干什么，其他的一切在回调中处理，可以理解成websocket的```emit``` 和 ```on```事件。


**开发口诀：**  
创建的本地预览视图和屏幕共享视图(例如：```onSetLocalVideoCapturerResult```中创建的本地预览视图)，在退会或自己被踢出会议等等需要移除本地预览视图的情况时自己移除，否则，从```onRTCCloseVideoRender```回调中移除（自己的事情，自己干）

## 创建应用 

> 说明：  
登录anyRTC用户平台，前往[管理中心](https://www.anyrtc.io/manage)创建一个**网站应用**    
填写应用相关信息，等待审核通过获取到```开发者ID```，```AppID```，```AppKey```，```AppToken```等应用信息。  
如果需要屏幕共享需要安装anyRTC-ScreenShare Chrome拓展程序,并且引用```Screen-Capturing.js```，使用anyRTC-ScreenShare Chrome拓展程序，本地环境请使用```127.0.0.1```,使用```localhost```将不能使用拓展程序。

## 安装SDK  

> [下载SDK文件](https://www.anyrtc.io/download/demo_sdk_web/anyrtc_RTMeet.zip)到项目中直接引用。（Web SDK现仅支持直接引用，后续支持```npm```安装）    
按照以下顺序应用

```
<script src="https://www.anyrtc.io/websdk/RTMeet/2.4.0/adapter.js"></script>
<script src="https://www.anyrtc.io/websdk/RTMeet/2.4.0/xmlhttp.js"></script>
<script src="https://www.anyrtc.io/websdk/RTMeet/2.4.0/anyrtc.js"></script>
<script src="https://www.anyrtc.io/websdk/RTMeet/2.4.0/RTMeetKit.js"></script>
<!-- 如果需要屏幕共享 -->
<script src="https://www.anyrtc.io/websdk/RTMeet/2.4.0/Screen-Capturing.js"></script>
```

## 创建实例  

```
//创建会议实例对象
var RTMeetkit = RTMeetKit || window.RTMeetKit;
var meetKit = new RTMeetkit();
```

> ## RTMeetKit SDK初始化 

> API文档中的初始化方法，必须在调用joinRTC方法之前，API方法则必须在joinRTC成功之后调用。填写刚刚创建的网页app信息和开发者ID。

```
var RTMeetkit = RTMeetKit || window.RTMeetKit;
var meetKit = new RTMeetkit();
    
/**
 *  配置开发者信息
 *  @params[0]      strDeveloperId      平台开发者信息
 *  @params[1]      strAppId            平台应用appid
 *  @params[2]      strAppKey           平台应用appkey
 *  @params[3]      strAppToken         平台应用appToken
 *  @params[4]      strDomain           平台应用绑定的网站域名
 **/
meetKit.initEngineWithAnyRTCInfo();
```

## 配置

> 配置非必须调用方法，均有默认值

> #### 设置音频模式

```
/**
*  设置会议音频模式
*  @param 	          bAudioOnly  默认false
*  API说明                设置音频会议模式
**/
meetKit.setAudioModel(true);
```

> ### 设置视频质量

```
/**
*  设置视频会议质量
*  @params strVideoMode       默认：AnyRTCVideoQuality_Low2
*  参数说明：
*  AnyRTCVideoQuality_Height3   1280*720    2048 kbps
*  AnyRTCVideoQuality_Height2   1280*720    1280 kbps
*  AnyRTCVideoQuality_Height1   640*480     1024 kbps
*  AnyRTCVideoQuality_Medium3   640*480     768 kbps
*  AnyRTCVideoQuality_Medium2   640*480     512 kbps
*  AnyRTCVideoQuality_Medium1   640*480     384 kbps
*  AnyRTCVideoQuality_Low3      352*288     384 kbps
*  AnyRTCVideoQuality_Low2      352*288     256 kbps
*  AnyRTCVideoQuality_Low1      320*240     128 kbps
**/
meetKit.setVideoMode();

```

## 加入会议 

> 加入会议流程：    
采集本地摄像头--（采集本地摄像头成功）-->加入会议--（监听其他与会者加入会议、监听其他会议人视频流回调）

```
var roomId = '9527', jUserData = {};
//采集本地摄像头
meetKit.setLocalVideoCapturer();
//监听采集本地摄像头结果
meetKit.on('onSetLocalVideoCapturerResult', function(nCode, videoRender, stream){
    if (nCode === 0) {
        jUserData["userId"] = 'u9527';
        jUserData["nickName"] = 'UN9527';
        jUserData["headUrl"] = '';
        //加入会议,未使用主持人模式时，第二个参数默认选择false
        meetKit.joinRTC(roomId, false, jUserData["userId"], JSON.stringify(jUserData));
        //自己的事情自己干，在退会或自己被踢出会议等等需要移除本地预览视图的情况时自己移除
        document.body.appendChild(videoRender);
    } else if (nCode == 7) {
        alert("打开摄像头错误：：" + videoRender);
    }
});
//加入会议成功
meetKit.on('onRTCJoinMeetOK', function(){
    console.log('onRTCJoinMeetOK');
});
// 远程人员加入
meetKit.on("onRTCOpenVideoRender",function(strPeerId, strPubId, dRander, strUserData){
    //设置视频窗口
    dRander.id = strPubId;
    dRander.style.width = 'auto';
    dRander.style.height = '100%';
    //创建视频容器
    document.body.appendChild(dRander);
});
// 收到远程人员视频流
meetKit.on("onRTCRemoteStream",function(oStream, strPeerId, strPubId){
    //给之前添加的视频容器绑定视频实时流
    meetKit.setRTCVideoRender(oStream, document.getElementById(strPubId));
});
// 远程人员离开
meetKit.on("onRTCCloseVideoRender",function(strPeerId, strPubId){
    document.getElementById(strPubId).remove();
});
```

> ## 说明
> 回调监听最好写在初始化实例之后，调用方法之前  
> 公网环境下网站必须为HTTPS协议，网站如果要上公网需要配置SSL证书（Chrome浏览器）

> ## 技术支持 
> 加QQ技术咨询群：580477436 (一群) 554714720 (二群)
> 欢迎加入<a href="https://bbs.anyrtc.io" target="_blank">anyRTC社区</a> 和我们一起探讨WebRTC技术以及解决集成问题。  
