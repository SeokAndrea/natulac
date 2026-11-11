# Mapa del proyecto

Guía para ubicar cada pantalla y su código. Todas las rutas son relativas a la raíz del
repositorio. Se armó revisando el código el 2026-11-11: si algo no coincide con lo que se ve
en el repositorio, vale el código.

Documentos relacionados: [README.md](README.md) (cómo levantar todo),
[docs/ARQUITECTURA.md](docs/ARQUITECTURA.md) (cómo está armado y por qué) y
[supabase/ESQUEMA.md](supabase/ESQUEMA.md) (tablas y funciones de la base).

## 1. Raíz del repositorio

| Ruta | Qué hay |
|---|---|
| `src/` | La app web (React + TypeScript). Ver sección 2. |
| `supabase/migrations/` | Cada cambio de la base, en orden por nombre. La lógica de negocio vive aquí. |
| `plc-bridge/` | Servicio que lee el PLC de Preparación y lo reenvía por WebSocket. Tiene su manual (`Manual-plc-bridge.pdf`) y `bridge.conf`. |
| `correo/` | Servicio que manda por SMTP los avisos de Solicitudes de Inspección / Intervención. |
| `archivos/` | Servicio que guarda y sirve los PDF de las actas. |
| `scripts/` | `aplicar-migracion.sh` (aplica migraciones al PostgreSQL de planta), `revisar-personal-puestos.sql` (consulta de solo lectura del procedimiento de Personal). |
| `public/` | Íconos y fondo de la app (`IconoNatulac.png`, `Logo.webp`, `fondo.webp`, `manifest.webmanifest`). |
| `docs/` | Arquitectura y diseños de los módulos. |
| `docker-compose.yml`, `Dockerfile` | Cómo se levanta todo en el servidor (app, API, archivos, correo, plc-bridge). |
| `vite.config.ts` | Configuración de Vite. Reenvía `/api` a PostgREST y `/archivos` al servicio de PDF. Escribe la versión del build en `version.json`. |
| `index.html` | Página base: título de la pestaña y aplicación del tema y del tamaño de letra antes de cargar. |
| `vitest.config.ts`, `.oxlintrc.json` | Pruebas y revisión de estilo. |
| `components.json` | Configuración de los componentes base (shadcn). |
| `PRODUCT.md`, `DESIGN.md` | Qué es el producto, para quién y cómo debe verse. |

## 2. Cómo está organizado `src/`

| Carpeta o archivo | Qué hay |
|---|---|
| `main.tsx` | Arranque y proveedores: sesión (`auth`), identidad del turno (`sesionTurno`) y catálogos en vivo (`catalogosLive`). |
| `App.tsx` | **Todas las rutas.** Cada pantalla se carga al abrirla. |
| `pages/` | `Login`, `Hub` (inicio con tarjetas), `Ajustes`, `PrimerIngreso`. En `pages/apps/` hay una página por pantalla. |
| `components/` | Piezas de las pantallas. Una carpeta por módulo (ver sección 3), `ui/` con los componentes base y `nav/` con el menú lateral de PC y el botón de navegación. |
| `lib/` | Lógica y acceso a la base. Ver sección 5. |
| `index.css` | Estilos globales, colores (bloques `:root` y `.dark`) y utilidades de los paneles. |

## 3. Pantallas

Para agregar una pantalla nueva: sumar su tarjeta en `src/lib/apps.tsx` (título, ruta, permiso),
crear la página en `src/pages/apps/` y registrar la ruta en `src/App.tsx` con
`<ProtectedRoute app="slug">`. La ruta toma los mismos criterios de la tarjeta.

Los permisos son los de `src/lib/permisos.ts`. Cuando hay varios, basta con tener uno. Las
tarjetas «atajo» aparecen arriba del inicio. «Solo en Pruebas» significa que la tarjeta solo
se muestra en el área de Pruebas.

### Turno (sección Producción)

