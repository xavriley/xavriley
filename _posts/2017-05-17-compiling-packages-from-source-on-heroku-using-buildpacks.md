---
title: 'Compiling packages from source on Heroku using buildpacks'
date: 2017-05-17
permalink: /posts/2017/05/compiling-packages-from-source-on-heroku-using-buildpacks
redirect_from:
  - /compiling-packages-from-source-on-heroku-using-buildpacks
tags:
  - heroku
---

##### (Warning: whilst I work for Heroku, this isn't official supported - it's just something I discovered in my spare time. Hopefully this helps someone but leave me any feedback on Twitter `@xavriley`)

What do you do if you're deploying an app on Heroku but you have a package your app depends on? The first thing is to check the list of default packages here https://devcenter.heroku.com/articles/stack-packages - if it already exists there then you don't need to worry.

The next thing to checkout is the list of third party buildpacks here https://elements.heroku.com/buildpacks These are contributed by the community and it's fairly likely that someone has gone down this path before.

If there's no third party buildpack, or if the ones you've found are no longer maintained, then the next option is the `Apt` buildpack https://elements.heroku.com/buildpacks/heroku/heroku-buildpack-apt This allows you to create a file named `Aptfile` at the root of your project which lists packages that you'd like to install. Bear in mind that this doesn't resolve dependencies in quite the same way as `apt-get install` so you'll need to list all the dependencies you want.

Right - with those out of the way, what about if none of them work? I came across this recently where someone wanted to use a version of `curl` that had support for HTTP/2 requests.

### Building `curl` with HTTP/2 support from source

First, add the following buildpacks:

```
heroku buildpacks:add --index 1 https://github.com/heroku/heroku-buildpack-apt.git
heroku buildpacks:add https://github.com/weibeld/heroku-buildpack-run.git
```

Now install the dependencies - create a file named `Aptfile` in the root of the project with the following contents:

```
# Aptfile
nghttp2
libnghttp2-dev
```

To trigger the compilation (on each build unfortunately) create a file named `buildpack-run.sh` in the root of the project with the following contents:

```
# buildpack-run.sh
wget http://curl.haxx.se/download/curl-7.46.0.tar.bz2
tar -xvjf curl-7.46.0.tar.bz2
cd curl-7.46.0
./configure  --disable-shared --with-nghttp2 --prefix=/app/.apt/usr
make
```

> #### Finding libraries in the right places
>
> The missing piece of the puzzle was to use the `--disable-shared` flag with `./config`. When the app is building on Heroku it uses funky folders in `/tmp` and the compiled binary tries to link against those paths. By disabling shared libs, it pulls in all the necessary code to the resulting binary so you can move it around the system without problems.


You will then need to add these new files and deploy. This should build a version of curl for you in the root of the project.

You can then test this in a one-off dyno as follows:

```
$ heroku run bash
...
~ $ ./curl-7.46.0/src/curl --http2 -I https://nghttp2.org/
HTTP/2.0 200
date:Thu, 27 Apr 2017 11:21:13 GMT
content-type:text/html
last-modified:Mon, 24 Apr 2017 13:50:42 GMT
etag:"58fe02b2-19ff"
accept-ranges:bytes
content-length:6655
x-backend-header-rtt:0.001139
strict-transport-security:max-age=31536000
server:nghttpx
via:2 nghttpx
x-frame-options:SAMEORIGIN
x-xss-protection:1; mode=block
x-content-type-options:nosniff
```

The `curl` binary and object files are located in `/app/curl-7.46.0/src/` (Heroku deploys your code to the `/app` folder by default).

Note that the dyno now has two versions of `curl` installed! If you need to use the one with HTTP/2 support you have to be careful and specify the full path `./curl-7.46.0/src/curl --http2 -I https://nghttp2.org/` or add it to the front of the `$PATH` variable.

