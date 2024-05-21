---
title: 'Reviving a Roland VS880 with an SD card'
date: 2023-01-18
permalink: /posts/2023/01/reviving-a-roland-vs880-with-an-sd-card
tags:
  - vs880
  - roland
  - hardware
---

This is more of a reminder to myself than a blog post, but it might prove useful to others.

I have an old Roland VS880 multitrack recorder which looks like this:

![Roland VS880 - what a beast](https://static.roland.com/assets/images/products/gallery/vs-880_angle_gal.jpg)

After letting it languish in my loft for a few years I had a need to do some location recording recently and decided to fix it up.

Following the instructions here https://untidymusic.com/roland-vs880/adventures-with-the-roland-vs880-cf-card-and-cf-to-ide-reader I bought myself the following:

- a cheap Compact Flash to IDE adapter [link](https://www.amazon.co.uk/dp/B07NS5JTP5?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- a more expensive Compact Flash to IDE adapter [link](https://www.amazon.co.uk/dp/B0036DDXUM?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- an 8Gb Compact Flash card [link](https://www.amazon.co.uk/dp/B008W2QHYQ?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- a 32Gb SD card [link](https://www.amazon.co.uk/dp/B08GYG6T12?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- a Compact Flash to SD adapter [link](https://www.amazon.com/QUMOX-Compact-Memory-Adapter-Reader/dp/B019REDBY6/ref=sr_1_3?dchild=1&keywords=sd%2Bto%2Bcf%2Badapter&qid=1621264224&sr=8-3&th=1)

Basically, I tried every combination of the above but nothing worked. Whatever I did resulted in `SYS Init = No DRV` being shown after booting the VS880.

People on the [VS Planet](http://www.vsplanet.com/) forum were complaining of similar issues, suggesting that manufacturers had silently changed the components which caused these parts to lose compatibility with the VS880 hardware.

I was ready to give up, but when trying to recover data from my (broken) old IDE drive that I pulled out, I noticed that the formatting was unusual

```
/dev/disk5 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *2.2 GB     disk5
   1:                 DOS_FAT_16 VS-880ST25B             1.1 GB     disk5s1
   2:                 DOS_FAT_16 VS-880ST25B             1.1 GB     disk5s2
   3:               DOS_FAT_16_S VS-880ST25B             20.2 MB    disk5s3
```

I thought I'd try formatting the SD card to match this using a Mac first and then see if the VS880 would recognize it. After some googling I came up with this command:

```
diskutil partitionDisk /dev/disk2 MBR "MS-DOS FAT16" "VS880" 2048M free "" 0B
```

where `disk2` was the ID of my SD card - if you decide to run this, check where your SD card is mounted by looking at the output of `mount` first!

I think the `2048M` part here was important, as the next time I plugged it into the VS880 everything worked! I could follow the instructions to initialize the drive from here: https://www.sweetwater.com/sweetcare/articles/roland-vs-880-force-initializing-hard-drive-vs-880/

At the time of writing it is 17% of the way into a surface scan but I'm more hopeful that it's back now.

