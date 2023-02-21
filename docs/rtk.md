# configuring RTK


setup used is a singleboard computer connected to a U-Blox ZED-F9P. In our case we used a raspberry with an extra port enabled via USB to Serial. This way only the ttl level UART's were used of the U-Blox ZED-F9p


## install ports

Attach the two serial ports `UART1` and `UART2` to the raspberry. Attach `UART1` to `serial0` and `UART2` to a ttl<->usb converter. In our setup this corresponds to `ttyUSB0`.


## configure ublox with native u-blox client

Configure the f9p to use:

- UART1 for GPSD
- UART2 for RTCMv3 messages

### PRT (PORT)
UART1 set to:
```
protocol in:     none
protocol out:    1 - NMEA
Baudrate:        38400
```

UART2 set to:
```
protocol in:      0+1+5 UBX+NMEA+RTCM3
protocol out:     0+1 UBX+NMEA
Baudrate:         115200
```

### MSG (Messages)

02-13 RXM-SFRBX -> UART2 & USB
02-15 RXM-RAWX  -> UART2 & USB


## configure gpsd

Change `/etc/default/gpsd` to use the `serial0` port.

```
# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/serial0"

# Other options you want to pass to gpsd
GPSD_OPTIONS=""

# Automatically hot add/remove USB GPS devices via gpsdctl
USBAUTO="false"
```

Configure gpsd to always set the correct port speed. For this we change the systemd service file.
Add (or edit) `/etc/systemd/system/gpsd.service` to the following content:
```
[Unit]
Description=GPS (Global Positioning System) Daemon
Requires=gpsd.socket
# Needed with chrony SOCK refclock
After=chronyd.service

[Service]
Type=forking
EnvironmentFile=-/etc/default/gpsd
ExecStartPre=/usr/bin/stty -F /dev/serial0 38400
ExecStart=/usr/sbin/gpsd $GPSD_OPTIONS $OPTIONS $DEVICES

[Install]
WantedBy=multi-user.target
Also=gpsd.socket
```

Reload the service file
```
sudo systemctl daemon-reload
```

Restart gpsd:
```
sudo systemctl restart gpsd
```

## install and run str2str from RTKLib

Download code from github:

```
git clone https://github.com/rtklibexplorer/RTKLIB.git
cd RTKLIB/app/consapp/str2str/gcc
make
sudo cp str2str /usr/local/bin/
```

Try out to see if the program works. You should get similar output:

```
sudo str2str -in ntrip://kpniot01:5739@62.41.137.66:2101/06GPSVRSGLO31 -b 1 -out serial://ttyUSB0
stream server start
2023/02/21 12:43:30 [WC---]          0 B       0 bps (0) connecting... (1) /dev/ttyUSB0
2023/02/21 12:43:35 [CC---]       1356 B    2389 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:43:40 [CC---]       2865 B    2389 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:43:45 [CC---]       4649 B    2913 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:43:50 [CC---]       6158 B    2387 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:43:55 [CC---]       7811 B    2381 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:00 [CC---]       9320 B    2388 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:05 [CC---]      10973 B    2392 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:10 [CC---]      12482 B    2384 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:15 [CC---]      14266 B    2916 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:20 [CC---]      15775 B    2383 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:25 [CC---]      17428 B    2388 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:30 [CC---]      18937 B    2390 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
2023/02/21 12:44:35 [CC---]      20590 B    2392 bps (0) 62.41.137.66/06GPSVRSGLO31 (1) /dev/ttyUSB0
...
```

When verified, cancel this program and make sure it is started at startup. For this we create the service file:
`/etc/systemd/system/rtkclient`

with the following content:
```
[Unit]
Description=RTK client program
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/default/rtkclient
ExecStartPre=/usr/bin/stty -F /dev/ttyUSB0 115200
ExecStart=/usr/local/bin/str2str -in ntrip://${USERNAME}:${PASSWD}@${SERVER}:${PORT}/${STREAM} -b 1 -out serial://${DEVICE}

[Install]
WantedBy=multi-user.target
```

Set defaults for rtkclient and write them to: `/etc/default/rtkclient`
```
# Set variables for RTK Client
# these are used by /etc/systemd/system/rtkclient.service

# username of the rtk service
USERNAME="username"

# password of the rtk service
PASSWD="passwd"

# stream mount point.
STREAM="06GPSVRSGLO31"

# Address of the RTK service
SERVER="62.41.137.66"

# Port to use:
PORT="2101"

# device to read location and write RTCM message to
DEVICE="ttyUSB0"
```



Reload the service files:
```
sudo systemctl daemon-reload
```

