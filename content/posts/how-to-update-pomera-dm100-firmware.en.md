---
title: "How to Update Pomera DM100 Firmware"
date: 2017-11-22T06:13:35+07:00
categories: ["Gear"]
tags: ["Pomera DM100"]
draft: false
aliases: [/post/2017-11-22-how-to-update-pomera-dm100-firmware/]
---

Pomera DM100 is a distraction-free digital writing tool. Think mini laptop with only text editor application loaded.

It comes from Japan. It means its manual is in all Japanese. It becomes hard when you want to do the crucial procedure (like **upgrading firmware**) **AND** Google Translate doesn't help you much.

I can only follow some steps in [its updating guide web page](https://translate.google.co.id/translate?hl=en&sl=ja&u=http://www.kingjim.co.jp/support/pomera/software/dm100&prev=search) and the rest of important steps in [PDF](http://www.kingjim.co.jp/resource/media/support/pomera/update_dm100.pdf) are not translatable by Google Translate.

In the end, I have successfully updated my Pomera DM100 to the latest firmware.

Here are the steps:

<!--more-->

# Prerequisites
1. An SD Card.
	
	Make sure you have backed up your data in this SD card **AND** in the internal memory of Pomera DM100
2. An USB 2.0 A to Mini-B cable.

# Steps to Update Firmware

1. Download the latest firmware : <https://www.kingjim.co.jp/resource-files/download/pomera/dm100/DM100_V15000.zip>

2. Extract the zip file

	```
	$ unzip DM100_V15000.zip
Archive:  DM100_V15000.zip
  inflating: DM100_V15000/Pomera_Update_Dm100_V15000.bin
	```

3. Format the SD card as **MS-DOS (FAT)** and set name to **POMERA**. 
On Mac :
	1. Open **Disk Utility**
	2. In sidebar, make sure you choose your SD Card
	3. Click `Erase` on header bar menus
	4. Set `Name` to **POMERA**
	5. `Format` as **MS-DOS (FAT)**
	6. Click `Erase` button
![disk-utility](https://user-images.githubusercontent.com/55460/33103742-c3281bee-cf56-11e7-9d20-126f85a8b514.png)

4. Copy `Pomera_Update_Dm100_V15000.bin` to the SD card
	
	```
	$ cp DM100_V15000/Pomera_Update_Dm100_V15000.bin /Volumes/POMERA/
	```

4. Connect Pomera DM100 to PC using USB A to Mini-B cable
5. Update firmware on Pomera DM100
	1. Open Pomera DM100
	2. Click `Menu` button on the device
	3. `Setting` > `About DM100`
	4. Press `F1`
5. Updating process will begin. Wait until following dialog appears. (*Update is complete. Disconnect the USB cable to restart the DM100*)
![reduced-1-updated-complete](https://user-images.githubusercontent.com/55460/33152572-d9be37da-d00f-11e7-8c84-0f1e4d38a86f.jpg)
6. Disconnect the USB cable. Wait until the reboot has finished.
7. After restarted, a dialog appears. It means we have to **connect again USB cable to the PC**
![reduced-2-connect-again](https://user-images.githubusercontent.com/55460/33152517-a06595be-d00f-11e7-89cd-afe24386e61d.jpg)
9. Updating process continues.
![reduced-3-update-continues](https://user-images.githubusercontent.com/55460/33152704-9191d2b8-d010-11e7-960d-9df3b3a54153.jpg)
11. If succeeded, the main screen is displayed. We can unplug the USB cable. **DONE**

# Change Display Language

But it seems reverted back to the original setting, Japanese as display language. To set English as display language :

1.  Click `Menu` button on the device
2.  Choose top-right menu then choose `Language`. Pick `English` then click `Enter`
