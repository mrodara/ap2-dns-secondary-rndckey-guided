# Usa Ubuntu para disponer del cliente ISC DHCP y herramientas conocidas.
FROM ubuntu:26.04

# Recibe desde Compose el hostname que este cliente enviará al servidor DHCP.
ARG CLIENT_HOSTNAME

# Detiene la construcción si Compose olvidó proporcionar el hostname.
RUN test -n "$CLIENT_HOSTNAME"

# Actualiza el índice de paquetes dentro de la imagen.
RUN apt-get update

# Instala el cliente DHCP, dig y herramientas de red visibles para el alumnado.
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends isc-dhcp-client bind9-dnsutils iproute2 iputils-ping procps

# Elimina el índice descargado para no conservar datos innecesarios.
RUN rm -rf /var/lib/apt/lists/*

# Escribe un hostname estable; evita publicar el ID aleatorio del contenedor en DNS.
RUN printf 'send host-name "%s";\nrequest subnet-mask, broadcast-address, routers, domain-name, domain-name-servers;\n' "$CLIENT_HOSTNAME" > /etc/dhcp/dhclient.conf

# Mantiene el contenedor disponible hasta que ejecutemos dhclient manualmente.
CMD ["sleep", "infinity"]