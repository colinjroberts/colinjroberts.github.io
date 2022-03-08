---
layout: post
title:  "CrossXwalk"
date:   2022-01-01 05:05:05 -0800
categories: art
published: true
lede: "A crosswalk that signals more than walking - an interactive art project"
---

crossXwalk (pronounced "crosswalk") is an interactive art piece made for Critical Northwest, an event put on by [Ignition NW](https://ignitionnw.org/). It was a collaboration between myself, Ian, K, G, and Sabina. You can find the code used on the [crossXwalk github](https://github.com/iwsmith/crosswalk).

The idea was to recreate a crosswalk that looks and acts like a regular crosswalk (i.e. you push a button, wait for the signal to change from a red hand to a white silhouette of a walking person, then walk across as it eventually changes to a flashing red hand with a countdown). The major difference is that instead of always giving a signal to walk, our art piece would give signals to do other activities like run, jump, gallop, dance, and stand on one leg. This is the finished product:

{% 
include caption-img.html 
image="crossXwalk-8.jpg" 
caption="The completed crosswalk in action" 
alt = "A closeup of a black crosswalk box attached to a pole with a light on it in the forest. The sign displays a red hand."
%}

G and Sabina took on the task of creating the pole by welding metal poles to sheet metal plates. 

G and Ian figured out how to get the RaspberryPis to drive the LED matrix and developed a system for triggering and displaying the content. The code needed to be able to synchronize the boxes so regardless of which button was pushed, both boxes would be triggered at the same time, play the same images, and count down the same amount.

I created the content displayed on the crosswalks. The implementation of the code is explained on the [crossXwalk github](https://github.com/iwsmith/crosswalk), so this post will serve more as a showcase for the content.

## Design and Construction
The first challenge was mocking up the physical design. To make the box, we would need screens, a case with a transparent front that would protect the electronics from rain, and a way to run power to them and a button. The case and button would need to be mounted on some kind of post. 

{% include img-gallery.html filenames="crossXwalk-1.jpg,crossXwalk-2.jpg,crossXwalk-3.jpg,crossXwalk-4.jpg,crossXwalk-5.jpg,crossXwalk-6.jpg"%}

The box was a plastic scrapbook paper box with a few holes drilled in it. Each box contained a RaspberryPi, a power strip, two 64x32 RGB LED Matrix - 5mm pitch, and a bunch of cables connecting everything (including a sensor on the button box). The button (with an LED inside for visibility) was attached to a wooden box along with a sign we had made that says "Push button to cross". The speaker that played all of the audio was attached to the pole.

{% 
include caption-img.html 
image="crossXwalk-7.jpg" 
caption="The signal and button boxes" 
alt = "The crosswalk box and small button box on a table."
%}

## Creating Images and Audio

Like a real crosswalk, ours needed to have a hand showing you should stop/not walk, a walk symbol which stays on for some amount of time, then a number countdown and a blinking hand showing how much time you have left to cross. Near us, many of the crosswalks also make sounds or talk to you. Typically, when you press the button, it will say "wait" and start clicking. When the signal changes to walk, the clicking changes to a different sound and, at some intersections, a voice will say something like "now crossing main street".

We started by brainstorming as many actions as we could: walk, run, jump, cartwheel, hop, play tag, air guitar, crawl, hydrate, sing karaoke, act like an animal (snake, horse, dog, cat), the list goes on. In later iterations, we added walk signals in different languages and longer scheduled events like yoga, meditation and workouts. With a long list of actions, I set about making sounds and images that would be happen when the button was pushed. 

Each pi was effectively running a 64x64 LED matrix, so the simplest approach was to design svgs that I could use to make 64x64px gifs. I created the gifs in Photoshop which allows one to set the length of each frame. I designed a few of them myself, but the majority of the shapes were bought from the [Noun Project](https://thenounproject.com/). I Photoshop's Timeline window to create, edit, and time the frames to be used in each gif. Adobe's tutorial [here](https://www.adobe.com/creativecloud/photography/discover/animated-gif.html) / [archive](https://web.archive.org/web/20211215220447/https://www.adobe.com/creativecloud/photography/discover/animated-gif.html) shows how to do that. The finished gifs were pushed to the Github repo, then later we'd pull down the repo from the pi on each crosswalk box.

For the audio, we wanted the crosswalk to be similar to a real crosswalk, but our art piece wasn't going to be placed on a street that the voice could announce. Instead, we wanted the voice to specifically announce the action you were being invited to perform. We settled on the phrase "____ sign is on. ____ now." For example, if the signal was to walk, it would say "Walk sign is on. Walk now". 

We wanted a voice that was consistent across all the walks, not too human, and not too robotic. We settled on using the `say` command a Mac using the Samantha voice. Most audio files were created using the following command then edited in Audacity to match the length of the gif and add any extra sounds:

```
say -v "Samantha" "Test sign is on. Test now." --data-format="LEF32@22050" -o "test.wav"
```

For the system to work smoothly, the gif and the audio need to be the same length, so sometimes the gif was made longer to fit the audio, but most often, the audio was extended with silence match the length of the gif. We found 3-9 seconds to be a good range. 

When the button was pressed, the system would randomly choose a walk. If it wasn't one of the language walks, it would randomly choose one of the wait signals (which were different lengths). You would be told to wait a few times, then the walk would play, eventually counting down until the red hand shows again.

Here is what a full walk signal cycle would look like.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/qU0uRPJr9ZM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

In the end, we made over 200 different actions, not including ones with repeats. For example, in the rock paper scissors walk, the crosswalk challenges you to 3 games of rock paper scissors. We made 27 different variations so that the average user would most likely experience a different game each time they played.

I've converted a handful of the images and audio into videos, but they will only show the walk animation/sound. Please use your imagination to add the wait signal before the signal and the countdown after. Before showing those examples, I want to document the process I used to create them from the gif and wav files the crosswalk uses.

## Generating Videos of the Content
The videos before were generated using gifsicle and ffmpeg. It uses the actual gifs and audio files used in the crosswalk, hence the low pixel quality as the files are all 64x64, but are blown up for the videos. The following code assumes a files structure like the following:

```
.
└── files
    ├── img
    │   ├── walks
    │   │   ├── bowl.gif
    │   │   ├── cartwheel.gif
    │   │   ├── chestbump.gif
    │   │   ├── cowboy.gif
    │   │   ├── dance-twist.gif
    │   │   ...
    │   │   └── wok.gif
    │   └── intros
    │       ├── wait2.gif
    │       ├── wait4.gif
    │       ├── wait6.gif
    │       └── wait8.gif
    ├── snd
    │   ├── bowl.wav
    │   ├── cartwheel.wav
    │   ├── chestbump.wav
    │   ├── cowboy.wav
    │   ├── dance-twist.wav
    │   ...
    │   ├── wait2.wav
    │   ├── wait4.wav
    │   ├── wait6.wav
    │   ├── wait8.wav
    │   └── wok.wav
    └── conversion
        ├── scaled_gifs
        └── final_videos
```

The script loops through all of the gifs in the walks directory, runs gifsicle to scale up the gif and saves it in the scaled_gifs directory. Then it runs ffmpeg to combine the gif and wav into an mp4 and saves it in the final_videos directory (with a counter to make it easier to rate limit).

```bash
#!/bin/bash

# Set directory variables
WALK_DIR=files/img/walks
WAIT_DIR=files/img/intros
AUDIO_DIR=files/snd
SCALED_DIR=conversion/scaled_gifs
VIDEO_DIR=conversion/final_videos

# For each walk 
COUNTER=0
START=0
MAX_FILES=200

for walk in "$WALK_DIR"/*.gif; do

    if [ "$COUNTER" -ge "$START" ] && [ "$COUNTER" -lt "$MAX_FILES" ]; then

        echo $walk
        WALK_NAME=$( echo "$walk" | cut -d "/" -f12)
        FILE_NAME_NOEXT=$( echo "$WALK_NAME" | cut -d "." -f1)

        # scale up using gifsicle
        gifsicle --resize 512x512 $walk > "$SCALED_DIR"/$WALK_NAME   

        # Combine resized gif with audio of the same name
        AUDIO_NAME=$AUDIO_DIR/$FILE_NAME_NOEXT.wav
        ffmpeg -f gif -i "$SCALED_DIR"/$WALK_NAME -i "$AUDIO_NAME" -c:v libx264 -crf 50 -c:a aac -pix_fmt yuv420p -max_muxing_queue_size 4000 "$VIDEO_DIR"/"$FILE_NAME_NOEXT".mp4 

    fi

    ((COUNTER++))
done
```

## Other-Than-Walks

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/videoseries?list=PLStnh-lzh70P37qolg62q5VFKok38jGlM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The videos in this playlist show a number of signals people were shown in addition to "walk".
- walk
- applaud
- baseball
- bowl
- cartwheel
- click heels
- dog
- duck
- hadouken
- head pat, belly hub
- laugh
- mime
- order by height
- Emily Dickinson poetry
- rock paper scissors
- swashbuckle

## Languages

The second year we took crossXwalk to Critical, we updated it with many more signals than it had the first year. It was a goal of mine to add some signals in a few different languages.

I had seen [ampelmänner](https://www.ampelmann.de/en/) before in street photos of Berlin, so I set out to try to find other interesting pedestrian crossing lights. From what I could find with a few internet searches, crosswalks seem pretty similar around the world. I suspect that the more unique ones I found were probably art of their own, only used in a single location, or used for a limited amount of time. Also, most of the more unique signs I could find didn't have any audio attached, and it is surprisingly challenging to find videos of crosswalks with good audio from around the world, so we made up our own instead. We were able to translate some on our own and crowdsourced some other translations. Although the images and the sounds might not be perfect, I'm pretty pleased with how they turned out.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/videoseries?list=PLStnh-lzh70PwcLR0Pr51OVqbQvAdzF8T" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The videos in the playlist above show the wait and walk signals we made that match our style while adapting elements of actual signs I found in Taiwan, Japan, Germany, Denmark, the Netherlands, and Dubai.

## Scheduled Events

One of the other additions we added the second year were scheduled activities including meditation, workouts, and yoga. The crosswalk doesn't have the most soothing or even understandable voice, but these were pretty fun to make and to do at the event. The videos in the following playlist show one of each type of event. 

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/videoseries?list=PLStnh-lzh70MBAYnKHYOwtR9-lSvARlde" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
