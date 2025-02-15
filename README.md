# TeslaRouter
Wi-Fi Router for Tesla vehicles based on Raspberry Pi Zero 2W and OpenWRT. Especially useful for vehicles imported outside of home country.

This project will utilize a Raspberry Pi Zero 2W, Waveshare USB/ethernet hub, LTE Modem, and USB Bluetooth adapter to create a wifi router that will provide an internet connection for a Tesla vehicle. The device can be powered from a 12-9V to 5v voltage converter connected to the constant 12-9v found in the rearview mirror pod to make sure the connection is always available. In addition, the device can run MQTT2TeslaBLE to enable remote waking of the vehicle.

# Requirements:
1) Raspberry Pi Zero 2W: https://www.amazon.de/Raspberry-Pi-Zero-2-W
2) Waveshare Eth/USB hub box for Pi 2W: https://www.amazon.de/Waveshare-Ethernet-Raspberry-Compatible-Boards/dp/B09K5DLR17
3) Linux compatible USB-Bluetooth LE adapter supporting BT>4.0: https://www.amazon.de/Bluetooth-Kopfh%C3%B6rer-Controller-unterst%C3%BCtzt-Windows11
4) 12-9v to 5v step-down power supply: https://www.amazon.de/APKLVSR-DC-DC-Converter-Power-Supply
5) USB LTE modem with MBIM or QMI and linux capatibility. Recommend: https://www.m2mgermany.de/shop/produkt/alcatel-linkkey-ik41ve-lte-eu
6) 12v automotive wire of sufficient guage, T-tap wire connectors, usb A to micro cable, command velcro mounting strips
7) MicroSD card of at least 32GB size and reader
8) An MQTT server connected to your Tailnet (preferably one running Home Assistant)

# Assemble hardware

Attach the usb/eth hub to the Pi using the provided screws and risers. Place in case and secure with screws. Insert the USB BLE adapter into one of the USB ports and put the LTE Modem into one of the other ports. Insert an activated SIM card into the slot on the USB modem.

# Flash SD Card

Download Raspberry Pi Imager to your PC/Mac. Insert the MicroSD Card into the card reader and insert the reader into an available USB port on your mac/PC

On your Mac/PC go to the OpenWRT firmware selector. Choose the following options:

      -Device: "Raspberry Pi A/A+/B/B+/CM/Zero/ZeroW"
      -Release: "24.10.0 RCX"      
*NOTE: older releases do not have the kernel drivers needed to properly support the USB BLE Adapter

click the dropdown next to "Customize installed packages and/or first boot script"

Add to the end of Installed Packages: 

      tailscale kmod-usb-serial-qualcomm picocom kmod-usb-serial kmod-usb-net kmod-usb-serial-wwan kmod-usb-serial-option kmod-usb-net-qmi-wwan kmod-usb-net-cdc-mbim modemmanager luci-proto-modemmanager kmod-bluetooth bluez-libs bluez-utils kmod-usb-core kmod-usb-uhci kmod-usb2 usbutils nano cfdisk

