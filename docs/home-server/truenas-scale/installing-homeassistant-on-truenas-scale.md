---
tags:
  - home server
  - truenas scale
  - home assistant
---

# Installing Home Assistant OS on Truenas Scale

_2022-10-29 Update_

There's a newer version of HAOS available, so the below guide has been updated to use that. The original used this:

> Use wget to get the ova file:

> `wget https://github.com/home-assistant/operating-system/releases/download/6.6/haos_ova-6.6.ova`

> Extract the ova file using tar:

> `tar -xvf haos_ova-6.6.ova`

_2022-03-08_

Mostly taken from https://www.truenas.com/community/threads/home-assistant-vm-on-scale.91058/post-666766 and cleaned up.

Make sure to use a location on your data pool as a working directory, don't use any system directory. I made a `hass` folder on my data pool `plex-media`:

`cd /mnt/plex-nas/plex-media/hass`

Use wget to get the ova file:

`wget https://github.com/home-assistant/operating-system/releases/download/9.3/haos_ova-9.3.ova`

Extract the ova file using tar:

`tar -xvf haos_ova-9.3.ova`

Convert the vmdk to a raw image file, I had to use the full working directory for the source:

`qemu-img convert -f vmdk -O raw /mnt/plex-nas/plex-media/hass/home-assistant.vmdk hassos.img`

Create a Zvol using the TrueNas Scale GUI - Be sure to make it large enough for dd to complete, I used 35 Gib.

Use dd to write the image file to your zvol

`dd if=hassos.img of=/dev/plex-nas/home-assistant`

Create a virtual machine using the gui and attach the zvol you just created as the hdd.

Minimum recommended assignments:

- 2GB RAM
- 32GB Storage
- 2vCPU

All these can be extended if your usage calls for more resources.
