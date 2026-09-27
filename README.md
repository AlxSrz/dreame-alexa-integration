# Dreame D10 Plus Gen 2 → Alexa (vía Home Assistant en tu VPS)

Guía paso a paso para controlar el robot por voz con Alexa, sin depender de una
skill oficial de Dreame (que ya no existe para México). Arquitectura:

```
Alexa (voz) → Skill privada → AWS Lambda (gratis) → HTTPS (Nginx + Let's Encrypt)
            → Home Assistant (Docker, en tu VPS) → integración dreame-vacuum (HACS)
            → cuenta Dreamehome → robot D10 Plus
```

Archivos de este repo:
- `docker-compose.yml` — levanta Home Assistant en el VPS.
- `nginx/homeassistant.conf` — reverse proxy HTTPS.
- `homeassistant/configuration-snippet.yaml` — bloques a fusionar en la config de HA.

---

## Paso 1 — Levantar Home Assistant en el VPS

Conéctate por SSH a tu VPS de DigitalOcean y, en la carpeta donde subas este
proyecto (o clonando este repo si lo subes a git):

```bash
mkdir -p homeassistant/config
docker compose up -d
```

Verifica que responda localmente:

```bash
curl -I http://localhost:8123
```

Entra por primera vez a `http://IP_DEL_VPS:8123` (temporalmente, antes del HTTPS)
para crear tu usuario admin de Home Assistant.

## Paso 2 — Dominio + HTTPS válido

1. Crea un registro DNS tipo A: `ha.tudominio.com` → IP de tu VPS.
2. Instala Nginx y Certbot en el VPS:
   ```bash
   sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
   ```
3. Copia `nginx/homeassistant.conf` a `/etc/nginx/sites-available/homeassistant`,
   reemplaza `TUDOMINIO.com` por tu dominio real, y enlázalo:
   ```bash
   sudo ln -s /etc/nginx/sites-available/homeassistant /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```
4. Emite el certificado (Certbot edita el `.conf` automáticamente):
   ```bash
   sudo certbot --nginx -d ha.tudominio.com
   ```
5. Confirma que `https://ha.tudominio.com` carga Home Assistant con candado
   válido (sin advertencias del navegador).
6. Aplica el snippet `homeassistant/configuration-snippet.yaml` dentro de
   `homeassistant/config/configuration.yaml` (bloque `http:` con
   `trusted_proxies`) y reinicia:
   ```bash
   docker compose restart homeassistant
   ```

## Paso 3 — Vincular el robot (integración Dreame Vacuum vía HACS)

1. Instala **HACS** en Home Assistant siguiendo https://hacs.xyz/docs/use/download/download/
   (requiere reiniciar HA una vez instalado).
2. En Home Assistant: **Configuración → Dispositivos y servicios → HACS →
   Integraciones → Explorar y descargar repositorios**, busca
   `Tasshack/dreame-vacuum` (si no aparece, añádelo como repositorio
   personalizado con la URL `https://github.com/Tasshack/dreame-vacuum`).
3. Reinicia Home Assistant tras instalar.
4. **Configuración → Dispositivos y servicios → Añadir integración → Dreame
   Vacuum**. Cuando pida cuenta, usa el **correo y contraseña de la app
   Dreamehome** (modo cloud — el VPS no está en tu red local, así que no uses
   el modo "local/IP").
5. Debería aparecer una entidad `vacuum.dreame_...` y entidades de habitación
   generadas automáticamente. Pruébalo desde el dashboard de HA (iniciar,
   pausar, limpiar una zona) antes de seguir con Alexa.

## Paso 4 — Configurar el bloque `alexa:` en Home Assistant

Ya está en `homeassistant/configuration-snippet.yaml` — pero **espera al Paso
5** para llenar `client_id`/`client_secret` con los valores reales que generes
en la skill, luego reinicia Home Assistant.

## Paso 5 — Crear la skill privada en Alexa Developer Console

1. Crea/entra a tu cuenta en https://developer.amazon.com/alexa/console/ask
   (gratis, con tu cuenta de Amazon).
