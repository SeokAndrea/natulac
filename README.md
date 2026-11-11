# Natulac · Aséptico

App web de la planta Aséptico de Natulac: turnos, preparación de tanques, líneas, producto
terminado, paradas, calidad, indicadores, programación y paneles en vivo.

## Documentación

| Documento | Para qué sirve |
|---|---|
| [docs/ARQUITECTURA.md](docs/ARQUITECTURA.md) | Cómo está armado todo y las reglas que conviene conocer antes de tocar algo. **Empezar por aquí.** |
| [MAPA.md](MAPA.md) | Dónde está cada pantalla y su código. |
| [supabase/ESQUEMA.md](supabase/ESQUEMA.md) | Tablas y funciones de la base. |
| [docs/disenos/](docs/disenos/) | Decisiones de diseño de los módulos más recientes. |
| [docs/flujos-lineas-y-producto-terminado.md](docs/flujos-lineas-y-producto-terminado.md) | Flujos de Líneas y Producto Terminado (foto del 2026-09-30). |
| [PRODUCT.md](PRODUCT.md), [DESIGN.md](DESIGN.md) | Qué es el producto, para quién y cómo debe verse. |
| [plc-bridge/Manual-plc-bridge.pdf](plc-bridge/Manual-plc-bridge.pdf) | Manual del servicio que lee el PLC. |

## Cómo está armada

- **App:** React + TypeScript + Vite + Tailwind (`src/`).
- **Base de datos:** PostgreSQL propio en el servidor de planta (`db_control_nectares`; antes
  estaba en Supabase). La app llama funciones de la base por PostgREST. Casi toda la lógica vive
  en esas funciones (`supabase/migrations/`); las tablas no se leen ni escriben directo.
- **Archivos:** los PDF de las actas los guarda un servicio propio (`archivos/`).
- **Correo:** un servicio (`correo/`) manda los avisos de Solicitudes de Inspección / Intervención.
- **PLC:** `plc-bridge/` lee el PLC de Preparación y lo reenvía por WebSocket.
- **Sesión:** login propio (`verificar_login`); cada petición lleva una credencial en el header
  `x-sesion`.

## Requisitos

- Node 22 y npm (es la versión que usa el `Dockerfile`).
- Docker y Docker Compose, para el servidor.
- Acceso al PostgreSQL de planta (host, base, usuario y clave).
- Para el PLC: que el equipo que corre `plc-bridge` vea la red del PLC.

## Configuración

Copiar `.env.example` como `.env.local` (ya está en `.gitignore`) y completar:

- `DB_*`: conexión a PostgreSQL. Los usa solo el contenedor de PostgREST; nunca llegan al navegador.
- `SMTP_*`, `CORREO_DE`, `CORREO_PARA`, `APP_URL`: correo de los avisos. Vacíos = no se manda nada. Por ahora no están configurados: la cuenta de correo la definirá informática.
- `VITE_PLC_BRIDGE_WS_URL`: opcional; si no se define usa `ws://192.168.10.143:4036`.

Nunca subir `.env.local` ni ningún `.env`.

## Desarrollo

```
npm install
npm run dev      # servidor de desarrollo (puerto 5173)
npm test         # pruebas (Vitest)
npm run lint     # revisión de estilo (oxlint)
npm run build    # compilación de producción
```

En desarrollo, Vite reenvía `/api` y `/archivos` a los servicios del servidor de planta
(`vite.config.ts`). Se pueden cambiar con las variables `API_DESTINO` y `ARCHIVOS_DESTINO`.

## En el servidor (Docker)

```
docker compose --env-file .env.local --profile preview up -d --build
```

| Servicio | Puerto | Qué es |
|---|---|---|
| `preview` | 4035 | La app. |
| `plc-bridge` | 4036 | WebSocket del PLC. |
| `api` | 4037 | PostgREST. |
| `archivos` | 4038 | PDF de las actas. |
| `correo` | — | Avisos por SMTP. |

Los PDF de las actas quedan en `../natulac-actas` y los registros del plc-bridge en
`../natulac-plc-datos`, al lado de la carpeta del proyecto. Quedan fuera del proyecto a
propósito, para que sincronizarlo no los borre.

## Base de datos

- `supabase/migrations/`: cada cambio de la base, en orden por nombre. Se aplican en ese orden y
  no se editan una vez aplicadas. Un cambio nuevo es una migración nueva.
- Aplicar una migración al PostgreSQL de planta, desde la carpeta del proyecto en el servidor:
  `bash scripts/aplicar-migracion.sh supabase/migrations/<archivo>.sql`. Usa los `DB_*` de
  `.env.local`, corre cada archivo en una sola transacción y recarga PostgREST al final.
- `scripts/revisar-personal-puestos.sql`: consulta de solo lectura que usa el procedimiento de
  aplicación de Personal (ver `supabase/ESQUEMA.md`).

## PLC

`plc-bridge/` corre en modo simulación (datos inventados) o real. La configuración (IP del PLC,
puerto, tags) está en `plc-bridge/bridge.conf`; el manual explica cada ajuste. En una PC de
prueba se arranca con `plc-bridge/plc-bridge.bat`.
