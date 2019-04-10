# Cross compiling Qt 5.11 for RPi3 on macOS 10.14 Mojave or macOS 10.13 High Sierra


This guide will show you how to prepare a development environment for Qt with cross compiling for RPi3 on macOS Mojave or High Sierra.

### Prerequisites
- macOS High Sierra 10.13.6 or Mojave 10.14.4 installed
- Xcode 8 or above
- Raspberry Pi 3 with Raspbian stretch, kernel version ```4.14.79-v7+ #1159``` or later and SSH access enabled.

## Step 1 - Prepare the toolchain

1. **[on host]**  Create a working folder where you will place everything you need (example `/Users/me/raspi`) and enter on it.

2. **[on host]**  Create a DMG file that will contains the artifacts, so open the DiskUtility app and create an empty image named **xtool-build-env** with at least of 10GB disk size and select *AFPS file system with case sensitive* as format. Place this file in your working folder and mount it.

Also, should you make a disk that is not the correct size, you can invoke the resize command to fix it:

```
hdiutil resize -size <size>G ~/Desktop/<filename>.dmg
```

3. **[on host]**  Next step is to prepare the toolchain you use to make cross compiling. Download the built artifacts from this link: <https://github.com/yc2986/armv8-rpi3-linux-gnueabihf-gcc-8.1.0-macos-high-sierra> in your working folder. These artifacts works for both macOS High Sierra and Mojave versions.

4. **[on host]**  Copy the root folder ```armv8-rpi3-linux-gnueabihf``` from the toolchain into the DMG image you created.

At this point you will have ready a toolchain with the next tools:

- GNU make 4.2.1
- GNU Compiler Collection (*gcc*) 8.1.0
- GNU C Library (*glibc*) 2.27
- GNU Binutils 2.28
- GNU m4 1.4.18
- GNU Debugger (*gdb*) 8.1
- GNU Build system: autoconf 2.69, automake 1.15, libtool 2.4.6

**IMPORTANT**: In the RPi you must have the same package version for *glibc* that the toolchain have (v2.27). Otherwise you will not able to run your code generated. To check the version number you have installed in the RPi for this package run:

```
apt-cache policy libc6
```
Whether the version installed is prior to the required by the toolchain then you have to update that package by enabling the ***testing*** repositories for raspbian stretch and configure the stable branch by default. The package required is *Libc6 2.27* and it lives in the ***testing*** branch.

To update *libc6*:

1. **[on RPi]** Open */etc/apt/sources.list* and add the next at the end of the file:

	```
	deb http://raspbian.raspberrypi.org/raspbian/ testing main contrib non-free rpi
	deb-src http://raspbian.raspberrypi.org/raspbian/ testing main contrib non-free rpi
	```
2. **[on RPi]** Create or modify the file */etc/apt/preferences* by adding:
	
	```
	Package: *
	Pin: release a=stretch
	Pin-Priority: 900

	Package: *
	Pin: release a=testing
	Pin-Priority: 50
	```
	This will ensure you get the stable packages by default always, but bringing you access to the testing repositories.
	
3. **[on RPi]** Run ```sudo apt-get install -t testing libc6``` to install the required package from the ***testing*** repo with the dependencies needed.
	At this point you will be prompted about to restart some services you have installed. Proceed to restart them.
	
4. **[on RPi]** After restart the required services, proceed to restart your RPi: ```sudo reboot```

At this point you will have installed the required *libc6* library and you'll be able to run your generated code from the toolchain you will provide to Qt.
## Step 2 - Configure Qt for cross compiling

1. **[on RPi]** (optional) Run ```raspi-config``` in your Raspberry Pi, change it to boot to the console instead of X, change the GPU memory to 256 MB.

	```
	sudo raspi-config
	```

2. **[on RPi]** For Raspbian Stretch you need also to update your RPi.

	```
	sudo rpi-update
	reboot
	```
	
3. **[on RPi]** Install a bunch of development files (for simplicity we use ```build-dep```, not everything is really needed, but it is easier this way).

	3.1. Edit sources list in ```/etc/apt/sources.list``` and uncomment the *deb-src* line:

		```
    	sudo nano /etc/apt/sources.list
    	```

    3.2. Update your system and install required libraries:

		```
	    sudo apt-get update
	    sudo apt-get build-dep qt4-x11
   		sudo apt-get build-dep libqt5gui5
    	sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0
    	```

