# OpenWrt en TP-Link Omada EAP225 v4.0 (firmware stock 5.2.3)

**Dispositivo:** TP-Link Omada EAP225 — Hardware v4.0  
**Firmware stock:** 5.2.3 Build 20250709 Rel. 67584(5553)  
**SO del operador:** Fedora Silverblue (con toolbox/distrobox)  
**Fecha:** Marzo 2026

---

## Respuesta corta

**OpenWrt soporta el EAP225 v4** oficialmente desde 2022 (target `ath79/generic`).
**El firmware 5.2.3 BLOQUEA el método estándar de instalación por software.**
Se necesita acceso físico al hardware (consola serial) para flashear.

---

## Qué dice Gemini vs. la realidad

Gemini te dio una guía que corresponde a firmwares **anteriores al 5.2.x** y presenta
varios errores importantes:

| Afirmación de Gemini | Veredicto | Realidad |
|---|---|---|
| "SSH + `cliclientd stopcs` debería funcionar" | **INCORRECTO** | El firmware 5.2.x reemplazó el shell Linux por una CLI propietaria restrictiva. `cliclientd stopcs` devuelve "Invalid command". No hay acceso a shell Linux por SSH. |
| "Como es un build reciente, el método por SSH debería funcionar perfectamente" | **INCORRECTO** | Es exactamente al revés: cuanto más nuevo el firmware, más bloqueado está. El 5.2.x es el que rompió el método. |
| "Plan B: TFTP con botón reset" | **NO CONFIRMADO para EAP225 v4** | TP-Link documenta el recovery TFTP por botón de reset solo para EAP225 **v5**, EAP245 v3/v4, EAP265HD v1, y modelos 802.11AX. El EAP225 **v4 no aparece en esa lista**. |
| No menciona consola serial | **OMISIÓN GRAVE** | La consola serial + U-Boot es el único método confirmado para flashear el EAP225 v4 cuando el método por software no funciona. |
| Implica que se puede hacer downgrade a 5.0.3 | **INCORRECTO** | El firmware 5.2.x prohíbe explícitamente el downgrade: "After upgrading this firmware, rolling back to any of the previous version is forbidden." |
| IP por defecto `192.168.0.254` | Correcto | Es la IP del EAP en modo standalone. OpenWrt después usa `192.168.1.1`. |

**En resumen: la guía de Gemini NO funciona con tu firmware 5.2.3.**

---

## Especificaciones del EAP225 v4

| Componente | Detalle |
|---|---|
| SoC | QCA9563 @ 775 MHz |
| RAM | 128 MiB DDR2 |
| Flash | 16 MiB SPI-NOR |
| WiFi 2.4 GHz | b/g/n 3x3 (integrado en SoC, driver `ath9k`) |
| WiFi 5 GHz | a/n/ac 2x2 MU-MIMO (QCA9886, driver `ath10k`) |
| Ethernet | 1× GbE (PHY: RTL8211FS), 802.3at PoE |
| Serial | PCB J4 — Pin 1: TXD, Pin 2: RXD, Pin 3: GND, Pin 4: VCC |
| Serial config | **3.3V**, 115200 baud, 8N1 |

---

## Qué cambió en el firmware 5.2.x

En versiones anteriores (hasta la 5.1.6), al conectar por SSH se obtenía un shell
Linux completo donde se podía ejecutar `cliclientd stopcs` y después flashear
OpenWrt por la interfaz web.

A partir del firmware 5.2.x, TP-Link:

1. **Eliminó el shell Linux** por SSH — ahora se accede a una CLI propietaria
   (prompt `EAP>` / `EAP#`) tipo Cisco.
2. **Eliminó el comando `cliclientd`** — no existe en la nueva CLI.
3. **Bloqueó el downgrade** — no se puede volver a ninguna versión anterior.
4. Comandos Linux (`ls`, `cat`, `sh`, `/bin/sh`) no funcionan.
5. El modo `enable` solo expone comandos `show` de diagnóstico.

---

## MÉTODO: Consola serial + TFTP + U-Boot

Este es el método que usó el desarrollador original que agregó soporte del EAP225 v4
a OpenWrt. Bypasea completamente el firmware stock accediendo al bootloader U-Boot
a través del puerto serial del PCB.

### Materiales necesarios

**Hardware:**

- **Adaptador USB a UART (3.3V)** — CP2102, FT232RL, o CH340G.
  **CRÍTICO: debe ser de 3.3V. A 5V podés dañar el SoC.**
- **Cables dupont** (jumper wires)
- **Soldador + estaño** — los pads J4 probablemente no tengan pines.
  Necesitás soldar pin headers o cables.
