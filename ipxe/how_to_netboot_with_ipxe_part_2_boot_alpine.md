[Read Article on medium.com](https://medium.com/@peter.bolch/how-to-netboot-with-ipxe-6191ed711348) 

How to Netboot with iPXE Part 2
===============================

Booting Alpine
--------------

![](https://miro.medium.com/max/1400/0*Im4HYlTDlK2AtpOM)Photo by [Kaidi Guo](https://unsplash.com/@kaidi_guo?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral)

In “[How to netboot with iPXE Part 1: Basics](https://medium.com/@peter.bolch/how-to-netboot-with-ipxe-6a41db514dee)” we’ve learned how to build an iPXE.iso and embed a boot script. Furthermore we loaded an iPXE boot script over HTTP. We used the python HTTP module and qemu to test it on our local machine.

Now we want to use HTTP to provide an iPXE boot script that actually boots Alpine Linux. We will use qemu once again. We choose Alpine because it has some nice resources [describing how to netboot](https://wiki.alpinelinux.org/wiki/PXE_boot) it and a “[netboot server](https://boot.alpinelinux.org/)” with infos and a full featured iPXE boot script so you can start exploring on your own.

What will I learn
-----------------

*   Creating a boot script to boot Alpine linux
*   Add a kernel option (somewhat Alpine specific)

Download Alpine Netboot
-----------------------

Alpine provides a netboot version on the [download page](https://alpinelinux.org/downloads/). There are builds for several architectures. We load the x86_64 Version to our HDD.

Once downloaded we create a folder called _web_. Extract the alpine-netboot-X.Y.ZZ-x86_64.tar.gz to this folder. Later on the folder will also hold the iPXE script which is provided over HTTP.

Now we have two “netbootable” versions of Alpine in our _web_ folder. The *-lts and the *-virt version of Alpine.  
For this tutorial it is not important to know the differences between the lts and the virt version. If you want to know more about the versions visit: [Alpine Wiki Kernel Page](https://wiki.alpinelinux.org/wiki/Kernels).

In your webfolder you should see the following files:

![](https://miro.medium.com/max/368/1*pEIL0qTQzkSs7M8-Zrg98g.png)

iPXE Boot Script for Alpine Linux
---------------------------------

First of all we want to provide a boot script over HTTP to boot Alpine Linux. To perform the boot we have to load a kernel and an initial ram file system (initramfs) or initial ramdisk (initrd). A minimal iPXE bootscript may look like this:

```
#!ipxe  
kernel url/to/kernel/vmlinuz [kernel arguments] initrd=initrd.img  
initrd url/to/initrd.img  
boot
```

On the A[lpine Netboot Wiki Page](https://wiki.alpinelinux.org/wiki/PXE_boot) we can see which kernel arguments are available and which are required.

**Required kernel arguments:**

1.  ip — Provide a static ip or simply use dhcp to get an ip address
2.  alpine_repo —A list of repositories for /etc/apk/repositories

Using _ip=dhcp_, the official _alpine repo url_ and the _alpine files_ (namely the kernel and the initramfs of the lts version) from our web folder, we end up with the following iPXE boot script:

```
#!ipxe  
kernel http://10.0.2.2/vmlinuz-lts ip=dhcp alpine_repo=http://dl-cdn.alpinelinux.org/alpine/v3.15/main initrd=initramfs-lts  
initrd http://10.0.2.2/initramfs-lts  
boot
```

Or a little smoother with variables

```
#!ipxe  
  
set local_address http://10.0.2.2:5001  
set alpine_repo http://dl-cdn.alpinelinux.org/alpine/v3.15/main  
  
kernel ${local_address}/vmlinuz-lts ip=dhcp alpine_repo=${alpine_repo} initrd=initramfs-lts  
initrd ${local_address}/initramfs-lts  
  
boot
```

> Remember: We use qemu and the python http server on the host machine, therefore we use http://10.0.2.2:5001 to call the host machine on port 5001.

Now we create the alpine_boot.ipxe file in the _web_ folder next to the Alpine files and paste the script above into this file. As we’ve learned in “[How to netboot with iPXE Part 1: Basics](https://medium.com/@peter.bolch/how-to-netboot-with-ipxe-6a41db514dee)” we provide our iPXE script by navigating into the web folder and start the python HTTP server with:

```
$ sudo python -m http.server 5001
```

Loading the iPXE script over HTTP and actually boot Alpine
----------------------------------------------------------

> One may ask: Peter why not embed the boot script in the ipxe.iso file instead of loading it over HTTP?  
> Answer: It is much easier to edit the boot script and load it over HTTP than building the iso image everytime you’ve edited the boot script.  
> But shure, if you want to, just build your ipxe.iso with the embeded alpine_boot.ipxe boot script.

Create a boot.ipxe file with the following content:

```
#!ipxe  
chain [http://10.0.2.2:5001/alpine_boot.ipxe](http://10.0.2.2/boot.ipxe)
```

Now lets build our ipxe.iso as shown in “[How to netboot with iPXE Part 1: Basics](https://medium.com/@peter.bolch/how-to-netboot-with-ipxe-6a41db514dee)”:

```
$ cd ipxe/src  
$ make bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi EMBED=boot.ipxe  
$ ./util/genfsimg -o ipxe.iso bin/ipxe.lkrn bin-x86_64-efi/ipxe.efi
```

Time to boot Alpine on qemu over HTTP:

```
$ qemu-system-x86_64 -boot d -cdrom ipxe.iso -m 512
```

You should see something like this

![](https://miro.medium.com/max/1400/1*YWO9dXsAWfwsQJJJm5xXLQ.png)

But what’s this? ERROR????  
Yes! As you can see the error says ‘modloop failed to start’. This is, because we don’t tell the kernel where to find the modloop file. The modloop file provides a full kernel module tree as SquashFS. With our boot_alpine.ipse script we just loaded the minimal kernel from the initramfs to boot. You can play with it, but it will be somewhat limited.

As we can see in the file list above, Alpine Netboot ships with a modloop file. Lets have a look at the A[lpine Docs](https://wiki.alpinelinux.org/wiki/PXE_boot) and we’ll find the modloop option.

> modloop
> 
> If the specified file protocol is http/ftp or https (if wget is installed), the modloop file will be downloaded to the /lib directory and will be mounted afterwards e.g. modloop=http://192.168.1.1/pxe/alpine/grsec.modloop.squashfs in the append section of your [bootloader](https://wiki.alpinelinux.org/wiki/Bootloaders)

So lets give it a try and add this to our alpine_boot.ipxe

```
modloop=${local_address}/modloop-lts
```

Change your boot_alpine.ipxe script to match this:

```
#!ipxe  
  
set local_address http://10.0.2.2:5001  
set alpine_repo http://dl-cdn.alpinelinux.org/alpine/v3.15/main  
  
kernel ${local_address}/vmlinuz-lts modloop=${base-url}/modloop-lts ip=dhcp alpine_repo=${alpine_repo} initrd=initramfs-lts  
initrd ${local_address}/initramfs-lts  
  
boot
```

Another qemu boot will show us something like this:

![](https://miro.medium.com/max/1400/1*iIj6U8nhVSvvTyWsphb6dg.png)

And: tadaaaa. No more errors. Alpine is up and running.

Final thoughts
--------------

We’ve leraned how to boot Alpine with an iPXE boot script. Therefore we provided the Alpine netboot files and the iPXE boot script to boot Alpine over an local HTTP server. We’ve learned how to add boot options to the kernel command by discovering the modloop option.  
As one might imagine, there is a lot more to discover. Alpine gives you the possibility to add an [apk overlay (apkovl)](https://wiki.alpinelinux.org/wiki/Alpine_local_backup) to specify which software should be installed. You can even provide files with an apkovl file.  
So we can not only boot a specific Alpine version, we are also able to boot a Alpine version with a defined set of software and files.

Whats next?
-----------

In “How to Netboot with iPXE Part 3: Creating a menu and failure handling” we will learn how to create a menu with iPXE to boot either the lts version or the virt version of Alpine. Furthermore we will explore some error handling.

Future
------

I’ve decided to go on with this series. Some of the topics I want to discover and share with you are:

*   How to verify kernel and initramfs (imgverify)
*   How to use isset, iseq and some other iPXE commands
*   How to provide software and os updates for edge devices with an iPXE setup.

Disclaimer: The list above is just a rough idea. There is no guarantee that the articles are ever written. ;)

Links and Resources
-------------------

1.  Alpine Netboot Info Page: [https://boot.alpinelinux.org/](https://boot.alpinelinux.org/)
2.  Alpine Netboot Wiki Page: [https://wiki.alpinelinux.org/wiki/PXE_boot](https://wiki.alpinelinux.org/wiki/PXE_boot)
3.  Difference between virt and lts alpine kernels [https://wiki.alpinelinux.org/wiki/Kernels](https://wiki.alpinelinux.org/wiki/Kernels)
4.  apkovl [https://wiki.alpinelinux.org/wiki/Alpine_local_backup](https://wiki.alpinelinux.org/wiki/Alpine_local_backup)
5.  ipxe commands [https://ipxe.org/cmd](https://ipxe.org/cmd)