4. **[on RPi]** Prepare our target directory:

	```
	sudo mkdir /usr/local/qt5pi
	sudo chown pi:pi /usr/local/qt5pi
	```
	
5. **[on host]** Enter in your working directory and get a toolchain:

	```
	cd ~/raspi
	git clone https://github.com/raspberrypi/tools
	```
6. **[on RPi]** Download and compile *openssl v1.0.2l* (For some incompatibility reason Qt sources cannot compile with *openssl v1.1.0* linked):
	
	```
	cd ~
	wget https://www.openssl.org/source/openssl-1.0.2l.tar.gz
	tar xzf openssl-1.0.2l.tar.gz
	cd openssl-1.0.2l
	./Configure os/compiler:arm-linux-gnueabihf
	make CC="arm-linux-gnueabihf-gcc" AR="arm-linux-gnueabihf-ar r" RANLIB="arm-linux-gnueabihf-ranlib"
	make install
	```
7. **[on host]** Create a sysroot. Using rsync we can properly keep things synchronized in the future as well. Replace raspberrypi.local with the IP address of the Pi and set the port number you have configured:

	```
	mkdir sysroot sysroot/usr sysroot/opt
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/lib sysroot
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/usr/include sysroot/usr
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/usr/lib sysroot/usr
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/opt/vc sysroot/opt
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/usr/local/ssl/lib sysroot/usr
	rsync -avz -e "ssh -p 22" pi@raspberrypi.local:/usr/local/ssl/include sysroot/usr
	```

8. **[on host]** Adjust symlinks to be relative. Use provided script, because the old *fixQualifiedLibraryPaths* is not working properly:

	```
	cd ~/raspi
	wget https://raw.githubusercontent.com/riscv/riscv-poky/priv-1.10/scripts/sysroot-relativelinks.py
	chmod +x sysroot-relativelinks.py
	./sysroot-relativelinks.py /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot
	```
9. **[on host]** Copy openssl libraries and headers to the DMG file:

	```
	cp -r /Users/me/raspi/sysroot/usr/include/openssl /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/include/
	cp /Users/me/raspi/sysroot/usr/lib/libssl.a /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/lib/
	cp /Users/me/raspi/sysroot/usr/lib/libcrypto.a /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/lib/
	cp /Users/me/raspi/sysroot/usr/lib/pkgconfig/* /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/lib/pkgconfig/
	```
10. **[on host]** Get qtbase and configure Qt.
	
	```
	cd ~
	git clone git://code.qt.io/qt/qtbase.git -b 5.11
	cd qtbase
	./configure -release -no-opengl -openssl-linked OPENSSL_LIBS="-I /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/include -L /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot/usr/lib -lssl -lcrypto" -device linux-rasp-pi3-g++ -device-option CROSS_COMPILE=/Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/bin/armv8-rpi3-linux-gnueabihf- -sysroot /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot -opensource -confirm-license -make libs -prefix /usr/local/qt5pi -extprefix /Users/me/raspi/qt5pi -hostprefix /Users/me/raspi/qt5 -v -no-use-gold-linker
	make
	make install
	```
	If you failed, you can clear everything with:
	
		git clean -dfx
		
11. **[on host]** Deploy Qt to the device. We simply sync everything from ```~/raspi/qt5pi``` to the prefix we configured above.
		
	```
	cd ~/raspi
	rsync -avz -e "ssh -p 22" qt5pi pi@raspberrypi.local:/usr/local
	```
	
12. **[on RPi]** Update the device to let the linker find the Qt libs:
	
	```
	echo /usr/local/qt5pi/lib | sudo tee /etc/ld.so.conf.d/qt5pi.conf
	sudo ldconfig
	```
13. **[on RPi]** Install the gdbserver:
	
	```
	sudo apt-get install gdbserver
	```
	
14. **[on host]** Install gdb:

	```
	brew install gdb
	```

