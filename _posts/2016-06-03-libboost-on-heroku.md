---
title: 'Installing Boost C++ libraries (libboost) on Heroku'
date: 2016-06-03
permalink: /posts/2016/06/libboost-on-heroku/
redirect_from:
  - /libboost-on-heroku
tags:
  - heroku
---

A quick note for anyone else who runs into this.

You can try this experimental [Heroku Apt buildpack](https://github.com/heroku/heroku-buildpack-apt). The documentation is a little out of date as it's best to use it with the [multiple buildpacks support](https://devcenter.heroku.com/articles/using-multiple-buildpacks-for-an-app). Buildpacks are order dependent so you'll want to add this one first in the list:

```
heroku buildpacks:add https://github.com/heroku/heroku-buildpack-apt --index 1
```

You'll need to add a file name `Aptfile` at the root of the project with the following contents:

```
libstdc++-4.8-dev
libboost1.42-dev
libboost1.46-dev
libboost1.48-dev
libboost1.49-dev
libboost1.50-dev
libboost1.52-dev
libboost1.53-dev
libboost-atomic1.54-dev
libboost-chrono1.54-dev
libboost-context1.54-dev
libboost-coroutine.54-dev
libboost-date-time1.54-dev
libboost-exception1.54-dev
libboost-filesystem1.54-dev
libboost-graph-parallel1.54-dev
libboost-graph1.54-dev
libboost-iostreams1.54-dev
libboost-locale1.54-dev
libboost-log.54-dev
libboost-math1.54-dev
libboost-mpi-python1.54-dev
libboost-mpi1.54-dev
libboost-program-options1.54-dev
libboost-python1.54-dev
libboost-random1.54-dev
libboost-regex1.54-dev
libboost-serialization1.54-dev
libboost-signals1.54-dev
libboost-system1.54-dev
libboost-test1.54-dev
libboost-thread1.54-dev
libboost-timer1.54-dev
libboost-wave1.54-dev
libboost1.54-doc
libboost1.54-tools-dev
libmpfrc++-dev
libntl-dev
```

Then you can do an empty deploy and boost should be available:

```
$ git add Aptfile
$ git commit -m "Add libboost"
$ git push heroku master
```

You'll see a lot of error messages but yolo...

The files are installed to `/app/.apt/usr/lib/x86_64-linux-gnu/`

### Why not just use `libboost-all-dev`?

I'm so glad you asked. The Heroku Apt buildpack isn't the same as `apt-get` or `aptitude` so it doesn't have any dependency resolution. That means you have to specify *every* dependency in the `Aptfile`.

