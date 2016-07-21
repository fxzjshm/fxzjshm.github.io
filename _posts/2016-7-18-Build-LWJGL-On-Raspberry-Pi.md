---
layout: post  
title: Build LWJGL on Raspberry Pi
date: 2016-7-18
author: Fxzjshm
category: RPi
tags: [open-source, minecraft]
---

I am trying run Minecraft on the Raspberry Pi. But something is nessary for me --- build LWJGL on ARM Linux. OK, let's do it.  

First of first, clone LWJGL 2.X from Github.  

```bash
pi@raspberrypi:~/workspace $ git clone https://github.com/LWJGL/lwjgl --depth=1
```

To avoid the huge history data, I used "--depth=1"  

> Cloning into 'lwjgl'...  
> remote: Counting objects: 1506, done.  
> remote: Compressing objects: 100% (1047/1047), done.  
> remote: Total 1506 (delta 845), reused 764 (delta 419), pack-reused 0  
> .............  

```bash
pi@raspberrypi:~/workspace $ cd lwjgl/
pi@raspberrypi:~/workspace/lwjgl $ ls
```

> Current (2016-7-18) files.  

|                |       |           | |
| -------------- | ----- | --------- |
| applet         | doc   | libs      |
| platform_build | res   | build.xml |
| eclipse-update | maven | README.md |
| src            |       |           |

As the README.md says, try running:  

> * ant generate-all  
> * ant compile  
> * ant compile_native  

Ignored all the `WARNING`s.  

Everything is fine with the first and second one.  
But when it is compiling native code...... ?!  

> [apply] /usr/bin/ld: cannot find -ljawt  
> [apply] collect2: error: ld returned 1 exit status  
>   
> BUILD FAILED  

It seems that ...... `libjawt` is missing.  

```bash
sudo apt-get install libjawt
```

> E: Package libjawt not found.  

```bash
sudo apt-cache search libjawt
```

> (Nothing here.)  

.............  

**Suddenly** I realized that --- awt is part of Java!  
So look at what I found:  

> /usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt/lib/arm/libjawt.so  

So  

```bash
pi@raspberrypi:~/Desktop $ sudo mv libjawt.so /usr/lib
```
Then redo  

```bash
ant compile_native
```

Then everything is normal.  

> BUILD SUCCESSFUL  

OK, open `bin/lwjgl` and that's it.  
