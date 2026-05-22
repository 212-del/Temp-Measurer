Okay so here is the making process of the temp measure to be able to measure we start from very basic first we need to know what is actually measuring the temp i mean who is mesruing the temp

In kali for this

We need to do
```bash
xfce4-panel --preferences
```

if you have set the temp by below method 

- Right click on taskbar 
- Tap on Add new items you will be able to see it after hovering it over panel.
- Once after it you will see a window opne you will then select sensor pluglin.


If you have set the temp by this method then the program that is currectly measuring you temp is 

```text
xfce4-sensors-plugin
```

As i went to reach the source code of the program i was trying to understand the program

the program is https://gitlab.xfce.org/panel-plugins/xfce4-sensors-plugin 

To understand it we will use the method 1 that is in the repo of GitNexus put github.dev instead of github.com this will giveyou a vscode window in  your current brower tab you can explore now with more ease and faster.

To understand we are going to use a github project named https://github.com/abhigyanpatwari/GitNexus

which will help us to understand the code better. after do understanding it.

We got the below lesson after deep diving into the progaram

The Xfce4 Sensors Plugin is only a frontend/UI reader.

It does NOT directly communicate with the CPU hardware.

Instead, it reads values exposed by:

/sys/class/hwmon/
libsensors
lm-sensors

which ultimately come from the kernel driver (coretemp).

To check your kernal driver you could run

```bash
for hw in /sys/class/hwmon/hwmon*; do
    echo "=== $hw ==="
    cat $hw/name
done
```

you could see 2 values

- coretemp > for intel processor
- k10temp  > For AMD   Processor

Now we need to find the source code of coretemp

For this we started going deep and deep about the coretemp driver.

if you look at the ouput of the script 

```bash
for hw in /sys/class/hwmon/hwmon*; do
    echo "=== $hw ==="
    cat $hw/name
done
```

you could  be able to locate path something like mine that is 

/sys/class/hwmon/hwmon0

After seeing this path you should be now able to know that this is exact hwmon directory.

hwmon is a shortform stands for hardware monitor

As per your real hwmon directory you have to now find where the actual path is.

The command is 'readlink -f /sys/class/hwmon/hwmon0/device'

it is used to find the real physical path behind a symbolic link. 

As per my case it gave me the result '/sys/devices/platform/coretemp.0'

The kernel device itself is named:

coretemp.0

That tells you:

the hwmon sensor belongs to the coretemp platform driver.

by doing the command 'modinfo coretemp'

I was being able to get the file of kernal driver that is at the location

'/lib/modules/6.18.12+kali-amd64/kernel/drivers/hwmon/coretemp.ko.xz'

In my case.
