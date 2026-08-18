# Correo de solo recepción — relay Postfix (IP pública) + docker-mailserver (Tailscale)

Setup de un servidor de correo de solo recepción usando dos máquinas conectadas por una red
privada [Tailscale](https://tailscale.com):

- **Relay** (Ubuntu, IP pública): Postfix en modo relay puro (sin buzones) recibe en el puerto 25
  desde internet y reenvía por Tailscale hacia el servidor local. También aloja el webmail
  (Roundcube + nginx + Let's Encrypt).
- **Servidor local** (sin IP pública): [`docker-mailserver`](https://github.com/docker-mailserver/docker-mailserver)
  (Postfix + Dovecot) almacena los correos reales, solo alcanzable vía Tailscale.

La guía completa, paso a paso, está en [`setup-correo-relay-tailscale.md`](./setup-correo-relay-tailscale.md).

## Contenido de este repo

```
docker/
  srv-wb/docker-compose.yml           # docker-mailserver (servidor local)
  oracle-roundcube/docker-compose.yml # Roundcube (relay)
nginx/
  mail.example.com.conf               # proxy nginx + Certbot para el webmail
systemd/
  oracle-socks-proxy.service          # túnel SOCKS5 (workaround de red, ver Fase 1.4 del setup)
  tailscaled-override.conf            # override de tailscaled para usar el proxy solo en el control-plane
setup-correo-relay-tailscale.md       # guía completa del despliegue, fase por fase
```

## Qué NO está en este repo (y por qué)

- **Buzones reales / `docker-data/`**: contenido de correo real de usuarios.
- **`postfix-accounts.cf`, `dovecot-masters.cf`**: hashes de contraseñas de las cuentas.
- **Certificado privado (`*-key.pem`)**: llave privada del self-signed de docker-mailserver.
- **Llave SSH privada**: usada para administrar el relay. El setup solo referencia su ruta,
  nunca su contenido.
- **`recipient-bcc-map`**: mapeo real de cuentas internas.

Todos esos quedan excluidos por [`.gitignore`](./.gitignore) y, además, nunca se generaron dentro
de esta carpeta — viven en el volumen de datos del servidor local, fuera de este repo.

## Valores a reemplazar

Este repo es público, así que las IPs reales, el dominio y las credenciales fueron reemplazados
por placeholders (`<IP_PUBLICA_ORACLE>`, `<IP_TAILSCALE_SRV-WB>`, `mail.example.com`, etc.). Para
reusar este setup, sustituye esos placeholders por tus propios valores — nunca los subas de vuelta
a un repo público.
