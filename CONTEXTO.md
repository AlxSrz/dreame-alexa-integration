# Contexto del proyecto: Dreame D10 Plus Gen 2 → Alexa

**Objetivo:** controlar por voz con Alexa un robot aspiradora **Dreame D10 Plus
Gen 2** (iniciar limpieza, limpiar zonas específicas, usarlo en rutinas de
Alexa). La skill oficial de Dreame para Alexa ya no existe/no está disponible
en México, y publicar una skill propia que la reemplace no es viable (las
skills smart-home de Alexa deben hablar con la nube privada real del
fabricante).

**Por qué esta ruta:** se intentó vincular el robot con la app Xiaomi Home /
Mi Home primero — falló, porque el D10 Plus Gen 2 usa la nube/app propia de
Dreame (**Dreamehome**), separada de Xiaomi, no Mi Home.

## Arquitectura elegida (confirmada con investigación web, sept. 2026)

```
Alexa (voz) → skill privada de Alexa Smart Home → AWS Lambda (puente, Python, free tier permanente)
            → HTTPS (dominio propio + Let's Encrypt) → Nginx reverse proxy en el VPS del usuario
            → Home Assistant (Docker, en el VPS) → integración dreame-vacuum (HACS)
            → cuenta Dreamehome → robot Dreame D10 Plus
```

- La integración comunitaria de Home Assistant `Tasshack/dreame-vacuum` (vía
  HACS) soporta explícitamente `dreame.vacuum.r2205` (D10 Plus) — en modo
  cloud usando las credenciales de Dreamehome (no modo local, porque Home
  Assistant corre en un VPS remoto, no en la misma red que el robot).
- El componente nativo `alexa.smart_home` de Home Assistant permite crear una
  skill de Alexa **privada** (nunca publicada) usando los endpoints OAuth
  propios de Home Assistant (`/auth/authorize`, `/auth/token`).
- El puente requiere una función **AWS Lambda** (código oficial de Home
  Assistant en https://gist.github.com/matt2005/744b5ef548cc13d88d0569eea65f5e5b)
  — gratis indefinidamente a este nivel de uso (free tier permanente de
  Lambda: 1,000,000 invocaciones/mes).
- El usuario ya tiene un **VPS en DigitalOcean** — no se necesita hardware
  nuevo (ej. Raspberry Pi).
- Alternativa descartada: Nabu Casa / Home Assistant Cloud (~$6.5 USD/mes) —
  más simple (sin configurar Lambda/OAuth) pero el usuario prefirió la ruta
  gratuita al self-hosted, ya que tiene el VPS.
- **Pendiente conocido:** el modelo de dispositivo "vacuum" de Alexa solo
  soporta encender/apagar, no "limpia la zona X" por nombre libre. Plan:
  exponer cada zona como un `switch` helper en Home Assistant, cada uno
  disparando un script que llame al servicio de limpieza por zona de la
  integración Dreame, y luego incluir esos switches en el filtro del bloque
  `alexa:`. **Aún no implementado — sigue pendiente.**

## Archivos del proyecto (en esta misma carpeta)

- `docker-compose.yml` — contenedor de Home Assistant.
- `nginx/homeassistant.conf` — reverse proxy HTTPS, con placeholder de dominio
  `ha.TUDOMINIO.com` pendiente de reemplazar.
- `homeassistant/configuration-snippet.yaml` — bloques `http:` y
  `alexa: smart_home:`, con placeholders de `client_id`/`client_secret` de la
  skill pendientes de rellenar.
- `README.md` — guía completa de 7 pasos: Home Assistant en el VPS → dominio
  + HTTPS → integración HACS + Dreame → configuración `alexa:` → skill privada
  en Alexa Developer Console → función AWS Lambda puente → pruebas.

## Estado actual

Nada de esto se ha ejecutado todavía en el VPS real ni en las consolas de
AWS/Amazon — solo existen los archivos de configuración locales y la guía.

**Antes de continuar, hace falta:**
- Confirmar qué dominio se va a usar.
- Confirmar si HACS ya está instalado en Home Assistant.
- Confirmar el estado de las cuentas Dreamehome / AWS / Amazon Developer.

Con esas respuestas se sabe en qué paso del `README.md` retomar el trabajo.
