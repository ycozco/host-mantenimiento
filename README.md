# host-mantenimiento

Despliegue NATS + JetStream para la sincronización de Decolumax.

Despliegue inicial de un broker NATS con JetStream en disco, TLS de extremo a extremo y límites conservadores. Nginx Proxy Manager (NPM) reenvía TCP; no termina TLS. No incluye API, base de datos ni certificados privados.

## Qué protege y qué limita

- El puerto NATS `4222` no se publica directamente por Compose: NPM debe compartir la red Docker indicada por `NPM_DOCKER_NETWORK`.
- El certificado TLS se instala en NATS. La entrada de NPM es un **Stream TCP passthrough** al host `decolumax-nats`, puerto `4222`; no crear un Proxy Host HTTP ni terminar TLS en NPM.
- JetStream usa un volumen persistente. El servidor limita su almacenamiento a 128 MB y memoria JetStream a 64 MB. La configuración sugerida del stream limita eventos a 128 KiB, 96 MiB y siete días.
- El contenedor conserva el perfil de laboratorio de 256 MiB / 0,50 CPU. En el ensayo local NATS + JetStream llegó a un pico muestreado de 123,2 MiB; esa medición no incluía TLS y credenciales individuales. No es una garantía de consumo en VPS.
- La clave de TLS y `.env` son secretos operativos; no están incluidos. No expongas el puerto de monitorización `8222`.

## Requisitos

Docker Compose v2 en el VPS, NPM ya operativo, un DNS `A/AAAA` para el subdominio apuntando al VPS, y una red Docker compartida entre NPM y este contenedor. Usa un nombre como `sync.tu-dominio.example`; debe coincidir con el SAN del certificado.

### 1. Preparar directorio y secreto

Sube esta carpeta al VPS. Dentro de ella, copia `.env.example` a `.env`, genera una contraseña aleatoria larga y cambia `NATS_SYNC_PASSWORD`. Configura `NPM_DOCKER_NETWORK` con el nombre real de la red de NPM. No uses el valor de ejemplo en producción.

En Linux, restringe `.env` y la clave privada: `chmod 600 .env certs/privkey.pem`.

### 2. Instalar certificado

Coloca el certificado válido para el subdominio en `certs/fullchain.pem` y su clave en `certs/privkey.pem`. Puedes emitirlo con Let's Encrypt desde un flujo DNS-01/HTTP-01 y copiarlo con renovación automatizada. Después de renovar, reinicia NATS para cargar el certificado actualizado:

```sh
docker compose restart nats
```

No pongas estos archivos en Git ni los compartas con los socios.

### 3. Conectar NPM

En el panel de NPM, crea una entrada **Streams** TCP:

- Puerto de entrada público: `4222`.
- Destino: `decolumax-nats`.
- Puerto destino: `4222`.

NPM y NATS deben estar en la misma red Docker. Si NPM no ofrece Streams, configura su módulo `stream` TCP (`proxy_pass`) o usa un proxy L4 compatible. No configures un Proxy Host HTTP para NATS TCP. El cliente se conecta a `sync.tu-dominio.example:4222` y valida TLS usando ese hostname. Abre en el firewall solo el puerto TCP 4222 para esta ruta; no publiques 8222.

### 4. Arrancar el broker

```sh
docker compose config
docker compose up -d
docker compose logs --tail=100 nats
```

Confirma el log de inicio y luego prueba desde un cliente externo con TLS, hostname correcto y las credenciales `.env`. No consideres verificado el despliegue hasta comprobar conexión, publicación, recepción, ACK y reconexión tras reinicio.

### 5. Crear el stream acotado

Usa NATS CLI desde un equipo administrativo, con TLS y las credenciales iniciales. El archivo `stream.json` define almacenamiento en disco, una réplica, descarte de mensajes nuevos cuando llega a cuota, máximo de 96 MiB, 128 KiB por mensaje y retención de siete días. Ajusta `subjects` y permisos antes de incorporar más organizaciones.

En una sesión Linux, carga las variables del archivo local antes de ejecutar la CLI:

```sh
set -a
. ./.env
set +a
nats --server tls://sync.tu-dominio.example:4222 --user "$NATS_SYNC_USER" --password "$NATS_SYNC_PASSWORD" stream add --config stream.json
```

Si la versión instalada de NATS CLI no admite `--config`, usa `nats stream add DECO_SYNC` y proporciona los mismos valores interactivamente. Valida el resultado con `nats stream info DECO_SYNC`.

## Invitaciones y credenciales de socios

Esta configuración es una base de transporte y TLS, no implementa el registro de organizaciones ni la aceptación de invitaciones. El usuario configurado aquí es un principal de bootstrap compartido: **no distribuyas estas credenciales en la app ni a todos los socios**. Antes de producción, reemplázalo por identidades NATS por dispositivo o usuario, permisos de subjects por organización y un proceso administrativo de alta/revocación. El código de invitación debe ser aleatorio, de un solo uso, con vencimiento y validado/finalmente aprobado por la organización; no debe convertirse directamente en contraseña NATS.

En el diseño sin base de datos, el servidor aún conserva el estado mínimo que requiere JetStream (mensajes cifrados hasta su expiración) y NATS puede conservar metadatos públicos de autorización. La clave raíz de firma no debe residir en móviles ni en el VPS expuesto.

## Copias y operación

El volumen `nats_data` contiene el estado persistente de JetStream. Haz snapshots consistentes siguiendo el procedimiento de respaldo de NATS, guarda la copia fuera del VPS y prueba una restauración. La cuota del stream no sustituye respaldos. Si la cola alcanza 96 MiB, `discard: new` rechaza mensajes nuevos: el cliente debe conservarlos en su outbox local y reintentarlos/avisar, nunca eliminarlos al recibir un error. Los mensajes expiran a los siete días; los dispositivos que vuelvan después requieren un mecanismo de snapshot/reconciliación.

Revisa mensualmente espacio del volumen, uso y antigüedad de mensajes, logs y vencimiento del certificado. La configuración de un stream no limita por sí sola todos los streams futuros: provisiona y audita streams con una herramienta administrativa y no habilites creación arbitraria para clientes.

## Referencias

- [NATS TLS](https://docs.nats.io/running-a-nats-service/configuration/securing_nats/tls)
- [JetStream streams y límites](https://docs.nats.io/nats-concepts/jetstream/streams)
- [Autenticación descentralizada NATS](https://docs.nats.io/using-nats/developer/connecting/decentralized_auth)
- [Docker Compose networking](https://docs.docker.com/compose/how-tos/networking/)
