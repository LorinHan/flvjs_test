### 实现摄像头的直播功能其实有许多方案，像是安装vlc插件、rtsp转rtmp然后使用videojs通过flash播放rtmp，以及hls .m3u8等方式
### 然而现今的浏览器对于vlc插件几乎都不再支持了，flash在2020年也将被chrome停止支持，而.m3u8的方案用来做直播的话似乎延迟很高
### 经过一番查找，最终决定使用B站（bilibili）开源的flvjs作为解决方案，其原理是后端用ffmpeg将rtsp视频流转换为flv，然后通过websocket传输flv视频流，然后前端通过websocket获取到视频流后，使用flvjs对视频流再一次处理并进行播放，这是一套无插件无flash免费的视频直播解决方案。
---
# 1.使用vlc等工具测试，确保rtsp流可连接
# 2.后端环境：node express
### 2.1 安装第三方依赖
`npm install express express-ws fluent-ffmpeg websocket-stream -S -D`
### 2.2 编写代码 index.js
> 其中setFfmpegPath这里是指明了ffmpeg的安装路径，如果没有安装，请看第4点

```
var express =  require("express");
var expressWebSocket = require("express-ws");
var ffmpeg = require("fluent-ffmpeg");
ffmpeg.setFfmpegPath("D:/ffmpeg-20191031-7c872df-win64-static/ffmpeg-20191031-7c872df-win64-static/bin/ffmpeg");
var webSocketStream = require("websocket-stream/stream");
var WebSocket = require("websocket-stream");
var http = require("http");
function localServer() {
    let app = express();
    app.use(express.static(__dirname));
    expressWebSocket(app, null, {
        perMessageDeflate: true
    });
    app.ws("/rtsp/:id/", rtspRequestHandle)
    app.listen(8888);
    console.log("express listened")
}
function rtspRequestHandle(ws, req) {
    console.log("rtsp request handle");
    const stream = webSocketStream(ws, {
        binary: true,
        browserBufferTimeout: 1000000
    }, {
        browserBufferTimeout: 1000000
    });
    let url = req.query.url;
    console.log("rtsp url:", url);
    console.log("rtsp params:", req.params);
    try {
        ffmpeg(url)
            .addInputOption("-rtsp_transport", "tcp", "-buffer_size", "102400")  // 这里可以添加一些 RTSP 优化的参数
            .on("start", function () {
                console.log(url, "Stream started.");
            })
            .on("codecData", function () {
                console.log(url, "Stream codecData.")
             // 摄像机在线处理
            })
            .on("error", function (err) {
                console.log(url, "An error occured: ", err.message);
            })
            .on("end", function () {
                console.log(url, "Stream end!");
             // 摄像机断线的处理
            })
            .outputFormat("flv").videoCodec("copy").noAudio().pipe(stream);
    } catch (error) {
        console.log(error);
    }
}
localServer();
```
### 2.3 启动后端，node index.js
---
# 3.前端环境，采用vue
### 3.1 vue的搭建就不赘述了，构建好一个vue项目之后，npm install flv.js -S -D
### 3.2 编写代码
> video标签中的muted属性，是因为在视频流加载好之后，autoplay属性无法自动播放，加上这个属性后就可以实现了

```
<template>
    <div>
        <video class="demo-video" ref="player" muted autoplay></video>
    </div>
</template>
<script>
import flvjs from "flv.js";
export default {
    data () {
        return {
          id: "1",
          rtsp: "rtsp://admin:12345@192.168.0.101:554/h264/ch1/main/av_stream",
            player: null
        }
    },
    mounted () {
        if (flvjs.isSupported()) {
            let video = this.$refs.player;
            if (video) {
                this.player = flvjs.createPlayer({
                    type: "flv",
                    isLive: true,
                    url: `ws://localhost:8888/rtsp/${this.id}/?url=${this.rtsp}`
                });
                this.player.attachMediaElement(video);
                try {
                    this.player.load();
                    this.player.play();
                } catch (error) {
                    console.log(error);
                };
            }
            // if (video) {
            //     this.player = flvjs.createPlayer({
            //         type: "flv",
            //         url: `/static/test.flv`
            //     });
            //     this.player.attachMediaElement(video);
            //     try {
            //         this.player.load();
            //         this.player.play();
            //     } catch (error) {
            //         console.log(error);
            //     };
            // }
        }
    },
    beforeDestroy () {
        this.player.destory();
    }
}
</script>
<style>
    .demo-video {
        max-width: 880px; 
        max-height: 660px;
    }
</style>
```
---
# 4.到此为止，如果你有ffmpeg的环境，应该可以在前端看到画面了，如果没有ffmpeg的话请进行安装
### 4.1 访问官网[ffmpeg.zeranoe.com/builds/](https://ffmpeg.zeranoe.com/builds/)，根据操作系统自行选择安装
### 4.2 下载好后进行解压，在后端设置ffmpeg路径的代码中指向解压路径下的bin目录下的ffmpeg
```
var ffmpeg = require("fluent-ffmpeg");
ffmpeg.setFfmpegPath("D:/ffmpeg-20191031-7c872df-win64-static/ffmpeg-20191031-7c872df-win64-static/bin/ffmpeg");
```