---
layout: post
title: "Ubuntu下Nginx+nginx-rtmp-module搭建"
date: 2017-12-27
categories: nginx rtmp
---

## 下载
[Nginx](http://nginx.org/en/download.html)下载

[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)下载

## 编译安装
nginx-rtmp-module的github主页上有介绍，如果需要其他组件，其他支持，自行添加

```bash
sudo ./configure --add-module=/path/to/nginx-rtmp-module --with-http_ssl_module
sudo make
sudo make install
```

## 添加配置
默认的安装Nginx在 **/usr/local** 下，更改 **/usr/local/nginx/conf** 下的nginx.conf文件。[nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)下也有较好的例子，如：

```Nginx
rtmp {
    server {
        listen 1935;

        application mytv {
            live on;
        }
    }
}
```

然后使配置生效

```bash
sudo ./nginx -c nginx.conf
```

## 测试
访问本地地址测试NGINX是否安装成功并启动，访问 **localhost** 即可。

利用ffmpeg生成rtmp流进行测速：

```bash
sudo ./ffmpeg -re -i /path/to/File/test.mp4 -vcodec libx264 -acodec aac -f flv rtmp://127.0.0.1:1935/mytv/test
```

播放rtmp流，如在vlc中播放地址： **rtmp://127.0.0.1:1935/mytv/test** (1935默认端口号可省略)