| Pantalla (ruta) | Qué hace | Permiso | Página | Piezas | Lógica |
|---|---|---|---|---|---|
| Comenzar Turno (`/turno`) | Registra el inicio del turno | `TURNO_ASUMIR` | `ComenzarTurno.tsx` | `RevisionInicioTurno.tsx`, que reúne `EstadoPlantaTabs` y `LineasEstadoPlanta` | `lib/ajustesTurnos.ts`, `lib/turno12x12.ts`, `lib/sesionTurno.tsx` |
| Preparación (`/preparacion`) | Iniciar y liberar tanques; corregir lo heredado | `TURNO_CARGAR` | `Preparacion.tsx` | `EstadoPlantaTabs.tsx`, `tanques/` (un archivo por panel y formulario), `pixel/` | `lib/preparacion/`, `lib/desvases.ts` |
| Líneas (`/lineas`) | Activar, pausar y detener corridas; **Modo Paradas** (parar, anotar, cubrir, terminar fallas) | `TURNO_CARGAR` o `PARADAS_MANTENIMIENTO` | `Lineas.tsx` | `LineasEstadoPlanta.tsx`, `lineas/` (`ModoParadas`, `FormArrancarLinea`, `FormCipLinea`…) | `lib/produccion/`, `lib/paradas.ts`, `lib/estadoLineas.ts`, `lib/lineas.ts`, `lib/cubrirTurno.ts`, `lib/mantenimientoSesion.ts` |
| Producto Terminado y Contador (`/producto-terminado`) | Carga paletas, cajas y contador de envases por corrida | `TURNO_CARGAR` | `ProductoTerminado.tsx` | `producto-terminado/` | `lib/productoTerminado.ts`, `lib/ptTurnoAnterior.ts`, `lib/equivalenciaEnvases.ts` |
| Robot Tavil (`/robot-tavil`) | El operador cuenta las paletas y escribe el Resto; es la precarga que el supervisor confirma | `ROBOT_TAVIL_CARGAR` | `RobotTavil.tsx` | `robot-tavil/TarjetaLineaRobot.tsx`, `producto-terminado/AvisoPrecarga.tsx` | `lib/robotTavil.ts` |
| Finalizar Turno (`/finalizar-turno`) | Revisa lo pendiente, cierra el turno y genera el acta | `TURNO_ASUMIR` | `FinalizarTurno.tsx` | `finalizar-turno/`, `acta/VisorActa.tsx`, `NovedadesTurno.tsx` | `lib/actaPdf.ts`, `lib/novedades.ts`, `lib/reportes/`, `lib/eficiencia.ts` |
| Inventario diario (`/inventario`) | Materia prima y material de empaque, mañana y tarde | `INVENTARIO_CARGAR` | `InventarioDiario.tsx` | `inventario/` | `lib/inventario.ts`, `lib/inventarioDemo.ts` |
| Calculadoras (`/calculadoras`) | Fórmula de producto, bobina y conteo por peso | `CALCULADORAS` | `Calculadoras.tsx` | `calculadoras/` | `lib/formulaMateriaPrima.ts`, `lib/bobina.ts`, `lib/conteoPeso.ts` |
| Reportes de paradas (`/reportes-paradas`) | Revisar paradas y abrir su reporte para imprimir; vista Resumen | `REPORTES_PARADAS_VER` | `ReportesParadas.tsx` | `paradas/` | `lib/paradas.ts`, `lib/reportesResumen.ts`, `lib/ventanasParadas.ts` |
| Solicitudes de Intervención (`/solicitudes-intervencion`) | Fallas repetidas para Mantenimiento: Inspección a la 3.ª vez, Intervención a la 4.ª | `SIF_GESTIONAR`, `PARADAS_REGISTRAR`, `PARADAS_MANTENIMIENTO` o `AUDITORIA_VER` | `SolicitudesIntervencion.tsx` | `sif/` | `lib/sif.ts`, `lib/sifPdf.ts`; el correo lo manda `correo/servidor.mjs` |

### Auditoría