- **Cable Ethernet** — conexión directa PC ↔ EAP225.
- **Inyector PoE o switch PoE** — para alimentar el EAP225.

**Software (todo FLOSS, dentro de un toolbox):**

- `picocom` — terminal serial
- `dnsmasq` — servidor TFTP
- Imágenes de OpenWrt para EAP225 v4

---

### Paso 0: Preparar el entorno en Fedora Silverblue

```bash
# Crear un toolbox dedicado
toolbox create openwrt-flash
toolbox enter openwrt-flash

# Instalar herramientas
sudo dnf install -y picocom dnsmasq wget openssh-clients
```

Permisos del puerto serial (esto se hace en el **HOST**, fuera del toolbox):

```bash
# En el HOST:
sudo usermod -aG dialout $USER
# Cerrar sesión y volver a iniciar para que tome efecto.
# Los dispositivos /dev/ttyUSB* son visibles dentro del toolbox,
# pero los permisos del grupo dialout se resuelven en el host.
```

---

### Paso 1: Descargar las imágenes de OpenWrt

Desde el toolbox:

```bash
mkdir -p ~/openwrt-eap225v4 && cd ~/openwrt-eap225v4

# Imagen initramfs (boot temporal desde RAM)
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-tplink_eap225-v4-initramfs-kernel.bin \
  -O initramfs.bin

# Imagen sysupgrade (instalación permanente)
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-tplink_eap225-v4-squashfs-sysupgrade.bin \
  -O sysupgrade.bin
```

> **Verificar versión:** chequeá la última release estable en
> https://firmware-selector.openwrt.org/ buscando "EAP225 v4" y ajustá los URLs.

---

### Paso 2: Abrir el EAP225 y conectar el adaptador serial

1. **Desconectar la alimentación** (desenchufar el PoE).
2. **Abrir la carcasa** — quitar los tornillos de la parte trasera.
3. **Localizar el header J4** en el PCB — son 4 pads alineados:

```
J4: [ Pin1: TXD ] [ Pin2: RXD ] [ Pin3: GND ] [ Pin4: VCC ]
```

4. **Puentear resistencias si es necesario** — puede ser necesario puentear las
   resistencias no pobladas **R225** (línea TXD) y **R237** (línea RXD).
5. **Soldar pines o cables** al header J4.
6. **Conectar al adaptador USB-UART:**

```
EAP225 J4 Pin 1 (TXD)  →  RXD del adaptador USB-UART
EAP225 J4 Pin 2 (RXD)  →  TXD del adaptador USB-UART
EAP225 J4 Pin 3 (GND)  →  GND del adaptador USB-UART
EAP225 J4 Pin 4 (VCC)  →  NO CONECTAR (el EAP se alimenta por PoE)
```

---

### Paso 3: Abrir la terminal serial

Desde el toolbox:

```bash
# Verificar que el adaptador aparezca:
ls /dev/ttyUSB*

# Abrir terminal serial:
picocom -b 115200 /dev/ttyUSB0
```

Para salir de picocom: `Ctrl+A` y después `Ctrl+X`.

**No alimentar el EAP todavía** — lo hacemos después de configurar TFTP.

---

### Paso 4: Configurar la red y el servidor TFTP

Conectá tu PC al EAP225 con cable Ethernet directo.

**Configurar IP estática (en el HOST, fuera del toolbox):**

```bash
# Identificar tu interfaz Ethernet:
ip link show
# Buscar algo como: enp0s25, eno1, enp3s0, etc.

# Crear conexión temporal con NetworkManager:
sudo nmcli con add type ethernet ifname TU_INTERFAZ con-name eap-flash \
  ipv4.method manual ipv4.addresses 192.168.1.10/24
sudo nmcli con up eap-flash
```

**Abrir el firewall para TFTP (en el HOST):**

```bash
sudo firewall-cmd --add-service=tftp
```

**Levantar el servidor TFTP (en el toolbox):**

```bash
# Preparar directorio
mkdir -p /tmp/tftp
cp ~/openwrt-eap225v4/initramfs.bin /tmp/tftp/

# Levantar dnsmasq solo como servidor TFTP (sin DHCP):
sudo dnsmasq --no-daemon --port=0 --enable-tftp \
  --tftp-root=/tmp/tftp --listen-address=192.168.1.10
```

**Dejar esa terminal corriendo.** Abrir otra terminal/pestaña en el toolbox para
los pasos siguientes.

---

### Paso 5: Acceder al bootloader U-Boot

