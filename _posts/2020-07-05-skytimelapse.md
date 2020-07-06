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

In the current form, it takes a couple steps to get a video produced.
1. Kick off the ```raspistill```
2. Start the ```wget``` to download the stills
3. Begin the ```ffmpeg``` to render the webm
4. Access the webm to enjoy the video. 

That's a lot of pain for a pleasant cloud video! Let's automate

## Automation

In my work as a consultant I have begin using [Ansible](https://www.ansible.com/), an open source software, for automating IT tasks. We use Ansible to deploy, upgrade, and backup Alteryx, Tableau, and MongoDB. Maybe there are other, simpler ways to do this automation but I know Ansible pretty well at this point so that's what I went with. 

Ansible is a simple ```apt-get ansible``` away on Ubuntu and then I put a few lines in ansible.cfg and into the default ```hosts``` file.

Here is the playbook ```timelapse.yml```. One thing I especially enjoy about Ansible is how self-documenting it is. By having the user give each play a name, it forces us into better documentation habits.

```
---
- hosts: pi
  name: Collect timelapse on the Pi
  become: yes
  gather_facts: no
  tasks:

  - name: Delete existing files...
    shell: rm -rf /var/www/html/*


#The bracketed duration and interval are jinja2 variables that are passed in from the command line 
  - name: Begin timelapse capture...
    shell: raspistill -w 1920 -h 1200 -t {{ duration }} -tl {{ interval }} -o /var/www/html/frame%04d.jpg
    async: 7300
    poll: 5
  
  - name: TAR the files...
    archive: 
      path: /var/www/html/
      dest: /var/www/html/images.tgz

- hosts: localhost
  name: Process video and tweet from the laptop
  connection: local
  gather_facts: yes
  become: yes
  tasks:
    - name: Download images archive from the Pi...
      get_url:
        url: http://192.168.1.213/images.tgz
        dest: /home/mark/timelapse/files/images.tgz

    - name: Unarchive tgz...
      unarchive:
        src: /home/mark/timelapse/files/images.tgz
        dest: /home/mark/timelapse/files/

    - name: Compile the files into an mp4...
      shell: ffmpeg -i /home/mark/timelapse/files/frame%04d.jpg -crf 18 -c:v libx264 /home/mark/timelapse/videos/output.mp4

    - name: Run tweetvideo.py
      shell: python3 /home/mark/timelapse/tweetvideo.py
      register: python_output
    - debug:
        var: python_output

    - name: Rename output to output and timestamp...
      shell: mv /home/mark/timelapse/videos/output.mp4 /home/mark/timelapse/videos/output-{{  ansible_date_time.iso8601_basic_short }}.mp4

    - name: Remove images and TAR...
      shell: rm -rf /home/mark/timelapse/files/*
```

If you look closely... I output an mp4 here rather than the webm. That is because Twitter will only accept an mp4. The playbook kicks off tweetvideo.py. This is a script I lifted almost word for word from the [Twitterdev Github](https://github.com/twitterdev/large-video-upload-python/).

```
import os
import sys
import time

import json
import requests
from requests_oauthlib import OAuth1


MEDIA_ENDPOINT_URL = 'https://upload.twitter.com/1.1/media/upload.json'
POST_TWEET_URL = 'https://api.twitter.com/1.1/statuses/update.json'

CONSUMER_KEY ="Key"
CONSUMER_SECRET ="Secret"
ACCESS_TOKEN ="Token"
ACCESS_TOKEN_SECRET ="Secret"

VIDEO_FILENAME = "/home/mark/timelapse/videos/output.mp4"


oauth = OAuth1(CONSUMER_KEY,
  client_secret=CONSUMER_SECRET,
  resource_owner_key=ACCESS_TOKEN,
  resource_owner_secret=ACCESS_TOKEN_SECRET)


class VideoTweet(object):

  def __init__(self, file_name):
    '''
    Defines video tweet properties
    '''
    self.video_filename = file_name
    self.total_bytes = os.path.getsize(self.video_filename)
    self.media_id = None
    self.processing_info = None


  def upload_init(self):
    '''
    Initializes Upload
    '''
    print('INIT')

    request_data = {
      'command': 'INIT',
      'media_type': 'video/mp4',
      'total_bytes': self.total_bytes,
      'media_category': 'tweet_video'
    }

    req = requests.post(url=MEDIA_ENDPOINT_URL, data=request_data, auth=oauth)
    media_id = req.json()['media_id']

    self.media_id = media_id

    print('Media ID: %s' % str(media_id))


  def upload_append(self):
    '''
    Uploads media in chunks and appends to chunks uploaded
    '''
    segment_id = 0
    bytes_sent = 0
    file = open(self.video_filename, 'rb')

    while bytes_sent < self.total_bytes:
      chunk = file.read(4*1024*1024)
      
      print('APPEND')

      request_data = {
        'command': 'APPEND',
        'media_id': self.media_id,
        'segment_index': segment_id
      }

      files = {
        'media':chunk
      }

      req = requests.post(url=MEDIA_ENDPOINT_URL, data=request_data, files=files, auth=oauth)

      if req.status_code < 200 or req.status_code > 299:
        print(req.status_code)
        print(req.text)
        sys.exit(0)

      segment_id = segment_id + 1
      bytes_sent = file.tell()

      print('%s of %s bytes uploaded' % (str(bytes_sent), str(self.total_bytes)))

    print('Upload chunks complete.')


  def upload_finalize(self):
    '''
    Finalizes uploads and starts video processing
    '''
    print('FINALIZE')

    request_data = {
      'command': 'FINALIZE',
      'media_id': self.media_id
    }

    req = requests.post(url=MEDIA_ENDPOINT_URL, data=request_data, auth=oauth)
    print(req.json())

    self.processing_info = req.json().get('processing_info', None)
    self.check_status()


  def check_status(self):
    '''
    Checks video processing status
    '''
    if self.processing_info is None:
      return

    state = self.processing_info['state']

    print('Media processing status is %s ' % state)

    if state == u'succeeded':
      return

    if state == u'failed':
      sys.exit(0)

    check_after_secs = self.processing_info['check_after_secs']
    
    print('Checking after %s seconds' % str(check_after_secs))
    time.sleep(check_after_secs)

    print('STATUS')

    request_params = {
      'command': 'STATUS',
      'media_id': self.media_id
    }

    req = requests.get(url=MEDIA_ENDPOINT_URL, params=request_params, auth=oauth)
    
    self.processing_info = req.json().get('processing_info', None)
    self.check_status()


  def tweet(self):
    '''
    Publishes Tweet with attached video
    '''
    request_data = {
      'status': '#stlwx #timelapse #raspberrypi #ansible',
      'media_ids': self.media_id
    }

    req = requests.post(url=POST_TWEET_URL, data=request_data, auth=oauth)
    print(req.json())


if __name__ == '__main__':
  videoTweet = VideoTweet(VIDEO_FILENAME)
  videoTweet.upload_init()
  videoTweet.upload_append()
  videoTweet.upload_finalize()
  videoTweet.tweet()
```

## One command

Now, with one Ansible command I'm able to orchestrate the work of the Pi and the laptop and leverage the Twitter API to post my tweet on @Stltimelapse. Stephanie and I both follow this account from our normal Twitter accounts, so the timelapse will just pop up in our feed. A pleasant surprise!

```ansible-playbook timelapse.yml -k --extra-vars '{"duration":"7200000","interval":"8000"}'```