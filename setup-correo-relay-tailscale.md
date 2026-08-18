# Arquitectura: correo de solo recepción — relay en Oracle Cloud + docker-mailserver local vía Tailscale

## Resumen

Sistema de correo electrónico de solo recepción, repartido en dos máquinas conectadas por una
red privada [Tailscale](https://tailscale.com) (mesh VPN sobre WireGuard):

- **Relay** (VM Oracle Cloud, capa Always Free, con IP pública): recibe el correo desde internet
  y lo reenvía; también expone el webmail.
- **Servidor local** (`srv-wb`, sin IP pública): almacena los correos reales y hace todo el
  procesamiento pesado (filtrado de spam, DKIM/DMARC, IMAP).

Ninguna de las dos máquinas por sí sola resolvía el problema completo, así que la arquitectura
nace de combinar lo que cada una sí puede ofrecer.

---

## Por qué esta arquitectura

**El problema de fondo:** para recibir correo de internet hace falta una IP pública fija (el
registro MX del dominio tiene que apuntar a algo alcanzable). La máquina con mejor hardware
disponible para este proyecto es un servidor local, sin IP pública — está detrás del NAT de un
ISP residencial, con IP dinámica y, además, con el puerto 25 de salida bloqueado (algo común en
conexiones residenciales, pensado para frenar spam saliente de equipos comprometidos).

La opción para conseguir una IP pública fija y gratuita es una VM de **Oracle Cloud Always Free**.
Pero ese tier gratuito tiene recursos muy limitados (CPU y RAM compartida/reducida, pensada para
cargas ligeras) — insuficiente para correr con margen un stack de correo completo
(Postfix + Dovecot + Amavis + SpamAssassin + OpenDKIM/OpenDMARC + almacenamiento de buzones) de
forma sostenida.

**La solución adoptada — dividir responsabilidades por lo que cada máquina puede aportar:**

| Máquina | Aporta | Hace |
|---|---|---|
| VM Oracle Cloud (Always Free) | IP pública fija | Solo relay SMTP puro (sin buzones) + proxy del webmail |
| Servidor local | Recursos (CPU/RAM/disco) | Todo el stack de correo real: Postfix + Dovecot + filtrado + almacenamiento |

La VM de Oracle nunca ve ni guarda el contenido de los correos — solo los recibe en el puerto 25
y los reenvía de inmediato. Todo el trabajo pesado (antispam, firma/verificación DKIM, entrega a
buzones IMAP) ocurre en el servidor local, que tiene el hardware para sostenerlo. El puente entre
ambas es Tailscale: le da al servidor local una identidad de red alcanzable de forma privada,
sin necesitar su propia IP pública ni abrir puertos hacia internet.

---

## Topología de red

```
                            INTERNET
                                │
                     DNS: MX/A → IP pública Oracle
                                │
                                ▼
                 ┌───────────────────────────┐
                 │   VM Oracle Cloud          │
                 │   (Always Free, IP pública)│
                 │                            │
                 │  :25  Postfix (relay puro) │
                 │  :443 nginx + Let's Encrypt│──┐
                 │        → Roundcube :8080   │  │ proxy_pass local (127.0.0.1)
                 └─────────────┬──────────────┘  │
                                │                 │
                     túnel privado Tailscale      │
                    (WireGuard, red 100.64.0.0/10)│
                                │                 │
                                ▼                 ▼
                 ┌───────────────────────────────────┐
                 │   Servidor local (srv-wb)          │
                 │   (sin IP pública, más recursos)   │
                 │                                    │
                 │  docker-mailserver, expuesto SOLO   │
                 │  en la IP de Tailscale:             │
                 │  :25   Postfix (recepción real)    │
                 │  :143/993  Dovecot IMAP/IMAPS      │
                 │                                    │
                 │  Amavis + SpamAssassin + OpenDKIM  │
                 │  + OpenDMARC + buzones en disco    │
                 └────────────────────────────────────┘
```

**Puntos clave de esta topología:**

- El servidor local **nunca expone puertos a la red local ni a internet**: `docker-mailserver`
  publica sus puertos ligados específicamente a la IP de Tailscale de la máquina (no `0.0.0.0`),
  así que solo es alcanzable desde dentro del tailnet.
- El relay de Oracle **no tiene información de las cuentas ni de los buzones** — solo sabe a qué
  IP de Tailscale reenviar el dominio completo (`transport_maps` de Postfix).
- El webmail (Roundcube) corre en Oracle —no en el servidor local— porque el navegador del
  usuario siempre llega primero a la IP pública; Roundcube actúa ahí como cliente IMAP/SMTP
  hacia el servidor local, igual que cualquier otro cliente de correo, pero conectando por la
  red privada de Tailscale en vez de por internet.
- El certificado público (Let's Encrypt) solo es viable en la VM de Oracle, porque el reto
  HTTP-01 de Let's Encrypt necesita que el dominio resuelva a una IP donde efectivamente se pueda
  responder la verificación — y esa IP es la de Oracle, no la del servidor local.
- Tailscale usa NAT traversal (P2P directo cuando es posible) con caída a relés DERP de la propia
  red de Tailscale cuando hay NAT/firewalls intermedios que impiden la conexión directa —
  transparente para las aplicaciones, el túnel WireGuard se mantiene igual en ambos casos.

---

## Flujo de un correo entrante

1. Alguien en internet envía un correo a `usuario@mail.example.com`.
2. El DNS resuelve el registro MX del dominio hacia la IP pública de la VM Oracle.
3. Postfix en Oracle acepta la conexión SMTP en el puerto 25 (sin guardar el correo — no tiene
   `mydestination` local, solo `relay_domains` + `transport_maps`).
4. Postfix reenvía el mensaje por el túnel de Tailscale hacia la IP privada del servidor local,
   puerto 25.
5. Postfix en el servidor local recibe el mensaje y lo pasa por Amavis (SpamAssassin + verificación
   DKIM/DMARC).
6. Si pasa el filtrado, se entrega vía LMTP a Dovecot, que lo guarda en el buzón del usuario.

## Flujo de acceso al webmail / cliente de correo

1. El usuario visita `https://mail.example.com` (o configura un cliente IMAP apuntando ahí).
2. nginx en Oracle termina TLS con el certificado de Let's Encrypt y hace proxy hacia Roundcube
   (que solo escucha en `127.0.0.1`, no accesible directamente desde fuera).
3. Roundcube se conecta como cliente IMAPS/SMTPS al servidor local, usando su IP de Tailscale —
   el mismo patrón que el relay de correo, pero en sentido lectura en vez de entrega.

---

## Estructura del repositorio

```
docker/
  srv-wb/docker-compose.yml
  srv-wb/fail2ban-jail.cf
  oracle-roundcube/docker-compose.yml
  daemon.json
nginx/
  mail.example.com.conf
  conf.d/rate-limit.conf
fail2ban/
  roundcube-auth.conf
  mail.local
systemd/
  oracle-socks-proxy.service
  tailscaled-override.conf
```

### `docker/srv-wb/docker-compose.yml`

Levanta [`docker-mailserver`](https://github.com/docker-mailserver/docker-mailserver) (imagen que
empaqueta Postfix + Dovecot + Amavis + SpamAssassin + OpenDKIM/OpenDMARC + fail2ban) en el
servidor local. Puntos relevantes de la configuración:

- Los puertos (`25`, `143`, `993`) se ligan explícitamente a la IP de Tailscale de la máquina —
  no a `0.0.0.0` — para que el servicio solo sea alcanzable desde el tailnet.
- `SSL_TYPE=self-signed`: como esta máquina no tiene IP pública, no puede pasar el reto HTTP-01
  de Let's Encrypt, así que usa un certificado autofirmado para IMAPS (Roundcube, al conectarse
  por Tailscale, desactiva la verificación de ese certificado — ver más abajo).
  Nota: Tailscale puede emitir certificados TLS reales (`tailscale cert`) para nodos del tailnet;
  es una alternativa mejor al self-signed si se habilita esa función en la cuenta.
- `ENABLE_CLAMAV=0`: desactivado por presupuesto de RAM del equipo; no es crítico porque el
  servicio es solo de recepción interna, no relay saliente de terceros.
- El `hostname` del contenedor es distinto al `myhostname` del relay de Oracle a propósito: si
  ambos nodos se identifican igual en el saludo SMTP (HELO/EHLO), Postfix interpreta que el
  correo "vuelve a sí mismo" y lo rebota — son dos identidades SMTP distintas aunque compartan
  dominio de correo.

### `docker/oracle-roundcube/docker-compose.yml`

Levanta [Roundcube](https://roundcube.net) en la VM de Oracle como interfaz web de correo. Solo
publica el puerto `8080` en `127.0.0.1` (nginx es el único punto de entrada público — ver
siguiente archivo). Se conecta al servidor local vía IMAPS (`993`) y SMTPS (`465`) sobre la IP de
Tailscale, nunca en texto plano: Dovecot exige TLS antes de aceptar credenciales.

### `nginx/mail.example.com.conf`

Proxy inverso en Oracle: termina TLS con un certificado real de Let's Encrypt (gestionado por
Certbot, con renovación automática) y redirige todo el tráfico HTTP a HTTPS. Reenvía las
peticiones a Roundcube (`127.0.0.1:8080`). Este es el único servicio de los cuatro que necesita
—y puede tener— un certificado público válido, porque es el único punto con IP pública real.

### `systemd/oracle-socks-proxy.service`

Túnel SSH en modo SOCKS5 (`ssh -D`) desde el servidor local hacia la VM de Oracle, como servicio
systemd persistente (`Restart=always`). Se usa para que el tráfico de control de Tailscale
(autenticación, coordinación de la red) del servidor local pueda salir por una ruta alterna
cuando la red local interfiere con esas conexiones — el tráfico de datos entre nodos (relés DERP)
no pasa por este túnel, solo el plano de control (ver el siguiente archivo).

### `systemd/tailscaled-override.conf`

Override del servicio `tailscaled` que configura `HTTPS_PROXY`/`HTTP_PROXY` apuntando al túnel
SOCKS5 anterior, pero solo para el dominio de control de Tailscale — la lista `NO_PROXY` excluye
explícitamente todos los relés DERP públicos de Tailscale (para que esos sigan yendo directo,
sin pasar por el túnel SSH, que no soporta el protocolo DERP).

### `nginx/conf.d/rate-limit.conf` + `limit_req` en `nginx/mail.example.com.conf`

Límite de tasa por IP sobre el login del webmail (`1r/s`, ráfaga de 20, con `429` como respuesta).
Es la primera línea de defensa contra fuerza bruta: no distingue usuario legítimo de atacante,
simplemente limita cuántas peticiones por segundo acepta una IP — barato de mantener y no depende
de parsear logs de aplicación.

### `docker/daemon.json`

Desactiva el **userland-proxy** de Docker (`"userland-proxy": false`) — necesario en **ambas**
máquinas. Por defecto, cuando un contenedor publica un puerto ligado a una IP específica (como
hace este setup, ver más abajo), Docker no solo hace NAT vía `iptables`: además levanta un proceso
`docker-proxy` que acepta la conexión externa y abre una *segunda* conexión propia hacia el
contenedor. El contenedor nunca ve al cliente real — ve al `docker-proxy`, cuyo origen es la IP
del gateway del bridge de Docker. Esto rompe cualquier cosa que dependa de la IP real del cliente:
filtrado por IP, geolocalización, y sobre todo **fail2ban**, que terminaría baneando la puerta de
enlace de Docker en vez del atacante (ver la nota de `fail2ban-jail.cf` más abajo). Con
`userland-proxy` desactivado, Docker usa únicamente reglas de `iptables` (DNAT vía conntrack), que
sí preservan la IP de origen real de principio a fin. Requiere reiniciar el daemon
(`systemctl restart docker`) para aplicarse — reinicia todos los contenedores del host, no solo
los de este proyecto.

Nota: esto **no** resuelve el caso de nginx (en Oracle) hablando con Roundcube por
`127.0.0.1:8080` — ahí el "cliente" es el propio host conectándose a su loopback, un hairpin NAT
que Docker necesita enmascarar sí o sí para que el enrutamiento de vuelta funcione, sin relación
con el userland-proxy. Por eso el filtro de `fail2ban/roundcube-auth.conf` no usa la IP de origen
de la conexión TCP, sino la cabecera `X-Real-IP` que nginx ya venía agregando.

### `docker/srv-wb/fail2ban-jail.cf`

Override de `fail2ban` para `docker-mailserver`, con una lista de `ignoreip` que nunca debe
banearse: el rango de bridges de Docker (`172.16.0.0/12`) y, más importante, **todo el rango CGNAT
de Tailscale (`100.64.0.0/10`)**. La razón: todas las conexiones de correo y todas las conexiones
IMAP que hace Roundcube en nombre de los usuarios del webmail llegan desde la misma IP de
Tailscale del relay. Sin este `ignoreip`, unos pocos logins fallidos de *cualquier* usuario del
webmail (típicamente 5-10 en el `bantime`/`findtime` por defecto de varios días de
`docker-mailserver`) terminan baneando esa IP — lo que corta el correo y el webmail para *todos*
los usuarios a la vez, no solo para quien se equivocó de contraseña. La defensa real contra fuerza
bruta tiene que vivir en el borde (el relay, que sí distingue IPs de clientes reales), no en el
servidor local, que solo ve un único origen confiable por diseño.

### `fail2ban/mail.local` + `fail2ban/roundcube-auth.conf`

Jails de fail2ban para la VM de Oracle (no vienen instalados por defecto):

- **`postfix`**: banea IPs que intentan usar el relay para reenviar correo a dominios externos
  (`relay access denied`) o que generan rechazos repetidos — backend `systemd`, porque estas
  instancias no traen `rsyslog`/`/var/log/mail.log`, Postfix loguea directo a `journald`.
- **`roundcube-auth`**: banea fuerza bruta contra el login del webmail. El filtro no usa el campo
  `from` del log de Roundcube (que muestra la IP del gateway de Docker por el hairpin NAT
  explicado arriba), sino el valor de `X-Real-IP` que Roundcube registra entre paréntesis. Requiere
  que Roundcube tenga `log_driver = 'file'` y `log_logins = true` en su `config.inc.php` (por
  defecto viene en `stdout` y sin loguear intentos de login), y que el volumen `./logs` esté
  montado (ver `docker/oracle-roundcube/docker-compose.yml`).

Importante: Ubuntu trae por defecto `/etc/fail2ban/jail.d/defaults-debian.conf` con
`backend = systemd` como valor global — sin `backend = auto` explícito en `[roundcube-auth]`, el
jail intenta leer del *journal* en vez del archivo de log de Roundcube y nunca detecta nada.

---

## Notas de diseño

- La comunicación entre las dos máquinas es exclusivamente por Tailscale — en ningún punto el
  servidor local expone un puerto directamente a internet.
- Si el servidor local se apaga, el relay de Oracle simplemente reintenta la entrega en cola
  (comportamiento estándar de Postfix, durante varios días) — no se pierde correo, se retrasa.
- Esta arquitectura es de **solo recepción**. No está pensada para enviar correo saliente hacia
  terceros (requeriría calentamiento de IP, SPF/DKIM más estrictos y gestión de reputación, fuera
  del alcance de este setup).
