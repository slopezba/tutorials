# Sharing PC internet with Raspberry/Jetson/robot miniPCs

Quick guide to let a PC with Wi-Fi provide internet access to miniPCs connected through Ethernet.

## Example network

| Device | IP | Typical interface |
|---|---:|---|
| PC, robot/cable side | `192.168.1.158` | `enp2s0` |
| PC, internet/Wi-Fi side | `192.168.1.23` | `wlp3s0` |
| Robot 1 / Raspberry | `192.168.1.9` | `eth0` |
| Robot 2 / Jetson | `192.168.1.97` | `eno1` or `eth0` |
| Robot 3 / miniPC | `192.168.1.78` | `eth0` |

> Adjust interface names if they are different on your system. On the PC, check them with:
>
> ```bash
> ip -br addr
> ip route get 8.8.8.8
> ```

---

## 1. Commands on the PC

### 1.1 Check interfaces

On the PC:

```bash
ip -br addr
ip route get 8.8.8.8
```

Expected example:

```text
enp2s0    UP    192.168.1.158/24
wlp3s0    UP    192.168.1.23/24
8.8.8.8 via 192.168.1.1 dev wlp3s0
```

Interpretation:

```text
LAN_IF = enp2s0   # Ethernet/cable side towards the robots
WAN_IF = wlp3s0   # Wi-Fi interface with internet access
```

---

### 1.2 Enable IPv4 forwarding temporarily

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Check:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

It should return:

```text
1
```

---

### 1.3 Add NAT for the three robots

On the PC:

```bash
WAN_IF="wlp3s0"
LAN_IF="enp2s0"

ROBOT1_IP="192.168.1.9"
ROBOT2_IP="192.168.1.97"
ROBOT3_IP="192.168.1.78"
```

#### Robot 1 / Raspberry `192.168.1.9`

```bash
sudo iptables -t nat -C POSTROUTING -s ${ROBOT1_IP}/32 -o ${WAN_IF} -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -I POSTROUTING 1 -s ${ROBOT1_IP}/32 -o ${WAN_IF} -j MASQUERADE

sudo iptables -C FORWARD -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT1_IP}/32 -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT1_IP}/32 -j ACCEPT

sudo iptables -C FORWARD -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT1_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT1_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

#### Robot 2 / Jetson `192.168.1.97`

```bash
sudo iptables -t nat -C POSTROUTING -s ${ROBOT2_IP}/32 -o ${WAN_IF} -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -I POSTROUTING 1 -s ${ROBOT2_IP}/32 -o ${WAN_IF} -j MASQUERADE

sudo iptables -C FORWARD -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT2_IP}/32 -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT2_IP}/32 -j ACCEPT

sudo iptables -C FORWARD -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT2_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT2_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

#### Robot 3 / miniPC `192.168.1.78`

