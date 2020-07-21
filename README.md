# FCREPLAY
This is sort of a mad scientist project to record and upload [fightcade](https://www.fightcade.com/) replays to archive.org: [Gino Lisignoli - Archive.org](https://archive.org/search.php?query=creator%3A%22Gino+Lisignoli%22)

## How it works
Using some python (and a janky bash) scripts I scrape the fightcade replay urls from the site. Store them in a database with the relevant information.

A replay is then selected and the capture script is started. Fightcade is started with wine, and then OBS is started to record the match. After the match is finished (based on the length of the match), OBS is killed the python script takes over again.

The script then takes over and:
 1. Renames the file to the fightcade id
 1. Runs ffmpeg to correct it since killing OBS might break the file
 1. Runs ffmpeg again to generate a thumbnail
 1. Tries to determine which characters players are using OpenCV
 1. Uploads the file to archive.org with the relevant metadata
 1. Uploads the file to youtube.com with the relevant metadata 
 2. Removes the generated files.

## Character Detection
OpenCV is used to analise the video and match character names from the health bars. It does this by template matching the included character name images against the video every 60 frames. Depending on your OBS recording settings you might need to regenerate the images.

### A few more notes:
To trigger OBS I am capturing the screen, looking for a the split windows, then checking for differences in the screen capture. This might seem crude, but without memory inspection it doesn't seem like there is a way to tell when the emulator has started playing the replay.

Because fightcade doesn't have the ability to record to a video file you need to use some sort of capture software.

For this project I use [Open Broadcaster Software](https://obsproject.com/). While it doesn't have a lot of scripting ability out of the box, it seemed to work fairly well. I did initially try to just use ffmpeg and capture x11 with audio, but for some reason it would always drop to 20fps or less.

I use the i3 window manager to ensure that the ggpofba-ng window is always in the same place, and have preconfigured OBS to record that area.

This is all done in a headless X11 session

## Todo
 - Remove a bunch of jank. There is a lot of late night coding just to make things work.
 - Better exception handling.
 - Support for games other than 3rd strike.
 - Better installation.
 - Find something that might be more lightweight than OBS.
 - Possibly find a way to do multiple recordings at once.
 - Thumbnails are kinda broken

## Goal
The goal of this was to make fightcade replays accessible to anyone to watch without the need of using an emulator.

## Requirements
To run this yourself you need:
 1. A VM or physical machine.
     1. With at least 4 Cores (Fast ones would be ideal)
     1. With at least 2GB Ram
     1. With at least 20GB of storage (Though you probably won't use that much)
 1. Running Fedora 32 (you can probably make this work in other distributions as well)
     1. You really want to use a minimal installation
 1. Some familiarity with linux will help

There is also a bunch of code included to run this in a google cloud project. More documentation to come.

## Configuration
The defaults should work if you follow the guide below.

If you are using the ansible playbook, you will also want to have ready the following files:
 - Configuration file: `config.json`
 - Appened description: `description_append.txt`
 - Google cloud storage credentials: `.stroage_creds.json`
 - Google api client secrets: `.client_secrets.json`
 - Youtube upload credentials: `.youtube-upload-credentials.json`
 - Archive.org secrets: `.ia`

### Uploading to archive.org
To upload files to archive.org, set the configuration key `upload_to_ia` to `true` and configure the ia section in the configuration file. You will also need to have your `.ia` secrets file in your users home directory. This can be generated by running `ia configure` from the command line once you have setup the python virtual environment.

# Installation
I've include a basic ansible playbook for the installation, you will need to have ssh access and a deployment user with root access.

```
ansible-playbook -i <host>, -u <deployment_user> --diff playbook.yml
```

Login and switch to the fcrecorder user, then create a x11vnc password as the fcrecorder user (It will be stored in ~/.vnc/passwd):
```commandline
x11vnc -storepasswd 
```

Now you need to start the dummy X server, configure fightcade and configure OBS.
As the fcrecorder user:
```commandline
# Run tmux, and split so you have two panes
# In the first pane, run startx
startx
# In the second pane, run x11vnc
x11vnc --rfbauth ~/.vnc/passwd -noxfixes -noxdamage -noxrecord
```

Start the pulseaudio server
```commandline
pulseaudio --start
```

Check that it's working by run pavucontrol
```commandline
pavuconrol
```

In the i3 session, run ggpofba-ng:
```commandline
cd ~/fcreplay/pyqtggpo-master
wine ggpofba-ng.exe
```

And configure it with:
 1. Select blitter: Enhanced.
 1. Blitter options, mark these:
     1. Enable Pre-scale
     1. Pre-scale using SoftFX
     1. RGB effects
     1. Advanced settings: Force 16-bit emulation & Use DirectX texture management
 1. Stretch: Correct aspect ratio

Close ggpofba, and run OBS:
```commandline
obs
```

Configure OBS for your systems performance. You want to change the video output directory to `~/fcreplay/videos` with a static filename (set in the advanced section) as `replay`. Make sure to disable recording of the mouse cursor as well

Now you need to configure the scene. I found the best way to do this was to have ggpofba running, then switch OBS to a i3 floating window (win+shift+space).

 * I'm using a canvas resolution of 512x384 and a output resolutuon of 512.384.
 * For the screen capture I'm using the following crop/pad filter settings:
   * Left: 514
   * Top: 204
   * Right: 2
   * Bottom: 184

# Running automatically on startup
To run fcreplay automatically on startup you need to enable the service, and uncommet the i3 line:
```commandline
systemctl enable fcrecord
sed -i 's/^# exec "xterm/exec "xterm/' .config/i3/config
```

# Running only once
After doing all this you can start recording. Activate the python environment with:
```commandline
cd ~/fcreplay
source ./venv/bin/activate
```

Now grab some replays:
```commandline
fcreplayget <fightcade profile>
```

This will download quite a few replays, and place them in a sqlite3 database (replays.db).
Now you can trigger the recording loop, Run:
```commandline
fcreplayloop
````

Once a loop has started you can close the vnc session and leave it running.

You can also run `fcreplayloop --debug` to only run for a single iteration. Useful for testing.

# Manual install
The manual steps of the anisble script are:

Install the packages:
```
dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
dnf install \
        alsa-utils \
        firewalld \
        gcc \
        git \
        i3 \
        jq \
        libpq-devel \
        obs-studio \
        pavucontrol \
        portaudio \
        portaudio-devel \
        pulseaudio \
        pulseaudio-libs \
        python3 \
        python3-devel \
        rsync \
        tmux \
        unzip \
        vim \
        wine.i686
        x11vnc \
        xinit \
        xorg-x11-drv-dummy \
        xterm \
```

Enable firewalld:
```commandline
systemctl enable firewalld
systemctl start firewalld
```

Add a firewalld rule to allow x11vnc:
```commandline
firewall-cmd --zone=public --permanent --add-port=5900/tcp
firewall-cmd --zone=public --add-port=5900/tcp
```

Add the fcrecorder user:
```commandline
useradd -m -s /usr/bin/bash fcrecorder
```

Add user to the pulse, pulse-access and input groups:
```commandline
usermod -a -G pulse,pulse-access,input fcrecorder
```

Switch to the fcrecorder user and install the package in a virtual env:
```commandline
su fcrecorder -
git clone https://github.com/glisignoli/fcreplay.git ~/fcreplay_install
mkdir ~/fcreplay
cd ~/fcreplay
python3 -m venv ./venv
source ./venv/bin/activate
cd ~/fcreplay_install
python3 setup.py install
```

Install youtube-upload
```
cd ~/fcreplay
source ./venv/bin/activate
pip install --upgrade google-api-python-client oauth2client progressbar2
cd ~/
git clone https://github.com/tokland/youtube-upload.git youtube-upload_install
cd youtube-upload_install
python setup.py install
```

Edit the config file and replace the defaults
```commandline
vi ~/fcreplay/config.json
```

Download pyqtggpo-0.42 and unzip it as the fcrecorder user:
```commandline
cd ~/fcreplay
git clone https://github.com/poliva/pyqtggpo.git pyqtggpo-master
```

Copy the xorg.conf and Xwrapper.conf file to /etc/X11 as the root user.
These files are installed in the python package installation directory
```commandline
cp /home/fcrecorder/fcreplay/venv/lib/python3.8/site-packages/fcreplay-*-py*.egg/fcreplay/data/{Xwrapper.config,xorg.conf} /etc/X11/
```

Create the default .xinitrc file
```commandline
cat << EOF > ~/.xinitrc
exec i3
EOF
```

Using a vnc client, connect to your server. Once you have authenticated you will be prompted to generate the default i3
config. Hit win+return to spawn a terminal. 

Add the following line to the fcrecorder users i3 config:
```
# Add the following to ~/.config/i3/config
exec --no-startup-id "/usr/bin/xterm"
```