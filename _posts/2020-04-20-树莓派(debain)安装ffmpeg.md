---
title: 树莓派(debain)安装ffmepg
layout: post
categories: linux软件安装 问题记录
tags: ffmpeg 软件安装
excerpt: 
---

### 写在前面
家里有个树莓派， 想着在上面安装一个ffmpeg达到下载视频的目的

### 安装步骤
1. 配置阿里云镜像

2.更新源并安装git
```shell
    sudo apt-get update
    sudo apt-get install git
```
3.x264配置脚本config_x264_rpi.sh，放进x264目录
```shelll
    #!/bin/sh
    ./configure \
    --disable-shared --enable-static \
    --enable-strip \
    --disable-cli
```
4.下载x264源码并编译安装(https://www.videolan.org/developers/x264.html)
```shell
    git clone https://code.videolan.org/videolan/x264.git
    cd x264
    mv ../config_x264_rpi.sh ./
    chmod +x config_x264_rpi.sh
    ./config_x264_rpi.sh
    make -j4
    sudo make install
```
5.ffmpeg配置脚本config_ffmpeg_rpi.sh，放进ffmpeg目录
```shell
    #!/bin/sh
    PREFIX=/usr/local
    ./configure \
    --enable-gpl    --enable-version3 --enable-nonfree \
    --enable-static --disable-shared \
    \
    --prefix=$PREFIX \
    \
    --disable-opencl \
    --disable-thumb \
    --disable-pic \
    --disable-stripping \
    \
    --enable-small \
    \
    --enable-ffmpeg \
    --enable-ffplay \
    --enable-ffprobe \
    \
    --disable-doc \
    --disable-htmlpages \
    --disable-podpages \
    --disable-txtpages \
    --disable-manpages \
    \
    --enable-libx264 \
    --enable-encoder=libx264 \
    --enable-decoder=h264 \
    --enable-encoder=aac \
    --enable-decoder=aac \
    --enable-encoder=ac3 \
    --enable-decoder=ac3 \
    --enable-encoder=rawvideo \
    --enable-decoder=rawvideo \
    --enable-encoder=mjpeg \
    --enable-decoder=mjpeg \
    \
    --enable-demuxer=concat \
    --enable-muxer=flv \
    --enable-demuxer=flv \
    --enable-demuxer=live_flv \
    --enable-muxer=hls \
    --enable-muxer=segment \
    --enable-muxer=stream_segment \
    --enable-muxer=mov \
    --enable-demuxer=mov \
    --enable-muxer=mp4 \
    --enable-muxer=mpegts \
    --enable-demuxer=mpegts \
    --enable-demuxer=mpegvideo \
    --enable-muxer=matroska \
    --enable-demuxer=matroska \
    --enable-muxer=wav \
    --enable-demuxer=wav \
    --enable-muxer=pcm* \
    --enable-demuxer=pcm* \
    --enable-muxer=rawvideo \
    --enable-demuxer=rawvideo \
    --enable-muxer=rtsp \
    --enable-demuxer=rtsp \
    --enable-muxer=rtsp \
    --enable-demuxer=sdp \
    --enable-muxer=fifo \
    --enable-muxer=tee \
    \
    --enable-parser=h264 \
    --enable-parser=aac \
    \
    --enable-protocol=file \
    --enable-protocol=tcp \
    --enable-protocol=rtmp \
    --enable-protocol=cache \
    --enable-protocol=pipe \
    \
    --enable-filter=aresample \
    --enable-filter=allyuv \
    --enable-filter=scale \
    --enable-libfreetype \
    \
    --enable-indev=v4l2 \
    --enable-indev=alsa \
    \
    --enable-omx \
    --enable-omx-rpi \
    --enable-encoder=h264_omx \
    \
    --enable-mmal \
    --enable-hwaccel=h264_mmal \
    --enable-decoder=h264_mmal \
    \
    --enable-openssl \
    --enable-protocols \
    --enable-protocol=https
```
6.在FFmpeg官网获取源码 http://ffmpeg.org/download.html 或 http://ffmpeg.org/releases/, 下载最新的版本
```shell
    wget http://ffmpeg.org/releases/ffmpeg-4.2.2.tar.bz2
    tar jxvf ffmpeg-4.2.2.tar.bz2
    cd ffmpeg-3.3.2
    mv ../config_ffmpeg_rpi.sh ./
    chmod +x config_ffmpeg_rpi.sh
    ./config_ffmpeg_rpi.sh
    make -j4 && make install 
```

### 注意事项
1. 一定不能添加disable-everything, 不然会有很多东西没有安装
2. 用make -j4 比make的速度要快许多。如果cpu是多核的话建议用这个
3. --enable-openssl --enable-protocols --enable-protocol=https 这几个一定要添加不然无法使用https协议