15. **[on host]** Build other Qt modules as desired, the steps are always the same (you need to adjust ***qt-module***):

	```
	cd ~/raspi/qtbase
	git clone git://code.qt.io/qt/<qt-module>.git -b 5.11
	cd <qt-module>

	~/raspi/qt5/bin/qmake
	make
	make install
	```
	
	Then go to the folder you have your ***qt5pi*** dir and deploy new these files by running:
	
	```
	cd ~/raspi
	rsync -avz -e "ssh -p 22" qt5pi pi@raspberrypi.local:/usr/local
		
At this point you have a Qt development toolchain ready to use for cross compiling to RPi3.

## Step 3 - Setup Qt Creator for cross compiling

Once Qt is on the device, Qt Creator can be set up to build, deploy, run and debug Qt apps directly on the device with one click. 

1. **[on host]** Add the RPi device:
	Go to **Preferences --> Devices** and select the tab named 'Devices', press on ```Add``` button and:

	* Select *Generic Linux Device*.
	* Enter the IP address for your RPi, port to connect, user & password.
   	* Press on 'Finish' button.
	
	Here you can also use advanced options to connect to your RPi.
	
2. **[on host]** Add the compilers:

	Go to **Preferences --> Kits** and select the tab named 'Compilers', press on ```Add``` button and:
   
   * Select **GCC --> C++** and add as copiler path: ```/Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/bin/armv8-rpi3-linux-gnueabihf-g++```
    	
   * Select **GCC --> C** and add as compiler path: ```/Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/bin/armv8-rpi3-linux-gnueabihf-gcc```

3. **[on host]** Add the debugger:
 
	Go to **Preferences --> Kits** and select the tab named 'Debuggers', press on ```Add```button and:
  		
	* Add as debugger path: ```/usr/local/Cellar/gdb/8.2.1/bin/gdb```

4. **[on host]** Add the custom Qt version built:

	Go to **Preferences --> Kits** and select the tab named 'Qt Versions', then check if an entry with the path ```~/raspi/qt5/bin/qmake``` shows up. If not, add it.


5. **[on host]** Add a new kit:

	Go to **Preferences --> Kits** and select the tab named 'Kits', press on ```Add```button and fill the next fields:

	* **Name:** A custom name to identify the kit.
	* **Device type:** *Generic Linux Device*
	* **Device:** The one we just created.
   * **Sysroot:** ```/Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/armv8-rpi3-linux-gnueabihf/sysroot```
	* **Compiler:** The one we just created.
   * **Debugger:** The one we just created.
   * **Qt version:** The one we saw under Qt Versions.
   * **Qt mkspec:** Leave empty.

**IMPORTANT:** You need to have mounted the DMG image file **BEFORE** to launch the QTCreator. What you can do is to automount the DMG file when you log in into the system by going to **System Preferences --> Users & Groups --> Login Items.** Press "+" button to add a new login item and choose the DMG image file. Logged out then in and you'll get your DMG image file mounted. 

## Step 4 - Configure a new empty project in Qt Creator for RPi3

To create an empty project that compile and runs on your RPi follow the next steps:

1. Create a new project by selecting *Console Application* type.
2. Select the kit you created above.
3. Open the .PRO file and modify the *target.path* variable from ```/opt``` to ```/home/pi``` or another desired. The reason for this is because the ```/opt``` folder only have root access, and you would need to run your binary as root user).
4. Code and Run :)

At this point you should be able to create a new project, using your new kit to build and deploy to the RPi. 

## Step 5 (Optional) - Configure Boost libraries to be used in Qt

To install Boost on your host machine to make cross compiling to your RPi follow the next steps:

1. Download **Boost** stable from the main page (Currently the last stable version is *v1.69.0*)
2. extract the downloaded file into a desired folder: ```tar xjf boost_1_69_0.gz```
3. Enter in the directory: ```cd boost_1_69_0```
4. Create the bootstrap: ```./bootstrap.sh --prefix=/Users/me/raspi/boost```
5. Create the ```user-config.jam``` in the user directory: ```nano ~/user-config.jam```
6. add the next in the file: ```using gcc : arm : /Volumes/xtool-build-env/armv8-rpi3-linux-gnueabihf/bin/armv8-rpi3-linux-gnueabihf-g++ ;```
7. Compile boost: ``` ./b2 --enable-unicode=ucs4 --no-samples --no-tests toolset=gcc-arm link=static cxxflags=-fPIC install```
8. Delete the user config file: ```rm ~/user-config.jam```
9. The following directory should be added to compiler include paths: ```/Users/me/raspi/boost/include```
10. The following directory should be added to linker library paths: ```/Users/me/raspi/boost/lib```

Use ```./b2 clean``` to start a new build.
