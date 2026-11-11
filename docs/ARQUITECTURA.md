# Arquitectura

Cómo está armada la app de la planta Aséptico y por qué. Para ubicar una pantalla o un
archivo concreto, ver [MAPA.md](../MAPA.md). Para las tablas y funciones de la base, ver
[supabase/ESQUEMA.md](../supabase/ESQUEMA.md).

## Piezas

```
Navegador (React)
      │  mismo servidor (sin CORS)
      ▼
Vite preview :4035 ──► /api       ──► PostgREST :4037 ──► PostgreSQL (db_control_nectares)
                  └──► /archivos  ──► servicio archivos :4038 ──► carpeta de PDF de actas

plc-bridge :4036 ◄── WebSocket ── navegador (pantalla Preparación PLC)
plc-bridge ──► PLC S7-1200 (lectura de tanques y válvulas)
correo ──► SMTP (avisos de Solicitudes de Inspección / Intervención)
```

| Pieza | Carpeta | Qué hace |
|---|---|---|
| App web | `src/` | React + TypeScript + Vite + Tailwind. Todas las pantallas. |
| Base de datos | `supabase/migrations/` | PostgreSQL propio. Casi toda la lógica vive en funciones SQL. |
| API | contenedor `api` | PostgREST: expone las funciones de la base como HTTP. |
| Archivos | `archivos/` | Guarda y sirve los PDF de las actas. Sin dependencias, solo Node. |
| Correo | `correo/` | Manda los avisos por SMTP si `SMTP_*` y `CORREO_PARA` están puestos. |
| PLC | `plc-bridge/` | Lee el PLC de Preparación y lo reenvía por WebSocket. Tiene modo simulación. |

Todo se levanta con `docker-compose.yml` (ver [README.md](../README.md)).

## Reglas que conviene conocer antes de tocar algo

1. **La lógica vive en la base, no en el navegador.** Las tablas tienen RLS activado y sin
   políticas: nadie las lee ni escribe directo. Todo pasa por funciones (`supabase.rpc(...)`)
   que validan permisos y reglas de negocio. El navegador solo muestra y llama.
2. **Login propio, no de Supabase.** `verificar_login` entrega una credencial al entrar y la app
   la manda en cada petición (header `x-sesion`, ver `src/lib/sesionToken.ts`). La base la
   usa para saber quién llama; no confía en el usuario que manda la pantalla.
3. **Permisos por puesto.** Cada persona tiene un puesto (área + nivel + permisos) y puede
   tener permisos extra. La lista está en `src/lib/permisos.ts` y se valida en el servidor con
   `tiene_permiso()`. En el navegador solo sirve para mostrar u ocultar tarjetas.
   **El usuario maestro** está un nivel por encima del Super Administrador y se pensó como
   responsable técnico. En el código y en la base se llama «dueño» (`es_dueno()`, `esDueno`,
   columna `usuarios.ve_errores`). Nadie más lo edita, desactiva ni elimina; da o quita el rol
   Super Administrador, ve la pantalla de Errores y cambia los ajustes de planta
   (`configuracion_app`). Por eso muchos comentarios dicen «solo el usuario maestro». Es un
   indicador, no un rol (ver `20261079090000_dueno.sql`).
4. **Las migraciones son el historial del esquema.** Están en `supabase/migrations/`, se
   aplican en orden por nombre y no se editan una vez aplicadas en el servidor. Un cambio
   nuevo es una migración nueva. `supabase/ESQUEMA.md` es solo un resumen de consulta.
5. **Hora de planta.** Toda fecha y hora pasa por `src/lib/tiempoPlanta.ts`
   (America/Caracas, UTC−4). No usar `new Date()` suelto ni `now()` de Postgres sin convertir.
   La jornada va de 7:00 a 7:00: la madrugada pertenece al turno 3 del día anterior.
6. **El nombre `supabase` es histórico.** La base antes estaba en Supabase. La app conserva el
   nombre `supabase` en `src/lib/supabase.ts` para no cambiar decenas de importaciones, pero
   hoy usa solo `@supabase/postgrest-js` contra el PostgreSQL propio.

## Dominio: tres módulos y una capa de reportes

El turno es la unidad de trabajo. Dentro de un turno hay tres módulos, cada uno responsable de sus
tablas y de su carpeta en `src/lib/`:

| Módulo | Responsable de | Código |
|---|---|---|
| Preparación | tanques y lotes | `src/lib/preparacion/` |
| Producción | líneas, corridas y contadores | `src/lib/produccion/` |
| Producto Terminado | paletas y cajas por corrida | `src/lib/productoTerminado.ts` |

- Dentro de Preparación y Producción, `nucleo.ts` tiene el ciclo normal (iniciar → liberar;
  activar → pausar → terminar) y `ajustes.ts` lo que sale distinto de lo planeado (ajustes,
  transferencias, correcciones). `tipos.ts` y `mapear.ts` definen y convierten los datos.
- **Reportes** (`src/lib/reportes/`) no tiene tablas propias: lee los tres módulos y calcula
  merma, meta y rendimiento. Los módulos entregan números crudos; la resta y la división
  se hacen solo acá.
- `src/lib/sesionTurno.tsx` da a los tres módulos la identidad del turno (cuál es, quién lo
  tiene, de qué área). Es lo único compartido entre ellos.
- Una corrida es una fila de `turno_lineas`: cada cambio de sabor o presentación en una línea
  abre una corrida nueva.

Otros módulos, más pequeños, siguen el mismo patrón (pantalla en `src/pages/apps/`, lógica en
`src/lib/`, funciones en la base): Paradas, Mantenimiento, Solicitudes de Intervención (SIF),
Calidad, Inventario, Personal, Indicadores, Programación, Validar jornada y Robot Tavil.

## Tareas programadas

Corren dentro de PostgreSQL con `pg_cron`:

- `cerrar-turnos-vencidos`: cada 5 minutos, cierra los turnos que pasaron su hora.
- `relevo-de-respaldo`: cada 5 minutos, cubre un relevo que no se hizo.
- `paradas-mueven-lineas`: cada minuto, aplica las paradas al estado de las líneas.

## Decisiones de diseño por módulo

Las decisiones más recientes y su motivo están en [docs/disenos/](disenos/). Cada migración
nueva cita el diseño que la originó en su comentario inicial.

## Pruebas

- `npm test`: pruebas de la app con Vitest (`src/**/*.test.ts(x)`).
- `npm run lint`: revisión de estilo con oxlint.
- Las funciones de la base no tienen pruebas automáticas en este repositorio.
