# Usa la misma distribución que los servidores DNS para simplificar el laboratorio.
FROM ubuntu:26.04

# Actualiza el índice de paquetes de Ubuntu.
RUN apt-get update

# Instala ISC DHCP, utilidades DNS/TSIG y herramientas de diagnóstico.
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends isc-dhcp-server bind9-utils bind9-dnsutils iproute2 procps

# Elimina el índice descargado para no conservar datos innecesarios.
RUN rm -rf /var/lib/apt/lists/*

# Copia la configuración explicada en el siguiente apartado.
COPY --chown=root:root --chmod=0644 dhcpd.conf /etc/dhcp/dhcpd.conf

# Crea el directorio donde dhcpd escribirá su PID.
RUN install -d -o root -g root -m 0755 /run/dhcpd

# Crea el directorio persistente de concesiones.
RUN install -d -o root -g root -m 0755 /var/lib/dhcp

# Crea una base vacía que Docker copiará a un volumen nuevo.
RUN touch /var/lib/dhcp/dhcpd.leases

# Ejecuta DHCP en primer plano para que Docker pueda observar sus logs y su estado.
CMD ["dhcpd", "-4", "-f", "-d", "-cf", "/etc/dhcp/dhcpd.conf", "-lf", "/var/lib/dhcp/dhcpd.leases", "-pf", "/run/dhcpd/dhcpd.pid", "eth0"]