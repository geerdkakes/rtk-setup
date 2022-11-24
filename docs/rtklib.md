# using rtklib

RTKlib can be used to retrieve RTK correction data via ntrip.

RTKLib can be installed via the commandline using the following commands:
```
cd /usr/local/src
sudo wget "https://github.com/tomojitakasu/RTKLIB/archive/rtklib_2.4.3.zip"
cd /usr/local/src/RTKLIB-rtklib_2.4.3/lib/iers/gcc
sudo make
cd /usr/local/src/RTKLIB-rtklib_2.4.3/app/consapp
sudo make
sudo make install
```

