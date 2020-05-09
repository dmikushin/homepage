---
layout: post
title: "Setting up Jitsi Meet"
tags:
- Building
- Software Engineering
- Jitsi
- Web Conferencing
thumbnail_path: blog/2020-05-07-setting-up-jitsi-meet/jitsi.jpg
pytorch_url: "https://github.com/jitsi/jitsi-meet"
pytorch_fork_url: "https://github.com/dmikushin/jitsi-meet"
---

Web-conferencing platforms are on the raise during these unprecedented times. On the other side, the vulnerablilities of Zoom and lack of privacy motivates us to explore possible alternatives. In this post I will run through the setup of [Jitsi]({{ page.jitsi_url }}) - a Java-based technology stack for privately-held web-conferencing.

![alt text](\assets\img\blog\2020-05-07-setting-up-jitsi-meet\jitsi_window.jpg)

In a nutshell, Jitsi is similar to Google Meet or Skype. Group meetings are organized into "rooms" with audio, video and text functionality. The principal advantage of Jitsi is platform independency. One can host an own Jitsi Meet server in a private cloud or even on the local laptop with internet address gateway. That is, no Microsoft, Google or Facebook being an unexpected attendee of your family dinner. Participants connect to the private web-conference using the server URL that you can distribute with a messenger.

Being Java-based, Jitsi quickly saturates the available resources. One can reportedly expect one server to hold up to 30 participants.   

# Server setup

Setup Jitsi repository and install packages:

```sh
sudo apt install build-essential
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
sudo sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"
sudo apt-get -y update
sudo apt-get -y install jitsi-meet
```

If you are running a VPS with a limited RAM, consider enabling swap partition:

```
sudo dd if=/dev/zero of=/swapfile count=2048 bs=1M
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile   none    swap    sw    0   0' | sudo tee -a /etc/fstab
```

Install Let's Encrypt SSL certificate and restart the Nginx server:

```
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
sudo service nginx restart
```

## Server customization

Clone our fork of Jitsi Meet:

```sh
git clone https://github.com/dmikushin/jitsi-meet.git
```

Setup a recent version of Node.js:

```sh
sudo npm cache clean -f
sudo npm install -g n
sudo n stable
sudo apt remove nodejs
sudo npm install webpack-bundle-analyzer
sudo npm audit fix
sudo npm install -g webpack webpack-cli
```

Execute webpack and build the server frontend bundle:

```sh
webpack -p
make source-package
make install
```

## Allowing mobile browser sessions

By default, Jitsi expects a mobile application to be used on Android and iOS, which sometimes is not desirable. A useful workaround is to uncomment the following line in `/etc/jitsi/meet/*config.js`:

```java
disableDeepLinking: true
```

After this change (coupled with Nginx restart), mobile phone should open rooms in a web browser, just like for the desktop version (note that it will only work on Android).

# Android client

## Prerequisites

Prepare Java 8 and set it as default manually (Java 11 is incompatible with Android SDK):

```sh
sudo apt install openjdk-8-jdk
sudo update-java-alternatives --set /usr/lib/jvm/java-1.8.0-openjdk-amd64
``` 

Download the latest Android SDK:

```sh
cd /opt
sudo wget https://dl.google.com/android/repository/sdk-tools-linux-4333796.zip
sudo mkdir android-sdk-linux
sudo chown $USER android-sdk-linux
cd android-sdk-linux/
unzip ../sdk-tools-linux-4333796.zip 
cd tools
yes | bin/sdkmanager --licenses
bin/sdkmanager --update
```

Set environment variables:

```sh
sudo sh -c "echo 'export PATH=$PATH:/opt/android-sdk-linux/platform-tools' >> /etc/profile.d/android.sh"
sudo sh -c "echo 'export ANDROID_TOOLS=/opt/android-sdk-linux' >> /etc/profile.d/android.sh"
sudo sh -c "echo 'export ANDROID_HOME=/opt/android-sdk-linux' >> /etc/profile.d/android.sh"
source /etc/profile.d/android.sh
```

## Building

Before building, we need to fix react-native-calendar’s ContextCompat problems:

```
cd jitsi-meet
npm install --save-dev jetifier
npx jetify
```

Also we need to relax system's file watchers limit:

```sh
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

As an example of functional change, we alter the default server name setting:

```diff
--- a/android/app/src/main/java/org/jitsi/meet/MainActivity.java
+++ b/android/app/src/main/java/org/jitsi/meet/MainActivity.java
@@ -89,7 +89,7 @@ public class MainActivity extends JitsiMeetActivity {
         JitsiMeetConferenceOptions defaultOptions
             = new JitsiMeetConferenceOptions.Builder()
                 .setWelcomePageEnabled(true)
-                .setServerURL(buildURL("https://meet.jit.si"))
+                .setServerURL(buildURL("https://sip.your-server.org"))
                 .setFeatureFlag("call-integration.enabled", false)
                 .build();
         JitsiMeet.setDefaultConferenceOptions(defaultOptions);
```

Finally, build the Jitsi Meet Android package:

```
cd jitsi-meet/android
./gradlew
./gradlew assembleRelease
```

## Troubleshooting

The following message is an indication of insufficient system RAM capacity:

```
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
```

For the reference, here is a [Jitsi Meet fork]({{ page.jitsi_fork_url }}) that demonstrates the performed customizations.

