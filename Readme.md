# Laboratorio de DNS secundario, DHCP dinámico y clave TSIG

Este repositorio levanta con Docker Compose un laboratorio IPv4 completo con dos servidores DNS autoritativos, un servidor DHCP y dos clientes. El servidor DHCP asigna direcciones, publica registros directos e inversos mediante DDNS firmado con TSIG y BIND replica las zonas del DNS primario al secundario.

> La clave TSIG se genera durante la ejecución y se guarda en un volumen de Docker. No se incluye ningún secreto en el repositorio ni en las imágenes.

## Puesta en marcha rápida

### Requisitos

- Docker Engine o Docker Desktop con el complemento `docker compose`.
- BuildKit y Buildx habilitados. Los Dockerfiles usan `COPY --chmod`, que no admite el constructor clásico.
- Acceso a Internet para descargar la imagen base y los paquetes de Ubuntu.
- Los puertos del anfitrión no son necesarios: todas las pruebas se ejecutan dentro de la red de Compose.
- La subred `10.53.0.0/24` debe estar libre y no solaparse con otra red local, VPN o red de Docker.

Comprueba las herramientas:

```bash
docker version
docker compose version
```

### Arranque limpio

Desde la raíz del repositorio:

```bash
# Valida la sintaxis y la resolución del fichero Compose.
docker compose config -q

# Construye todas las imágenes.
docker compose build

# Crea la red, los volúmenes y los contenedores.
docker compose up -d

# Muestra también el trabajo tsig-init, que ya habrá finalizado.
docker compose ps -a
```

El estado esperado es:

- `tsig-init`: finalizado con código `0` (`Exited (0)`). Este servicio no aparece en `docker compose ps` sin la opción `-a`.
- `primary`, `secondary` y `dhcpd`: `Up` y `healthy`.
- `client1` y `client2`: `Up`.

Si todavía aparece `health: starting`, espera unos segundos y repite:

```bash
docker compose ps -a
```

## Arquitectura

```text
                         volumen tsig-material
                      (clave privada dhcp-ddns.)
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
      DHCP 10.53.0.2 ── DDNS firmado ──► DNS primario 10.53.0.100
                                                  │
                                                  │ AXFR/IXFR + NOTIFY
                                                  ▼
                                      DNS secundario 10.53.0.200

      cliente1                         cliente2
      IP inicial 10.53.0.20            IP inicial 10.53.0.21
      DHCP: 10.53.0.120-170            DHCP: 10.53.0.120-170
```

Todos los contenedores comparten la red bridge `lab` (`10.53.0.0/24`). Las direcciones `10.53.0.20` y `10.53.0.21` solo permiten arrancar los clientes; durante la prueba se eliminan para solicitar una concesión real por DHCP.

### Servicios

| Servicio | Dirección | Función | Persistencia |
|---|---:|---|---|
| `tsig-init` | Sin dirección fija | Genera una sola vez la clave `dhcp-ddns.` con HMAC-SHA256 | `tsig-material` |
| `primary` | `10.53.0.100` | DNS autoritativo primario, acepta DDNS y notifica cambios | `primary-zones` |
| `secondary` | `10.53.0.200` | DNS autoritativo secundario, recibe las zonas del primario | `secondary-zones` |
| `dhcpd` | `10.53.0.2` | Asigna direcciones y actualiza los registros A, PTR y DHCID | `dhcp-leases` |
| `client1` | Inicial: `10.53.0.20` | Cliente DHCP con hostname `cliente1` | No |
| `client2` | Inicial: `10.53.0.21` | Cliente DHCP con hostname `cliente2` | No |

## Flujo de funcionamiento

1. `tsig-init` crea `/run/tsig/named.conf.key` si el volumen aún no contiene una clave.
2. El DNS primario arranca cuando la clave está disponible y carga las zonas semilla.
3. El DNS secundario espera a que el primario esté sano y solicita las zonas directa e inversa.
4. `dhcpd` arranca con la misma clave montada en modo de solo lectura.
5. Un cliente solicita una concesión y envía su hostname.
6. DHCP asigna una dirección entre `10.53.0.120` y `10.53.0.170`.
7. DHCP firma una actualización DDNS para crear:
   - un registro `A` en `aulasri.test`;
   - un registro `PTR` en `0.53.10.in-addr.arpa`;
   - un registro `DHCID` para detectar conflictos de nombre.
8. El primario guarda el cambio en su journal y avisa al secundario.
9. El secundario transfiere la nueva versión y responde con los mismos datos.

## Configuración, fichero a fichero

### `compose.yml`

Define el orden de arranque, la red, las direcciones fijas, los volúmenes y las comprobaciones de salud.