Click the cog in the bottom right of the UCI-defaults box and uncomment/edit the following:
       
       wlan_name="Tesla Wifi"
       wlan_password="changeme"
       root_password="changeme"
       lan_ip_address="192.168.100.1"

 Click "Request Build" and wait

 When build is complete, click the button next to "Factory (SQUASHFS)" and download to your PC/Mac

 Open Raspberry Pi Imager. Select Pi Zero W for the machine. For image choose "custom" and select the file you downloaded. Select the USB device with your MicroSD card and then flash. If prompted to apply customizations say "no" (they aren't compatible with OpenWRT).

# Configure Router

 When flash is complete, remove the card from the reader and place into the slot on your Pi. Connect the Pi to power and wait for it to boot up. Once you see a new wifi network named "Tesla Wifi" it should be ready. Connect your Mac/PC to this network. Open a Terminal (Linux/MacOS) or Putty (Windows) and connect to 192.168.100.1 with the user "root". Enter the password from above (default "changeme").

     ssh root@192.168.100.1

Once logged in via SSH we will need to expand the filesystem on the Pi (from https://oriolrius.cat/2022/08/04/resize-squasfs-ext4-partition-of-openwrt-in-a-raspberry-pi/)
   
     cfdisk /dev/mmcblk0
     # change partition size using the UI
     opkg install losetup resize2fs
     BOOT="$(sed -n -e "/\s\/boot\s.*$/{s///p;q}" /etc/mtab)"
     DISK="${BOOT%%[0-9]*}"
     PART="$((${BOOT##*[^0-9]}+1))"
     ROOT="${DISK}0p${PART}"
     LOOP="$(losetup -f)"
     losetup ${LOOP} ${ROOT}
     fsck.ext4 -y ${LOOP}
     resize2fs ${LOOP}
     reboot

Now that we have more space available, we can install the rest of the packages we will need:

    opkg install dockerd docker docker-compose luci-app-dockerman

Now lets set up the USB Modem. Assuming the modem is already in MBIM or QMI Mode we just need to open a web browser and point to 192.168.100.1, enter password "changeme" (or whatever put in above) and do the following:
      1) go to Network > Interfaces
      2) click "Add new interface"
      3) for name enter "wwan0" and for protocol select "ModemManager" and click "Create Interface"
      4) on the new interface and set the apn, pin, authentication type, and assign to the wan zone. clikc "save" and then "save and apply"
      5) if everything worked your new modem interface should be up and your router will now be online

Now lets install Tailscale so we can remotely access the router and so services on the router (like MQTT) can communicate with our tailnet. Go back to your SSH session and enter the following:

    tailscale up
    
Follow the instructions to connect the device to your tailnet

# Install Tesla BLE MQTT

Follow the instructions at this link to install Tesla BLE MQTT. For the server name I recommend using the Tailscale IP address of the server vs the DNS name as this will work even if there are DNS issues (tailscale IPs are static). 

      https://github.com/tesla-local-control/tesla_ble_mqtt_docker/blob/main/INSTALL.md

# Install the router in your vehicle

Now that our Router is ready and configured we will want to install it permanently in our vehicle. To do this we will need to tap the constant 9-12v power provided by the rearview mirror pod and run this to our 5v adapter which will in turn power our router even when the car is asleep and all other vehicle electronics are asleep. This is important so that the car always has an internet connection when it wakes (booting any router/hotspot would likely take too long and the vehicle will disable wifi if a drive starts before it is connected) but also so that we can always access the vehicle via the routers bluetooth in order to send commands and remotely wake the vehicle.

This is not an overly detailed description so I recommend checking out this YouTube video for greater detail on accessing the 12-9V connection: https://www.youtube.com/watch?v=QuSF3I0LpqE

1) Pull down the dome light console and find the 12V+ and 12V- wires
2) Attach a T-tap to each wire
3) Run wires from the taps through the hole to the headliner (follow the path of the existing wires) and pull out on the other side of the headliner against the windshield
4) Connect 12V+ and 12V- to the input on the voltage converter
5) Connect a USB A to micro cable to the USB port on the voltage converter
6) Wrap the voltage converter in electrical tape to prevent any parts from shorting out and stuff the assembly into the gap between the headliner and the front windshield/roof
7) Connect the USB micro end of the cable to the power port on the router
8) Attach a velcro command strip to the bottom of the router. Mount the router at the top of the windshield on the passenger side as close to the mirror pod as possible

With the router installed in your vehicle, connect your Tesla to the wifi network "Tesla Wifi" and set the option to "remain connected in drive". This will make sure the wifi is always connected and ready whenever the car is awake and while driving.

# Optional: Set up an automation on iPhone to wake the vehicle when you open the Tesla App

1)  make sure you have the Home Assistant configured and working on your phone
2)  In the Home Assistant app, create an action that will press the "wake" button from the MQTT integration
3)  In the shortcuts app on the iPhone create a new shortcut that will activate this button
4)  In the shortcuts app create an automation that will run the shortcut any time the Tesla app is opened

This automation will now wake your Tesla any time you open the Tesla app. If your car was asleep it will wake and be ready for commands within a few seconds of opening the App. If the App shows your car as asleep just wait a few seconds and pull down to refresh.




