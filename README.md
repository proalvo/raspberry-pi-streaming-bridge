# raspberry-pi-streaming-bridge

With streaming bridge you can receive RTMP stream e.g. from OBS and convert it to HDMI interface provided by Raspberry Pi 4. You have also option to stream to Youtube and Facebook, and record all streams. Setting up services you need to modify rtmp.conf

Steps
1. Install Raspberry OS
2. Configure HDMI interface
3. Installa and configure nginx
4. Install and configure stunne4 for the Facebook streaming

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

Install
```
sudo apt-get install stunnel4
```
Configure

```
sudo nano /etc/default/stunnel4
```
Add line 
```
ENABLED=1
```

Create new file 
```
sudo nano /etc/stunnel/stunnel.conf
```

Add lines
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
Server: rtmp://<IP-ADDRESS-OF-YOUR-RASPBERRY-PI>:1935/<facebok|youtube|hdmi>
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
## Resources
Streaming bridge  https://www.youtube.com/watch?v=MtETl23cnOA
Options for the HDMI are available at https://www.raspberrypi.org/documentation/computers/configuration.html#hdmi-configuration

