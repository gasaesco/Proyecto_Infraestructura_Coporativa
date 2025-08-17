# Samba en Docker (con Docker Compose)

> **Resumen rápido:** Este proyecto levanta un servidor **Samba** dentro de un contenedor Docker para compartir archivos en tu red.  
> **Importante:** Si ejecutas Docker **directamente en Windows (host)**, **no podrás exponer el puerto 445** (SMB) porque lo usa el propio Windows para su servicio de archivos. Por eso **no es posible** acceder a `\\IP_DE_TU_WINDOWS\archivos` cuando Samba está en un contenedor en ese mismo Windows.  
> **Soluciones recomendadas:** correr Samba en una **VM Linux (Ubuntu Server)** o en **WSL2 (Ubuntu dentro de Windows)** y luego acceder desde Windows a la IP de esa VM/WSL2.

---

## 🧭 Tabla de contenidos
- [Arquitectura y limitaciones en Windows](#arquitectura-y-limitaciones-en-windows)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Archivos](#archivos)
  - [`docker-compose.yml`](#docker-composeyml)
  - [`Dockerfile`](#dockerfile)
  - [`smb.conf`](#smbconf)
- [Despliegue](#despliegue)
- [Acceso desde Windows](#acceso-desde-windows)
- [Usuarios, contraseñas y permisos](#usuarios-contraseñas-y-permisos)
- [Configuración avanzada](#configuración-avanzada)
  - [Cambiar Workgroup, NetBIOS, protocolos SMB](#cambiar-workgroup-netbios-protocolos-smb)
  - [Restringir IPs/Interfaces](#restringir-ipsinterfaces)
  - [Agregar más recursos compartidos](#agregar-más-recursos-compartidos)
  - [Usar tu propio DNS](#usar-tu-propio-dns)
  - [Habilitar WINS (opcional)](#habilitar-wins-opcional)
- [Solución de problemas](#solución-de-problemas)
- [Comandos Git/GitHub](#comandos-gitgithub)
- [Licencia](#licencia)

---

## Arquitectura y limitaciones en Windows

Samba expone **SMB** en los puertos **137/UDP, 138/UDP, 139/TCP y 445/TCP**.  
En **Windows**, el puerto **445/TCP** ya lo usa el servicio **Server (LanmanServer)** del propio sistema para compartir archivos de Windows. **Docker no puede “republicar”/exponer ese puerto** hacia el host Windows porque ya está ocupado por el kernel/servicio de Windows.

**Consecuencia:** Si levantas el contenedor **en el host Windows**, no podrás acceder a `\\IP_DEL_HOST\archivos`, aunque el contenedor esté funcionando.  
**Alternativas viables:**  
1. **VM Linux (Ubuntu Server)** en la misma red (p. ej. IP `192.168.100.10`).  
   - Levantas Docker + Samba **dentro** de la VM.  
   - Desde Windows entras a `\\192.168.100.10\archivos`.  
2. **WSL2 (Ubuntu en Windows)**.  
   - Levantas Docker + Samba **dentro** de WSL2.  
   - Desde Windows entras a `\\<IP_WSL2>\archivos` (ver nota en [Solución de problemas](#solución-de-problemas) por particularidades de red en WSL2).

> 💡 Si solo quieres mover archivos entre Windows y el contenedor, puedes usar **volúmenes** de Docker y operar la carpeta del **host Windows** directamente, sin usar Samba.

---

## Estructura del proyecto

```
samba-docker/
├─ docker-compose.yml
├─ Dockerfile
├─ smb.conf
└─ data/
   └─ archivos/     # aquí residen los archivos compartidos en el host
```

---

## Archivos

### `docker-compose.yml`
```yaml
version: "3.8"

services:
  samba:
    build: .
    container_name: samba
    restart: unless-stopped
    environment:
      - TZ=America/Guayaquil
    ports:
      - "137:137/udp"
      - "138:138/udp"
      - "139:139/tcp"
      - "445:445/tcp"
    volumes:
      - ./data/archivos:/share/archivos
    # Opcional: fija DNS del contenedor
    # dns:
    #   - 1.1.1.1
    #   - 8.8.8.8
    # Opcional: mapea nombres internos a IPs
    # extra_hosts:
    #   - "samba.local:192.168.100.10"
```

### `Dockerfile`
```dockerfile
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends samba \
 && rm -rf /var/lib/apt/lists/*

# Carpeta compartida
RUN mkdir -p /share/archivos && chown -R nobody:nogroup /share && chmod -R 0775 /share

# Usuario de Samba "admin" con contraseña "admin" (cámbiala luego)
RUN useradd -M -s /usr/sbin/nologin admin \
 && (echo 'admin'; echo 'admin') | smbpasswd -s -a admin

# Configuración
COPY smb.conf /etc/samba/smb.conf

EXPOSE 137/udp 138/udp 139 445
CMD ["bash","-lc","testparm -s && smbd --foreground --no-process-group"]
```

### `smb.conf`
```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Docker
   security = user
   passdb backend = tdbsam
   map to guest = Bad User
   log file = /var/log/samba/log.%m
   max log size = 50
   server role = standalone server
   dns proxy = no

   # Recomendado: fuerza SMB2/3 (evita SMB1)
   server min protocol = SMB2
   client min protocol = SMB2

   # Limita el acceso a tu LAN (ajusta según tu red)
   hosts allow = 192.168.100. 127.

   # (Opcional) Asigna un nombre NetBIOS propio del servidor
   # netbios name = SAMBA-DOCKER

[archivos]
   path = /share/archivos
   browseable = yes
   read only = no
   writable = yes
   guest ok = no
   valid users = admin
   force user = nobody
   create mask = 0664
   directory mask = 0775
```

---

## Despliegue

1. Crea la carpeta de datos si no existe:
   ```bash
   mkdir -p data/archivos
   ```

2. Levanta el stack:
   ```bash
   docker compose up -d --build
   docker compose ps
   docker logs -f samba   # Ctrl+C para salir
   ```

3. Verificaciones dentro del contenedor:
   ```bash
   docker exec -it samba testparm -s
   docker exec -it samba smbstatus
   docker exec -it samba pdbedit -L   # lista usuarios (debe aparecer 'admin')
   ```

---

## Acceso desde Windows

1. **Asegúrate** de que el contenedor corre en **una VM Linux o WSL2** con una **IP alcanzable** desde Windows (ej.: `192.168.100.10` en VM o `172.x.x.x` en WSL2).
2. En Windows, abre `Win + R` y escribe:
   ```
   \\<IP_DEL_SERVIDOR>\archivos
   ```
   Ejemplos:
   - VM Linux: `\\192.168.100.10\archivos`
   - WSL2: `\\172.20.123.45\archivos` (IP de `eth0` en WSL2)

3. Credenciales:
   - Usuario: `admin`
   - Contraseña: `admin` (cámbiala después).

4. **Mapear unidad de red** (opcional): Explorador → *Este equipo* → *Conectar a unidad de red…* → Carpeta: `\\IP\archivos` → *Conectar con otras credenciales* → `admin/admin`.

> Si Windows recuerda credenciales antiguas:
> ```cmd
> net use * /delete
> ```

---

## Usuarios, contraseñas y permisos

- Cambiar la contraseña del usuario `admin`:
  ```bash
  docker exec -it samba smbpasswd admin
  ```

- Crear un nuevo usuario `juan`:
  ```bash
  docker exec -it samba useradd -M -s /usr/sbin/nologin juan
  docker exec -it samba smbpasswd -a juan
  # Añádelo a 'valid users' en smb.conf (o quítalo para permitir varios)
  docker restart samba
  ```

- Ajustar permisos en la carpeta del host si no puedes escribir:
  ```bash
  sudo chown -R nobody:nogroup data/archivos
  sudo chmod -R 0775 data/archivos
  ```

---

## Configuración avanzada

### Cambiar Workgroup, NetBIOS, protocolos SMB
En `[global]` del `smb.conf`:
```ini
workgroup = MI_GRUPO
netbios name = FILESERVER01
server min protocol = SMB2
client min protocol = SMB2
```
> Evita **SMB1** por seguridad. Si una máquina muy antigua lo requiere, evalúa riesgos antes de habilitarlo.

### Restringir IPs/Interfaces
Limita quién puede conectarse (segmentos confiables):
```ini
hosts allow = 192.168.100. 127.
# hosts deny = 0.0.0.0/0
```
Para forzar interfaces (si la VM/host tiene varias):
```ini
interfaces = eth0 lo
bind interfaces only = yes
```

### Agregar más recursos compartidos
Ejemplo de otro share de solo lectura:
```ini
[publico]
   path = /share/publico
   browseable = yes
   read only = yes
   guest ok = yes
```
Recuerda crear la carpeta y ajustar permisos:
```bash
mkdir -p data/publico
# y mapea en docker-compose: - ./data/publico:/share/publico
```

### Usar tu propio DNS
**Objetivo 1: Resolver nombres dentro del contenedor**  
En `docker-compose.yml` puedes fijar **resolvers**:
```yaml
services:
  samba:
    dns:
      - 192.168.100.1   # tu DNS local
      - 1.1.1.1
```
**Objetivo 2: Que Windows resuelva un nombre para el servidor Samba**  
- Crea un **registro A** en tu DNS local (por ej. `files.empresa.local -> 192.168.100.10`) y conecta con `\\files.empresa.local\archivos`.
- Alternativa rápida: agrega al *hosts* de Windows (administrador):
  ```
  C:\Windows\System32\drivers\etc\hosts
  192.168.100.10   files.empresa.local
  ```
- Si usas WSL2 y la IP cambia, considera un nombre que apunte a la IP real (o script para actualizar).

### Habilitar WINS (opcional)
Si tu red aún usa **NetBIOS** intensivamente, Samba puede actuar como **WINS server**:
```ini
[global]
wins support = yes
```
Luego, apunta clientes al WINS (DHCP u opciones manuales). En redes modernas con DNS, normalmente **no es necesario**.

---

## Solución de problemas

- **No puedo abrir `\\IP\archivos` desde Windows**  
  - ¿El contenedor corre en **Windows HOST**? → No funcionará por el puerto 445 reservado. Usa VM/WSL2.
  - ¿Firewall en la VM? Abre 137/udp, 138/udp, 139/tcp, 445/tcp.
  - ¿Versión SMB? Fuerza SMB2/3 en `smb.conf` y actualiza clientes.

- **WSL2: no veo el share**  
  - Obtén la IP real de WSL2:
    ```bash
    ip addr show eth0 | grep "inet "
    ```
  - Conéctate a esa IP desde Windows: `\\IP_DE_ETH0\archivos`.
  - Revisa que `smbd` esté escuchando y que Windows no esté bloqueando conexiones salientes a 445 por políticas.

- **Credenciales incorrectas / cacheadas**  
  ```cmd
  net use * /delete
  ```

- **Validar configuración Samba**  
  ```bash
  docker exec -it samba testparm -s
  docker exec -it samba smbstatus
  docker logs -f samba
  ```

---

## Comandos Git/GitHub

Desde tu carpeta del proyecto:
```bash
git init
git add .
git commit -m "Samba en Docker con Compose"

# Conecta con tu repo (HTTPS)
git branch -M main
git remote add origin https://github.com/<tu-usuario>/samba-docker.git

# Primer push (si el remoto está vacío)
git push -u origin main
```

> Si GitHub pide contraseña, usa un **Personal Access Token (PAT)**.  
> Si el remoto ya tenía commits (README, etc.), ejecuta primero:
> ```bash
> git pull origin main --allow-unrelated-histories
> git push -u origin main
> ```

---

## Licencia

Este proyecto se ofrece “tal cual”, sin garantías. Úsalo, modifícalo y adáptalo según tus necesidades y políticas de tu organización.