```bash
sudo iptables -t nat -C POSTROUTING -s ${ROBOT3_IP}/32 -o ${WAN_IF} -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -I POSTROUTING 1 -s ${ROBOT3_IP}/32 -o ${WAN_IF} -j MASQUERADE

sudo iptables -C FORWARD -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT3_IP}/32 -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${LAN_IF} -o ${WAN_IF} -s ${ROBOT3_IP}/32 -j ACCEPT

sudo iptables -C FORWARD -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT3_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
sudo iptables -I FORWARD 1 -i ${WAN_IF} -o ${LAN_IF} -d ${ROBOT3_IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---

### 1.4 Check NAT rules on the PC

```bash
sudo iptables -t nat -S POSTROUTING
sudo iptables -S FORWARD
```

You should see rules for:

```text
192.168.1.9
192.168.1.97
192.168.1.78
```

---

## 2. Commands on each robot / miniPC

Each robot must use the PC cable-side IP as its default gateway:

```text
PC gateway = 192.168.1.158
```

---

### 2.1 Robot 1 / Raspberry `192.168.1.9`

On the Raspberry:

```bash
sudo ip route replace default via 192.168.1.158 dev eth0
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
```

Check:

```bash
ip route
ping -c 3 192.168.1.158
ping -c 3 8.8.8.8
ping -c 3 github.com
```

You should see:

```text
default via 192.168.1.158 dev eth0
```

---

### 2.2 Robot 2 / Jetson `192.168.1.97`

First check the interface name:

```bash
ip -br addr
```

If the interface is `eno1`:

```bash
sudo ip route replace default via 192.168.1.158 dev eno1
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
```

If the interface is `eth0`:

```bash
sudo ip route replace default via 192.168.1.158 dev eth0
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
```

Check:

```bash
ip route
ping -c 3 192.168.1.158
ping -c 3 8.8.8.8
ping -c 3 github.com
```

---

### 2.3 Robot 3 / miniPC `192.168.1.78`

If the interface is `eth0`:

```bash
sudo ip route replace default via 192.168.1.158 dev eth0
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
```

If the interface has another name, for example `enp1s0`, use:

```bash
sudo ip route replace default via 192.168.1.158 dev enp1s0
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
```

Check:

```bash
ip route
ping -c 3 192.168.1.158
ping -c 3 8.8.8.8
ping -c 3 github.com
```

---

## 3. Make it persistent on the PC

`iptables` rules are lost after reboot. To avoid that, create a script and a systemd service.

### 3.1 Create NAT script

On the PC:

```bash
sudo nano /usr/local/sbin/blueboat-nat.sh
```

Paste:

```bash
#!/usr/bin/env bash
set -e

WAN_IF="wlp3s0"
LAN_IF="enp2s0"

ROBOT_IPS=(
  "192.168.1.9"
  "192.168.1.97"
  "192.168.1.78"
)

sysctl -w net.ipv4.ip_forward=1

