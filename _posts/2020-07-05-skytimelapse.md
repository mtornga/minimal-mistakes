---
layout: posts
title: "From timelapse to Twitter with Ansible"
header:
  teaser: "/assets/images/avatar.png"
---

Start with the final product, the tweet:

<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://twitter.com/hashtag/stlwx?src=hash&amp;ref_src=twsrc%5Etfw">#stlwx</a> <a href="https://twitter.com/hashtag/timelapse?src=hash&amp;ref_src=twsrc%5Etfw">#timelapse</a> <a href="https://t.co/Yt0PJ8TaIx">pic.twitter.com/Yt0PJ8TaIx</a></p>&mdash; SouthamptonTimelapse (@StlTimelapse) <a href="https://twitter.com/StlTimelapse/status/1279837308360024064?ref_src=twsrc%5Etfw">July 5, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

How did we get here? First, I impluse-bought the new [Raspberry Pi HQ camera sensor](https://www.raspberrypi.org/products/raspberry-pi-high-quality-camera/) without much of a plan.

Here's what that looks like when set up:

![Camera Setup](/assets/images/camerasetup.jpg "The pi and wide angle lens")

The inspiring and worthwhile [Official Raspberry Pi Camera Guide](https://www.raspberrypi.org/products/camera-guide/) provides a simple one-liner to use the raspistill package to capture a timelapse. Raspistill is included in the basic Raspian OS install, which is nice. 

```raspistill -w 1920 -h 1200 -t 7200000 -tl 6000 -o /var/www/html/frame%04d.jpg```

This command will create a 1920x1200 pixel image every 6,000 milliseconds for 7,200,000 milliseconds and save them in the filepath /var/www/html/. They will be named frame0001, frame0002, frame0003, and so on. 

You are also provided a command to compile the jpgs into a webm video.

```sudo avconv -i /var/www/html/frame%04d.jpg -crf 4 -b:v 10M /var/www/html/video.webm```

This one renders the video at a 25fps frame rate, a high bitrate, and a low Constant Rate Factor to produce a good-quality video. I'll admit that tinkering with video settings is something I have low patience for.

Rendering is highly resource intensive and the Raspberry Pi takes about 4 hours to render a 30-second video. Anyway...

Success! I have a video that I can share with Stephanie and others. But its a little slow and manual. Let's see what we can do to improve the process.

## Speed

The Pi guide suggests moving the images to another, more powerful computer before rendering the video. To do this, I set up an [Apache Tomcat](http://tomcat.apache.org/) HTTP web server that my laptop can reach over the home wifi network. The frames of timelapse are saved to the folder the the web server displays to the web. 

Tomcat does not by default display the directory listing, so the listing parameter has to be changed to true:

```<init-param>
    <param-name>listings</param-name>
    <param-value>true</param-value>
</init-param>```

Here is what the web server looks like from a web browser:

![Webserver](/assets/images/tomcat.PNG "A screengrab of the listing of frames")

A simple ```wget``` or ```curl``` will pull the images down from the server to laptop.

Last, I used ffmpeg in place of avconv to render the video. I run [Ubuntu](https://ubuntu.com/) on the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) (or WSL) on my Windows 10 OS. This allows me to sometimes avoid using Virtualbox.

Here is what WSL looks like:

![WSL](/assets/images/WSL.PNG "A screengrab of Linux on Windows")

Its a lot smoother way to use Linux on Windows. 

Anyway, avconv wasn't available on Ubuntu for some reason so I did some digging and it looks like ffmpeg does the same thing and has the same syntax. Over the course of the research I also discovered that there was a developer fight over at ffmpeg leading to multiple versions: [The real ffmpeg vs the fake one](https://stackoverflow.com/questions/9477115/what-are-the-differences-and-similarities-between-ffmpeg-libav-and-avconv)

