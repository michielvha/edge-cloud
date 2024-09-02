# **Firewall Setup for ClusterMode Project**

## **Introduction**

The goal of this clustermode project is to create a multi site kubernetes cluster to be able to create 1 giant or multiple smaller clusters at multiple sites and linking them through Rancher.

To achieve a multi-site setup we'll ofcourse need a firewall capable of creating a Virtual Private Network. I'll probably be using `wireguard` for this. I had previously setup a firewall using [opensense](https://docs.opnsense.org/manual/how-tos/wireguard-client.html) but for some reason this will no longer boot.

Untill I can try flashing the hardware with latest firmware I was thinking about using OpenWRT instead, as it offers the same functionality and is more lightweight. Since we will have multiple sites and possibily need firewalls everywhere it looks best to me to base it on as few resources as possibly needed.


## **Install OpenWRT**

1. **Create Bootable USB**:
    obtain the [image](https://downloads.openwrt.org/) and flash it onto a USB.

2. **Flash the firmware**:
    Consult this [link](https://openwrt.org/toh/views/toh_standard_all) to find your device and correct firmware. Afterwards follow this [guide](https://openwrt.org/docs/guide-quick-start/factory_installation)

3. **Install [OpenWRT](https://openwrt.org/docs/guide-quick-start/start)**:
    
    3.1 **Identify Target Device**:
        Since lsblk is not available, you can use `df` or `cat /proc/partitions` to get more information about your available disks.

    ```bash
    cat /proc/partitions
    ```

    3.2 **Prepare the Target Disk**:
    Assuming you have identified your internal storage (let's say it's `/dev/sda`), the next step is to copy the OpenWRT root filesystem from the USB drive to the internal storage.

    ```bash
    dd if=/dev/sdb of=/dev/sda bs=4M
    ```

    3.3 **Sync and Reboot**:
    After copying, ensure all data is written to the disk:

    ```bash
    sync
    ```

    3.4 **Set Root Password**:
    set a root password to secure access.:

    ```bash
    passwd
    ```

    Then, reboot your machine:

    ```bash
    reboot
    ```

## Configure interfaces

### 1. Determine the Current LAN Interface Configuration

First, you need to check the current configuration of the LAN interface.

```bash
uci show network.lan
```

This command will display the current settings for the LAN interface, including the IP address and subnet.

### 2. Change the LAN IP Address and Subnet

You can update the LAN interface to use a new IP range. For example, if you want to change the LAN IP address to `192.168.2.1` with a subnet mask of `255.255.255.0` (or `/24`), you can do so with the following commands:

```bash
uci set network.lan.ipaddr='192.168.2.1'
uci set network.lan.netmask='255.255.255.0'
```

If you want to change the IP range to something else, just replace `192.168.2.1` and `255.255.255.0` with the desired IP address and subnet mask.


### 3. Update the DHCP Range

If you are using DHCP on the LAN interface, you should also update the DHCP range to match the new IP range:

```bash
uci set dhcp.lan.start='100'
uci set dhcp.lan.limit='150'
uci set dhcp.lan.leasetime='12h'
uci set dhcp.lan.dhcp_option='3,192.168.2.1'
```

Replace `192.168.2.1` with your new LAN IP address, and adjust the `start`, `limit`, and `leasetime` values as needed.

### 4. Apply and Commit the Changes

After making the changes, you need to commit the configuration and restart the network service for the changes to take effect:

```bash
uci commit network
uci commit dhcp
service network restart
service dnsmasq restart
```

---

## Configure USB Wi-Fi Adapter

### 1. Obtain Correct Drivers

I have a ``TP-Link AC1300 model Archer T4U``. After some research, I've found that this chipset is based on ``Realtek RTL8812AU``.

### 2. Install the Driver on OpenWrt

To get this USB Wi-Fi adapter working on OpenWrt, follow these steps:

#### Step 1: Update the Package List

First, make sure your package list is up-to-date by running the following command:

```bash
opkg update
```

#### Step 2: Install the Appropriate Driver

Install the driver that supports the Realtek RTL8812AU chipset:

```bash
opkg install kmod-rtl8812au-ct
```

If this driver doesn't work, you can alternatively try:

```bash
opkg install kmod-rtl8812au
```

but it is highly recommended to use the ``CT packages`` as this stands for ``Community Tuned``. These packages are tuned by the community for OpenWRT and thus are considerd to be stable.

#### Step 3: Reboot Your Device

After the driver is installed, reboot your OpenWrt device to load the new driver:

```bash
reboot
```

### 3. Verify That the Device is Recognized

After the reboot, you need to verify that the USB Wi-Fi adapter is recognized and functioning correctly.

#### Step 1: Check USB Device Recognition

You can check if the device is recognized by running:

```bash
dmesg | grep -i usb
```

Look for output lines that indicate your USB Wi-Fi adapter, such as references to the Realtek chipset or similar.

#### Step 2: Check Wireless Interfaces

To ensure that the wireless interface has been correctly initialized, run:

```bash
iw dev
```

This command lists all wireless interfaces. You should see a new interface, typically named `wlan0` or similar, representing your USB Wi-Fi adapter.

#### Step 3: Verify Interface in LuCI

Finally, you can also verify the presence of the wireless interface via the OpenWrt web interface (LuCI):

1. Log into your OpenWrt LuCI web interface.
2. Navigate to **Network > Wireless**.
3. You should see your USB Wi-Fi adapter listed as an available wireless device.

If all these checks pass, your TP-Link Archer T4U Wi-Fi adapter is successfully recognized and ready to be configured for your network.

### 4. Configure the USB Adapter as an Access Point in OpenWRT via the CLI

To enable our router to advertise a wireless connection we will configure the USB Adapter as an access point. This will be usefull for connecting devices to troubleshoot the router.

#### Step 1: SSH into Your OpenWrt Device

First, you'll need to SSH into your OpenWrt device from a terminal on your computer.

```bash
ssh root@192.168.10.1
```

Replace `192.168.10.1` with the IP address of your OpenWrt device if it's different.

#### Step 2: Verify the Wireless Interface

List your network interfaces to identify the wireless interface:

```bash
iw dev
```

Typically, the wireless interface will be named something like `wlan0`.

#### Step 3: Install Necessary Packages (If Not Already Installed)

Ensure that the necessary packages for Wi-Fi functionality are installed:

```bash
opkg update
opkg install wpad kmod-rtl8812au-ct
```

If you already have these installed, you can skip this step.

#### Step 4: Configure the Wireless Interface

Edit the wireless configuration file to set up the interface as an access point:

```bash
vi /etc/config/wireless
```

In this file, you should see an existing configuration block for your `wlan0` interface. If not, you can create one. The configuration should look something like this:

```bash
config wifi-device 'radio0'
        option type 'mac80211'
        option path 'pci0000:00/0000:00:1d.0/usb1/1-1/1-1.2/1-1.2:1.0'
        option channel '36'  # Keep 36 if using 5GHz; change to a 2.4GHz channel like 6 if using 2.4GHz
        option band '5g'  # Use '2g' if switching to 2.4GHz
        option htmode 'VHT20'  # Or VHT40/VHT80 for 5GHz, HT20/HT40 for 2.4GHz
        option disabled '0'  # Enable the Wi-Fi

config wifi-iface 'default_radio0'
        option device 'radio0'
        option network 'lan'
        option mode 'ap'
        option ssid 'OpenWrt'  # Change to your desired SSID
        option encryption 'psk2'  # WPA2 encryption
        option key 'yourpassword'  # Replace with a strong password
```

### Explanation of Key Options:
- **`channel`**: Choose a channel that is not heavily used in your environment. Channel 36 is for 5GHz; if using 2.4GHz, you might select channel 6.
- **`band`**: Specifies the frequency band (`5g` for 5GHz, `2g` for 2.4GHz).
- **`htmode`**: High Throughput mode. For 5GHz, use `VHT20`, `VHT40`, or `VHT80`. For 2.4GHz, use `HT20` or `HT40`.
- **`ssid`**: Set this to the name you want your Wi-Fi network to broadcast.
- **`encryption` and `key`**: Use `psk2` for WPA2 encryption and set a strong password.

#### Step 5: Configure the Network Interface

This step is not needed because your `br-lan` device already acts as a bridge, automatically including all interfaces assigned to it, including the Wi-Fi interface, which we have linked to the LAN by setting the `option network 'lan'` in the wireless setup.

#### Step 6: Commit Changes and Restart the Network

After making these changes, commit the changes and restart the network services:

```bash
/etc/init.d/network restart
```

This command will apply the changes and restart the network services on your OpenWrt device.

#### Step 7: Verify the Wireless AP

Finally, you can verify that the Wi-Fi AP is up and running by checking the status of the wireless interface:

```bash
iw dev wlan0 info

# or Troubleshoot
logread | grep wlan0
```

This command should display information about the Wi-Fi interface, including its SSID, mode, and operational channel. You can also scan for available networks from a laptop or another device to confirm that your new Wi-Fi network is visible and accessible.


---

## **Configure Wireguard [Server](https://openwrt.org/docs/guide-user/services/vpn/wireguard/server)**

### 1. Preparation

Install the required packages and specify configuration parameters for the VPN server.

#### Install Packages
```bash
opkg update
opkg install wireguard-tools
```

#### Configuration Parameters
```bash
VPN_IF="vpn"
VPN_PORT="51820"
VPN_ADDR="192.168.9.1/24"
VPN_ADDR6="fd00:9::1/64"
```

### 2. Key Management

Generate and exchange keys between the server and client.

#### Generate Keys
```bash
umask go=
wg genkey | tee wgserver.key | wg pubkey > wgserver.pub
wg genkey | tee wgclient.key | wg pubkey > wgclient.pub
wg genpsk > wgclient.psk
```

#### Store Keys
```bash
# Server private key
VPN_KEY="$(cat wgserver.key)"

# Pre-shared key
VPN_PSK="$(cat wgclient.psk)"

# Client public key
VPN_PUB="$(cat wgclient.pub)"
```

### 3. Firewall Configuration

Consider the VPN network as private. Assign the VPN interface to the LAN zone to minimize firewall setup. Allow access to the VPN server from the WAN zone.

#### Configure Firewall
```bash
uci rename firewall.@zone[0]="lan"
uci rename firewall.@zone[1]="wan"
uci del_list firewall.lan.network="${VPN_IF}"
uci add_list firewall.lan.network="${VPN_IF}"
uci -q delete firewall.wg
uci set firewall.wg="rule"
uci set firewall.wg.name="Allow-WireGuard"
uci set firewall.wg.src="wan"
uci set firewall.wg.dest_port="${VPN_PORT}"
uci set firewall.wg.proto="udp"
uci set firewall.wg.target="ACCEPT"
uci commit firewall
service firewall restart
```

### 4. Network Configuration

Configure the VPN interface and peers.

#### Configure Network
```bash
uci -q delete network.${VPN_IF}
uci set network.${VPN_IF}="interface"
uci set network.${VPN_IF}.proto="wireguard"
uci set network.${VPN_IF}.private_key="${VPN_KEY}"
uci set network.${VPN_IF}.listen_port="${VPN_PORT}"
uci add_list network.${VPN_IF}.addresses="${VPN_ADDR}"
uci add_list network.${VPN_IF}.addresses="${VPN_ADDR6}"
```

#### Add VPN Peers
```bash
uci -q delete network.wgclient
uci set network.wgclient="wireguard_${VPN_IF}"
uci set network.wgclient.public_key="${VPN_PUB}"
uci set network.wgclient.preshared_key="${VPN_PSK}"
uci add_list network.wgclient.allowed_ips="${VPN_ADDR%.*}.2/32"
uci add_list network.wgclient.allowed_ips="${VPN_ADDR6%:*}:2/128"
uci commit network
service network restart
```

## **Configure Wireguard [Client](https://openwrt.org/docs/guide-user/services/vpn/wireguard/client)**