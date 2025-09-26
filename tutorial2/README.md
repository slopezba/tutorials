Como tu PC tiene la IP `192.168.1.100`, la configuraremos como un servidor NTP local.

## 🔹 1. Instalar `chrony` en la PC

Ejecuta en tu PC con Ubuntu:

```bash
sudo apt update
sudo apt install chrony -y
```

## 🔹 2. Configurar `chrony` como servidor

Edita el archivo de configuración:

```bash
sudo nano /etc/chrony/chrony.conf
```

Luego, agrega al final del archivo:

```bash
allow 192.168.1.0/24
local stratum 10
```

### 📌 Explicación:

- `allow 192.168.1.0/24` → Permite que dispositivos en la red local (como la Jetson) se sincronicen.
- `local stratum 10` → Hace que la PC actúe como servidor NTP de baja prioridad.

Guarda el archivo (`CTRL+X`, `Y`, `ENTER`).

## 🔹 3. Reiniciar `chrony` en la PC

Ejecuta:

```bash
sudo systemctl restart chrony
sudo systemctl enable chrony
```

Para verificar que tu PC está funcionando como servidor NTP:

```bash
chronyc sources -v
```

Debería mostrar que está en **Stratum 10**.

---

# Configurar la Jetson Nano como Cliente

## 🔹 1. Editar `chrony.conf` en la Jetson

En la Jetson, edita el archivo de configuración:

```bash
sudo nano /etc/chrony/chrony.conf
```

Agrégale al final del documento:

```bash
server 192.168.1.100 iburst
```

Guarda (`CTRL+X`, `Y`, `ENTER`).

## 🔹 2. Reiniciar y Forzar Sincronización en la Jetson

Ejecuta en la Jetson:

```bash
sudo systemctl restart chrony
sudo chronyc tracking
sudo chronyc sources
```

Si `chronyc tracking` muestra **Stratum 10**, significa que la Jetson ya está sincronizada con tu PC.

Si no se sincroniza, fuerza la actualización con:

```bash
sudo chronyc makestep
sudo chronyc burst 4/4
sudo chronyc waitsync
```

Para verificar:

```bash
chronyc tracking
chronyc sources -v
```

---

# 📌 Resumen

### 🔹 En la PC (`192.168.1.100`):

1. Instalar `chrony`:
    
    ```bash
    sudo apt install chrony -y
    ```
    
2. Editar `/etc/chrony/chrony.conf` y agregar:
    
    ```bash
    allow 192.168.1.0/24
    local stratum 10
    ```
    
3. Reiniciar `chrony`:
    
    ```bash
    sudo systemctl restart chrony
    ```
    
4. Verificar que está corriendo:
    
    ```bash
    chronyc sources -v
    ```
    

### 🔹 En la Jetson Nano:

1. Editar `/etc/chrony/chrony.conf` y asegurarse de que tenga:
    
    ```bash
    server 192.168.1.100 iburst
    ```
    
2. Reiniciar `chrony`:
    
    ```bash
    sudo systemctl restart chrony
    ```
    
3. Verificar sincronización:
    
    ```bash
    chronyc tracking && chronyc sources -v
    ```