1. Tener picocom abierto en otra terminal.
2. **Conectar la alimentación PoE** al EAP225.
3. **Inmediatamente presionar `Ctrl+B` repetidamente** en la terminal serial.
4. Si lo lográs interrumpir, aparece el prompt de U-Boot:

```
ath>
```

> Si se te pasa y arranca el firmware stock, desconectá el PoE, esperá unos
> segundos, y volvé a conectar mientras spameás `Ctrl+B`.

---

### Paso 6: Cargar OpenWrt por TFTP

En el prompt `ath>` de U-Boot:

```
ath> setenv ipaddr 192.168.1.1
ath> setenv serverip 192.168.1.10
ath> tftp 0x80800000 initramfs.bin
ath> bootelf $fileaddr
```

Qué hace cada comando:
- `setenv ipaddr` — IP que usa el EAP225 para la transferencia
- `setenv serverip` — IP de tu PC (servidor TFTP)
- `tftp` — descarga la imagen a la RAM del EAP225
- `bootelf` — arranca OpenWrt desde la RAM

En la terminal de dnsmasq deberías ver la transferencia del archivo.
En la terminal serial vas a ver el kernel de Linux arrancando.

**Esto es temporario** — OpenWrt corre desde la RAM, no tocó la flash todavía.

---

### Paso 7: Instalar OpenWrt permanentemente

Ahora el EAP225 corre OpenWrt (temporal) y es accesible por red en `192.168.1.1`.

**Opción A: Por SSH (recomendada)**

```bash
# Verificar conectividad:
ping -c 3 192.168.1.1

# Copiar la imagen sysupgrade:
scp ~/openwrt-eap225v4/sysupgrade.bin root@192.168.1.1:/tmp/

# Conectar por SSH:
ssh root@192.168.1.1

# Dentro del EAP225, flashear:
sysupgrade -v /tmp/sysupgrade.bin
```

**Opción B: Por interfaz web (LuCI)**

1. Abrir `http://192.168.1.1` en el navegador.
2. Ir a System → Backup/Flash Firmware.
3. Subir `sysupgrade.bin`.
4. Confirmar.

El equipo se reinicia. Esta vez arranca OpenWrt desde la flash interna.

---

### Paso 8: Primer ingreso a OpenWrt

1. Esperar a que el LED se estabilice.
2. Ir a `http://192.168.1.1`.
3. Usuario: `root`, sin contraseña.
4. **Configurar contraseña inmediatamente** en System → Administration.

---

### Paso 9: Limpiar el entorno

```bash
# En el HOST - quitar la conexión de red temporal:
sudo nmcli con delete eap-flash

# Restaurar firewall:
sudo firewall-cmd --remove-service=tftp

# Cerrar picocom: Ctrl+A, Ctrl+X
# Cerrar dnsmasq: Ctrl+C en la terminal donde corre

# Opcional - eliminar el toolbox:
toolbox rm openwrt-flash
```

---

## Resumen de viabilidad

| Firmware stock | SSH + `cliclientd` + web | Consola serial + U-Boot | Recovery TFTP (botón reset) |
|---|---|---|---|
| ≤ 5.1.6 | **Funciona** | Funciona | No documentado para v4 |
| 5.2.x | **BLOQUEADO** | **Funciona** ← tu camino | No documentado para v4 |

---

## Alternativa de último recurso: Programador SPI

Si por alguna razón U-Boot también estuviera bloqueado (no debería), se puede
escribir directamente al chip de flash SPI-NOR con un programador externo
(clip SOIC-8 + CH341A a 3.3V, o Raspberry Pi + `flashrom`). Hacer backup
completo de la flash antes de tocar nada.

---

## Seguimiento

Hay discusiones activas intentando escapar la CLI del 5.2.x. A marzo de 2026,
nadie lo ha logrado. Seguí estos hilos:

- https://forum.openwrt.org/t/installing-openwrt-on-eap225v3-with-new-firmware/242003
- https://forum.openwrt.org/t/eap225-outdoor-v1-fw-5-2-3-cliclientd-invalid-command/244922

---

## Links de referencia

- **Firmware selector OpenWrt (EAP225 v4):**
  https://firmware-selector.openwrt.org/?version=23.05.5&target=ath79/generic&id=tplink_eap225-v4
- **Wiki OpenWrt del EAP225:**
  https://openwrt.org/toh/tp-link/eap225
- **Commit de soporte v4:**
  https://git.openwrt.org/7e21ce8e2b0b73f145af96958511f761544942d7
- **Recovery mode TP-Link (NO incluye EAP225 v4):**
  https://www.tp-link.com/us/support/faq/3186/
