---
layout: post
title: Linux下录屏并输出为GIF格式
category: howto
tags: [linux, capture, convert]
images: /assets/capture-screen-export-gif-ffmpeg/feature.png
descriptions: 使用FFMPEG工具的x11drag编码器录制屏幕。将录制结果输出为每帧图像。使用ImageMagick将图像合并为GIF格式动画。
---

## TL;DR

使用FFMPEG工具的x11drag编码器录制屏幕。将录制结果输出为每帧图像。使用ImageMagick将图像合并为GIF格式动画。

{% highlight bash %}
$ sleep 2; ffmpeg -y -video_size 683x384 -framerate 15 -f x11grab -i :0.0+0,0 output.mp4
$ ffmpeg -i output.mp4 -ss 10 -t 54 myclip.mp4; ffplay myclip.mp4
$ ffmpeg -i myclip.mp4 -vf fps=6 frames/ffout%03d.png
$ convert -delay 7 -loop 0 -dither None -colors 3 "frames/ffout*.png" -fuzz "10%" -layers OptimizeFrame "output.gif"
{% endhighlight %}

## 0. 准备

安装二进制包`ffmpeg`, `ImageMagick`。


## 1. 录制屏幕

> X11 display -> output.mp4

{% highlight shell %}
sleep 10 # 在录制操作开始前等待10秒
ffmpeg -y -video_size 1366x680 -framerate 25 -f x11grab -i :0.0+0,80 /tmp/output.mp4
## -y:                             如果输出文件output.mp4存在，则强行覆盖文件内容
## -video_size <width>x<height>:   设定录制区域大小
## -framerate:                     设定录制采样速率
## -f x11grab:                     使用内置的X11录屏编码器
## -i <a>.<b>+<left>,<top>:        <a>.<b>是对应屏幕的${DISPLAY}环境变量;<left>,<top>是相对左上角的录制区域偏移
{% endhighlight %}

## 1.5. 观看并剪辑视频文件 [可选步骤]

> output.mp4 -> finclip.mp4

{% highlight shell %}
ffplay /tmp/output.mp4 # 使用FFMPEG系列工具（一般随二进制包附带）播放录制结果
ffmpeg -y -i /tmp/output.mp4 -ss 3 -t 11 /tmp/finclip.mp4
## -i:         指定原始录制文件
## --ss <n>:   跳过原始录制文件的前<n>秒内容
## -t <d>:     制定输出视频的时长为<d>秒（裁剪原始录制文件的结束位置）
{% endhighlight %}

## 2. 将录制视频按帧转换为图片

> finclip.mp4 -> abc.PNG, xyz.PNG...

{% highlight shell %}
rm -rf /tmp/clipframes; mkdir /tmp/clipframes # 清理临时文件夹
ffmpeg -i /tmp/finclip.mp4 -vf scale=320:-1:flags=lanczos,fps=10 /tmp/clipframes/ffout%03d.png
## scale=<width>:-1:   将每帧画面的宽度缩放为<width>大小，高度自动计算等比缩放
## flags=<whatever>:   一些优化参数，MAGIC 根据需要改动
## fps=<x>:            将视频以<x>帧每秒播放并截图
## ffout%03d.png:      生成的每帧图像文件名将会是ffout001.png, ffout002.png...
{% endhighlight %}

## 3. 将每帧画面合并为GIF动画

> PNGs -> output.GIF

{% highlight shell %}
convert -delay 3 -loop 0 -dither None -colors 80 "/tmp/clipframes/ffout*.png" -fuzz "10%" -layers OptimizeFrame "/tmp/output.gif"
## -delay <f>:      将输入的每<f>帧图像设定为GIF关键帧，结果会使GIF动画播放更快，大小更小
## -loop 0:         使GIF动画循环无限次数
## -color <c>:      将GIF动画的调色板缩小为<c>个颜色
## -fuzz "10%":     融合GIF动画的相似颜色，根据需要改动
## -layers:         一些优化参数，MAGIC 根据需要改动
{% endhighlight %}

## 结果演示 -- 54秒MP4->约20秒GIF, 900KiB

![demo of result][convert-result-pic]


## FYI:    
[Capture screen in Linux][ffmpeg-wiki-capturing]

[How to trim first seconds with ffmpeg][so-trim]

[How to convert with reasonable quality][so-convert]

[Fuzz Factor - Matching Similar/Multiple Colors][imagemagick-color-fuzz]

[so-convert]: http://superuser.com/questions/556029/how-do-i-convert-a-video-to-gif-using-ffmpeg-with-reasonable-quality
[so-trim]: http://superuser.com/questions/556029/how-do-i-convert-a-video-to-gif-using-ffmpeg-with-reasonable-quality
[ffmpeg-wiki-capturing]: https://trac.ffmpeg.org/wiki/Capture/Desktop
[compiling ffmpeg]: https://trac.ffmpeg.org/wiki/CompilationGuide
[convert-result-pic]: /assets/capture-screen-export-gif-ffmpeg/result.gif
[origin post at v2ex]: http://v2ex.com/t/226531
[imagemagick-color-fuzz]: http://www.imagemagick.org/Usage/color_basics/#fuzz