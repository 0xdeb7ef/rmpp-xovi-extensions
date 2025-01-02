# Disclaimers and general info

I am NOT responsible for any damages that you may incur to your device using my software. If you end up permanently bricking your device, or damaging the screen, you cannot expect me to fix it. I have written these extensions for myself, and I'm sharing them in hopes that someone might also find them useful.

You are installing this at your own risk.

XOVI is a tethered extension manager - after a restart, the tablet will get back to the stock OS. To get XOVI back, simply run `xovi/start` from the SSH prompt. Your tablet will restart, and you will get XOVI back.
There is additionally now a permanent way to install xovi, if this is preferred by the user. Scroll down to find the instructions.

## To install XOVI:
- Download [this](https://github.com/asivery/xovi/releases/latest/download/xovi.so) file
- You need the `screen` binary on your device
- Over SSH, create the following directory structure:
```
/home/root
    |
    \->xovi
       |
       |->extensions.d
       |
       |->exthome
       |
       \->xovi.so (copy the file here)
```

## Temporary Setup
This will setup the xovi loader in a temporary fashion. After a reboot or shutdown & poweron xovi will not get loaded.

**This setup means you as the user have to start xovi manually per SSH.**
- Run the following commands (copy and paste into SSH shell):
```
echo "#!/bin/bash" > /home/root/xovi/_start
echo "#!/bin/bash" > /home/root/xovi/start
echo "systemctl stop xochitl; killall xochitl 2>/dev/null; bash -c '/home/root/xovi/_start'" >> /home/root/xovi/start
echo "QML_DISABLE_DISK_CACHE=1 LD_PRELOAD=/home/root/xovi/xovi.so xochitl" >> /home/root/xovi/_start
chmod a+x /home/root/xovi/{_start,start}
```

### To install extensions:
- Before installing an extension, read its readme to make sure you also installed all of its dependencies. An extension without dependencies will either fail to load, or crash on startup.
- Download an extension's `.so` file (in case of this repository, download the `extensions.zip` file, then unpack it to get all the currently available xovi extensions)
- Copy the `.so` file to `/home/root/xovi/extensions.d/` on the tablet
- Run `xovi/start` to restart xovi

### To update hashtab (required for UI mods)
If your device updates, or you're installing qt-resource-rebuilder for the first time, be sure to update the hashtab file. To do that, issue the following commands:
```
mkdir -p /home/root/xovi/exthome/qt-resource-rebuilder
rm -f /home/root/xovi/exthome/qt-resource-rebuilder/hashtab
killall xochitl
QMLDIFF_HASHTAB_CREATE=/home/root/xovi/exthome/qt-resource-rebuilder/hashtab xovi/_start
```
After that, you should start seeing logs scroll up your screen. Enter your passcode on the rM, then watch the logs. When you see the line:
```
[qmldiff]: Hashtab saved to /home/root/xovi/exthome/qt-resource-rebuilder/hashtab
```
scroll up your screen on the PC, press CTRL+C.
After that, start xovi normally (`xovi/start`)

## Permanent Setup
This will setup the xovi loader in a permanent way. xovi will get automatically loaded on poweron or reboot of the remarkable.

** This will cleanup the temporary setup for not needed parts.**

Run the following commands (copy and paste into SSH shell):

1. remount rootfs writeable  
`mount -o remount,rw /`
2. unmount (recursive) the overlayfs from /etc to make the changes permanent  
`mount -R /etc`
3. create systemd override folder  
`mkdir -p /etc/systemd/system/xochitl.service.d`
4. create systemd override file  
```
cat <<"EOF" > /etc/systemd/system/xochitl.service.d/override.conf
[Unit]
After=data.mount home.mount

[Service]
Environment="QML_DISABLE_DISK_CACHE=1"
Environment="LD_PRELOAD=/home/root/xovi/xovi.so"
EOF
```
The `After=` part is important so the remarkable GUI process starts only **after** the encrypted home mount was successfully mounted. I had to factory-reset my remarkable because the xochitl started trying to preload from the `/home/root/xovi` folder when the mount was not ready yet.

5. reload systemd  
`systemctl daemon-reload`
6. start xochitl.service  
`systemctl restart xochitl.service`

This will override the `xochitl` service to load `xovi.so` on startup.

The stdout log of `xochitl` can be found in `journalctl -e -u xochitl.service`.

### more Troubleshooting (optional, only of needed)
For more troubleshooting you can create the two scripts from the temporary setup:
```
echo "#!/bin/bash" > /home/root/xovi/_start
echo "#!/bin/bash" > /home/root/xovi/start
echo "systemctl stop xochitl; killall xochitl 2>/dev/null; bash -c '/home/root/xovi/_start' &" >> /home/root/xovi/start
echo "QML_DISABLE_DISK_CACHE=1 LD_PRELOAD=/home/root/xovi/xovi.so xochitl" >> /home/root/xovi/_start
chmod a+x /home/root/xovi/{_start,start}
```

### To install extensions:
- Before installing an extension, read its readme to make sure you also installed all of its dependencies. An extension without dependencies will either fail to load, or crash on startup.
- Download an extension's `.so` file (in case of this repository, download the `extensions.zip` file, then unpack it to get all the currently available xovi extensions)
- Copy the `.so` file to `/home/root/xovi/extensions.d/` on the tablet
- Run `systemctl restart xochitl` to restart xovi

### To update hashtab (required for UI mods - better script for the permanent setup)
If your device updates, or you're installing qt-resource-rebuilder for the first time, be sure to update the hashtab file. To do that, issue the following commands:
```
cat <<"EOF" > /home/root/xovi/update-hashtab
#!/bin/bash

# stop systemwide gui process
systemctl stop xochitl.service

if pidof xochitl; then
  kill -15 $(pidof xochitl)
fi

# remove the actual hashtable
rm -f /home/root/xovi/exthome/qt-resource-rebuilder/hashtab

# start process
echo -e "#################################"
echo -e "Now we will run the GUI process with the arguments to build a new hashtable."
echo -e "\n\t[...]"
echo -e "\t[qmldiff] [Hashtab Rule Processor]: Hashed derived '/ark/icons/stroke_medium'"
echo -e "\t[qmldiff]: Hashtab saved to /home/root/xovi/exthome/qt-resource-rebuilder/hashtab"
echo -e "\nWhen we find this line in the output the process is stopped and the systemd service is started again."
read -p "Please press enter to continue:"
echo -e "\n\nOutput:"
sleep 5

QMLDIFF_HASHTAB_CREATE=/home/root/xovi/exthome/qt-resource-rebuilder/hashtab QML_DISABLE_DISK_CACHE=1 LD_PRELOAD=/home/root/xovi/xovi.so /usr/bin/xochitl 2>&1 | while IFS= read line; do
  echo -e "$line"
  if [[ "$line" == "[qmldiff]: Hashtab saved to /home/root/xovi/exthome/qt-resource-rebuilder/hashtab" ]]; then
    # yep, found the line we are wating for. exiting the process.
    echo -e "\n##############"
    echo -e "Found expected output. Killing gui process and restarting systemd service."
    kill -15 $(pidof xochitl)
  fi
done

# so, we seem to be finished.
# wait another 5 seconds, then run the normal gui process via systemd
sleep 7
systemctl start xochitl.service
EOF
```
This script will 
- stop the remarkable GUI process running in systemd,
- make sure the process is stopped
- remove the old hashtab file
- run the gui process interactive with the arguments to build new hashtab.  
  Errors like these one right at the start are ok here.
```
Output:
[qmldiff]: Iterating over directory /home/root/xovi/exthome/qt-resource-rebuilder/
[qmldiff]: Failed to load hashtab: No such file or directory (os error 2)
[qmldiff]: Loading file Edit.qmd
[qmldiff]: Failed to load file Edit.qmd: Error while parsing: expected String or identifier, got Keyword(Traverse)    <<<<---- HERE. Ignore this one.
[qmldiff]: Loading file simplified-3.15-3.16.qmd
[qmldiff]: Failed to load file simplified-3.15-3.16.qmd: Error while parsing: expected String or identifier, got Keyword(Import)    <<<<---- HERE. Ignore this one.
[qmldiff]: Configured hashtab rules.
[qmldiff]: Hashtab saver started!
[qmldiff]: Was asked to process the first slot. Sealing slots, entering postinit...
[qmldiff]: Hashing: /src/xofm/bundles/papertablet.json
[...]
```
- Search in each line written to stdout or stderr for the string `[qmldiff]: Hashtab saved to /home/root/xovi/exthome/qt-resource-rebuilder/hashtab`.
- If this string is found, SIGINT the gui process.
- wait 5 seconds
- start the gui process via the normal systemd service unit.
