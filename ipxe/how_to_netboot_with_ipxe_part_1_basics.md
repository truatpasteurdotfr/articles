[Read Article on medium.com](https://medium.com/@peter.bolch/how-to-netboot-with-ipxe-6a41db514dee)

How to Netboot with iPXE Part 1
===============================

The Basics
----------

![](https://miro.medium.com/max/1400/1*dmk-oUjHhMKpRUhEoFpVgw.png)

I recently wanted to try out PXE (Preboot Execution Environment). I came across iPXE, former known as gPXE. (For the whole story see: i[PXE Homepage](https://ipxe.org/start))

iPXE is an ”open source network boot firmware” with some nice features. Instead of booting over dhcp and deliver the OS over tftp([learn more](https://networkboot.org/)), one can use HTTP (even HTTPS but with outdated cipher suites) iCSE SAN, AoE SAN, etc. It ships with a command line, we can build for different architectures, we can build bootable usb sticks, iso images …..

What will I learn?
------------------

*   Build your very own iPXE .iso image to boot from and play around with the command line
*   Start the .iso image in qemu to play around
*   Write a boot script and embed it to the iPXE iso image
*   Load further boot instructions from a HTTP server

We focus on providing an iPXE boot script on a HTTP server load it with a iPXE and execute the commands. In this article I’m not booting an operating system , but you’ll get familiar with the iPXE scripting language, and loading a boot script over HTTP. In Part II (coming soon) I will show you how to boot Alpine with iPXE.

Getting started
---------------

To build iPXE we need to install the following packages on our system (for the full list and alternatives see: [https://ipxe.org/download](https://ipxe.org/download)):

```
$ dnf install gcc binutils make perl liblzma mtools mkisofs syslinux
```

Clone the iPXE repo from [https://github.com/ipxe/ipxe](https://github.com/ipxe/ipxe)

```
$ git clone [git@github.com](mailto:git@github.com):ipxe/ipxe.git 
```

For our testing environment we will need:

```
$ dnf install qemu python3
```

Build your first iPXE iso image
-------------------------------

First of all, we have to decide for  which system architecture we want to build the image. We choose a x86_64 machine.

Run the following two commands in the src/ folder to build your first iPXE iso image.

```
$ make bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi  
$ ./util/genfsimg -o ipxe.iso bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi
```

After that you’ll find the ipxe.iso file directly in the src folder.

_A word about embedded scripts:  
We could’ve pass a boot script at build time.This script would be executed once iPXE is running. But iPXE also ships with a nice little cli to play around with the iPXE commands. Therefore we don’t need to embed a boot script in the first place._

_A word about uefi and the build without parameters:  
If you only do a “make” in the src folder, like described in the iPXE README on github, you only get an iPXE iso which boots on standard bios, not uefi. You can then find the ipxe.iso in the /bin folder. This will also work for qemu. As I said: to test with qemu we don’t need uefi, but to show how one can build for a specific platform I decided to give you the build parameters for uefi, too. For cross compiling etc. refer to the iPXE documentation._

Test the image
--------------

Start qemu with your iso image as a cdrom (or use gnome boxes)

```
$ qemu-system-x86_64 -boot d -cdrom ipxe.iso -m 512
```

We should see something like this:

![](https://miro.medium.com/max/1400/1*1EeXYdQ0KRskqurqF81gEw.png)

As we can see, iPXE configures the network and does a dhcp call to get an ip (10.0.2.15/24 gw 10.0.2.2). By pressing Ctrl-B we get to the iPXE command line to try out some [iPXE commands](https://ipxe.org/cmd). For example, we can try to load the official iPXE test boot script by typing the command:

```
> chain http://boot.ipxe.org/demo/boot.php
```

If there is no network connection, we can simply type dhcp again and it should work.

Configuration
-------------

There are several options to customize our build. We’ll find the config files in /src/config/.  
For example, we can enable the german keymap by defining KEYBOARD_MAP de in the console.h config file, simply remove the hash.

Take a look around, there are some helpful config options in general.h, console.h or branding.h

How to embed a boot script into your iPXE iso
---------------------------------------------

Let’s assume we want to experiment with iPXE but we won’t use the iPXE command line anymore. Therefore we can dynamically provide a boot script over HTTP. So we need two boot scripts, one to bake into the iPXE iso image and another one which is provided over HTTP.

The first one simply asks for further boot instructions over HTTP. It’s kind of decoupling and makes things very flexible for prototyping and testing, because we can change the boot script, provided by the HTTP server, without the need of rebuilding the whole iso image.

So here is the boot script **boot.ipxe** for the iso image:

```
#!ipxe  
echo Configure dhcp ....  
dhcp  
chain http://10.0.2.2/boot-http.ipxe
```

Hint: 10.0.2.2 is the ip address of the host machine in qemu.

Now you can embed the script into your iPXE iso with the follwing commands.

```
$ make bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi EMBED=boot.ipxe
$ ./util/genfsimg -o ipxe.iso bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi
```

Now we need a bootscript which is served by a HTTP server. For testing purposes create the following boot script called **boot-http.ipxe**:

```
#!ipxe  
echo hello world!  
echo sleeping for 5 seconds ....  
sleep 5
```

This is perfectly fine for testing. iPXE will boot nothing but it says “hello world!” and is waiting for 5 seconds.

To provide the boot script over HTTP we navigate to the folder where your boot-http.ipxe is located (on the host machine). In this folder start a HTTP server, e.g. a python http server with the following command.

```
$ sudo python -m http.server 80
```

Now start qemu with the new ipxe.iso and we should see this:

![A boot screen with iPXE is booting, calling a http endpoint for further boot instructions and executes them which shows “Hello World” and “Slleping for 5 seconds”](https://miro.medium.com/max/1400/1*yh7YkHkYKr-ma4uV2eGLzQ.png)

As we can see, iPXE starts our boot script, gets the boot-http.ipxe over HTTP and executes the commands in the boot-http file.

Advanced boot script
--------------------

As we can see in the screenshot above, iPXE is going to halt after our boot-http script is executed. This is because we provide nothing bootable. After that iPXE gives back the control to the bios. In consequence the bios tries to boot from HDD. And for shure there is nothing to boot from and we get the message ‘No bootable device’.  
Now we have to reboot or force shutdown to test another boot script.

Lets make this more comfortable by using the following iPXE boot script.

```
#!ipxe  
set uri http://10.0.2.2/boot-http.ipxe
dhcp  
:loading  
prompt Press any key to continue loading from ${uri}  
echo loading from ${uri}  
chain ${uri}  
goto loading
```

**set uri**

With the set command we can assign a value to a variable and use it later on.

```
# syntax 'set [varname] [value]'  
set uri http://10.0.2.2/boot-http.ipxe
...  
# usage of the variable with ${varname}  
prompt Press any key to continue loading from ${uri}
```

**Label**

```
# This is a label, syntax ':<label>'  
:loading
```

**Prompt**

Just wait for the user to press any key. We use this to give control to the user.

```
prompt Press any key to continue loading from ${uri}
```

**goto**

```
goto loading
```

If the boot script isn’t booting into an OS, we came back to the iPXE script and jump to the label ‘:loading’ to start all over again. You can adjust your boot-http.ipxe and try again.

Final thoughts
--------------

We’ve build our own bootable iPXE iso, introduced config files to configure the build, introduced embedded boot scripts and finally show how to load boot instructions from a HTTP server. We’ve tested all of this with qemu and a simple HTTP server on the host machine.

Now we have a setup to play around with iPXE.

“[In How to Netboot with iPXE Part 2: Booting Alpine](https://medium.com/@peter.bolch/6191ed711348)” we will learn how to boot Alpine Linux with iPXE.

Links and Resources
-------------------

1.  Official Website: [https://ipxe.org/start](https://ipxe.org/start)
2.  FAQ: [https://ipxe.org/faq](https://ipxe.org/faq)
3.  What is Networkbooting: [https://networkboot.org/](https://networkboot.org/)
4.  Rom Burning [https://ipxe.org/howto/romburning](https://ipxe.org/howto/romburning)
5.  Downlaod page of iPXE [https://ipxe.org/download](https://ipxe.org/download)
6.  iPXE command reference [https://ipxe.org/cmd](https://ipxe.org/cmd)
7.  PXE boot Alpine: [https://wiki.alpinelinux.org/wiki/PXE_boot](https://wiki.alpinelinux.org/wiki/PXE_boot)
