---
layout: post
---

I've been living without internet recently. That hasn't stopped me writing code though. If anything it has accelerated the process. I'm mainly using Git these days. It's more than at home  working without a connection to my repos on GitHub. Every now and again I like to back things up though. That's where the "**Sneakernet Push**" comes in.

Start by creating a new empty git repository somewhere on the drive you're going to transfer your data with:

```bash
$ mkdir usb.git
$ cd usb.git
$ git init --bare
```

Add the new repo as a remote so you can push to it easily:

```bash
$ git remote add usb file:///Volumes/USB/usb.git
```

Now you can just push the branches you want to the new git repo on the USB stick:

```bash
$ git push usb master:master
```

The next step is a litre more difficult. Go forth and seek out some wild internet. From there you can get those precious bytes into the cloud with a couple of commands. Open up the usb.git folder, ad the internet remote and push the commits up to it.

```bash
$ git remote add origin http://github.com/myname/myrepo
$ git push origin master
```

Sit back, relax, grab a beverage and begin contemplating your next commit.