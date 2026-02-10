# Removed from reddit so sharing here to keep it available

## Original question
Hey yall!

I desperately need your advice what can i do to resolve this mess. I started my little smart home project with the following setup: Ubuntu, HA in docker, phyton-matter-server in docker, Aqara M100 as a border router and the new IKEA Myggbett as a test device (plus Android companion). My ISP not supporting Ipv6, my devices only getting link-local (fe80::) addresses but no ULA ones. If I try to connect my ikea sensor its looking good until the last step when the matter server trying to ping the device during the commissioning (with random ULA every time) when I see the "Network is unreachable" message in the server logs. I tried to hack the situation with radvd (local RA with prefix) even tried to set the matter prefix as default gateway, but no luck.

(one option I see is to call the ISP set the router to bridge mode and hook up a private router after that, but that too much hustle)

Do you guys have any advice what should I try next?


# Matter + IPv6 RA Setup (Docker, Ubuntu, HA container No HAOS)

I finally managed to solve it. The essence of the problem was that the router I received from my ISP did **not** have any settings to perform IPv6 Router Advertisements (RA).
Because of that, I compensated for this deficiency using a Linux (Ubuntu) server on the LAN.

Everything is running in **Docker**, **not HAOS + plugins**.

---

## ⚠️ Important

**Make sure you run your Matter server in *host network mode*!**

---

## Prerequisites (Linux Host)

Edit sysctl settings:

```bash
sudo nano /etc/sysctl.conf
```

Add / ensure the following:

```ini
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.eth0.accept_ra=1
net.ipv6.conf.eth0.accept_ra_rt_info_max_plen=64
```

Restart your host (recommended, just in case).

---

## Docker IPv6 Configuration

Edit Docker daemon config:

```bash
/etc/docker/daemon.json
```

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/64"
}
```

> The prefix is random — you can use anything for ULAs.

---

## Matter Server

* Clean install **python-matter-server**
* Remove the old Matter fabric
  → **delete the data folder**

---

## Router Advertisements (radvd)

Install `radvd`:

```bash
sudo apt install radvd
```

Edit config:

```bash
sudo nano /etc/radvd.conf
```

```conf
interface eno1 {
    AdvSendAdvert on;
    AdvDefaultLifetime 0;
    prefix fd00:dead:beef::/64 {
        AdvOnLink on;
        AdvAutonomous on;
    };
};
```

Allow mDNS (UDP 5353):

```bash
sudo ufw allow in proto udp from any to any port 5353
```

Restart `radvd`:

```bash
sudo systemctl restart radvd
```

---

## IPv6 Address on Interface

Assign a static IPv6 address:

```bash
sudo ip -6 addr add fd00:dead:beef::10/64 dev eno1
```

---

## Home Assistant Cleanup

Delete **all previous Matter-related data**:

* Matter integration
* Thread integration
  ➡️ Complete wipe

---

## M100 Border Router Setup

1. Disconnect M100 from power
2. Reboot
3. Reset (hold button **10 seconds** until yellow flashing)
4. Add M100 using **Aqara Home App**
5. Get the **Matter pairing code** (valid for ~10 minutes)

---

## Pairing with Home Assistant

* Use **Home Assistant Companion App (Android)**
* Add **Matter integration**
* Default `localhost` link is OK (running next to matter server)
* Pair M100 using the pairing code

  > Notice: link-local IPv6 addresses are used

### Expected Matter Logs

```text
2026-02-10 20:10:49.764 (MainThread) INFO [matter_server.server.device_controller] Starting Matter commissioning using Node ID 1 and IP fe80::56ef:44ff:fe8d:13d7.
2026-02-10 20:10:49.965 (Dummy-2) INFO [chip.ChipDeviceCtrl] Established secure session with Device
2026-02-10 20:10:50.734 (Dummy-2) INFO [chip.ChipDeviceCtrl] Commissioning complete
2026-02-10 20:10:50.735 (MainThread) INFO [matter_server.server.device_controller] Matter commissioning of Node ID 1 successful.
2026-02-10 20:10:50.735 (MainThread) INFO [matter_server.server.device_controller] Interviewing node: 1
2026-02-10 20:10:51.174 (MainThread) INFO [matter_server.server.device_controller] <Node:1> Setting-up node...
2026-02-10 20:10:51.176 (MainThread) INFO [matter_server.server.device_controller] <Node:1> Setting up attributes and events subscription.
2026-02-10 20:10:51.286 (MainThread) INFO [matter_server.server.device_controller] <Node:1> Subscription succeeded with report interval [1, 60]
2026-02-10 20:10:51.287 (MainThread) INFO [matter_server.server.device_controller] Commissioning of Node ID 1 completed.
```

✅ Border router setup complete.

---

## Adding the First Sensor (Failure Case)

When adding the first Matter sensor:

```text
2026-02-10 20:12:59.102 (MainThread) INFO [matter_server.server.device_controller] Starting Matter commissioning using Node ID 2 and IP fdd6:c91d:ff09:1:37c7:a670:34b3:38db.
2026-02-10 20:12:59.107 (Dummy-2) CHIP_ERROR [chip.native.IN] SendMessage() to UDP:[fdd6:c91d:ff09:1:37c7:a670:34b3:38db]:5540 failed: OS Error 0x02000065: Network is unreachable
2026-02-10 20:12:59.108 (Dummy-2) CHIP_ERROR [chip.native.SC] Failed during PASE session pairing request
2026-02-10 20:12:59.109 (MainThread) ERROR [matter_server.server.client_handler] Commissioning failed for node 2.
```

⚠️ Note the **random IPv6 prefix from the Thread network**.

---

## Manual IPv6 Routing Fix

Add a route toward the Matter / Thread prefix:

```bash
sudo ip -6 route add fdd6:c91d:ff09:1::/64 via fe80::56ef:44ff:fe8d:13d7 dev eno1
```

---

## Retry Pairing

✅ Works.

**PROFIT.** 💰

