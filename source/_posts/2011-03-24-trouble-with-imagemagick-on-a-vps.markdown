---
layout: post
title: "Trouble With ImageMagick on a VPS"
date: 2011-03-24 17:45
comments: true
categories: [VPS, ImageMagick, OpenMP]
---

This week I was adding a galleries to a rails site that I am working on and for some reason on my production server, each time I tried to upload an image it failed and I got returned a 502 response from my server. The thing was that even though I was looking at my logs I saw no request even being made to the server. I finally got an image to upload and could see in my logs that the action took in excess of one minute to complete which led my Rails worker (unicorn) to time out and return the aforementioned 502 error response. As I went through the call stack I could determine that the bottleneck was ImageMagick and especially the calls to the convert method.

I did some testing and could confirm what I had determined in the logs:
```
$ time utilities/convert 'image.jpg' -resize "x60" -crop "60x60" +repage 'thumb'
real    0m11.602s
user    0m11.414s
sys     0m0.069s
```

__11 seconds to scale and crop an image, that’s just insanly slow!!!__

I did some googleing and found rumors that openmp, that is activated by default in ImageMagick and helps to make use of modern CPUs multi core architecture sometimes could cause issues in a virtualized environment wich is exactly what I am running on.

So I went out and got the latest version of the ImageMagick source, compiled it with the —disable-openmp flag and redid the test with this binary:
```
$ time utilities/convert 'image.jpg' -resize "x60" -crop "60x60" +repage 'thumb'
real    0m0.077s
user    0m0.058s
sys     0m0.019s
```

That’s more like it, don’t you agree. All I know had to do was to install this version on to my server. However as I am running Ubuntu 10.04 so I’d like to keep with the package system. For some reason I was not able to get the 10.04 package of ImageMagick to accept the —disable-openmp flag so, in the end I backported the package from Ubuntu 10.10 instead. please find the steps below:

###1. Download the following files:
* [imagemagick_6.6.2.6-1ubuntu1.1.dsc](http://archive.ubuntu.com/ubuntu/pool/main/i/imagemagick/imagemagick_6.6.2.6-1ubuntu1.1.dsc)
* [imagemagick_6.6.2.6.orig.tar.bz2](http://archive.ubuntu.com/ubuntu/pool/main/i/imagemagick/imagemagick_6.6.2.6.orig.tar.bz2)
* [imagemagick_6.6.2.6-1ubuntu1.1.debian.tar.bz2](http://archive.ubuntu.com/ubuntu/pool/main/i/imagemagick/imagemagick_6.6.2.6-1ubuntu1.1.debian.tar.bz2)

###2. Unpack
```
$ dpkg-source -x imagemagick_6.6.2.6-1ubuntu1.1.dsc
```

###3. Edit the build rules
```
$ cd imagemagick-6.6.2.6
$ vim debian/rules
```

Add the the following line to the ./configure statement on line 25-39. I added mine on line 34.
```
34: —disable-openmp \
```

###4. Add dependencies and build ( I needed these dependencies)
```
$ sudo apt-get install liblqr-1-0-dev librsvg2-dev
$ dpkg-buildpackage -b
```

###5. Out with the old, in with the new
```
$ sudo apt-get remove —purge imagemagick
$ sudo dpkg -i libmagickcore3_6.6.2.6-1ubuntu1.1_amd64.deb
$ sudo dpkg -i libmagickwand3_6.6.2.6-1ubuntu1.1_amd64.deb
$ sudo dpkg -i imagemagick_6.6.2.6-1ubuntu1.1_amd64.deb
```

So there we are, my gallery is now up and running and preforming as should.
