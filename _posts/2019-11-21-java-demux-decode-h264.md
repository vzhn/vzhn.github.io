---
layout: post
title: Demuxing and decoding h264 video with java
---

How to decode h264 video stream with java
<!--more-->

Initialize AVFormatContext
```java
private AVFormatContext openInput(String file) throws IOException {
    avfmtCtx = new AVFormatContext(null);
    BytePointer filePointer = new BytePointer(file);
    int r = avformat.avformat_open_input(avfmtCtx, filePointer, null, null);
    filePointer.deallocate();

    if (r < 0) {
        avfmtCtx.close();
        throw new IOException("avformat_open_input error: " + r);
     }
    return avfmtCtx;
}
```

Find video stream
```java
private void findVideoStream() throws IOException {
    int r = avformat_find_stream_info(avfmtCtx, (PointerPointer) null);
    if (r < 0) {
        avformat_close_input(avfmtCtx);
        avfmtCtx.close();
        throw new IOException("error: " + r);
    }

    PointerPointer<AVCodec> decoderRet = new PointerPointer<>(1);
    int videoStreamNumber = av_find_best_stream(avfmtCtx, AVMEDIA_TYPE_VIDEO, -1, -1, decoderRet, 0);
    if (videoStreamNumber < 0) {
        throw new IOException("failed to find video stream");
    }

    if (decoderRet.get(AVCodec.class).id() != AV_CODEC_ID_H264) {
        throw new IOException("failed to find h264 stream");
    }
    decoderRet.deallocate();
    st =  avfmtCtx.streams(videoStreamNumber);
}
```

Setup h264 decoder
```java
private void initDecoder() {
    codec = avcodec_find_decoder(AV_CODEC_ID_H264);
    codecoContext = avcodec_alloc_context3(codec);

    if((codec.capabilities() & avcodec.AV_CODEC_CAP_TRUNCATED) != 0) {
        codecoContext.flags(codecoContext.flags() | avcodec.AV_CODEC_CAP_TRUNCATED);
    }

    avcodec_parameters_to_context(codecoContext, st.codecpar());
    if(avcodec_open2(codecoContext, codec, (PointerPointer) null) < 0) {
        throw new RuntimeException("Error: could not open codec.\n");
    }
}
```

Allocate yuv420p frame
```java
private void initYuv420Frame() {
    yuv420Frame = av_frame_alloc();
    if (yuv420Frame == null) {
        System.err.println("Could not allocate video frame\n");
        System.exit(1);
    }
}
```

Allocate RGB frame
```java
private void initRgbFrame() {
    rgbFrame = av_frame_alloc();
    rgbFrame.format(AV_PIX_FMT_RGB24);
    rgbFrame.width(codecoContext.width());
    rgbFrame.height(codecoContext.height());
    int ret = av_image_alloc(rgbFrame.data(), rgbFrame.linesize(), rgbFrame.width(), rgbFrame.height(), rgbFrame.format(), 32);
    if (ret < 0) {
        System.err.println("could not allocate buffer!");
    }

    img = new BufferedImage(rgbFrame.width(), rgbFrame.height(), BufferedImage.TYPE_3BYTE_BGR);
}
```

Initialize sws context
```java
    private void getSwsContext() {
        sws_ctx = swscale.sws_getContext(
                codecoContext.width(), codecoContext.height(), codecoContext.pix_fmt(),
                rgbFrame.width(), rgbFrame.height(), rgbFrame.format(),
                0, null, null, (DoublePointer) null);
    }
```

And finally we are ready to decode video
```java
private void start(String[] argv) throws IOException {
    av_log_set_level(AV_LOG_VERBOSE);

    openInput(argv[0]);
    findVideoStream();
    initDecoder();
    initRgbFrame();
    initYuv420Frame();
    getSwsContext();

    avpacket = new avcodec.AVPacket();
    while ((av_read_frame(avfmtCtx, avpacket)) >= 0) {
        int ret = avcodec.avcodec_send_packet(codecoContext, avpacket);
        if (ret < 0) {
            System.err.println("Error sending a packet for decoding\n");
            System.exit(1);
        }
        while (ret >= 0) {
            ret = avcodec.avcodec_receive_frame(codecoContext, yuv420Frame);
            if (ret == AVERROR_EAGAIN() || ret == AVERROR_EOF()) {
                continue;
            } else
            if (ret < 0) {
                System.err.println("error during decoding");
            }

            swscale.sws_scale(sws_ctx, yuv420Frame.data(), yuv420Frame.linesize(), 0, yuv420Frame.height(), rgbFrame.data(), rgbFrame.linesize());

            DataBufferByte buffer = (DataBufferByte) img.getRaster().getDataBuffer();
            rgbFrame.data(0).get(buffer.getData());
        }
    }

    free();
}
```

