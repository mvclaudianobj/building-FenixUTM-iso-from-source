# How to: building a FenixUTM .iso from sources

Here are the steps for building a pfSense-CE ISO file. I tried to follow [the guide of PiBa-NL](https://github.com/PiBa-NL/PiBa-NL-WIKI/wiki/How-to-building-a-pfSense-.iso-from-sources) firstly, but there was missing things so I made my own guide. This guide has been written for 2.7.2, but may works for other versions

*Like for PiBa-NL guide, small disclaimer: Stuff might be missing, and stuff will change over time, and might not be updated.*

# First things first: choose a name for your product, and update repositories accordingly

Netgate doesn't allow you to build a product using the name "pfSense" (because of trademark). 
Before building a .iso file, you will have to choose a custom name for your firewall.
I'll use libreSense for this tutorial, but you could use whatever you want.


You will then have to fork 3 repositories: 
- [pfSense GUI](https://github.com/pfsense/pfsense)
- [FreeBSD Ports](https://github.com/pfsense/freebsd-ports) (Ports collection is the FreeBSD package management system, it works like `apt` or `rpm` on Linux)
- [FreeBSD Kernel Source](https://github.com/pfsense/freebsd-src)

You will also need to apply the following changes:

### FreeBSD Source
- Checkout to the branch you would like to build (`devel-main` for dev version, `RELENG_2_7_2` for stable version).
- In the folder `/release/conf/`, rename files starting with `pfSense` to `libreSense` (eg, `pfSense_install_src.conf` => `libreSense_install_src.conf`)
- Rename the file `/sys/amd64/conf/pfSense` to `/sys/amd64/conf/libreSense`
- Edit the file `/tools/tools/crypto/Makefile`: remove `cryptotest cryptostats` from the `PROGS` command

### FreeBSD Ports
- Checkout to the the branch you would like to build (`devel` for dev version, `RELENG_2_7_2` for stable version).
- Edit the file `/sysutils/pfSense-upgrade/Makefile`: remove the line `RUN_DEPENDS+= pfSense-repoc>=0:sysutils/pfSense-repoc`.
- Edit the file `/sysutils/pfSense-repo/Makefile` : change the line `MIRROR_TYPE?= srv` by `MIRROR_TYPE?= none`.
- Edit the file `/security/pfSense/pkg-plist`:  Remove all lines starting with `%%DATADIR%%/keys`.
- Edit the file `/security/pfSense/Makefile`:
  - Rename the variable `USE_GITLAB` to `USE_GITHUB`.
  - Rename the variable `GL_ACCOUNT` to `GH_ACCOUNT`. Also, change the variable content to your actual GitHub username.
  - Rename the variable `GL_COMMIT` to `GH_TAGNAME`.
  - Change the line `GL_PROJECT= ${PFSENSE_SRC_REPO}` to `GH_PROJECT= pfsense`.
  - Remove the line `GL_SITE= https://gitlab.netgate.com`.
  - Remove the line `pfSense-gnid>=0:security/pfSense-gnid \` from the `RUN_DEPENDS` variable, if it exists.

### pfSense GUI
- Checkout to the branch you would like to build (`master` for dev version, `RELENG_2_7_2` for stable version).
- Go to the folder `/tools/templates/pkg_repos/` and rename the file `pfSense-repo.conf` to `libreSense-repo.conf` (if you are building a stable ISO) or `libreSense-repo-devel.conf` (if you are building a dev version).
- Edit the file `/src/etc/inc/globals.inc` : replace the content of `product_name` by `libreSense`, and the content of `pkg_prefix` by `libreSense-pkg-`.
- In the folder `/src/usr/local/share/`, rename the folder `pfSense` to `libreSense`.
- In the folder `/src/etc/`, rename the files `pfSense-ddb.conf` and `pfSense-devd.conf` to `libreSense-ddb.conf` and `libreSense-devd.conf`.
- Edit the file `/tools/templates/core_pkg/base/pkg-plist`: Remove the line `share/%%PRODUCT_NAME%%/initial.txz` from the file.
- Edit the file `/tools/builder_common.sh`: apply the [following patches](https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/patches/builder-common.sh.diff) to the file.
- Edit the file `/tools/builder_defaults.sh`:
  - Remove`drm2` and `ndis` from the variable `MODULES_OVERRIDE_amd64`.
  - Change variable `PKG_REPO_BRANCH_RELEASE` to `v2_7_2`.
  - If you are building a stable version, change variable `PFSENSE_DEFAULT_REPO` to `"${PRODUCT_NAME}-repo"`.
 
## A deeper look into Netgate build environment

Finally, it is important to understand how we are going to build our ISO and what will be the build environment of this tutorial.

Netgate seems to be using a quite heavy and complex build environment, designed to match the goals and the strategy/direction of the company.

![netgate environment for build](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/netgate_env.png?raw=true)

Because this tutorial doesn't aim to set up an industrialized build farm:
- The build server will also act as web server for delivering ports (The build system is actively using ports repository when making an ISO).
- Signing scripts for the ports repository will be located on the build server
- Files won't be sent to an external NFS server after being built


With that said, let's setup our build server.

# Setup a proper build environment

- You will need to download and install a FreeBSD server that matches the version you want to build. pfSense CE 2.7.2 [require FreeBSD 14.0](https://docs.netgate.com/pfsense/en/latest/releases/versions-of-pfsense-and-freebsd.html) and the `devel` version [currently require FreeBSD 15.0](https://github.com/pfsense/FreeBSD-src/blob/devel-main/sys/conf/newvers.sh#L53), both in AMD64.

This server can be either a VM or a physical machine. High amount of HDD isn't needed (at least 50 Gb is recommended) but high number of CPU core is advised (otherwise build time may be very long) and high memory amount is required (>20Gb recommended. 20 Gb should be fine....16 Gb is not enough and may cause your build to crash). [You can download a FreeBSD ISO here](https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/).

This FreeBSD machine will also have to be connected to the internet and will have to be reachable by your workstations in SSH and HTTP.

During installation, **please partition your disk in ZFS**. 

![Disk partition](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/ZFS.png?raw=true)

## Make some configurations on the build server

Once the server has been installed some updates and configs need to be made. Login as root and execute following commands:

```
# Allow SSH using root, if you want it.
echo PermitRootLogin yes >> /etc/ssh/sshd_config
service sshd restart

# Required for configuring the server
pkg install -y pkg vim nano emacs

# Required for installing and building ports
pkg install -y git nginx poudriere-devel rsync sudo

# Required for building kernel and iso
pkg install -y vmdktool curl qemu-user-static gtar xmlstarlet pkgconf openssl portsnap

# Required for building iso
portsnap fetch extract

# not required but advised for building/monitoring/debugging
pkg install -y htop screen wget mmv

# Only install this if your FreeBSD is a virtual machine
pkg install -y open-vm-tools

# Create an 16G swap device (Not required but advised : will be useful if you don't have enough memory)
dd if=/dev/zero of=/root/swap.bin bs=1M count=16384
chmod 0600 /root/swap.bin
mdconfig -a -t vnode -f /root/swap.bin -u 0 
echo 'swapfile="/root/swap.bin"' >> /etc/rc.conf
swapon /dev/md0
```



Then you need to configure nginx for PKG hosting and poudriere monitoring:
```
# pfSense_gui_branch represents the branch of pfSense GUI that will be compiled, with "RELENG" replaced by "v" : master for a development ISO, v2_7_2 for a stable ISO
# pfSense_port_branch represents the branch of FreeBSD ports that will be compiled, using the same replacement ("RELENG"=>"v") : devel for a development ISO, v2_7_2 for a stable ISO
# product_name represents the name of your product.

pfSense_gui_branch=v2_7_2 # Replace with the version you want to build
pfSense_port_branch=v2_7_2 # Replace with the version you want to build
product_name=libreSense # Replace with your product name


cd /usr/local/www/nginx/
rm -rf *
mkdir -p packages
# Ports web server for core PKGs (pfSense-base, pfSense-rc, etc...)
ln -s /root/pfsense/tmp/${product_name}_${pfSense_gui_branch}_amd64-core/.latest packages/${product_name}_${pfSense_gui_branch}_amd64-core
# Ports web server for other PKGs
ln -s /usr/local/poudriere/data/packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} 
# Web server for monitoring ports build
ln -s /usr/local/poudriere/data/logs/bulk/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch}/latest poudriere

# Allow directory indexing, and configure nginx to start automatically on boot
sed -i '' 's+/usr/local/www/nginx;+/usr/local/www/nginx; autoindex on;+g' /usr/local/etc/nginx/nginx.conf
pw group mod wheel -m www
echo nginx_enable=\"YES\" >> /etc/rc.conf
service nginx restart
```

## Setup a signing key

As mentioned above, we will setup a signing key in the build server. Execute these commands to generate the signing key:
```
mkdir -p /root/sign/
cd /root/sign/
openssl genrsa -out repo.key 2048
chmod 0400 repo.key
openssl rsa -in repo.key -out repo.pub -pubout
printf "function: sha256\nfingerprint: `sha256 -q repo.pub`\n" > fingerprint
curl -o /root/sign/sign.sh https://raw.githubusercontent.com/freebsd/pkg/master/scripts/sign.sh
sed -i "" 's+ repo\.+ /root/sign/repo\.+g' /root/sign/sign.sh
chmod +x /root/sign/sign.sh
```

## Configure how pfSense will be built


Now that your server is configured, we will configure how pfSense will be compiled. Clone your fork of pfSense GUI, checkout to the branch that will be built, and configure your fork to use your signing key
```
cd /root
git clone https://github.com/{your username}/pfsense.git
cd pfsense
git checkout RELENG_2_7_2 # Replace by the branch of pfSense GUI to build.

# Ports repositories signing key
rm src/usr/local/share/${product_name}/keys/pkg/trusted/*
cp /root/sign/fingerprint src/usr/local/share/${product_name}/keys/pkg/trusted/fingerprint
```


Let's then create a file called ```build.conf``` in the folder of pfSense GUI.
```
export PRODUCT_NAME="libreSense" # Replace with your product name
export FREEBSD_REPO_BASE=https://github.com/{your username}/FreeBSD-src.git # Location of your FreeBSD sources repository
export POUDRIERE_PORTS_GIT_URL=https://github.com/{your username}/FreeBSD-ports.git # Location your FreeBSD ports repository

export FREEBSD_BRANCH=RELENG_2_7_2 # Branch of FreeBSD sources to build
export POUDRIERE_PORTS_GIT_BRANCH=RELENG_2_7_2 # Branch of FreeBSD ports to build


# Netgate support creation of staging builds (pre-dev, nonpublic version)
unset USE_PKG_REPO_STAGING # This disable staging build
# The kind of ISO that will be built (stable or development) is defined in src/etc/version in pfSense GUI repo

export DEFAULT_ARCH_LIST="amd64.amd64" # We only want to build an x64 ISO, we don't care of ARM versions

# Signing key
export PKG_REPO_SIGNING_COMMAND="/root/sign/sign.sh ${PKG_REPO_SIGN_KEY}"

# This command retrieves the IP address of the first network interface
export myIPAddress=$(ifconfig -a | grep inet | grep '.'| head -1 | cut -d ' ' -f 2)

export PKG_REPO_SERVER_DEVEL="http://${myIPAddress}/packages"
export PKG_REPO_SERVER_RELEASE="http://${myIPAddress}/packages"

export PKG_REPO_SERVER_STAGING="http://${myIPAddress}/packages" # We need to also specify this variable, because even
# if we don't build staging release some ports configuration is made for staging

export SRCCONF="/root/pfsense/tmp/FreeBSD-src/release/conf/${PRODUCT_NAME}_build_src.conf"
``` 

# Building the ISO

Now that we have a working environment, let's start the heavy work.

First, start a screen on your build server, using command ```screen -S build```. You may leave this screen using ctrl+A then D, and you may enter this screen again using command ```screen -r build```. The purpose of the screen is to keep your work running if you disconnect from the build server.

### Setup Jails
Execute the command ```./build.sh --setup-poudriere```. This will setup the environment and create the FreeBSD jail necessary for your build. You can then leave the screen and `tail -f logs/poudriere.log` to check what's going on. Expect the command to run for ~3 hours on a 16 core CPU.

If something goes wrong: 
- You may add `set -x` at the beginning of `build.sh` to debug what's happening in there.
- You can list created jails with command ```poudriere jail -l```, and you can remove created jails using `poudriere jail -d -j {jailName} -C clean`.
- You may also verify if you didn't enter a wrong URL for FreeBSD ports the `build.conf`. This would result in the command being stuck (A message `please enter username from "https://github.com":` will be displayed in the logs in such case)


### Build ports
Then execute ```./build.sh --update-pkg-repo``` to compile the ~550 FreeBSD ports of pfSense.
You may want to monitor the build environment on your server using HTTP ( http://ipOfYourServer/poudriere ). Expect the build to take around 6 hours.

![Disk partition](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/poudriere_build.png?raw=true)

In case something goes wrong: Logs files can be seen using HTTP or directly on the build server, in the folder ```/usr/local/poudriere/data/logs/bulk/```. You need to analyze the logs of each failed port to understand exactly what's the problem for each of them.

Few possible root causes: 
- The `FreeBSD-port` branch that you are trying to build does not match the FreeBSD version of your build server.
- You are trying to build an out-of-date version of pfSense and some `dist` files are not available anymore on the [official FreeBSD distcache](http://distcache.FreeBSD.org/), resulting in errors at the `fetch` step. 
   - In this case you can update the [`distinfo`](https://github.com/pfsense/FreeBSD-ports/blob/devel/print/texinfo/distinfo) of each concerned port in your GitHub fork of `FreeBSD ports`, then you can run `./build.sh --update-poudriere-ports` to refresh the files on your build server. 
   - Alternatively, you can find any old dist mirror (such as [this one](http://distfiles.icmpv6.org/distfiles/)), download the files on the build server in their matching folder (using `wget`/`curl`), then continue the build

You need to build **ALL** ports before proceeding to the next step. If you don't want to build one port, you can exclude it by removing it in [poudriere_bulk](https://github.com/pfsense/pfsense/blob/master/tools/conf/pfPorts/poudriere_bulk).


### Build kernel and create ISO

 Finally, you can build your customized firewall: `./build.sh --skip-final-rsync iso`. This command will build the kernel, then install ports on top of it and create the ISO file. Expect the command to run for ~one hour.
 
The build can the monitored from the two files in the `logs/` directory of pfSense GUI: 
- `buildworld.amd64`, `installworld.amd64` and `kernel.libreSense.amd64.log` will contain logs relative to the build of FreeBSD kernel.
- `install_pkg_install_ports.txt` contain logs relative to the installation of the ports. They are retrieved from the URL specified in the `build.conf` file.
- `isoimage.amd64` and `cloning.amd64.log` contain logs relative to the build of the ISO itself

At the end of the build, a compressed iso file (`.iso.gz`) will be present in `~pfsense/tmp/${product_name}/installer/`. You can extract it using `gunzip *.gz` if you need the plain `.iso`.

### If your encounter errors during ports or kernel build: possible root causes, and how to fix them:

One very possible root cause to why your build is failing, is related to how `Makefile` system works correlated with an unlucky timing. What happens is usually close to the following: 

1. Netgate build port XXX from `FreeBSD-ports`. On internal Netgate servers, poudriere marks the port as built and won't attempt to re-build it unless an update is made on the code ("config bump")
3. Few months later, ports owners update their build requirement (`distfile` update) / Netgate update its build environment (e.g., `openssl => opensll111` )
2. Netgate does NOT keep `FreeBSD-ports` in sync with the official FreeBSD ports repository. At this point, any attempt of rebuilding ports will fail...But it is fine for Netgate : the built packages are already there, and no rebuild will be made as long as no config bump is done on the ports. 
4. When you try to build ports yourself, some ports fail during build and you don't understand why.

The same applies for the content of `FreeBSD-src`. The recommended way to fix this issue is to simply re-synchronize modules that are failing with upstream, so that you will have an up-to-date ISO.

Also, Netgate has been accused of [performing *delayed open-sourcing*](https://github.com/doktornotor/pfsense-still-closedsource/blob/master/img/screenshot_bug8155_rebuilding_pfsense_kernel.png) in the past on `Freebsd-src`. It is unclear if it was due to users misunderstanding on how the build system works, due to Netgate shady practices, or due to a temporary maintenance. 
I haven't noticed any *delayed open sourcing* myself, but if that ever happens, you can merge FreeBSD sources with your branch directly.


# Differences between a genuine pfSense and your ISO

Your ISO is built the same way as pfSense ISO distributed by Netgate, and does contain the same code, with two major differences: your ISO does not include GNID nor pfSense-repoc.

## GNID

GNID is a binary (located at `/usr/sbin/gnid`) which is managing Netgate license for pfSense. This binary basically generates a unique Netgate ID for each genuine pfSense. 

The generated unique ID then become part of the default "User-Agent" when making HTTP requests with PHP (HTTP requests are used for fetching bogons, installing packages, displaying the first copyright message...etc). 

It is also required for accessing Netgate services on pfSense (such as ACB, or professional support)

How does this binary works is known *(a quick look to this binary with `radare2` show that it basically tries to fetch the platform it is running on, and if the platform is not Netgate hardware then it computes an ID using sha256 and MAC addresses of the device)*, but this program is property of Netgate and is closed-source (it is stored in the internal GitLab of Netgate). 

## pfSense-repoc

pfSense-repoc is a package containing multiple binaries. These binaries are in charge of configuring the right PKG repositories, depending on your licence and product (pfSense-CE, plus, etc..).

Like GNID, this package is property of Netgate and is closed-source.


Because your ISO does not contain pfSense-repoc nor GNID, you may not be able to retrieve bogons feed from Netgate, to use ACB, to install packages from official repositories or to ask for professional support to Netgate.
You also won't receive latest pfSense updates (which makes sense, since your ISO can't be called *pfSense software*..).

