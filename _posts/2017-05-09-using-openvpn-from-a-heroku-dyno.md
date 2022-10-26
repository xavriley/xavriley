---
title: 'Using OpenVPN from a Heroku dyno'
date: 2017-05-09
permalink: /posts/2017/05/using-openvpn-from-a-heroku-dyno
redirect_from:
  - /using-openvpn-from-a-heroku-dyno
tags:
  - heroku
---

##### (Warning: whilst I work for Heroku, this isn't official supported - it's just something I discovered in my spare time. Hopefully this helps someone but leave me any feedback on Twitter `@xavriley`)

## Update: this won't work

After some more research, the advice below will allow you to install the relevant packages but the Heroku dynos won't have the necessary permissions to rewrite the outgoing network packets. For an alternative way of setting up a secure tunnel to a dyno, you could try looking at https://github.com/koding/tunnel or https://github.com/jpillora/chisel instead.

---------------------------

In terms of getting a VPN client working there's no official support for this as part of the default image. To get something working you will need to add another buildpack to your app. If you haven't worked with multiple buildpacks before, read up on the best approach for those here https://devcenter.heroku.com/articles/using-multiple-buildpacks-for-an-app

You can then try this experimental buildpack to add additional packages:

    heroku buildpacks:add --index 1 https://elements.heroku.com/buildpacks/heroku/heroku-buildpack-apt

You would then create a file named `Aptfile` at the root of your project with the following contents:

```
liblzo2-2
openvpn
```

You would then need to add those new files to git and deploy for those changes to take effect.

Once the deploy completes, you can try out the VPN connection using the following commands whilst running a `heroku run bash` session in a one-off dyno.

```
$ heroku run bash
...
# Bash session running on a dyno
# This line is necessary because the apt version of openvpn doesn't link against libzo correctly otherwise
~ $ export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/app/.apt/lib/x86_64-linux-gnu/"
~ $ .apt/usr/sbin/openvpn --client --remote vpnhost.ac.uk --auth-user-pass --dev tunX --capath /etc/ssl/certs --daemon
```

That will get you a VPN session running with a user/password prompt to start. From there you'll need to look at the various ways of configuring `openvpn` to make it do what you need.

To set the `LD_LIBRARY_PATH` for every dyno, you can add a file named `.profile` to the root of your project with the following contents:

```
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/app/.apt/lib/x86_64-linux-gnu/"
```

You can find out more about the `.profile` script on Heroku here: https://devcenter.heroku.com/articles/dynos#the-profile-file