- `depends_on` evita que BIND o DHCP arranquen antes de disponer de la clave y de un primario operativo.
- `NET_ADMIN` y `NET_RAW` permiten a DHCP y a los clientes realizar operaciones de red.
- `ip_range: 10.53.0.224/27` reserva para Docker su asignación automática y evita que interfiera con el rango DHCP `10.53.0.120-170`.
- Los clientes reciben `CLIENT_HOSTNAME` como argumento de construcción para generar un `dhclient.conf` estable.

### `bind/Dockerfile`

Construye la imagen común de los dos servidores DNS. Instala BIND, `dig`, validadores y herramientas de diagnóstico; prepara los directorios escribibles y copia las configuraciones y zonas iniciales.

### `bind/primary.conf`

Configura un servidor exclusivamente autoritativo:

- desactiva la recursión e IPv6 para acotar el laboratorio;
- incluye la clave TSIG generada en tiempo de ejecución;
- sirve la zona directa `aulasri.test` y la inversa `0.53.10.in-addr.arpa`;
- solo permite transferencias hacia `10.53.0.200`;
- autoriza a `dhcp-ddns.` a crear registros `A`, `DHCID` y `PTR`, sin conceder permisos generales sobre las zonas;
- envía los eventos de actualización y seguridad a los logs del contenedor.

### `bind/secondary.conf`

Declara las mismas zonas como secundarias y usa `10.53.0.100` como primario. Las copias recibidas se guardan en `/var/lib/bind/slave`; el secundario no permite que terceros vuelvan a transferirlas.

### `bind/seed/db.aulasri.test`

Zona directa inicial. Contiene el SOA, los dos NS y los registros fijos:

| Nombre | Tipo | Valor |
|---|---|---:|
| `ns1.aulasri.test` | A | `10.53.0.100` |
| `ns2.aulasri.test` | A | `10.53.0.200` |
| `dhcp.aulasri.test` | A | `10.53.0.2` |

### `bind/seed/db.0.53.10`

Zona inversa inicial para `10.53.0.0/24`. Publica los PTR de DHCP y de ambos servidores DNS.

### `dhcp/Dockerfile`

Instala ISC DHCP y las utilidades necesarias. Ejecuta `dhcpd` en primer plano para que Docker pueda supervisar el proceso y conservar sus logs.

### `dhcp/dhcpd.conf`

Configura DHCP y DDNS:

- rango de concesiones: `10.53.0.120-10.53.0.170`;
- concesión normal: 60 segundos; máximo: 180 segundos;
- renovación T1: 20 segundos; revinculación T2: 40 segundos;
- dominio: `aulasri.test`;
- DNS entregados al cliente: `10.53.0.100` y `10.53.0.200`;
- actualizaciones directas e inversas firmadas con `dhcp-ddns.`;
- detección de conflictos mediante DHCID;
- el servidor controla el nombre publicado y usa el hostname solicitado por el cliente.

### `client/Dockerfile`

Instala `dhclient`, `dig`, `ping` y herramientas de red. El argumento `CLIENT_HOSTNAME` genera `/etc/dhcp/dhclient.conf` y cada contenedor permanece activo para ejecutar las pruebas manualmente.

### `.gitignore`

Evita versionar secretos TSIG, claves privadas, concesiones DHCP, journals de BIND y ficheros `.env`.

## Verificación completa

Los comandos siguientes parten del laboratorio arrancado con `docker compose up -d`.

### 1. Validar BIND y DHCP

```bash
docker compose exec -T primary named-checkconf -z /etc/bind/primary.conf
docker compose exec -T secondary named-checkconf -z /etc/bind/secondary.conf
docker compose exec -T dhcpd dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf
```

Los tres comandos deben finalizar con código `0`. BIND mostrará que las zonas se han cargado correctamente; DHCP no debe mostrar errores de sintaxis.

### 2. Comprobar la clave sin revelar el secreto

```bash
docker compose exec -T primary stat -c '%U:%G %a %n' /run/tsig/named.conf.key
docker compose exec -T dhcpd stat -c '%U:%G %a %n' /run/tsig/named.conf.key
```

Se espera:

```text
primary: root:bind 440 /run/tsig/named.conf.key
dhcpd:   root:dhcpd 440 /run/tsig/named.conf.key
```

El volumen conserva el GID numérico, pero cada imagen puede resolverlo con un nombre de grupo distinto. La clave mantiene el modo `0440`, BIND puede leerla y DHCP se ejecuta como `root`.

### 3. Comprobar las zonas iniciales en ambos DNS

```bash
docker compose exec -T client1 dig @10.53.0.100 aulasri.test SOA +short
docker compose exec -T client1 dig @10.53.0.200 aulasri.test SOA +short
docker compose exec -T client1 dig @10.53.0.100 ns1.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.200 ns2.aulasri.test A +short
```

Las consultas SOA deben devolver el mismo número de serie. Las dos últimas deben devolver `10.53.0.100` y `10.53.0.200`, respectivamente.

