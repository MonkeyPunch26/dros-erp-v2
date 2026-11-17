# Servicios extra para dros-servidor (sobre el stack odoo-dros)

Contenido: Caddy, AdGuard Home, Uptime Kuma, Vaultwarden, Homepage, pg_dump diario y restic.
Ya están adaptados a tu repo (servicio `postgres`, red `erp-network`, volúmenes `odoo_data` y `n8n_data`).
pgAdmin y Portainer ya los tienes en tu compose, por eso no se repiten (solo se les pone nombre por Caddy).
Nada se ha probado en tu servidor: valida con `docker compose config` y revisa logs en el primer arranque.

## 1. Preparación
Deja esta carpeta (`dros-extras`) junto a tu repo, por ejemplo `/home/TU_USUARIO/dros-extras`.
```bash
cd ~/dros-extras
cp .env.example .env && nano .env
echo ".env" >> .gitignore                       # no subir claves a GitHub
docker network ls | grep erp                    # debe ser odoo-dros_erp-network (ERP_NETWORK)
docker volume ls | grep -E 'odoo_data|n8n_data'
sudo mkdir -p /mnt/respaldo/postgres /mnt/respaldo/restic   # en el disco de respaldo, ya montado
docker compose -f docker-compose.extras.yml config > /dev/null && echo OK
```

## 2. Aplicar los 3 cambios al ERP para que funcione detrás del proxy
Están en `erp-proxy.patch`. Desde la carpeta del repo:
```bash
cd ~/odoo-dros
git apply --check ~/dros-extras/erp-proxy.patch && git apply ~/dros-extras/erp-proxy.patch
docker compose up -d odoo n8n superset superset-worker superset-worker-beat
```
Qué hace: Odoo arranca con `--proxy-mode`; n8n recibe `N8N_PROXY_HOPS`, `WEBHOOK_URL` y `N8N_EDITOR_BASE_URL`;
Superset activa `ENABLE_PROXY_FIX`. Revísalo con `git diff` antes de hacer commit.
(Superset se reconstruye solo si cambia su Dockerfile; aquí solo cambió la config, que se monta como archivo.)

## 3. Liberar el puerto 53 (Ubuntu lo ocupa con systemd-resolved)
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNS=1.1.1.1\nDNSStubListener=no\n' | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

## 4. IP fija
Reserva en el router (DHCP estático) una IP para el servidor. Todo el DNS depende de ella.

## 5. Levantar
```bash
cd ~/dros-extras
docker compose -f docker-compose.extras.yml up -d
docker compose -f docker-compose.extras.yml ps
```

## 6. AdGuard Home y router
1. Abre `http://IP-DEL-SERVIDOR:3000` y haz el asistente. **Deja el puerto del panel en 3000** y DNS en 53.
2. Filtros > Reescrituras DNS: crea **una por subdominio** apuntando a la IP del servidor:
   `ventas`, `reportes`, `automatizacion`, `pgadmin`, `portainer`, `monitoreo`, `claves`, `dns`, `inicio` (todos `.dros.cl`).
   **No uses comodín `*.dros.cl`**: dejaría sin resolver `www.dros.cl` (Vercel) dentro del local.
3. En el DHCP del router pon como DNS la IP del servidor. Si agregas un DNS público de respaldo, algunos
   equipos lo usarán y no resolverán los subdominios; si no lo agregas, sin el servidor el local pierde internet.
   Elige según tu tolerancia (el UPS ayuda).

## 7. Certificado HTTPS interno
```bash
docker compose -f docker-compose.extras.yml cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-root.crt
```
- Windows (caja y notebooks): doble clic > Instalar certificado > Equipo local > "Entidades de certificación raíz de confianza".
- iPhone: envíate el archivo, instálalo como perfil y actívalo en Ajustes > General > Información > Confianza de certificados.

## 8. Cierra los puertos directos (después de comprobar que todo abre por nombre)
En tu `docker-compose.yml`, los puertos 8069, 5678, 8088, 5050 y 9443 están abiertos a toda la red del local.
Cuando ya entres por `https://...dros.cl`, cámbialos a `127.0.0.1:` (ej. `"127.0.0.1:8069:8069"`) para que solo se pueda pasar por Caddy.
Antes revisa que el mini PC de la caja y el flujo de Tailscale usen los nombres y no `IP:8069`.

## 9. Acceso remoto del dueño (opcional, sin probar)
Los subdominios apuntan a la IP del local, que desde fuera no se alcanza. Con Tailscale:
en el servidor `sudo tailscale up --advertise-routes=IP-DEL-SERVIDOR/32`, apruébala en la consola de Tailscale
y en DNS agrega un nameserver con la IP Tailscale del servidor, limitado al dominio `dros.cl` (split DNS).

## 10. Vaultwarden
1. Entra a `https://claves.dros.cl/admin` con la clave cuyo hash pusiste en `VW_ADMIN_TOKEN`.
2. Invita a los 4 usuarios (el registro abierto está desactivado).

## 11. Uptime Kuma
`https://monitoreo.dros.cl`: crea el admin y monitores HTTP para `http://odoo:8069`, `http://superset:8088/health` y `http://n8n:5678/healthz`,
más uno TCP a `postgres:5432`. Configura notificación por Telegram o correo.

## 12. Respaldos
- `pgbackup` guarda dumps de `odoo`, `n8n` y `superset` en `/mnt/respaldo/postgres/{daily,weekly,monthly,last}`.
- `restic` guarda copias cifradas e incrementales en `/mnt/respaldo/restic` de: filestore de Odoo, datos de n8n,
  los dumps y la **carpeta del repo, que incluye tu `.env`**. Ese `.env` guarda `N8N_ENCRYPTION_KEY` y
  `SUPERSET_SECRET_KEY`: sin ellas, aunque restaures las bases, las credenciales guardadas en n8n no se pueden descifrar.
  También incluye `odoo/addons` (tus módulos propios, como el de validación de Carabineros).
- Si restic no inicializa el repo solo: `docker compose -f docker-compose.extras.yml exec restic restic init`
- **Copia fuera del equipo**: para Backblaze B2 cambia `RESTIC_REPOSITORY` a `b2:NOMBRE-BUCKET:dros` y añade
  `B2_ACCOUNT_ID` y `B2_ACCOUNT_KEY` en `environment` del servicio restic. (Google Drive requiere rclone: otra configuración.)
- **Los dumps no incluyen los roles** (`odoo`, `n8n`, `superset`). En un servidor nuevo, tu `01-init-databases.sh` los crea al primer arranque de `postgres_data`; luego restauras cada base.
- **Prueba una restauración** antes de confiar en esto:
  ```bash
  ls /mnt/respaldo/postgres/last/
  docker exec -i erp-postgres createdb -U postgres -O odoo odoo_prueba
  zcat /mnt/respaldo/postgres/last/odoo-latest.sql.gz | docker exec -i erp-postgres psql -U postgres -d odoo_prueba
  ```
  (ajusta el nombre del archivo a lo que muestre `ls`; borra `odoo_prueba` al terminar).

## Pendientes recomendados
- Apagado automático por el UPS (NUT o apcupsd), instalado en el host, no en Docker.
- Fijar versiones: n8n (`N8N_VERSION`), pgAdmin y las imágenes `latest` de este compose, cuando todo funcione.
  Actualiza a mano y con respaldo previo, no con Watchtower.
