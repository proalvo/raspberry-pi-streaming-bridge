# raspberry-pi-streaming-bridge

With streaming bridge you can recieve a video steam from remote location and you can use the stream as a part of your own production.

With this streaming bridge you can receive RTMP stream e.g. from OBS and convert it to HDMI interface provided by Raspberry Pi 4. You have also option to stream to Youtube and Facebook, and record all streams. You need to modify rtmp.conf to set up your service for Youtube and Facebook, but rtmp.conf is ready to use if you stream to your HDMI output.

Steps
1. Install Raspberry OS
2. Configure HDMI interface
3. Install and configure nginx
4. Install and configure stunne4 for the Facebook streaming
5. Automatically log in as 'pi' user. This applies only for local session. If you use SSH to log in to your Raspberry then you have to input user and password.

# To do

- Raspberry Pi 4 has two HDMI output and stream is outputted to the first HDMI port. HDMI cable should be attached before you boot up the Raspberry. Check if you can force output always to the HDMI port even if cable is not connected during the boot.
- Add instructions for recording - streaming can be recorded also.

# Install Raspberry Pi

https://www.raspberrypi.org/software/

- change default password
- activate WiFi (better to use wired connection for live streaming but wifi is ok to configure/install the system)
- set up your language/keyboard layout
- activate SSH to access from other computers
- boot to CLI (command line interface, not GUI)

# Configure HDMI

```
sudo nano /boot/config.txt
```
Edit the following (uncomment following lines i.e. remove #)
```
disable_overscan=1
hdmi_group=1
hdmi_mode=16
hdmi_drive=2
```
Options for the HDMI are available at https://www.raspberrypi.org/documentation/computers/configuration.html#hdmi-configuration
`hdmi_mode=16` is 1080p 60Hz.

# Install and configure nginx

Install the nginx
```
sudo apt-get update
sudo apt-get install nginx libnginx-mod-rtmp
```

Update permissions:
```
sudo usermod -aG video www-data
```
Create a new file which is used to configure rtmp
```
sudo nano /etc/nginx/rtmp.conf
```
Copy the following to the rtmp.conf:
```
# Usage:
#   server: rtmp://<ip address>/<youtube|facebook|hdmi>
#   key: whatever is accepted, key is used in filename in case recording is activated 
#
rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        
# Youtube configuration is commented out because the nginx would fail on boot due to incorrect key 
#       application youtube {
#         live on;
#         push rtmp://a.rtmp.youtube.com/live2/YOUR-YOUTUBE-STREAMING-KEY;
#         record all;
#         recorder all {
#           record all;
#           record_path /home/pi/Videos;
#           record_unique on;
#           record_suffix _%d%m%Y_%H%M%S.flv;
#         }
#       }
        application facebook {
          live on;
          push rtmp://127.0.0.1:1936/rtmp/YOUR-FACEBOOK-STREAMING-KEY;
          record off;
        }
        application hdmi {
          live on;
          record off;   
          allow play 127.0.0.1;
          deny play all;
          exec omxplayer -o hdmi rtmp://127.0.0.1/hdmi/$name
        }
  }
}
```
Then we tell the nginx configuration about the RTMP file by modifying the “nginx.conf” file
```
sudo nano /etc/nginx/nginx.conf
```
Add the following line. The location in the file is not important:
```
include /etc/nginx/rtmp.conf;
```
Test the format of the configuration
```
sudo nginx -t 
```
If the format is ok, then you can start the nginx
```
sudo nginx -t start
```
or reload the configuration withour interruption of the nginx server
```
sudo nginx -s reload 
```

## Install and configure Stunnel

Install stunnel to enable streaming to Facebook. The Facebook uses SRTMP which is not supported bu nginx, so we use Stunnel to add 'S' to 'RTMP'.

Install the Stunnel
```
sudo apt-get install stunnel4
```

Configure the Stunnel
```
sudo nano /etc/default/stunnel4
```

Add line (to be honest, I am sure if this is needed).
```
ENABLED=1
```

Create new configuration file 
```
sudo nano /etc/stunnel/stunnel.conf
```

Add the following lines
```
pid = /var/run/stunnel4/stunnel.pid
output = /var/log/stunnel4/stunnel.log

setuid = stunnel4
setgid = stunnel4

# https://www.stunnel.org/faq.html
socket = r:TCP_NODELAY=1
socket = l:TCP_NODELAY=1

debug = 4

[facebook-live]
client = yes
accept = 1936
connect = live-api-s.facebook.com:443
verifyChain = no
```

```
sudo systemctl start stunnel4.service
sudo systemctl status stunnel4.service
```

In case you need to stop the stunnel
```
sudo systemctl start stunnel4.service
```

## OBS configuration

Use the following settings in OBS to stream to your streaming bridge.


Select File > Settings > Stream > 

Service: Custom
Server: rtmp://< IP-ADDRESS-OF-YOUR-RASPBERRY-PI >:1935/<facebook|youtube|hdmi>
Streamkey: <anykey>

Output: 
  Output mode: Advanced
        Streaming: 
            Bitrate: 6000 Kbps
            Keyframe interval: 2

Following can be used to find out the IP address of your Raspberry Pi
```
hostname -I
```
    
## Fine tuning

Start up texts are visible when you boot the Raspberry Pi. Good thing is you see everything is working, but texts are visible on HDMI output also. This may be unwanted behaviour. Also, if you stop streaming the latest screen before streaming is shown. Simple solution to make screen black is following. First you set prompt color to black, and clear screen. Input the following. Note that there is one annoying result: if you need to write something, you cannot see it because the text is black on black background. 

```
PS1="\e[0;30m"
clear
```
If you want to make this setting permanent, you can change the default prompt by editing '.bashrc' file which is used to configure your environment when you log in.
    
## Troubleshooting

See log files for the nginx
```
sudo tail /var/log/nginx/error.log -n 200
sudo tail /var/log/nginx/access.log -n 200
``` 
    
 
## Resources
Streaming bridge  https://www.youtube.com/watch?v=MtETl23cnOA
Options for the HDMI are available at https://www.raspberrypi.org/documentation/computers/configuration.html#hdmi-configuration