2. **Create Skill** → nombre (ej. "Mi Casa") → modelo **Smart Home** → método
   **Provision your own** (no publiques en la tienda, quedará privada/en modo
   desarrollo).
3. En la sección **Account Linking** de la skill, configura:
   - Authorization URI: `https://ha.tudominio.com/auth/authorize`
   - Access Token URI: `https://ha.tudominio.com/auth/token`
   - Client ID: `https://pitangui.amazon.com/` (o el dominio Alexa de tu
     región — revisa la tabla oficial en la doc de HA si usas otra región)
   - Client Secret: cualquier cadena que inventes (Home Assistant no la valida,
     pero debe coincidir con la que pongas en `configuration.yaml`)
   - Scope: `smart_home`
4. Copia el **Skill ID** (lo necesitas en el paso 6) y pega el mismo
   `client_id`/`client_secret` en `homeassistant/configuration-snippet.yaml`,
   luego reinicia Home Assistant.

## Paso 6 — Función AWS Lambda (puente, gratis)

1. Crea cuenta AWS si no tienes (https://aws.amazon.com/) — no se cobra
   mientras te mantengas en el free tier (ver nota de costos abajo).
2. En **IAM**, crea un rol para Lambda con la política gestionada
   `AWSLambdaBasicExecutionRole`.
3. En **Lambda**, crea una función nueva:
   - Runtime: Python 3.12
   - Región: **us-east-1** (Norteamérica/México) — debe coincidir con la
     región que use tu skill de Alexa.
   - Rol de ejecución: el creado en el paso anterior.
4. Reemplaza el código de ejemplo por el script oficial de Home Assistant:
   https://gist.github.com/matt2005/744b5ef548cc13d88d0569eea65f5e5b
5. Variables de entorno de la función:
   - `BASE_URL` = `https://ha.tudominio.com` (sin `/` al final)
   - (`DEBUG=True` opcional mientras pruebas)
6. Añade un **trigger "Alexa Smart Home"** e ingresa el Skill ID del paso 5.
7. Copia el **ARN** de la función Lambda y pégalo en el campo **Default
   endpoint** de la skill, en Alexa Developer Console (pestaña Smart Home).

## Paso 7 — Probar

1. En la app de Alexa (con la misma cuenta de Amazon del desarrollador),
   ve a **Más → Skills y juegos → Tus skills → Dev** y activa tu skill.
2. Completa el account linking (te pedirá login contra tu Home Assistant).
3. Di **"Alexa, descubre dispositivos"** — debería encontrar el robot (y sus
   zonas, si las expusiste como entidades separadas).
4. Prueba: *"Alexa, enciende [nombre del robot]"* para iniciar limpieza,
   *"Alexa, apaga [nombre del robot]"* para detener/regresar a base.
5. Crea una rutina en la app de Alexa que incluya el dispositivo del robot.

---

## Notas de costo

- **AWS Lambda**: free tier permanente de 1,000,000 invocaciones/mes — el uso
  doméstico de esta skill nunca se acerca a ese límite. Costo esperado: **$0/mes**.
- **AWS/Amazon Developer**: cuentas gratuitas.
- **Dominio**: si no tienes uno, ~$10-15 USD/año (único costo recurrente real
  de esta configuración, aparte del VPS que ya pagas).

## Notas importantes

- El robot usa comandos tipo "encender/apagar" en el modelo de dispositivo
  Alexa para vacuum (no hay un intent nativo de Alexa para "limpiar la sala X"
  por nombre libre; para eso conviene exponer zonas como `switch` helpers en
  Home Assistant, cada uno disparando un script que llame al servicio de
  limpieza por zona de la integración Dreame, y luego incluir esos switches en
  el `filter.include_entities` del bloque `alexa:`).
- Si cambias la IP del VPS o vence el certificado sin renovarse
  automáticamente (Certbot instala renovación automática vía cron/systemd
  timer, verifícalo con `sudo certbot renew --dry-run`), el control por voz
  dejará de funcionar hasta corregirlo.
