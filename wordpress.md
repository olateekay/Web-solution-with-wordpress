# Web Solution With WordPress

*Step 1 — Prepare a Web Server*

Launch an EC2 instance that will serve as “Web Server”. 

Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.

![alt text](image1.jpg)

![alt text](image2.jpg)

![alt text](image3.jpg)

Open up the Linux terminal to begin configuration
Use lsblk command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there - their names will likely be xvdf, xvdh, xvdg.
