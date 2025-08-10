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

👥 Créditos

Equipo ASSR — Infraestructura Corporativa. Cada integrante mantiene su servicio y README específico.

