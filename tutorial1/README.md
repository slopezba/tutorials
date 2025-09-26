## **1. Verificar la Configuración de IP**

Según `ifconfig`:

- **WiFi (`wlp47s0`) en Ubuntu:** `192.168.1.72`
- **Ethernet (`enp45s0`) en Ubuntu:** `192.168.1.100`
- **Ethernet (`enP8p1s0`) en la Jetson:** `192.168.1.123`

La configuración de IP está bien, pero falta habilitar la conexión compartida.

---

## **2. Activar el Reenvío de Paquetes en Ubuntu**

Esto permite que Ubuntu enrute los paquetes de la Jetson hacia Internet.
En tu computadora:
``` bash
sudo sysctl -w net.ipv4.ip_forward=1
```
Para hacerlo permanente:
``` bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## **3. Configurar `iptables` para Compartir Internet**

Ejecuta en Ubuntu:
``` bash
sudo iptables -t nat -A POSTROUTING -o wlp47s0 -j MASQUERADE
sudo iptables -A FORWARD -i enp45s0 -o wlp47s0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlp47s0 -o enp45s0 -j ACCEPT
```
## **4. Verificar la Puerta de Enlace en la Jetson Nano**

En la Jetson Nano, verifica la puerta de enlace con:
``` bash
ip route
```
Debe mostrar algo como
``` bash
default via 192.168.1.100 dev enP8p1s0
```
Si no aparece, agrégala manualmente:
``` bash
sudo ip route add default via 192.168.1.100 dev enP8p1s0
```
## **6. Probar la Conexión en la Jetson Nano**

Ejecuta en la Jetson:
``` bash
ping -c 5 8.8.8.8
```
## **7. (Opcional) Hacer Persistentes las Reglas de `iptables`**

Si quieres que las reglas de `iptables` persistan tras un reinicio:
``` bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
