# OpenWrt for Skyworth JioFiber JCOW407 (EcoNet EN7528)

This repository provides an open-source OpenWrt distribution with full GPON WAN, DSA Ethernet switching, and Dual-Band Wi-Fi support for the **Skyworth JioFiber JCOW407 / JCOW407-IN** optical network router.

---

## 1. Hardware Specifications

| Component | Specification |
| :--- | :--- |
| **Model** | Skyworth JioFiber JCOW407 (JCOW407-IN) |
| **Firmware Tag** | `SKYWTF1_JCOW407_R...` |
| **SoC** | EcoNet / Airoha EN7528 (Dual-core MIPS 1004Kc, `mipsel_24kc` Little-Endian @ 900MHz) |
| **RAM** | 256 MB / 512 MB DDR3 |
| **Flash** | 256 MB SPI NAND (`mtd0` TcBoot, `mtd1` romfile, `mtd2` Bank 1 / tclinux, `mtd5` Bank 2 / tclinux_slave) |
| **LAN Switch** | MediaTek MT7530 internal switch via DSA (`lan1`–`lan4` bridged into `br-lan`) |
| **Wi-Fi 2.4 GHz** | MediaTek MT7603E (802.11b/g/n, 2x2 MIMO) |
| **Wi-Fi 5 GHz** | MediaTek MT7615E / MT7663 (802.11a/n/ac, 4x4 MIMO, VHT80) |
| **Optical WAN** | Integrated EcoNet GPON MAC (`1fb60000.xmac`, netdev `pon0`), Airoha EN7571 BOSA controller |
| **USB** | USB 2.0 / 3.0 Host |
| **LEDs / Buttons** | RGB status LED (Red: GPIO 2, Green: GPIO 3, Blue: GPIO 4), Reset: GPIO 0, WPS: GPIO 1 |

---

## 2. GPON & OMCI Architecture

* **ITU-T G.984 GPON MAC:** Native hardware acceleration with dynamic Alloc-ID and GEM port provisioning directly from OLT grants.
* **ITU-T G.988 OMCI Stack:** In-kernel baseline OMCI stack with multi-packet MIB upload (`net/xpon/omci/agent.c`) and Radisys (`RSYS`) interoperability.
* **Carrier Datapath:** 802.1Q VLAN 1015 (`pon0.1015`), IPoE DHCP with mandatory Option 60 (Vendor Class Identifier) and Option 61 (Client Identifier). Option 12 (Hostname) is suppressed (`*`) to ensure compatibility with carrier BRAS gateways.

---

## 3. Out-Of-The-Box Defaults

On first boot after installation, the firmware automatically initializes with generic, secure defaults:

* **LAN Network:**
  * IP: `192.168.1.1`
  * Netmask: `255.255.255.0`
  * DHCP Server: Active (pool `192.168.1.100` – `192.168.1.250`)
* **Wi-Fi (Active by default):**
  * **2.4 GHz:** SSID `OpenWrt-JCOW407-2.4G` (Channel 6, HT20, WPA2-PSK: `openwrt123`)
  * **5 GHz:** SSID `OpenWrt-JCOW407-5G` (Channel 36, VHT80, WPA2-PSK: `openwrt123`)
* **WAN Interface:**
  * Device: `pon0.1015`
  * Protocol: DHCP (IPv4) + DHCPv6
  * Hostname: `*` (suppressed)
* **Identity:**
  * Factory Native Identity (derives hardware MAC from bootloader nvmem and hardware GPON serial from reservearea).

---

## 4. Subscriber Identity Configuration (One-Click Configurator)

The firmware includes a unified identity configuration tool (`/usr/sbin/jiofiber-config`):

### Mode A: Native Hardware Identity
If your JioFiber subscription is provisioned directly on this physical router:
```sh
jiofiber-config native
```
This automatically clears MAC overrides and uses the hardware's native burnt-in MAC and serial number.

### Mode B: Clone Subscribed Router
If you are running OpenWrt on a secondary router and wish to use the subscription from your primary router:
```sh
jiofiber-config clone <SUBSCRIBED_GPON_SN> <SUBSCRIBED_WAN_MAC> [PLOAM_PASS]
```

**Example:**
```sh
jiofiber-config clone SKYWXXXXXXXX AA:BB:CC:DD:EE:FF 00000001
```