| Pantalla (ruta) | Qué hace | Permiso | Página | Piezas | Lógica |
|---|---|---|---|---|---|
| Resumen Diario (`/resumen-dia`) | La jornada de 7:00 a 7:00: actas, cajas por sabor y por línea, paradas y novedades; la analista corrige el mensaje de WhatsApp | `RESUMEN_VER` | `ResumenDia.tsx` | `resumen-dia/` | `lib/resumenDia.ts`, `lib/resumenDiario.ts`, `lib/observacionesCorrida.ts` |
| Validar jornada (`/validar-jornada`) | La analista revisa el día (cajas, envases, semielaborado, paradas) y lo valida para los KPIs. **Solo en Pruebas** | `VALIDAR` | `ValidarJornada.tsx` | `validar-jornada/` | `lib/validarJornada.ts` |
| Mis Actas (`/mis-actas`) | Actas de los turnos propios (quien ve Auditoría, las de todos) | `TURNO_CARGAR` | `MisActas.tsx` | `acta/VisorActa.tsx` | `lib/historialTurnos.ts`, `lib/actaVisor.ts`, `lib/regenerarActa.ts` |
| Auditoría (`/auditoria`) | Línea de tiempo por turno, registro de cambios, actas y exportación del dataset | `AUDITORIA_VER` | `Historial.tsx` | `AuditoriaTurnos.tsx`, `RegistroCambios.tsx` | `lib/auditoria.ts`, `lib/auditoriaVista.ts`, `lib/historial.ts`, `lib/dataset.ts`, `lib/estadisticas.ts` |

### Paneles y accesos

| Pantalla (ruta) | Qué hace | Permiso | Página | Piezas | Lógica |
|---|---|---|---|---|---|
| Panel de Producción (`/panel-produccion`) | Tanques, meta y merma del turno en curso, en vivo, con selector de fecha y turno | sin permiso propio | `PanelProduccion.tsx` | `panel/` (una pieza por sección), `MarcoPanel.tsx` | `lib/panelProduccion.ts`, `lib/reportes/`, `lib/eficiencia.ts` |
| Panel de Paradas (`/panel-paradas`) | Dashboard de solo lectura del tiempo perdido por línea | sin permiso propio | `PanelParadas.tsx` | `PanelParadasVista.tsx`, `panel-paradas/` | `lib/paradasEnVivo.ts`, `lib/eficienciaPeriodo.ts`, `lib/paradasTurnoCompleto.ts` |
| Modo TV (`/modo-tv`) | Alterna Panel de Producción y Panel de Paradas, un minuto cada uno | el del Panel de Paradas | `ModoTv.tsx` | usa las dos páginas anteriores | |
| Indicadores (`/indicadores`) | Mes por línea y presentación: eficiencia, cajas, merma, paradas y semielaborado; cada mes AUTOMÁTICO o MANUAL | `INDICADORES_VER` o `VALIDAR` | `Indicadores.tsx` | `indicadores/` (el editor manual en `indicadores/editor/`) | `lib/indicadores/` |
| Programación (`/programacion`) | Qué se planificó producir, por sabor y en cajas, por día y por semana | `TURNO_CARGAR` o `PROGRAMACION_EDITAR` | `Programacion.tsx` | `programacion/` | `lib/programacion.ts`, `lib/programacionSemanal.ts` |
| Contador de Agua (`/preparacion-plc`) | Litros, caudal y válvulas en vivo del PLC de Preparación; abrir y cerrar válvulas auxiliares pide `VALVULA_AUX`. Solo el área Aséptico | sin permiso propio | `PreparacionPLC.tsx` | `TanqueVisual.tsx` | habla por WebSocket con `plc-bridge/` |
| Manual | Tarjeta «Manual de usuario». **Pendiente:** el manual estará listo en diciembre; la tarjeta se deja y todavía no tiene ruta ni archivo | — | — | — | — |

### Servicios de apoyo

| Pantalla (ruta) | Qué hace | Permiso | Página | Piezas | Lógica |
|---|---|---|---|---|---|
| Calidad (`/calidad`) | Analizar y liberar lotes: Brix, acidez y conformidad. **Solo en Pruebas** | `LOTE_LIBERAR` | `Calidad.tsx` | `TanqueVisual.tsx` | `lib/calidad.ts` |
| Servicios Industriales (`/servicios-industriales`) | Cargar temperatura del Quantum, agua osmotizada y gasoil | `LECTURAS_SI` | `ServiciosIndustriales.tsx` | | `lib/panelProduccion.ts` |
| Registros del Área (`/registros-servicios-industriales`) | Historial de esas lecturas | `LECTURAS_SI` | `RegistrosServiciosIndustriales.tsx` | | `lib/panelProduccion.ts` |

