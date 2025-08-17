Proyecto Infraestructura Corporativa (ASSR Lab)

Este repo contiene un laboratorio Docker portátil para la sustentación del proyecto ASSR. Cada servicio vive en su propia carpeta con su compose.yml. El objetivo es que cualquier integrante pueda clonar el repo y levantar los servicios sin pasos extra (plug-and-play).

🚦 Idea principal

Todos los servicios comparten una misma red por defecto llamada assr-net.

No fijamos IPs: los contenedores se encuentran por nombre de servicio (DNS interno de Docker).

Evitamos depender de la red física del lugar (WiFi/LAN) y reducimos conflictos de IP.

Ventaja clave para la sustentación: en la laptop del dueño del repo, basta con docker compose up por servicio y Docker creará/empleará la red assr-net automáticamente.

🧠 ¿Por qué usar la red default = assr-net?

Portabilidad máxima: no se requiere docker network create. La red se crea/reutiliza sola.

Nombres en vez de IPs: ping suricata, curl http://kali → funciona en cualquier máquina.

Sin choques con la red local: Docker usa subred interna automática (no interfiere con WiFi/LAN del lugar).

Menos pasos que memorizar: el día de la demo solo se corre docker compose up en cada servicio.

Escalable: nuevos servicios solo declaran networks: default y ya están dentro de assr-net.

🗂️ Estructura del repo

Proyecto_Infraestructura_Coporativa/
├─ README.md                # este archivo
├─ .gitignore               # para evitar subir logs, workdir, etc.
├─ suricata/
│  ├─ compose.yml           # usa 'assr-net' como red default
│  ├─ README.md             # guía específica (opcional)
│  ├─ rules/                # reglas locales de Suricata
│  └─ logs/                 # fast.log, eve.json (no se versionan)
└─ kali/
   ├─ compose.yml           # usa 'assr-net' como red default
   ├─ README.md             # guía específica (opcional)
   └─ work/                 # scripts y artefactos (no se versionan)

🧩 Composición de los compose.yml

Cada compose.yml define:

services.<nombre>: contenedor (imagen, capacidades, comando de arranque, volúmenes).

networks.default.name: assr-net: asegura que el default se llame assr-net y se comparta.

Sin version:: Compose V2 ignora ese campo y muestra warnings.

Ejemplo mínimo (Kali):
services:
  kali:
    image: kalilinux/kali-rolling
    container_name: kali
    privileged: true
    stdin_open: true
    tty: true
    volumes:
      - ./work:/root/work
    command: >
      bash -lc "apt-get update && apt-get install -y iproute2 iputils-ping curl wget netcat-traditional dnsutils && bash"

networks:
  default:
    name: assr-net


🤝 Convenciones para contribuir

Una carpeta por servicio (/servicio/compose.yml).

Commits descriptivos: feat: ..., fix: ..., docs: ....

Ramas de feature: feature/<servicio>-<autor> (ej. feature/suricata-elias).

Pull Request con resumen breve de cambios y cómo probarlos.

- Comandos PS a utilizar antes de lanzar Bind9/Mariadb/Roundcube/Mailserver

docker-compose -f dns/docker-compose.yml up -d
docker-compose -f mariadb/docker-compose.yml up -d
docker-compose -f mail/docker-compose.yml up -d
docker-compose -f roundcube/docker-compose.yml up -d
docker-compose -f openvpn/docker-compose.yml up -d

docker run --rm `
  -v "${PWD}\docker-mailserver-config:/tmp/docker-mailserver/" `
  -v maildata:/var/mail `
  docker.io/mailserver/docker-mailserver:latest setup email add usuario1@empresa.local 123456

docker run --rm `
  -v "${PWD}\openvpn-data\conf:/etc/openvpn" `
  kylemanna/openvpn ovpn_genconfig -u udp://REEMPLAZAR_IP

docker run --rm -it `
  -v "${PWD}\openvpn-data\conf:/etc/openvpn" `
  kylemanna/openvpn ovpn_initpki

docker run --rm -it `
  -v "${PWD}\openvpn-data\conf:/etc/openvpn" `
  kylemanna/openvpn easyrsa build-client-full cliente1 nopass

docker run --rm `
  -v "${PWD}\openvpn-data\conf:/etc/openvpn" `
  kylemanna/openvpn ovpn_getclient cliente1 > "${PWD}\cliente1.ovpn"

docker compose up -d openvpn

(Si llega a dar error, es posible que toque agregar la carpeta a la ruta)

👥 Créditos

Equipo ASSR — Infraestructura Corporativa. Cada integrante mantiene su servicio y README específico.

