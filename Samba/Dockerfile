FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

# Instala Samba
RUN apt-get update \
 && apt-get install -y --no-install-recommends samba \
 && rm -rf /var/lib/apt/lists/*

# Carpeta a compartir
RUN mkdir -p /share/archivos && chown -R nobody:nogroup /share && chmod -R 0775 /share

# Usuario de Samba "admin" con clave "admin" (cámbiala luego)
RUN useradd -M -s /usr/sbin/nologin admin \
 && (echo 'admin'; echo 'admin') | smbpasswd -s -a admin

# Configuración
COPY smb.conf /etc/samba/smb.conf

EXPOSE 137/udp 138/udp 139 445
CMD ["bash","-lc","testparm -s && smbd --foreground --no-process-group"]
