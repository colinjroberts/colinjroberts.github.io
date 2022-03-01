---
layout: post
title:  "CrossXwalk"
date:   2022-01-01 05:05:05 -0800
categories: plotter-art
published: true
lede: "A crosswalk that signals more than walking - an interactive art project"
---

crossXwalk (pronounced "crosswalk") is an interactive art piece made for Critical Northwest, an event put on by [Ignition NW](https://ignitionnw.org/). It was a collaboration between myself, I, G, and S. You can find the code used on the [crossXwalk github](https://github.com/iwsmith/crosswalk).

The idea was to create a crosswalk that it mostly looks and acts like a regular crosswalk in that you have to push the button, wait for the light to change to a walk signal, then walk across as it eventually changes to a flashing red hand with a countdown). The major difference is that instead of always giving a signal to walk, our art piece would give signals to do other activities like run, jump, gallop, dance, stand on one leg, etc. This is the finished product:

*PLACEHOLDER FOR WORKING IMAGE*

G and S took on the task of creating the pole by welding metal poles to sheet metal plates. 

G and I figured out how to get the RaspberryPis to drive the LED matrix and developed a system for triggering and displaying the content. The code needed to be able to synchronize the boxes so regardless of which button was pushed, both boxes would announce that they have been triggered, play the same images, and count down the same amount.

I created the content held on the boxes. The implementation of the code is explained on the [crossXwalk github](https://github.com/iwsmith/crosswalk), so this post will serve more as a showcase for the content.

## The Design
The first challenge was mocking up the physical design. To make the box, we would need some screens, a case they could be seen through, a way to run power to them and a button. The case and button would need to be mounted on some kind of post. 

*PLACEHOLDER FOR DESIGN IMAGES*

## The Basics
Each pi was effectively running a 64x64 LED matrix, so the simplest way of developing content seemed to be to design svgs that I could use to generate 64x64 pixel gifs. 

Like a real crosswalk, ours needed to have a hand showing you should stop/not walk, a walk symbol which stays on for some amount of time, then a number countdown and a blinking hand showing how much time you have left to cross. 

*PLACEHOLDER FOR BASIC HAND AND WALK IMAGES*

It also needed a voice telling you to stop, walk, etc. We wanted something that was consistent across all the walks, not too human, and not too robotic. We settled on using the `say` command a Mac using the Samantha voice. Most audio files were created with a command like the following:

```
say -v "Samantha" "Test sign is on. Test now." --data-format="LEF32@22050" -o "test.wav"
```

The next step was to create the gif to be used for the image in Photoshop which allows one to set the length of each frame. For the system to work smoothly, the gif and the audio need to be the same length, so sometimes the gif was made longer to fit the audio, but most often, the audio was extended with silence match the length of the gif. We found that walks of 3-9 seconds to be a good range for some longer and shorter ones. 

## Other-Than-Walks

## Languages

## Scheduled Activities

We've taken crossXwalk to a number of different events, will likely continue to do so here and there. Maybe you'll see it one day!

## Generating Videos of the Content
The videos in this post were generated using gifsicle and ffmpeg which can merge the audio files and the actual gifs that were used on the crosswalk (hence the low pixel quality...all of the vector files are saved somewhere, but it would be a lot of work to remake the gifs with the larger files).

```bash
# Set directory variables
WALK_DIR=files/img/walks
WAIT_DIR=files/img/intros
AUDIO_DIR=files/snd
SCALED_DIR=conversion/scaled_gifs
VIDEO_DIR=conversion/final_videos

WALK_NAME=$( echo "$walk" | cut -d "/" -f11)
FILE_NAME_NOEXT=$( echo "$WALK_NAME" | cut -d "." -f1)

# Scale up using gifsicle
gifsicle --resize 512x512 $walk > "$SCALED_DIR"/$WALK_NAME   

# Combine resized gif with audio of the same name
AUDIO_NAME=$AUDIO_DIR/$FILE_NAME_NOEXT.wav
ffmpeg -f gif -i "$SCALED_DIR"/$WALK_NAME -i "$AUDIO_NAME" -c:v libx264 -crf 50 -c:a aac -pix_fmt yuv420p -max_muxing_queue_size 4000 "$VIDEO_DIR"/"$FILE_NAME_NOEXT".mp4 
```