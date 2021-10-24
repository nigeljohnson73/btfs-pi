# btfs-pi
I'm sure that following the instructions at the [main website homepage](https://docs.btfs.io/docs/btfs-demo) works fine for most people, but, getting the BTFS stuff up and running on the Raspberry Pi is a little tricker, and documetation is... not easy to find...

For the record, I recommend the Raspberry Pi Imager as it has cool features if you use the <ctrl> + <shift> + 'X' key combo where you can configure the WiFi and hostname so the device will boot and you don't need to scramble around looking for it on your router. If you haven't configured your Pi to boot from SSD yet, there is an option there to burn an image to SD card to sort that out as well. 

This guide assumes you have a Pi and can log into it via SSH, and have enough storage on the boot device. My configuration is as follows
  
  * Raspberry Pi 4b (2GB RAM version) configued to boot from SSD
  * 1TB Western Digital blue M2 NVMe attached and loaded with the latest Raspian OS lite
  * I configured my Pi with the hostname `bttmnr.local` so I can connect to the host

Because I'm not using the desktop version of Raspian OS, I want to use the WebUI from my main computer, so I'll need to configure that in BTFS.

Right, let's get started...

# Working out your IP address

For me, having used the Raspberry Pi Imager, I set a hostname, so, connect to the Pi with that hostname, and get the IP directly.

```language-console
MyMac $ ssh pi@bttmnr.local
...
pi@bttmnr:~ $ hostname -I
192.168.1.185
pi@bttmnr:~ $ 
```

There is the IP address given to me by my router - '`192.168.1.185`' and I'll need to remember that for later. If you have multiple entries, it will be because you have additional interfaces configured (wireless and ethernet, or a VPN etc). Choose one and stick to it. I will assume that you know what you are doing here.

The BTFS config probably had a way to serve out on the hostname, possibly by adding a line in the CORS stuff, but I have not experimented enough with that. 

# Update the Pi

If you have a super fresh install, the first thing to do was to ensure your Pi is at least as up to date as it can be.

```language-console
sudo apt update -y
sudo apt full-upgrade -y
sudo apt autoremove -y
sudo rpi-eeprom-update -a -d
```

You can reboot if you like, but it won't impact things if you don't. Next download the installer and run it.

# Download and install BTFS

Download to the temp diretory so it's removed on reboot.

```language-console
cd /tmp
wget https://raw.githubusercontent.com/TRON-US/btfs-binary-releases/master/install.sh
bash install.sh -o linux -a arm # For Linux ARM
```

Now you can set up a couple of things like the PATH variable, and your editor of choice. You should probably also add these to you `.bashrc` file as well.

```language-console
export PATH=${PATH}:${HOME}/btfs/bin
export EDITOR=vi
```
# Configure your repo.

The default storage location is `~/.btfs` and you can change this later in the GUI. You will need to know you're local external IP address from above. Let's set that up so we can use it later.

```language-console
export LOCALIP="192.168.1.185"
```

Initialise the repo, storage host and set the CORS stuff so you can access the web GUI from a separate computer. The quoting is odd for the first line, because I don't know how to do the variable expansion inside single quotes - which are specifically used to stop special character expansion.

```language-console
btfs init
btfs config profile apply storage-host
btfs config --json API.HTTPHeaders.Access-Control-Allow-Origin "[\"http://${LOCALIP}:5001\", \"http://localhost:3000\", \"http://127.0.0.1:5001\"]"
btfs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["PUT", "GET", "POST", "OPTIONS"]'
btfs config --json API.HTTPHeaders.Access-Control-Allow-Credentials '["true"]'
btfs config Addresses.API "/ip4/${LOCALIP}/tcp/5001"
btfs config Addresses.Gateway "/ip4/${LOCALIP}/tcp/8080"
btfs config Addresses.RemoteAPI "/ip4/${LOCALIP}/tcp/5101"
btfs config --json Swarm.ConnMgr.HighWater '100'
btfs config --json Swarm.ConnMgr.LowWater '50'
btfs config Routing.Type 'dhtclient'
```
<!--
Edit the config with `btfs config edit`, then change the IP addresses to the local external address the PI is on:

```language-json
"Addresses": {
	"API": "/ip4/192.168.1.185/tcp/5001",
	"Announce": [],
	"Gateway": "/ip4/192.168.1.185/tcp/8080",
	"NoAnnounce": [],
	"RemoteAPI": "/ip4/192.168.1.185/tcp/5101",
	...
```

I found my router die a lot. I may have fixed it with adjusting the way my node connects to the network - i.e. dont allow too many peers and only be a client.

Find the Swarm High and LowWater. Change them as follows

```language-json
"Swarm": {
		"AddrFilters": null,
		"ConnMgr": {
			"GracePeriod": "20s",
			"HighWater": 100,
			"LowWater": 50,
			...
```

Find the Routing Type setting. Change it as follows

```language-json
	"Routing": {
		"Type": "dhtclient"
	},
```

A daemon can be run from the command line.
-->
```language-console
btfs daemon > /tmp/btfs.log 2>&1
```

You should probably add that to your `crontab` for reboot:

```language-console
@reboot sleep 15 && /home/pi/btfs/bin/btfs daemon > /tmp/btfs.log 2>&1
```

# Connect from your main computer

You will need to continue the host configuration using the IP address: http://192.168.1.185:5001/hostui
<!--
# Possible better configurations

I have not tried these, but they would reduce some of the complexity and knowledge. They came from the bottom of [this website](https://medium.com/tron-foundation/configure-btfs-daemon-bd97f9c2e7cd) and if they work will remove the requirement to edit the config directly, and, more importantly, not know your IP address.

```language-console
btfs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["*"]'
btfs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["PUT", "GET", "POST", "OPTIONS"]'
btfs config --json API.HTTPHeaders.Access-Control-Allow-Credentials '["true"]'

btfs config Addresses.API '/ip4/0.0.0.0/tcp/5001'
btfs config Addresses.Gateway '/ip4/0.0.0.0/tcp/8080'
```

Extending the notes above, these should work as well

```language-console
btfs config Addresses.RemoteAPI '/ip4/0.0.0.0/tcp/5101'
btfs config --json Swarm.ConnMgr.HighWater '100'
btfs config --json Swarm.ConnMgr.LowWater '50'
btfs config Routing.Type 'dhtclient'
```
-->