This single command automatically:
1. Configures the xPON OMCI agent with the subscribed GPON serial and password.
2. Sets the WAN MAC override (`network.wan.macaddr`).
3. Formats and sets DHCP Option 60 (`network.wan.vendorid`).
4. Formats and sets DHCP Option 61 (`network.wan.clientid`).
5. Commits UCI and reloads networking and xPON services in one operation.

---

## 5. Building the Firmware

### Method 1: GitHub Actions (Recommended, Cloud Build)
This repository includes a ready-to-use GitHub Actions workflow (`.github/workflows/build-jcow407.yml`).
1. Push this repository to your GitHub account.
2. Go to the **Actions** tab on GitHub.
3. Select **Build JCOW407 OpenWrt Firmware** and click **Run workflow**.
4. Once completed, download the firmware images (`.trx` and `sysupgrade.bin`) directly from the workflow artifacts!

### Method 2: Local Build (Linux / WSL2)

#### Prerequisites (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y build-essential libncurses-dev gawk git subversion libssl-dev \
  gettext unzip zlib1g-dev file python3 libelf-dev rsync
```

#### Build Commands
```bash
# 1. Update and install feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 2. Load JCOW407 default config
cp jcow407.defconfig .config
make defconfig

# 3. Compile firmware
make -j$(nproc)
```

The resulting firmware images will be located at:
* `bin/targets/airoha/en7528/openwrt-airoha-en7528-jiofiber_jcow407-squashfs-tclinux.trx` (Factory flash)
* `bin/targets/airoha/en7528/openwrt-airoha-en7528-jiofiber_jcow407-squashfs-sysupgrade.bin` (Sysupgrade)

---

## 6. Flashing Instructions

> [!CAUTION]
> **Dual-Bank Safety Rule:** Always flash to **Bank 1 (`mtd2 = tclinux`) ONLY**. 
> **NEVER** flash, erase, or overwrite `mtd0` (bootloader), `mtd1` (romfile), or `mtd5` (`tclinux_slave` / factory stock Jio OS). Keeping Bank 2 intact ensures your router remains 100% unbrickable.

---

### Method A: Flashing via UART Serial Console
Since Skyworth firmware may have differing web GUI vulnerabilities compared to Sercomm models:

1. **Connect UART:** Open the router casing and connect a 3.3V USB-to-TTL adapter to the UART header (Baud rate `115200`, 8N1). Connect only GND, TX->RX, RX->TX. **Do not connect VCC.**
2. **Interrupt Boot:** Power on the router and press any key within 1 second to enter the bootloader command console (`bldr>`).
3. **Backup NAND:** Always dump a full flash backup before writing custom images.
4. **Flash OpenWrt:** Flash `openwrt-airoha-en7528-jiofiber_jcow407-squashfs-tclinux.trx` to Bank 1 (`tclinux`).
5. **Boot:** Set bootflag to Bank 1 and reboot:
   ```sh
   reboot
   ```

---

### Method B: Flashing from Factory Stock Jio OS using JFxploit
If your JCOW407 stock firmware version is vulnerable to JFxploit:

```bash
# On your PC
git clone https://github.com/JFC-Group/JFxploit.git
cd JFxploit
uv sync
uv run jfxploit run 192.168.29.1

# Host the firmware
python3 -m http.server 8000

# Inside telnet 192.168.29.1
cd /tmp
wget http://192.168.29.xxx:8000/openwrt-airoha-en7528-jiofiber_jcow407-squashfs-tclinux.trx -O /tmp/fw.trx
mtd -f write /tmp/fw.trx tclinux
/usr/bin/sys bootflag swap
reboot
```

---

### Method C: Safety Fallback — How to Revert to Factory Jio OS

Because Bank 2 (`mtd5 = tclinux_slave`) remains untouched:

* **From OpenWrt shell:**
  ```sh
  fw_setenv bootflag 1
  reboot
  ```
* **Or via TcBoot Serial Console (`bldr>`):**
  ```text
  bldr> memwl 80020000 31
  bldr> flash dfc0000 80020000 1
  bldr> go
  ```
The router will immediately boot back into the factory stock Jio OS!

---

## 7. Credits & Acknowledgments

* **Uday Khare ([@UdayKhare09](https://github.com/UdayKhare09))** for the JCOW414 OpenWrt reference repository and build setup.
* **Anand ([@Asjohn720](https://github.com/Asjohn720))** for the foundational EN7528 device tree sources and OpenWrt bring-up.
* **OpenWrt Project** and all kernel contributors for upstream EcoNet/Airoha EN7528 platform support.
