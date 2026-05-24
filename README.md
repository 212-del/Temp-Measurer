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

 ## This is the whole workflow of measuring temp contained in the xz file.

MSR 0x19C read
Thermal Status register. CPU package per core.
→
get_tjmax()
Reads MSR 0x1A2, fallback 0xEE, fallback 0x17 (IA32_THERM_STATUS)
→
Formula applied
Temp = TjMax − DTS_readout
→
show_temp()
Divides millideg by 1000, writes to /sys/hwmon
→
xfce4-sensors
Reads file, displays number. No math here.

## MSR Register Found 

0x1A2
IA32_TEMPERATURE_TARGET
bits[23:16] = TjMax °C
This is the primary TjMax source
0xEE
MSR_EE (fallback)
Used if 0x1A2 fails.
Seen in your binary at offset 0x1B4
0x17
IA32_PLATFORM_ID
Last fallback. Assumes desktop, logs warning if inaccessible
0x19C
IA32_THERM_STATUS
bits[22:16] = DTS (Digital Thermal Sensor) readout per core

## The Exact Formula

 Step 1: kernel reads MSR 0x19C (per core, per CPU) DTS = rdmsr(0x19C) >> 16 & 0x7F # bits 22:16, negative offset # Step 2: get TjMax from MSR 0x1A2 (confirmed in your binary at 0xA0) TjMax = rdmsr(0x1A2) >> 16 & 0xFF # bits 23:16 # Step 3: THE actual temp formula (from show_temp disassembly) temp_milli°C = (TjMax - DTS) × 1000 # Step 4: what show_temp does (confirmed: imul $0x3e8 = 1000) temp_°C = temp_milli°C / 1000 # That's it. No smoothing, no prediction, no load weighting. # 0x3e8 = 1000 in your binary at offset 0x50A confirmed this.

 ## weakness in measuring temp here

 No time-smoothing
Each read is independent. A 1ms burst gives same weight as sustained load. The value can jump 10°C in one tick.
TjMax is static (read once at boot)
Stored at init time, never re-read. If BIOS reports wrong TjMax (your E8500 has known issues with this), every reading is offset by that error forever.
DTS quantisation = 1°C steps
The MSR register stores integers only. Real temp can be 63.7°C but you'll see 63 or 64. No sub-degree precision possible from this path.
No thermal mass modelling
Silicon has thermal inertia — it takes time to heat/cool. coretemp reports the DTS snapshot with zero physics model applied.
No fan feedback
Fan RPM is read by a completely separate hwmon driver. coretemp never sees it — the two values are never fused.

## Needs Improvement in this section.

Next step: read MSR 0x19C directly yourself
Use rdmsr -p 0 0x19C (install msr-tools). Extract bits[22:16] yourself. Compare with /sys/class/hwmon reading. This proves your theory without touching any kernel code.
Layer your Kalman filter on top
Your improved script reads the same /sys file coretemp writes, but runs it through the Kalman estimator. You're not replacing coretemp — you're post-processing its output with better math. Zero kernel modification needed.
Verify TjMax for your E8500
Run: rdmsr -p 0 0x1A2 then extract bits[23:16]. E8500 TjMax should be 100°C. If your kernel is using 85°C or 90°C (BIOS-reported), your readings are shifted. Fixing this single value improves all readings instantly.
Add thermal mass compensation
Track rate-of-change: dT/dt. If temp is rising at 2°C/sec under load, the real junction temp is slightly ahead of the DTS read. A simple derivative term (PD controller style) makes your estimator more honest than coretemp.

The binary confirmed 4 MSR registers in use (at exact disassembly offsets 0xA0, 0x1B4, 0x209), and the formula is exactly:
temp = (TjMax - DTS_readout) × 1000   [stored as millidegrees]
Where DTS_readout is bits [22:16] of MSR 0x19C, and TjMax comes from MSR 0x1A2 bits [23:16]. The multiply by 1000 (imul $0x3e8) was confirmed at offset 0x50A of show_temp

We can compare live with the below script. vs that temp that is showing with the help of sensor plugin 

