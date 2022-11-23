---
tags:
  - home server
  - truenas scale
  - smb
  - samba
  - nas
---

# Mounting SMB shares at boot on Truenas Scale

_2022-11-23_

Largely taken from [here](https://linuxconfig.org/how-to-mount-a-samba-shared-directory-at-boot) and saved for posterity.

To be able to mount the Samba share at boot, as a first thing we need to create a mountpoint on our local filesystem. For the sake of this article we will create and use the `/mnt/asustor-plex` directory for this purpose. To create the directory we can run:

`sudo mkdir /mnt/asustor-plex`

Our mountpoint is now ready. Next we need to create an entry in the `/etc/fstab` file for the Samba share. On any Linux system, the `/etc/fstab` file contains the instructions needed to mount filesystems at boot. 

For simplicity we will store the SMB credentials directly in the /etc/fstab file. For example, run `nano /etc/fstab` and add this line:

`//192.168.0.39/shared_data /mnt/asustor-plex cifs username=myusername,password=mypassword,noperm 0 0`

1. In the first entry field we reference the filesystem we want to mount. Normally, when dealing with standard filesystems, we reference them by using their UUID, LABEL or path. In this case, however, we need to provide the IP of the samba server together with the name of the Samba share.
2. In the second field of the entry we specify the mountpoint for the filesystem. 
3. The third field, instead, is used to specify the filesystem type: we need to use “cifs” as value here.
4. The fourth field is where we specify mount options: here, as we said above, we used the username and password options to pass our Samba share credentials. This way of specifying credentials has its obvious flaws, since everyone in the system is be able to read the file. Even if the file had more strict permissions, the mount options would be visible in the output of the mount command, which, when invoked without options returns a list of the mounted filesystems and the associated mount options.
5. `noperm` is used to allow all users on the local system to write to the share once it is mounted. This is not particularly secure, but does make things easier in a home lab setup.
6. The last two fields of the fstab entry are used to specify whether the filesystem should be dumped (boolean value) and in what order filesystem should be checked (a value of 0 disables the check altogether).

After we save the entry in the fstab file, to check that the Samba share is mounted without problems, we can simply run:

`sudo mount -a`

Now you can access the files from the share at `/mnt/asustor-plex` even after reboots.
