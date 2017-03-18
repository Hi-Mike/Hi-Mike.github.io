---
layout: post
title: "Android WebRTC初探"
date: 2017-03-18
categories: WebRTC
---

`知道` [WebRTC](https://webrtc.org/) 应该蛮久了，之前也断断续续的看过些资料。然而的然而，连基本的demo都没有实现过。这次呢，有朋友有点小需求，所以没事也再次尝试了下。现在的资料貌似多了不少，照着资料实现了基本的点对点视频通话，(⊙ˍ⊙！)，看，我居然能看到两个自己。先看效果是个好习惯。。。

![androidWebRTC.png](http://7xnzl2.com1.z0.glb.clouddn.com/androidWebRTC.png?imageView2/2/w/320)

## 情报

* 官方流程图

    [https://webrtc.org/native-code/native-apis/](https://webrtc.org/native-code/native-apis/)

* 代码已上传，先运行下，效果更加 [https://github.com/Hi-Mike/AndroidWebRTC](https://github.com/Hi-Mike/AndroidWebRTC)

* stun服务器从 [xirsys.com](http://xirsys.com)，自行获取配置

* 信令交换 [PubNub](https://www.pubnub.com)，自行注册配置

## 项目配置

1. 项目依赖

    要自己编译WebRTC的库的话，还是要费些事的。所以这里直接用别人编好的。这里用的还是老版本的库，新版本的一些函数已经改变，后面有空会去看下。

    ```gradle
    compile 'io.pristine:libjingle:9694@aar'
    ```

2. 权限

    先一股脑添加上去吧

    ```xml
    <uses-feature android:name="android.hardware.camera" />
    <uses-feature android:name="android.hardware.camera.autofocus" />
    <uses-feature android:glEsVersion="0x00020000" android:required="true" />

    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
    ```

## 实现步骤（后面的个别步骤可调整）

0. 初始化并创建PeerConnectionFactory

    ```java
    boolean initOK = PeerConnectionFactory.initializeAndroidGlobals(
                this,//上下文，可自定义监听
                true,//是否初始化音频，布尔值
                true,//是否初始化视频，布尔值
                true,//是否支持硬件加速，布尔值
                null);//是否支持硬件渲染，布尔值

    factory = new PeerConnectionFactory();
    ```

1. 获取VideoSource与AudioSource

    ```java
    String frontCameraName = VideoCapturerAndroid.getNameOfFrontFacingDevice();
    VideoCapturer videoCapturer = VideoCapturerAndroid.create(frontCameraName);
    
    VideoSource videoSource = factory.createVideoSource(videoCapturer, videoConstraints);
    AudioSource audioSource = factory.createAudioSource(new MediaConstraints());
    ```

2. 获取VideoTrack与AudioTrack

    ```java
    VideoTrack videoTrack = factory.createVideoTrack("ARDAMSv0", videoSource);
    AudioTrack audioTrack = factory.createAudioTrack("ARDAMSa0", audioSource);
    ```

3. 创建renderer并交给画布，将本地render给track

    ```java
    VideoRendererGui.setView(mGLSurfaceView, null);
    remoteRender = VideoRendererGui.create(
                REMOTE_X, REMOTE_Y,
                REMOTE_WIDTH, REMOTE_HEIGHT, scalingType, false);
    localRender = VideoRendererGui.create(
                LOCAL_X_CONNECTING, LOCAL_Y_CONNECTING,
                LOCAL_WIDTH_CONNECTING, LOCAL_HEIGHT_CONNECTING, scalingType, true);
    // just add local,the remoteRenderer will be added when remoteRender is ready in onAddStream
    videoTrack.addRenderer(new VideoRenderer(localRender));
    ```

4. 获取本地MediaStream，可以看到自己啦

    ```java
    MediaStream localMS = factory.createLocalMediaStream("ARDAMS");
    localMS.addTrack(videoTrack);
    localMS.addTrack(audioTrack);
    ```

5. 创建PeerConnection

    这里的iceServers是stun服务器列表，我是通过 [xirsys.com](http://xirsys.com) 配置获取的

    ```java
    pc = factory.createPeerConnection(
                iceServers,//ICE servers
                pcConstraints,//MediaConstraints
                this);//observer : SdpObserver, PeerConnection.Observer
    ```

6. 将本地MediaStream加入PeerConnection

    ```java
    pc.addStream(localMS);
    ```

7. 信令交换（SessionDescription与IceCandidate）
    
    在项目中我使用 [PubNub](https://www.pubnub.com) 做的信令交换，只要你能使两个client通信，你可以使用任何方式进行信令交换。

    1) SessionDescription交换

        这里借用下 [这里](http://blog.csdn.net/youmingyu/article/details/53192714) 的说法进行适当修改。A为发起端，B为响应端，A call B。

        * A向B发出一个“init”请求（我觉得这步没有也行）。
        * B收到后“init”请求后，调用pc.createOffer()方法创建一个offer SessionDescription（包含媒体信息，如分辨率、编解码能力等）。
        * offer SessionDescription创建成功后会调用SdpObserver监听中的onCreateSuccess()响应函数，在这里B会通过pc.setLocalDescription将offer SessionDescription赋给自己的pc对象，同时将offer SessionDescription发送给A 。
        * A收到B的offer SessionDescription后，利用pc.setRemoteDescription()方法将B的SessionDescription赋给A的pc对象。并且调用pc.createAnswer()获取一个answer SessionDescription。
        * answer SessionDescription创建成功后，同样A在onCreateSuccess()监听响应函数中调用pc.setLocalDescription将answer SessionDescription赋给自己的PC对象，同时将answer SessionDescription发送给B 。
        * B收到A的answer SessionDescription后，利用pc.setRemoteDescription()方法将A的answer SessionDescription赋给B的pc对象。
        * SessionDescription交换完毕

    2) IceCandidate交换

        IceCandidate的交换和SessionDescription交换是差不多的，区别为IceCandidate会在适当的时候自动在onIceCandidate中回调，直接传给对方就好。这里我看其他文章貌似会有一个说法问题，内网不需要IceCandidate交换？我的理解是，IceCandidate交换是建立P2P通道的，就算内网也需要交换以知道对方的地址才好建立通道。

8. Is anyone there?

    其实在pc.setRemoteDescription后，远程的mediaStream就会在onAddStream回调回来，这里将之与画布关联

    ```
    HAVE_REMOTE_OFFER
    mediaStream.videoTracks.get(0).addRenderer(new VideoRenderer(remoteRender));
    Logger.d("onAddStream：" + mediaStream.videoTracks.size());
    VideoRendererGui.update(remoteRender,
                REMOTE_X, REMOTE_Y,
                REMOTE_WIDTH, REMOTE_HEIGHT, scalingType, false);
    VideoRendererGui.update(localRender,
                LOCAL_X_CONNECTED, LOCAL_Y_CONNECTED,
                LOCAL_WIDTH_CONNECTED, LOCAL_HEIGHT_CONNECTED, scalingType, true);
    ```

9. 释放

    通信结束，该释放的还是要释放，毕竟调用的都是native api