for IP in "${ROBOT_IPS[@]}"; do
  iptables -t nat -C POSTROUTING -s ${IP}/32 -o ${WAN_IF} -j MASQUERADE 2>/dev/null || \
  iptables -t nat -I POSTROUTING 1 -s ${IP}/32 -o ${WAN_IF} -j MASQUERADE

  iptables -C FORWARD -i ${LAN_IF} -o ${WAN_IF} -s ${IP}/32 -j ACCEPT 2>/dev/null || \
  iptables -I FORWARD 1 -i ${LAN_IF} -o ${WAN_IF} -s ${IP}/32 -j ACCEPT

  iptables -C FORWARD -i ${WAN_IF} -o ${LAN_IF} -d ${IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
  iptables -I FORWARD 1 -i ${WAN_IF} -o ${LAN_IF} -d ${IP}/32 -m state --state RELATED,ESTABLISHED -j ACCEPT
done
```

Give it execute permissions:

```bash
sudo chmod +x /usr/local/sbin/blueboat-nat.sh
```

Test it:

```bash
sudo /usr/local/sbin/blueboat-nat.sh
```

---

### 3.2 Make `ip_forward` persistent

On the PC:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-blueboat-ip-forward.conf
sudo sysctl --system
```

Check:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

It should return:

```text
1
```

---

### 3.3 Create systemd service

On the PC:

```bash
sudo nano /etc/systemd/system/blueboat-nat.service
```

Paste:

```ini
[Unit]
Description=BlueBoat NAT for robot miniPCs
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/blueboat-nat.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable blueboat-nat.service
sudo systemctl start blueboat-nat.service
```

Check:

```bash
systemctl status blueboat-nat.service
```

A correct status may look like:

```text
active (exited)
```

That is normal because this is a `oneshot` service: it runs the script, applies the rules, and exits.

---

## 4. Make the gateway persistent on each robot

This depends on how each robot manages networking. Two common cases are `dhcpcd` and `NetworkManager`.

---

### 4.1 Raspberry with `dhcpcd`

Edit:

```bash
sudo nano /etc/dhcpcd.conf
```

Add at the end:

```conf
interface eth0
static ip_address=192.168.1.9/24
static routers=192.168.1.158
static domain_name_servers=8.8.8.8 1.1.1.1
```

Reboot:

```bash
sudo reboot
```

---

### 4.2 Jetson or miniPC with NetworkManager

Show active connection:

```bash
nmcli -t -f NAME,DEVICE con show --active
```

Example for Jetson with `eno1`:

```bash
CON=$(nmcli -t -f NAME,DEVICE con show --active | awk -F: '$2=="eno1"{print $1; exit}')

sudo nmcli con mod "$CON" ipv4.addresses 192.168.1.97/24
sudo nmcli con mod "$CON" ipv4.gateway 192.168.1.158
sudo nmcli con mod "$CON" ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli con mod "$CON" ipv4.method manual

sudo nmcli con down "$CON"
sudo nmcli con up "$CON"
```

Example for miniPC `192.168.1.78` with `eth0`:

```bash
CON=$(nmcli -t -f NAME,DEVICE con show --active | awk -F: '$2=="eth0"{print $1; exit}')

sudo nmcli con mod "$CON" ipv4.addresses 192.168.1.78/24
sudo nmcli con mod "$CON" ipv4.gateway 192.168.1.158
sudo nmcli con mod "$CON" ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli con mod "$CON" ipv4.method manual

sudo nmcli con down "$CON"
sudo nmcli con up "$CON"
```

---

## 5. Docker / containers on Raspberry or miniPCs

If a container cannot resolve DNS even though the host has internet, check:

```bash
cat /etc/resolv.conf
getent hosts github.com
curl -I https://github.com
```

If you see something like:

```text
nameserver 192.168.1.1
```

and DNS does not work, set DNS inside the container:

```bash
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" > /etc/resolv.conf
```

Test:

```bash
getent hosts github.com
curl -I https://github.com
```

---

## 6. Quick troubleshooting

### Case A: the robot cannot reach the PC

On the robot:

```bash
ping -c 3 192.168.1.158
```

If it fails, check:

- Ethernet cable,
- robot IP,
- interface name,
- whether the PC is really using `192.168.1.158`.

---

### Case B: the robot reaches the PC but not the internet

On the robot:

```bash
ping -c 3 192.168.1.158
ping -c 3 8.8.8.8
```

If the PC responds but `8.8.8.8` does not:

- NAT is missing on the PC,
- `ip_forward` is `0`,
- wrong LAN/WAN interface in the `iptables` rules.

On the PC check:

```bash
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -S POSTROUTING
sudo iptables -S FORWARD
ip -br addr
ip route get 8.8.8.8
```

---

### Case C: `8.8.8.8` works but `github.com` does not

This is DNS.

On the robot:

```bash
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
ping -c 3 github.com
```

---

### Case D: internet is lost after reboot

One of these may have been lost.

On the PC:

```bash
systemctl status blueboat-nat.service
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -S POSTROUTING
sudo iptables -S FORWARD
```

On the robot:

```bash
ip route
cat /etc/resolv.conf
```

---

## 7. Things to keep in mind

1. If the PC Ethernet interface changes, for example from `enx00e04c6801cb` to `enp2s0`, old rules will stop matching.
2. `wlp3s0` must be the interface that really has internet. Check with `ip route get 8.8.8.8`.
3. `enp2s0` must be the wired interface towards the robots.
4. The robots' gateway must be the PC IP on the wired network, usually `192.168.1.158`.
5. `/32` in `192.168.1.9/32` means exactly one IP, not the full subnet.
6. You could give internet to the full subnet with `192.168.1.0/24`, but using specific robot IPs is more controlled.
7. Docker can have its own DNS inside the container. If the host resolves DNS but the container does not, check `/etc/resolv.conf` inside the container.
8. To add another robot, add its IP to the `ROBOT_IPS` array in the NAT script and set its gateway to `192.168.1.158`.
