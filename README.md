# TimeMachineAutomount
Automatically mount and unmount an external drive before running a time machine backup

## Background ##

### Issue ###

* I have a Macbook pro connected to a thunderbolt dock and I hate unmounting my time machine drive every time I want to disconnect my computer to take it somewhere else.

* At the same time I don't want to unplug and plug in my drive everytime I want to do a backup.

More backups would also be nice instead of only when I decide to plug in my drive.

### Bad Solutions ###

#### Network Drives ####

* These are expensive and I already have a hard drive.

#### Put my harddrive on the network ####

* This is probably the ideal scenario however I had trouble with this.   I don't have a computer at my disposal to constantly have on connecting my drive the network.

* I tried using a raspberry pi to make a network drive but that was confusing and turned out unsucessful.

## Solution ##

### 1. Prevent the drive from mounting ###

Incase the script does not run inbetween the time of plugging in and disconnecting, simply never mount in the first place.  This will allow safe disconnection whenever you please (and are not currently backing up). [Source](https://apple.stackexchange.com/a/310578/228373)

For this example I will refer to the hard drive name as `DRIVENAME`.

1. Run `diskutil info "DRIVENAME"`

2. Find the line labaled `Volume Name`.
    
    This should be the same as `DRIVENAME`.  

    If not refer to Volume Name as `DRIVENAME`

3. Add the following line to `/etc/fstab`:

    ```bash
    LABEL=DRIVENAME none hfs rw,noauto
    ```

    Replace `hfs`  with the data under `Type (Bundle):` from `diskutil info`.

    Note: You can refer to the drive by the UUID instead of label if that is prefered.  Refer [here](https://apple.stackexchange.com/a/310578/228373) for the code.

### 2. Create a Script to mount and backup ###

The [source](https://somethinginteractive.com/blog/2013/07/24/time-machine-auto-mountunmount-drive-os-x/) uses 2 files to achieve what I thought was just as easy to achieve in a single file.  

1. create a plane text file named `timemachine_backup.sh` with the following contents:

    ```bash
    #!/bin/bash
    # name volume
    vol_mount="DRIVENAME"
	# check if disk is mounted
	# &> /dev/null redirects output so that it isn't printed
    if ! mount | grep "$vol_mount" &> /dev/null ; then
		# disk is not - attempt to mount
        if  diskutil mount "$vol_mount" | grep "Unable to find disk for" &> /dev/null ; then
			# unable to mount - most likely because disk is not plugged in
			# Give up
            echo "unable to mount"
            exit 1
        fi
    fi

    echo "mounted"
	# start backup
    tmutil startbackup -b
	# unmount disk for safe removal
    diskutil unmount "$vol_mount"
    ```

2. Change the file permissions to allow for execution.  Run `chmod 755 timemachine_backup.sh`.

    This allows user to read write and execute and everyone else read and execute.  

    You can change the permissions, all that matters is that everyone can execute.  Who cares if everyone can execute, all they're doing is backing up your computer for you.

3. Save the file to `/usr/local/bin`.

4. create a plain text file named `com.apple.TimeMachine_mount.plist` with the following contents:

    ```XML
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>StartCalendarInterval</key>
        <array>
            <dict>
                <key>Minute</key>
                <integer>0</integer>
            </dict>
        </array>
        <key>Label</key>
        <string>com.apple.TimeMachine_mount</string>
        <key>RunAtLoad</key>
        <true/>
        <key>Program</key>
        <string>/usr/local/bin/timemachine_backup.sh</string>
    </dict>
    </plist>
    ```

5. Save the file in `~/Library/LaunchAgents/`.

6. turn off auto backup in system prefs.

    Not Sure how necessary this is, but we are creating automatic backups so just turn it off.

7. Run the following terminal command:

```bash
launchctl load ~/Library/LaunchAgents/com.apple.TimeMachine_mount.plist
```

### 3. Update options ###

* The current code will backup every hour on the hour.   

* You can specify a specific time of day to update or on an interval ignoring the time.  [Here](https://killtheyak.com/schedule-jobs-launchd/) is a reference for editing when the script is run.  You can also google for more options.

### 4. Trouble shooting ###

1. Try running `./timemachine_backup.sh` in whatever folder it is located in (Hopefully `/usr/local/bin` 

    If this doesn't work the launch agent never will 

2. Change the 0 in

    ```XML
    <key>Minute</key>
    <integer>0</integer>
    ```

    to the current next minute.  This will run the script within a minute instead of waiting until th next hour.
  
    Note: you might have to run `launchctl unload ~/Library/LaunchAgents/com.apple.TimeMachine_mount.plist` then reload the file after updates.  That is what I did and I haven't tested to see if it works without reloading

## Functionality ##

* The script should run on the hour (or whenever you set it to) and on boot (from `<key>RunAtLoad</key><true/>`).

* If the hardrive is unconnected or unable to be accessed the script will quietly fail in the background

* Note: The drive is mounted during the time that the backup is taking place, so you cannot safely unplug at any time.  However most of the time it should be perfectly safe to simply unplug your drive.

## All references ##

* [Main Source](https://somethinginteractive.com/blog/2013/07/24/time-machine-auto-mountunmount-drive-os-x/)

* [Prevent a drive from mounting](https://apple.stackexchange.com/a/310578/228373)

* [Prevent a drive from mounting 2](https://discussions.apple.com/docs/DOC-7942)

* [Change When to run script](https://killtheyak.com/schedule-jobs-launchd/)