### Base de Datos (administración)

| Pantalla (ruta) | Qué hace | Permiso | Página | Piezas | Lógica |
|---|---|---|---|---|---|
| Personal (`/personal`) | Organigrama por área y nivel; panel de la persona con puesto, título y permisos extra. Cada jefe ve solo su área y los niveles menores | `PERSONAL_GESTIONAR` o `ESTRUCTURA_GESTIONAR` | `Personal.tsx` | `personal/` (`PersonalPantalla`, `OrganigramaArea`, `PanelPersona`…) | `lib/personal.ts`, `lib/estructura.ts` |
| Áreas y puestos (`/areas-puestos`) | Matriz de permisos por puesto; crear áreas y puestos | `ESTRUCTURA_GESTIONAR` | `AreasPuestos.tsx` | `personal/` (`MatrizPermisos`, `FormPuesto`, `FormArea`) | `lib/estructura.ts` |
| Edición de Datos (`/edicion-datos`) | Catálogos generales: sabores, presentaciones, líneas, velocidades, fórmulas y más. Incluye «Ajustes de planta» (solo el usuario maestro) | `EDICION_DATOS` | `EdicionDatos.tsx` | `edicion-datos/` | `lib/edicionDatos.ts`, `lib/catalogosLive.tsx`, `lib/sabores.ts` |
| Catálogo de Paradas (`/catalogo-paradas`) | Fallas de equipos (Equipo › Sistema › Subsistema › Falla) y otras paradas, con sus códigos | `CATALOGO_PARADAS_VER` o `CATALOGO_PARADAS` | `CatalogoParadas.tsx` | `catalogo-fallas/`, `catalogo-paradas/`, `CatalogoEquipos.tsx` | `lib/catalogoFallas/`, `lib/paradasEquipos.ts`, `lib/paradasCatalogo.ts`, `lib/otrasParadas.ts`, `lib/paradasGrupos.ts` |
| Códigos de Mantenimiento (`/codigos-mantenimiento`) | Equipos y subsistemas de Mantenimiento. Solo el usuario maestro | `CODIGOS_MTTO_EDITAR` | `CodigosMantenimiento.tsx` | | `lib/mttoCatalogo.ts` |
| Errores (`/errores`) | Errores que le salieron a alguien usando la app: quién, cuándo y qué intentaba hacer. Solo el usuario maestro | `usuarios.ve_errores` | `ErroresCliente.tsx` | | `lib/erroresCliente.ts` (los registra `lib/supabase.ts`) |

### Fuera del hub

| Ruta | Qué hace | Archivo |
|---|---|---|
| `/` | Inicio de sesión | `pages/Login.tsx` |
| `/hub` | Inicio con las tarjetas según los permisos; el rol Solo Vista va directo al Panel de Producción | `pages/Hub.tsx`, `components/InicioSegunRol.tsx` |
| `/ajustes` | Tamaño de letra, tema y qué tarjetas ve cada persona (al tocar las iniciales del encabezado) | `pages/Ajustes.tsx`, `lib/tamanoLetra.ts`, `lib/tema.ts`, `lib/tarjetasOcultas.ts` |
| (sin ruta propia) | Primer ingreso: confirmar datos y definir clave. Se muestra en lugar de cualquier pantalla hasta completarlo | `pages/PrimerIngreso.tsx`, `lib/credenciales.ts` |

### Rutas viejas que redirigen

| Ruta vieja | Va a |
|---|---|
| `/paradas`, `/paradas-mantenimiento` | `/reportes-paradas` |
| `/mantenimiento` | `/lineas?paradas=1` (Modo Paradas) |
| `/calculadora-bobina`, `/calculadora-formula`, `/calculadora-conteo-peso` | `/calculadoras?c=bobina`, `?c=formula`, `?c=peso` |
| cualquier otra | `/` |

