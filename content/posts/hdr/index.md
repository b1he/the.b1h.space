---
title: "HDR video on bright SDR"
date: 2019-02-26T00:36:58+11:00
draft: false
author: "Bill He"
description: "HDR on a bright SDR phone!"
images: ["img/hdr/CRW_0006.jpg"]
tags: ["HDR", "High Dynamic Range"]
categories: ["HDR"]
featuredImage: "img/hdr/CRW_0006.jpg"
license: CC BY-NC 4.0
---

I have a Samsung Galaxy S6. It doesn't support watching HDR10 video but it does have a very bright AMOLED display. What if you could convert the HDR video to make it work with the S6 and still get the full HDR experience? You can! There are two different methods that I have been experimenting with the conversion: FFmpeg and Adobe After Effects. The idea is that both programs will read the HDR video and we instruct it to convert or 'tone map' the image by adjusting the light level specifically tailored to our display. It's incredible! You can get awesome results just like a phone with 'real' HDR!

If you have another phone and know its maximum brightness and is bright enough for HDR video, you can also follow this 'guide'.

This trick is possible by maxing out the brightness of the display. But by default, the display will never be 100% bright unless the light sensor can detect that the sun is directly shining onto the display. In that case, 'outdoor mode' is activated which also activates a contrast enhancing feature which we don't want. There are two ways that I know of that we can force 100% brightness without the sunlight contrast adjustment. Both methods unfortunately require messing deep in your phone.