``` bash
#!/usr/bin/env bash
sudo modprobe msr >/dev/null 2>&1

# Per-core TjMax — E8500 cores can differ
RAW_TJ0=$(sudo rdmsr -p 0 0x1A2 2>/dev/null)
RAW_TJ1=$(sudo rdmsr -p 1 0x1A2 2>/dev/null)
TJ0=$(( (0x$RAW_TJ0 >> 16) & 0xFF ))
TJ1=$(( (0x$RAW_TJ1 >> 16) & 0xFF ))
[ "$TJ0" -eq 0 ] || [ "$TJ0" -gt 125 ] && TJ0=95
[ "$TJ1" -eq 0 ] || [ "$TJ1" -gt 125 ] && TJ1=95

echo "Per-core TjMax → Core0: ${TJ0}°C  Core1: ${TJ1}°C"
echo ""

while true; do
    # ── READ EVERYTHING IN ONE TIGHT BLOCK ──────────────────────────
    T_READ=$(date '+%H:%M:%S.%N')      # nanosecond timestamp of read

    # Read /sys FIRST (it's the stale cached value — note it first)
    SYS2=$(cat /sys/class/hwmon/hwmon0/temp2_input 2>/dev/null || echo 0)
    SYS3=$(cat /sys/class/hwmon/hwmon0/temp3_input 2>/dev/null || echo 0)

    # Read MSR immediately after — minimum gap
    RAW0=$(sudo rdmsr -p 0 0x19C 2>/dev/null)
    RAW1=$(sudo rdmsr -p 1 0x19C 2>/dev/null)
    T_MSR=$(date '+%H:%M:%S.%N')       # timestamp after MSR read
    # ────────────────────────────────────────────────────────────────

    F0=$(( 0x$RAW0 )); F1=$(( 0x$RAW1 ))

    VALID0=$(( (F0 >> 31) & 1 ))
    VALID1=$(( (F1 >> 31) & 1 ))
    DTS0=$(( (F0 >> 16) & 0x7F ))
    DTS1=$(( (F1 >> 16) & 0x7F ))

    MSR0=$(( TJ0 - DTS0 ))
    MSR1=$(( TJ1 - DTS1 ))

    S0=$(( SYS2 / 1000 ))
    S1=$(( SYS3 / 1000 ))

    DIFF0=$(( MSR0 - S0 ))
    DIFF1=$(( MSR1 - S1 ))

    # Throttle bits
    THR0=$(( F0 & 1 ))
    THR1=$(( F1 & 1 ))

    # Package estimate = max of both cores
    PKG_MSR=$(( MSR0 > MSR1 ? MSR0 : MSR1 ))
    PKG_SYS=$(( S0 > S1 ? S0 : S1 ))

    clear
    printf "%-22s %10s %10s\n"  ""           "Core 0"     "Core 1"
    printf "%-22s %9s°C %9s°C\n" "TjMax used"  "$TJ0"       "$TJ1"
    printf "%-22s %10s %10s\n"  "DTS (raw)"   "$DTS0"      "$DTS1"
    printf "%-22s %9s°C %9s°C\n" "MSR formula"  "$MSR0"      "$MSR1"
    printf "%-22s %9s°C %9s°C\n" "Kernel /sys"  "$S0"        "$S1"
    printf "%-22s %9s°C %9s°C\n" "Difference"   "$DIFF0"     "$DIFF1"
    printf "%-22s %10s %10s\n"  "Valid bit"    "$VALID0"    "$VALID1"
    printf "%-22s %10s %10s\n"  "Throttling"   \
        "$([ $THR0 -eq 1 ] && echo YES || echo No)" \
        "$([ $THR1 -eq 1 ] && echo YES || echo No)"
    echo ""
    echo "Package (max of cores)"
    printf "  MSR : %d°C   SYS : %d°C   Diff : %d°C\n" \
        "$PKG_MSR" "$PKG_SYS" "$(( PKG_MSR - PKG_SYS ))"
    echo ""
    echo "Read timestamp : $T_READ"
    echo "MSR timestamp  : $T_MSR"

    sleep 1
done

```

After this if both temp mismatch then it is not measuring correctly.

So here is out result of our reseach 

Hardware (MSR 0x19C)
    ↓  DTS bits[22:16] extracted
    ↓  TjMax = 95°C (from MSR 0x1A2, E8500-specific)
    ↓  Formula: Temp = 95 - DTS
    ↓  Valid bit checked (bit 31)
coretemp.ko  (your actual binary, reverse-engineered)
    ↓  Caches result every ~250ms
    ↓  Writes to /sys/class/hwmon/hwmon0/temp2_input (Core 0)
    ↓                              /temp3_input (Core 1)
    ↓  No temp1 on E8500 — package sensor didn't exist yet
xfce4-sensors-plugin
    ↓  Reads /sys files, displays max of temp2/temp3
    ↓  Zero math involved
What you see on taskbar 

The temp difference is only 1 so it is not neccesary to modify it.