## 4. Marca, estilo y textos

| Qué se quiere cambiar | Archivo |
|---|---|
| Ícono de la app y de la pestaña | `public/IconoNatulac.png` (reemplazar el archivo, mismo nombre) |
| Marca en el encabezado | `src/components/Logo.tsx`, `src/components/AppHeader.tsx` |
| Título de la pestaña | `index.html` (`<title>`) |
| Colores | `src/index.css` (bloques `:root` y `.dark`) |
| Modo oscuro / tamaño de letra | `src/lib/tema.ts`, `src/lib/tamanoLetra.ts` (también se aplican en `index.html` antes de cargar) |
| Botones, tarjetas, diálogos base | `src/components/ui/` |
| Qué tarjetas hay y su texto | `src/lib/apps.tsx` |
| Orden y títulos de las secciones del inicio | `src/lib/navegacion.ts`, `src/pages/Hub.tsx` |
| Textos de una pantalla | La página de la sección 3, y la carpeta de piezas de esa fila |

## 5. Lógica compartida (`src/lib/`)

| Tema | Archivos |
|---|---|
| Conexión con la base | `supabase.ts` (cliente PostgREST y registro de errores), `sesionToken.ts` (credencial `x-sesion`) |
| Sesión y permisos | `auth.tsx`, `permisos.ts`, `apps.tsx` (`puedeVerApp`), `rolVista.ts`, `soloLectura.ts`, `credenciales.ts` |
| Hora y turno | `tiempoPlanta.ts` (hora de planta; **usar siempre**), `sesionTurno.tsx` (turno activo), `turno.ts`, `turno12x12.ts`, `ajustesTurnos.ts`, `continuidadTurno.ts`, `turnoCorreccion.tsx` (corregir un turno cerrado) |
| Los tres módulos del turno | `preparacion/` (tanques y lotes), `produccion/` (líneas, corridas y contadores), `productoTerminado.ts`. En los dos primeros: `nucleo.ts` (ciclo normal), `ajustes.ts` (lo que sale distinto), `tipos.ts`, `mapear.ts` y el hook `usePreparacion` / `useProduccion` |
| Cuentas y reportes | `reportes/` (merma, meta y rendimiento; lee los tres módulos), `eficiencia.ts`, `minutosSinReportar.ts`, `agruparProduccion.ts`, `indicadores/` |
| Paradas | `paradas.ts`, `paradasCatalogo.ts`, `paradasEquipos.ts`, `paradasGrupos.ts`, `otrasParadas.ts`, `catalogoFallas/`, `paradasTramo.ts`, `paradasEnVivo.ts`, `estadoLineas.ts`, `sheetMantenimiento.ts` |
| Actas y documentos | `actaPdf.ts`, `actaIconos.ts`, `actaVisor.ts`, `actasPendientes.ts`, `regenerarActa.ts`, `sifPdf.ts`, `descargarArchivo.ts` |
| Catálogos | `catalogos.ts` (fijos: áreas, roles, turnos, grupos), `catalogosLive.tsx` (líneas, presentaciones y velocidades desde la base), `sabores.ts`, `coloresSabor.ts`, `edicionDatos.ts` |
| Utilidades | `utils.ts`, `texto.ts`, `useAccion.ts`, `tanques.ts` |

## 6. Base de datos

- `supabase/migrations/`: más de 290 archivos con nombre `AAAAMMDDHHMMSS_descripcion.sql`.
  Se aplican en orden y cada uno trae un comentario inicial que explica qué cambia y por qué.
- `supabase/ESQUEMA.md`: resumen de tablas y funciones.
- Tareas programadas (`pg_cron`): `cerrar-turnos-vencidos`, `relevo-de-respaldo` y
  `paradas-mueven-lineas`.
- Para saber qué hace una función, buscar su nombre en `supabase/migrations/` y leer la
  migración más reciente que la defina.

## 7. Pruebas

`npm test` corre Vitest. Cada archivo `X.test.ts(x)` está junto al archivo que prueba. La
configuración común está en `src/test/setup.ts`.