* Rooting the phone and using a paid app called [High Brightness Mode](https://play.google.com/store/apps/details?id=flar2.hbmwidget) to force 100% brightness.
* Installing a custom rom like LineageOS. In this case, 100% brightness can easily be achieved by moving the non-automatic brightness slider all the way to the right.

Ideally, I would be using 10-bit HEVC video to preserve as much detail during the conversion process but unfortunately, ... **no** 10-bit hardware decoding for the S6 😞. That's ok, we will work around it by 'dithering' which will mean huge file sizes. If your phone does support 10-bit video, then maybe you wouldn't need dithering and to use the noise plugin explained below.

## Finding and getting HDR video on YouTube
When searching for HDR video on YouTube, apart from entering 'hdr' in the search box which could result in non-HDR videos, they do provide an HDR filter under features.
![](img/hdr/YouTube_.png)

Using the CLI tool [youtube-dl](https://youtube-dl.org/), we can specify the exact format we want to download (1440p60 + Opus)

`youtube-dl https://www.youtube.com/watch?v=tO01J-M3g0U -f 336+251`

Or separately for the After Methods method by replacing the plus with a comma and downloading the AAC audio instead..

`youtube-dl https://www.youtube.com/watch?v=tO01J-M3g0U -f 336,140`

You can also view other formats and the corresponding numbers that the video provides

`youtube-dl https://www.youtube.com/watch?v=tO01J-M3g0U -F`

## Converting via FFmpeg

FFmpeg is a command line tool but also powerful. A single command will achieve our result while I tweaked the command from [this website](https://stevens.li/guides/video/converting-hdr-to-sdr-with-ffmpeg/).


`ffmpeg.exe -i INPUT -vf "zscale=w=2560:h=-1: t=linear:npl=784, format=gbrpf32le, zscale=p=bt709, tonemap=tonemap=clip:desat=0, zscale=t=bt709:m=bt709:r=tv: d=error_diffusion, format=yuv420p" -c:v libx265 -crf 15 -x265-params aq-mode=3 -tune grain OUTPUT.mp4`

...where npl is the display luminance in nits. We will have to resort to the internet since measuring it ourselves requires some expensive tools. Looking at [this](http://www.displaymate.com/Galaxy_S6_ShootOut_1.htm), I'll take in 784 nits.

The result? It looked pretty good so far, until I watched the very beginning of [Lifeline episode 1](https://www.youtube.com/watch?v=Ru4zkxNuJ_I) (It has a much darker grade than usual).

I saw some banding or blocky artefacts in the field. It was noticeable enough that I had to investigate. To find what was causing this issue, I used madVR on MPC-HC to compare the tone mapped images. I changed FFmpeg to output a single frame in a png format while I took a screenshot of MPC-HC and placed the three frames into Photoshop to compare. The images had the exposure increased by 10 stops to highlight the problematic dark scenes. Even without the video compression involved, FFmpeg is still producing inferior results.
My conclusion is that one step of the tone mapping process is done in 16-bit which I could replicate in After Effects by setting it's depth to 16-bits.


### Comparison...


![](img/hdr/ffmpeg.jpg?classes=caption "FFmpeg. 😐")
![](img/hdr/AE.jpg?classes=caption "After Effects. Added lots of noise so it can survive HEVC compression.")
![](img/hdr/madVR.jpg?classes=caption "madVR. Video renderer on PC only so can't be played on phone.")
![](img/hdr/Normal.jpg?classes=caption "What it would actually look like normally!")

## A different method, the After Effects way
I'm using After Effects instead of Premiere Pro because it lets us convert videos with different colour profiles. While it is possible to directly import VP9 video into After Effects works if [this plugin](https://www.fnordware.com/WebM/) is installed, but it's super slow and produces some black line artefacts. What I do is use FFmpeg to convert the video to ProRes first then import that. I am outputting ProRes in YUV 4:4:4 because the colours imported are more accurate. This does unfortunately result in massive file sizes. The World in HDR for example is 13.9GB when converted to ProRes.

`ffmpeg -i INPUT.webm -c:v prores -pix_fmt yuv444p10le OUTPUT.mov`

In the Project Settings > Colour Settings, we set the depth to 32 bits per channel (float). Then both the video and audio is imported separately into After Effects (separately so that After Effects doesn't have to conform the audio and read through the entire video once, ProRes can be huge!). We then make the composition with the video and audio in it, while the composition is set as 60fps so the dithering can apply to the entire refresh rate of the display, not the video frame rate.

We need to apply these plugins: Noise, Color Profile Converter and Exposure.
* The noise plugin is to 'dither' the video and remove any noticeable banding. 2% is a good balance while we leave 'Use Color Noise' checked because it helps hide the banding better. If you're wondering why I added the noise plugin before everything else, it's because I noticed that the noise is unbalanced as the shadows gets too much noise and there isn't enough noise in the mid-details.
* The colour profile converter then comes next. The input profile to set is Rec.2100. This is to convert the HDR to SDR in preparation for the last step...
* Exposure adjustment. This is for correcting the HDR video for the brightness of the display. Without modifying the exposure, anything higher than 100 nits will get clipped. To find out what value you would need to correct the exposure to your display. I have worked out a formula shown below. In my case, I would need to set the exposure value to -2.9709, so that anything above 784 nits will get clipped instead.

![](img/hdr/AEeffects.PNG)

To calculate the exposure value needed, here is the mathematical formula that took a while for me to figure out:

<script src='https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML' async></script>

$$exposure=-{log(nits)-log(100) \over log(2)}$$

<div style="text-align:center">
(High school maths is useful for once!)
 </div>

  <br>

Or if you don't have one of those 'ultimate' calculator or Google around, here's my custom made calculator below:

<div style="text-align:center">
Enter brightness in nits:  <input id="value1" type="text" onkeydown = "if (event.keyCode == 13) output()" />
<input type="submit" value="Calculate" onclick="output();">
<p id="result"> </p>

<script type="text/javascript" language="javascript" charset="utf-8">

function output(){
    var value1 = document.getElementById('value1').value;
    document.getElementById('result').innerHTML = "Exposure: " + -((Math.log(parseInt(value1))-Math.log(100))/Math.log(2));
}
</script>
</div>

We're now ready to send it to Media Encoder! I use the [Voukoder](https://www.voukoder.org/) plugin to encode HEVC because it is much better and a lot more customisable. The settings to change: Video Encoder: libx265 H.265 HEVC, Tuning: Grain (so the dithering doesn't get lost), Constant Rate Factor of 25. In the advanced tab, AQ Mode is enabled with auto-variance and bias to dark scene with a strength of 3. This tells the encoder to prioritise for the shadows as more of the details of the video is squeezed into the darker ends of the video signal. And that's how you do the After Effects way.

## HLG
A small amount of HDR videos on YouTube are created in Hybrid Log-Gamma instead of Perceptual Quantizer (PQ). I couldn't figure out a way for After Effects to convert these types of videos properly. The FFmpeg way will do just fine, since it can automatically detect HLG video. You can confirm if a video is in PQ or HLG by looking at the FFmpeg output while it's encoding. arib-std-b67 means HLG while smpte2084 means PQ.

![](img/hdr/hlg.png)

## How does this compare to a 'real' HDR phone?
Since I didn't have any 'real' HDR phones, I took a trip to JB Hi-Fi where they had demo phones that I could use to do my comparisons. The way the phones display HDR video are all over the place, just like HDR TVs I guess...

* Samsung Galaxy Note 9: Probably the best display I tested. Both the Note 9 and S6 had lots of different display profiles I could choose. The S6's standard mode looked very similar to the Note 9's adaptive mode. There were very slight colour differences and the Note 9's highlights did look a bit brighter.
![](img/hdr/CRW_0040.jpg)

* Oppo Find X: With it's cool bezel-less display, it wasn't as saturated as the Note 9. The S6 looked very similar while using the natural display profile. The Find X did have a video enhancer in the settings, but made the video look awful.
![](img/hdr/CRW_0048.jpg)

* Google Pixel 3 XL: Looked similar to the S6's standard mode, the highlights on the Pixel 3 XL were a lot less bright than the S6. I don't have a photo because I compared it on another day without bringing my camera.
* iPhone X: This I had to check with a friend's one because demo iPhones don't come with YouTube. It looked like the S6's standard mode. The highlights on the iPhone looked slightly less bright compared to the S6.

## HDRVR?

![](img/hdr/CRW_0076.jpg)

What if I could get the big screen HDR experience by using a virtual reality headset? It sounded like an awesome idea in theory and so I grabbed myself a Gear VR on eBay for just $1! Gear VR mode is great in that the headset has its own sensors and the phone's display can run in low persistence mode. The low persistence mode though means I won't be able to modify the display to run in full brightness, so I had to use Google Cardboard apps instead.

The result? I was disappointed. Running the display with full brightness made all the VR quirks look a lot worse: the screen-door effect, the chromatic aberration, the blurry edges and especially the halos on any bright object with a black background. I'm sure VR will get better over time ,but these limitations are just too noticeable for my liking. Though I did like the Gear VR mode provided by the S6 and headset because of low persistence and there's no drifting!

## Heard of YouTube Vanced?
A modded YouTube client that lets you force HDR mode on. It doesn't seem to produce correct looking results for this phone at least when running full brightness, so you won't be able to achieve the full effect like the methods above.

## What's next?
I will try to grade in HDR with the help of my phone! It won't be great, I don't have much experience in even grading/correcting in SDR. The footage I will use is not the best but should be good enough for an experiment!
