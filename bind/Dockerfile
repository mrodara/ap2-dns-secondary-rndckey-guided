# Ubuntu proporciona BIND y las utilidades de validación usadas en la práctica.
FROM ubuntu:26.04

# Actualiza el índice de paquetes dentro de la imagen.
RUN apt-get update

# Instala BIND, sus validadores, dig y herramientas de diagnóstico sin preguntas interactivas.
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends bind9 bind9-utils bind9-dnsutils iproute2 procps

# Elimina el índice descargado para no conservar datos innecesarios en la capa final.
RUN rm -rf /var/lib/apt/lists/*

# Crea el directorio escribible del primario y se lo entrega al usuario bind.
RUN install -d -o bind -g bind -m 0770 /var/lib/bind/dynamic

# Crea el directorio donde el secundario guardará las transferencias.
RUN install -d -o bind -g bind -m 0770 /var/lib/bind/slave

# Prepara el punto de montaje TSIG: root administra y el grupo bind puede leer.
RUN install -d -o root -g bind -m 0750 /run/tsig

# Copia las configuraciones; se seleccionará una u otra desde Compose.
COPY --chown=root:root --chmod=0644 primary.conf secondary.conf /etc/bind/

# Coloca la zona directa semilla en la ruta que poblará el volumen nuevo del primario.
COPY --chown=bind:bind --chmod=0660 seed/db.aulasri.test /var/lib/bind/dynamic/db.aulasri.test

# Coloca la zona inversa semilla con los mismos permisos escribibles.
COPY --chown=bind:bind --chmod=0660 seed/db.0.53.10 /var/lib/bind/dynamic/db.0.53.10

# Define un comando seguro por defecto; Compose lo hará explícito para cada rol.
CMD ["named", "-g", "-u", "bind", "-c", "/etc/bind/primary.conf"]