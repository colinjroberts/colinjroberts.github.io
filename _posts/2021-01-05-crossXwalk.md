---
layout: post
title:  "crossXwalk"
date:   2022-01-01 05:05:05 -0800
categories: plotter-art
published: false
---

crossXwalk (pronounced "crosswalk") is an interactive art piece made for Critical Northwest, an event put on by [Ignition NW](https://ignitionnw.org/). It was a collaboration between myself, I, G, and S. You can find the code used on the [crossXwalk github](https://github.com/iwsmith/crosswalk).

The idea was to create a crosswalk with a button that looks and acts like a regular crosswalk (you have to wait for the light to change, it gives you a signal to walk, then counts down with a flashing red hand) with one difference. Instead of always giving a signal to walk, our art piece would give signals to do other activities like run, jump, gallop, cartwheel, etc.

The first challenge was mocking up the physical design. To make the box, we would need some screens, a case they could be seen through, a way to run power to them and a button. The case and button would need to be mounted on some kind of post.

*PLACEHOLDER FOR DESIGN IMAGES*

G and S took on the task of creating the pole by welding metal poles to sheet metal plates. 

G and I programmed the RaspberryPis that run each box. The code needed to be able to synchronize the boxes so regardless of which button was pushed, both boxes would announce that they have been triggered, play the same images, and count down the same amount.

I created all of the content held on the boxes. The implementation of the code is explained on the [crossXwalk github](https://github.com/iwsmith/crosswalk), so this post will serve more as a showcase for the content.

## The Basics
Like a real crosswalk, ours needed to have a hand showing you should stop/not walk, a walk symbol which stays on for some amount of time, then a number countdown and a blinking hand showing how much time you have left to cross. 

*PLACEHOLDER FOR DESIGN IMAGES*

It also needed a voice telling you to stop, walk, etc. We wanted something that was consistent across all the walks, not too human, and not too robotic. We settled on using the `say` command a Mac using the Samantha voice. Most audio files were created with a command like the following:

```
say -v "Samantha" "Test sign is on. Test now." --data-format="LEF32@22050" -o "test.wav"
```

The next step was to create the gif to be used for the image in Photoshop which allows one to set the length of each frame. For the system to work smoothly, the gif and the audio need to be the same length, so sometimes the gif was made longer to fit the audio, but most often, the audio was extended with silence match the length of the gif. We found that walks of 3-9 seconds to be a good range for some longer and shorter ones. 

## Other-Than-Walks

## Languages

## Scheduled Activities

We've taken crossXwalk to a number of different events, will likely continue to do so here and there. Maybe you'll see it one day!