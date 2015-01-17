---
layout: post
title: "Execute shell script on connecting to home network to sync data to NAS"
date: 2014-11-14 09:25:33 +0100
comments: true
categories: 
- Linux
- Ubuntu
- Rsync
- Shell Script
---

# Idea

I was looking for a way to synchronize my notebook data to the NAS each time i connect to the home network.
The network manager in ubuntu offers a way to execute scripts that are located in /etc/NetworkManager/dispatcher.d/. 
The script receives two arguments, the first being the interface name of the device just activated, and second an action. 
You can find more detail info about NetworkManager here: http://manpages.ubuntu.com/manpages/utopic/man8/NetworkManager.8.html .

Thinking about the implementation of the script i want it to perform the following tasks: 

1. Determine SSID and verify if it matches my home network
2. Mount the shared folder from NAS
3. Sync home folder to a mounted NAS folder with rsync.
3. Write the progress of the synchronization into a log file. 

# Scripts

As i played with the initial implementation i found out that it is necessary to separate the rsync command from the NetworkManagers dispatcher script. Thats because the rsync can actualy take long time to finish the job and the NetworkManager has a timeout of 3 seconds that applies to each dispatcher script it runs. If the script exceeds the 3 seconds, the NetworkManager tries to kill the script. 

Here is the dispachter script

``` bash wlanIsOnHomeNetwork : look for the home network and execute sync script 

#!/bin/bash

INTERFACE=$1
STATUS=$2

DESIRED_INTERFACE="wlan0"
HOME_NETWORK="WeCanDoIt"
SYNC_SCRIPT="/home/teknik/.scripts/syncHomeFolderToNas.sh"

if [ "$INTERFACE" = "$DESIRED_INTERFACE" ]; then
    #  get the iwconfig info for the interface and extract the ssid name 
    CURRENT_NETWORK="$(iwconfig $DESIRED_INTERFACE | grep -o 'SSID:".*"' | sed 's/.*"\(.*\)"/\1/' | tr -d ' ')"
    if [ "$HOME_NETWORK" = "$CURRENT_NETWORK" ]; then
        case "$STATUS" in
            up)
                logger -s "## Home network up event detected. Triggering sync script ..."
                nohup $SYNC_SCRIPT &
            ;;
            down)
                # nothing to execute
            ;;
            pre-up)
                # nothing to execute
            ;;
            post-down)
                # nothing to execute
            ;;
            *)
            ;;
        esac
    fi
fi

exit 0

```
Notice line 17 where the dispatcher script executes the sync script as a background process.

Following values need to be set:
``` bash wlanIsOnHomeNetwork : variables
DESIRED_INTERFACE="wlan0"                                       # the wifi interface
HOME_NETWORK="WeCanDoIt"                                        # the name of the home network
SYNC_SCRIPT="/home/teknik/.scripts/syncHomeFolderToNas.sh"      # script that does the syncing

```
And here is the second part, the script that takes care of the synchronization of my home directory. 

``` bash syncHomeFolderToNas.sh : syncs data to NAS 

#!/bin/bash

SOURCE_DIRECTORY=/home/teknik/
TARGET_MOUNT_POINT=/mnt/exile

TARGET_DIRECTORY=$TARGET_MOUNT_POINT/4-notebook/
EXCLUDE_FOLDERS_FILE=/home/teknik/.rsync/.excludeFolders.txt
LOG_FOLDER=/home/teknik/.rsync
LOG_FILE_PREFIX=syncLog_$(date +"%d-%m-%y_%H:%M:%S")
LOG_FILE=$LOG_FOLDER/$LOG_FILE_PREFIX.log

logger -s "## starting sync job!"
logger -s "## mount NAS shares to local FS!"
mount -a
# check if mount point ready
if grep -qs $TARGET_MOUNT_POINT /proc/mounts ; then
    logger -s "## syncing files!"
    # sync and write output to log file
    rsync -rlptDzv --no-o --no-g --progress --delete --log-file=$LOG_FILE --exclude=$LOG_FILE --exclude-from=$EXCLUDE_FOLDERS_FILE $SOURCE_DIRECTORY $TARGET_DIRECTORY 
    logger -s "## successfully synced data!"
    # zenity --info --text 'Successfully synced data!'
else
    logger -s "## mount point $TARGET_MOUNT_POINT not ready !"
fi

```
In line 19 you can see the rsync command with its options. Each sync attempt is storred in a log file. The list of excluded directories and files is stored in a file (line 7). In line 21 i tried to create a dialog window  that informs me after the syncing was succesffully completed. But unfortunately since the job runs as root i was not able to display the window under the actual user context.

``` bash syncHomeFolderToNas.sh : variables
SOURCE_DIRECTORY=/home/teknik/                                  # the source directory that needs to be backed up
TARGET_DIRECTORY=/mnt/exile/4-notebook/                         # the target directory where the backup is copied
EXCLUDE_FOLDERS_FILE=/home/teknik/.rsync/excludeFolders.txt     #list of folders that will be ignored during backup
LOG_FOLDER=/home/teknik/.rsync                                  # folder for log files
```

# Location & Permissions

Now all we need to do is to put the dispatcher script into the directory /etc/NetworkManager/dispatcher.d/, change ownership to root and make it executable. I called it wlanIsOnHomeNetwork.


``` console wlanIsOnHomeNetwork : Change permissions and make executable 
sudo chown root:root /etc/NetworkManager/dispatcher.d/wlanIsOnHomeNetwork
sudo chmod +x /etc/NetworkManager/dispatcher.d/wlanIsOnHomeNetwork
```

# Links

http://www.techytalk.info/start-script-on-network-manager-successful-connection/
https://wiki.ubuntu.com/OnNetworkConnectionRunScript