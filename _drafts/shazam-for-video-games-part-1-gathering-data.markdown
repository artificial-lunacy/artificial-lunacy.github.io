---
title: 'Shazam for video games - Part 1: Gathering data'
date: 2017-09-02 12:39:00 Z
---

## Shazam for video games - Part 1: Gathering data


If you're not living under the rock for the past few years, you might already know what Shazam is. Shazam is a service that allows you to identify the song you are listening to. There are several scientific on this subject and many great posts have been written to demonstrate how this service works.

In this series of posts, we will try to do something similar and detects video games from a screenshot. If you are thinking that this is complete nonsense,  you are not alone. So why am I doing this? because I'm bored.

In this part, I will show few simple ways to gather necessary data to train a convolutional neural network for classifying screenshots. There are two sources of data for this task, 1. screenshots 2. gameplay videos. We will use videos and extract random frames from them. There are hundreds of thousands hour video footage of games all around the web. Twitch and Youtube are among the greatest sources for extracting videos.

`youtube-dl` is swiss army knife of ripping video contents from the internet. We can simply fire up `youtube-dl` against some URLs, download the whole video and after that, extract the frames we want. `FFmpeg` is another great tool for doing all sort of crazy thing with videos. the following command will extract a frame at time ``00:01:05`` and save it as a jpeg image.

```
ffmpeg -ss 00:10:20 -t 1  -i input.mp4 -f mjpeg output-frame.jpg
```

This approach is incredibly slow! First, we have to download the whole video file and process it afterward. Fortunately, ``FFmpeg`` is smart enough and we can pass URL of a video to it, and it will do the rest for us!  `youtube-dl` has a flag called `get-url` that returns the download URL of the video. putting it together we will have this:

```
ffmpeg -ss 00:01:05 -t 1  -i $(youtube-dl -f 22 --get-url https://www.youtube.com/watch?v=QkkoHAzjnUs) -f mjpeg output-frame.jpg
```

Another crazy feature of `youtube-dl` is the ability to search youtube! You can pass a query to `youtube-dl` and it will automatically search youtube for that given query.

```
youtube-dl "ytsearch:GTA V"
```


Both ``ffmpeg`` and ``youtube-dl`` has Python API and we can use them inside our application. See [here](https://pypi.python.org/pypi/ffmpy) and [here](https://github.com/rg3/youtube-dl/blob/master/README.md#embedding-youtube-dl) for more details.




```python
import argparse
import youtube_dl
import ffmpy
import os

sample_rate = 5
game_dic = {}
video_per_game = 3

def save_frame(url, time, game):
    output_file = os.path.join(game, game + '-' + str(game_dic[game]) + '.jpg')
    ff = ffmpy.FFmpeg(
        inputs={url: '-ss ' + str(time) + ' -t 1'},
        outputs={output_file: '-f mjpeg'})
    ff.run()
    game_dic[game] += 1

def search_and_extract(query, game_name, rate):
    print('Processing links for: {}'.format(game_name))
    ydl = youtube_dl.YoutubeDL({'outtmpl': '%(id)s%(ext)s'})
    with ydl:
        result = ydl.extract_info('ytsearch' + str(video_per_game) + ':' + str(query), download=False)
    
    if not os.path.exists(game_name):
        os.makedirs(game_name)

    for game_video in result['entries']:
        video_link = game_video['formats'][-1]['url']
        duration = game_video['duration']

        for i in range(1, duration, int(duration/rate)):
            save_frame(video_link, i, game_name)


parser = argparse.ArgumentParser(description='download pile of videos from youtube')
parser.add_argument('--file', dest='file', action='store', required=True)
args = parser.parse_args()

with open(args.file) as f:
    content = f.readlines()

for line in content:
    game_name, search_query = str(line).strip().split(', ')

    if game_name not in game_dic.keys():
        game_dic[game_name] = 0

    if search_query is not "":
        search_and_extract(search_query, game_name, sample_rate)

```