### 4. Obtener concesiones DHCP

```bash
docker compose exec -T client1 sh -ec '
  ip address flush dev eth0
  dhclient -4 -v eth0
  ip -4 -o address show dev eth0
'

docker compose exec -T client2 sh -ec '
  ip address flush dev eth0
  dhclient -4 -v eth0
  ip -4 -o address show dev eth0
'
```

Cada cliente debe recibir una dirección diferente comprendida entre `10.53.0.120` y `10.53.0.170`.

Guarda las direcciones para las pruebas siguientes:

```bash
CLIENT1_IP=$(docker compose exec -T client1 sh -c "ip -4 -o address show dev eth0 | awk '{print \$4}' | cut -d/ -f1")
CLIENT2_IP=$(docker compose exec -T client2 sh -c "ip -4 -o address show dev eth0 | awk '{print \$4}' | cut -d/ -f1")
printf 'cliente1=%s\ncliente2=%s\n' "$CLIENT1_IP" "$CLIENT2_IP"
```

### 5. Verificar DDNS en el primario

```bash
docker compose exec -T client1 dig @10.53.0.100 cliente1.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.100 cliente2.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.100 -x "$CLIENT1_IP" +short
docker compose exec -T client1 dig @10.53.0.100 -x "$CLIENT2_IP" +short
docker compose exec -T client1 dig @10.53.0.100 cliente1.aulasri.test DHCID +short
```

Se esperan las dos direcciones asignadas, los nombres `cliente1.aulasri.test.` y `cliente2.aulasri.test.`, y un valor DHCID no vacío.

### 6. Verificar la réplica en el secundario

```bash
docker compose exec -T client1 dig @10.53.0.200 cliente1.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.200 cliente2.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.200 -x "$CLIENT1_IP" +short
docker compose exec -T client1 dig @10.53.0.200 -x "$CLIENT2_IP" +short
```

El secundario debe devolver los mismos datos que el primario. Si la transferencia todavía está en curso, espera unos segundos y repite las consultas.

También puedes comparar los SOA:

```bash
docker compose exec -T client1 dig @10.53.0.100 aulasri.test SOA +short
docker compose exec -T client1 dig @10.53.0.200 aulasri.test SOA +short
```

### 7. Revisar concesiones y logs

```bash
docker compose exec -T dhcpd sh -c 'grep -E "^lease |client-hostname|binding state" /var/lib/dhcp/dhcpd.leases'
docker compose logs --no-color dhcpd primary secondary
```

Los logs permiten seguir la concesión DHCP, la actualización firmada, la notificación y la transferencia de zona.

### 8. Verificar la persistencia

La clave, las zonas dinámicas, las copias secundarias y las concesiones están en volúmenes. Elimina y recrea solo los contenedores:

```bash
docker compose down
docker compose up -d
```

Cuando los servicios vuelvan a estar sanos, los registros creados anteriormente deben seguir disponibles:

```bash
docker compose exec -T client1 dig @10.53.0.100 cliente1.aulasri.test A +short
docker compose exec -T client1 dig @10.53.0.200 cliente1.aulasri.test A +short
```

## Diagnóstico

### Consultar el estado y los logs

```bash
docker compose ps
docker compose logs --no-color tsig-init primary secondary dhcpd
docker compose logs --no-color --follow primary dhcpd
```

### Una zona no aparece en el secundario

Comprueba la conectividad, el SOA y los mensajes de transferencia:

```bash
docker compose exec -T secondary ping -c 1 10.53.0.100
docker compose exec -T secondary dig @10.53.0.100 aulasri.test SOA +short
docker compose logs --no-color primary secondary
```

### El cliente no obtiene una concesión

```bash
docker compose ps dhcpd
docker compose logs --no-color dhcpd
docker compose exec -T client1 ip -4 address show dev eth0
```

Asegúrate de haber eliminado la dirección inicial antes de ejecutar `dhclient` y de que `dhcpd` está sano.

### Conflicto con la subred del anfitrión

Si `10.53.0.0/24` coincide con una VPN u otra red, modifica de forma coherente:

- la red y las direcciones de `compose.yml`;
- `allow-transfer`, `also-notify` y `primaries` en BIND;
- los destinos DDNS, la subred y el rango en `dhcpd.conf`;
- los registros semilla directos e inversos.

No basta con cambiar únicamente la subred de Compose.

## Parada y limpieza

Detén el laboratorio conservando todos los datos:

```bash
docker compose down
```

Elimina también la clave, las zonas dinámicas, los journals y las concesiones para repetir la práctica desde cero:

```bash
docker compose down -v --remove-orphans
```

> `down -v` es destructivo para los datos del laboratorio. Úsalo solo cuando quieras reiniciar completamente el escenario.
