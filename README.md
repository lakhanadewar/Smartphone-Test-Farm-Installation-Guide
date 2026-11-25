# Android Device Farm with DeviceFarmer STF

A short, tested guide to running your own Android device farm in the browser, using [DeviceFarmer STF](https://github.com/DeviceFarmer/stf) on Ubuntu or Ubuntu under WSL2.

You plug in Android phones by USB, or connect them wirelessly, open a web page, and control them from any browser. Handy for manual testing, Appium and Selenium-style automation practice, and sharing a few real devices with a team.

This guide was inspired by Paul Stanescu's original walkthrough, [How easy it is to create your own Android device farm on Linux](https://paul-stanescu.medium.com/how-easy-it-is-to-create-your-own-android-device-farm-on-linux-c07023615aac). That article got me started, but it was written for older software versions, so I hit errors when I followed it today. This README keeps the same idea and replaces the old steps with current ones.

## What you get

- Phones listed in a web dashboard at `http://YOUR_IP:7100`
- Remote screen control, app install, shell and logs per device
- A setup that takes about 15 minutes on a clean Ubuntu

## Tested setup

- Ubuntu running under WSL2 on Windows, with a phone connected over Wi-Fi (wireless debugging)
- Node.js 20 installed with nvm
- STF installed from the GitHub source
- RethinkDB for STF's database

I used Wi-Fi because WSL2 does not see USB devices by default. On a normal Ubuntu server or desktop (bare metal), USB works straight away and is the more stable choice. This guide covers both, so pick the one that fits your setup.

| Your setup | Connect phones with |
|---|---|
| Ubuntu on bare metal (desktop, server, laptop) | USB cable, simplest and most stable |
| Ubuntu under WSL2 | Wireless debugging, or USB through usbipd-win |
| Phones in another room, or no free USB ports | Wireless debugging |

## Requirements

- Ubuntu with `sudo` access (22.04 or newer is a sensible choice)
- An Android phone with Developer options enabled
- A USB cable that carries data, or the phone and the computer on the same Wi-Fi network for wireless debugging
- About 2 GB of free RAM

## Install

Run everything as your normal user, not root.

### 1. Install system packages

```bash
sudo apt update
sudo apt install -y build-essential git curl wget python3 python3-pip pkg-config cmake ninja-build autoconf automake libtool libzmq3-dev libprotobuf-dev protobuf-compiler graphicsmagick yasm android-tools-adb
```

### 2. Install Node.js 20 with nvm

STF's dependencies do not support the newest Node versions, so use Node 20.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.nvm/nvm.sh
nvm install 20
```

If you already have another Node version on the machine that other tools need (for example Node 24 for Frappe), keep it as the default.

```bash
nvm alias default system
```

From now on, run `nvm use 20` in any terminal where you work with STF.

### 3. Install RethinkDB

```bash
source /etc/lsb-release
echo "deb https://download.rethinkdb.com/repository/ubuntu-$DISTRIB_CODENAME $DISTRIB_CODENAME main" | sudo tee /etc/apt/sources.list.d/rethinkdb.list
wget -qO- https://download.rethinkdb.com/repository/raw/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/rethinkdb.gpg
sudo apt update
sudo apt install -y rethinkdb
```

If `apt update` complains that the repository has no Release file for your Ubuntu version, skip this step and run RethinkDB in Docker instead.

```bash
docker run -d --name rethinkdb -p 28015:28015 -p 8080:8080 rethinkdb:2.4
```

### 4. Start RethinkDB

Skip this if you used Docker in step 3.

```bash
mkdir -p ~/stf-data && cd ~/stf-data
nohup rethinkdb --bind all > rethink.log 2>&1 &
```

### 5. Install STF from source

```bash
cd ~
git clone https://github.com/DeviceFarmer/stf.git
cd stf
nvm use 20
npm install
npm link
```

If `npm install` fails on a native module, clean up and try again.

```bash
rm -rf ~/.node-gyp node_modules
npm install
```

### 6. Check the setup

```bash
stf doctor
```

Everything should say OK. If protobuf is the only item flagged, you can ignore it.

### 7. Connect a phone

Pick one option. On the phone, first enable Developer options (Settings, About phone, tap Build number seven times).

#### Option A: USB (bare metal Ubuntu)

1. On the phone, open Developer options and turn on **USB debugging**.
2. Plug the phone into the computer with a data cable.
3. Run:

```bash
adb start-server
adb devices -l
```

4. Accept the **Allow USB debugging?** prompt on the phone. Tick "Always allow from this computer".

The phone should be listed as `device`. If it says `no permissions` or shows nothing, see [USB troubleshooting](#usb-device-not-listed-or-no-permissions). You do not need `--allow-remote` for USB devices.

#### Option B: Wireless debugging (WSL2, or no cable)

Turn on **Wireless debugging** in Developer options, then follow the pairing steps below carefully, because this is where most people get stuck.

```bash
adb start-server
adb pair PAIR_IP:PAIR_PORT       # asks for the 6-digit pairing code
adb connect CONNECT_IP:CONNECT_PORT
adb devices -l
```

The device should be listed as `device`. See [the two addresses explained](#pairing-address-vs-connect-address) below, because `PAIR_PORT` and `CONNECT_PORT` are different numbers.

You can mix both. STF shows USB and wireless phones side by side.

### 8. Start STF

```bash
cd ~/stf-data
nvm use 20
stf local --public-ip YOUR_IP
```

Add `--allow-remote` if any of your phones are connected over Wi-Fi (Option B). USB-only setups do not need it.

```bash
stf local --allow-remote --public-ip YOUR_IP
```

Open `http://YOUR_IP:7100` in a browser, log in with any name and email, and your phone appears in the list. It can take up to a minute.

## Pairing address vs connect address

Android shows two different IP and port pairs for wireless debugging. Mixing them up is the number one reason `adb connect` fails.

| | Pairing address | Connect address |
|---|---|---|
| Where you see it | Wireless debugging, then **Pair device with pairing code** (popup) | Main **Wireless debugging** screen, under "IP address & Port" |
| Used with | `adb pair` | `adb connect` |
| Also needs | The 6-digit pairing code on the popup | Nothing, you must already be paired |
| Lifetime | Disappears when you close the popup | Stays while Wireless debugging is on |
| Port | Random, changes every time | Different from the pairing port, and changes when you toggle Wireless debugging |

The IP address is usually the same, but the port is not. After `adb pair` succeeds, the popup closes. Do not reuse the pairing port. Go back to the main Wireless debugging screen, read the new port, and run `adb connect` with that one.

## How to find the IP addresses

| What | How to find it |
|---|---|
| Phone IP | Wireless debugging screen, or Settings, Wi-Fi, tap your network |
| Ubuntu or WSL IP (use for `--public-ip`) | `hostname -I` and take the first address |
| Windows PC IP (when using WSL and other devices must reach it) | `ipconfig` in PowerShell, IPv4 under your Wi-Fi adapter |

Quick check that the computer can reach the phone.

```bash
ping -c 3 PHONE_IP
```

## What `--allow-remote` does

STF treats a device connected over the network (an address like `192.168.4.22:46843` in `adb devices`) as a "remote" device and ignores it by default. You will see this in the log.

```
Filtered out remote device "192.168.4.22:46843", use --allow-remote to override
```

Start STF with `--allow-remote` and the device appears in the dashboard. USB-connected phones are local devices, so if you only use cables you do not need it.

## Daily start

```bash
cd ~/stf-data
nohup rethinkdb --bind all > rethink.log 2>&1 &
adb start-server
adb connect CONNECT_IP:CONNECT_PORT   # wireless only, skip for USB
nvm use 20
stf local --allow-remote --public-ip YOUR_IP   # drop --allow-remote for USB only
```

Under WSL the IP changes after a restart, so run `hostname -I` again. The phone's connect port also changes whenever Wireless debugging is switched off and on.

## Stop

```bash
pkill -f "stf local"
pkill rethinkdb
adb kill-server
```

## Update STF

```bash
cd ~/stf
git pull
nvm use 20
npm install
```

## Troubleshooting

### `adb devices` shows nothing

Restart adb and connect again.

```bash
adb kill-server
adb start-server
adb connect CONNECT_IP:CONNECT_PORT
adb devices -l
```

### USB device not listed, or `no permissions`

On bare metal Ubuntu.

- Use a cable that carries data. Many cheap cables only charge.
- Make sure **USB debugging** is on, and set the USB mode to File transfer instead of Charging only.
- Check that Linux sees the phone with `lsusb`.
- If adb says `no permissions` or `insufficient permissions`, add a udev rule and your user to the right group.

```bash
sudo apt install -y android-sdk-platform-tools-common
sudo usermod -aG plugdev $USER
```

Log out and back in, then run `adb kill-server` and `adb devices -l`.

### `adb devices` shows `unauthorized`

Accept the "Allow USB debugging" prompt on the phone, then run `adb devices` again. If the prompt never appears, revoke USB debugging authorisations in Developer options and reconnect.

### `adb pair` works, but `adb connect` fails or the phone says to check the network

- You probably used the pairing port. Use the address on the **main** Wireless debugging screen instead.
- Keep the phone screen on and stay on the Wireless debugging screen while connecting.
- Set the phone's Wi-Fi privacy to **Use device MAC** instead of randomised MAC, then toggle Wi-Fi off and on. A changing MAC often means a changing IP.
- Turn off any VPN, and turn off any option that switches to mobile data when Wi-Fi is weak.
- Make sure the phone and computer are on the same network. Some routers and guest networks isolate devices from each other.
- Reserve a fixed IP for the phone in your router settings (DHCP reservation).

### The phone connects in adb but does not show in STF

Look for the "Filtered out remote device" line in the STF log. Restart with `--allow-remote`.

```bash
pkill -f "stf local"
stf local --allow-remote --public-ip YOUR_IP
```

### `ERR/poorxy ... ECONNREFUSED 127.0.0.1:7106` right after starting

STF's proxy tried a service that had not finished starting. Wait a minute and refresh the page. If the error keeps repeating, stop STF and start it again.

### Cannot connect to RethinkDB (port 28015 refused)

```bash
pgrep -a rethinkdb
```

If nothing is listed, start it again (step 4, or `docker start rethinkdb` if you used Docker).

### `npm install` fails with a node-gyp or native module error

```bash
rm -rf ~/.node-gyp node_modules
nvm use 20
node -v
npm install
```

Check that `node -v` prints a 20.x version. A newer Node version is the most common cause. Also confirm the packages from step 1 installed, especially `build-essential`, `cmake`, `yasm`, `libzmq3-dev` and `graphicsmagick`.

### `apt update` says the RethinkDB repository has no Release file

Your Ubuntu version is not in RethinkDB's repository. Remove the source file and use the Docker option in step 3.

```bash
sudo rm /etc/apt/sources.list.d/rethinkdb.list
sudo apt update
```

### `stf doctor` reports a problem

Read which component is flagged and fix that one. Missing `protoc` only matters if something fails later. Install it with `sudo apt install protobuf-compiler`.

### The page does not load

- Check that `stf local` is still running in its terminal.
- Use the same IP you passed to `--public-ip`.
- Under WSL, `http://localhost:7100` usually works from the Windows browser. Other devices on your network cannot reach WSL directly without a Windows port forward and a firewall rule for port 7100 and the ports STF uses after it.

### The phone appears, then disappears

Wireless debugging turns itself off when the phone sleeps on some devices, or when the network changes. Turn it back on, check the connect port, run `adb connect` again, and refresh the dashboard. A USB cable avoids this.

### Using USB with WSL2

USB devices are not visible inside WSL by default. Install [usbipd-win](https://github.com/dorssel/usbipd-win) on Windows and attach the phone.

In PowerShell as Administrator.

```powershell
winget install --interactive --exact dorssel.usbipd-win
usbipd list
usbipd bind --busid BUSID
usbipd attach --wsl --busid BUSID
```

Then in WSL.

```bash
sudo apt install -y linux-tools-generic hwdata
sudo update-alternatives --install /usr/local/bin/usbip usbip $(ls /usr/lib/linux-tools/*/usbip | tail -n1) 20
adb kill-server
adb start-server
adb devices -l
```

You need to attach the phone again each time you replug it or restart WSL.

### Port conflicts

STF uses ports from 7100 upwards. RethinkDB uses 28015, 29015 and 8080. If another service on the machine uses any of these, stop it or change its port.

## Security note

STF has no real authentication by default. Anyone who can reach port 7100 can log in with any name and control your phones. Keep it on a private network, and put it behind a VPN or a proper login before exposing it anywhere public.

## Links

- STF on GitHub: https://github.com/DeviceFarmer/stf
- STF Docker image: https://hub.docker.com/r/devicefarmer/stf
- nvm: https://github.com/nvm-sh/nvm
- RethinkDB: https://rethinkdb.com
- usbipd-win (USB in WSL): https://github.com/dorssel/usbipd-win
- Original inspiration, by Paul Stanescu: https://paul-stanescu.medium.com/how-easy-it-is-to-create-your-own-android-device-farm-on-linux-c07023615aac

## Credits

Thanks to Paul Stanescu for the original guide that made this idea approachable, and to the DeviceFarmer community for keeping STF alive after OpenSTF stopped releasing.

## Licence

Use and adapt this guide however you like. STF itself is licensed under its own terms in its repository.
