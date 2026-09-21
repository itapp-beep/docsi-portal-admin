# Docsi — Portal Administrativo (Angular)

Aplicación Angular 19 (standalone components, signals) para el portal
administrativo de Docsi. Consume la API Ktor existente
(`/api/v1/...`) y respeta el modelo de permisos (RBAC) definido en el
backend.

Roles soportados en un mismo shell (menú y acciones se muestran u
ocultan según permisos, no hay una app distinta por rol):
`EJECUTIVO`, `ADMINISTRADOR`, `EMPLEADO_ATC`.

## Requisitos

- Node.js 20 o 22 (probado con 22.22.2)
- Angular CLI 19 (`npm i -g @angular/cli@19`, o usar `npx ng ...` sin
  instalar global — así se generó este proyecto)
- El backend Ktor corriendo (por defecto se espera en
  `http://localhost:8080`)

## Instalación

```bash
npm install
```

## Levantar en desarrollo

```bash
npx ng serve
```

Abre `http://localhost:4200`. El proxy de API no es necesario: el
backend Ktor ya tiene CORS abierto (`anyHost()`, ver
`plugins/Monitoring.kt`) para desarrollo local, así que el navegador
llama directo a `http://localhost:8080/api/v1` según
`src/environments/environment.ts`.

Si el backend corre en otra URL o puerto, edita `apiUrl` en
`src/environments/environment.ts`.

## Build de producción

```bash
npx ng build --configuration production
```

Salida en `dist/docsi-portal-admin/`. Antes de desplegar:

1. Ajusta `apiUrl` en `src/environments/environment.prod.ts` a la URL
   real de la API en producción.
2. En el backend, reemplaza el CORS `anyHost()` por una lista explícita
   de orígenes permitidos (el dominio donde se sirva este build) — el
   `anyHost()` actual es solo apto para desarrollo.
3. La optimización de fuentes de Angular (`optimization.fonts`) está
   desactivada en `angular.json` porque el entorno de build no tenía
   salida a internet para pre-descargar Google Fonts; en un entorno de
   CI/build con acceso a internet se puede reactivar (`fonts: true`)
   sin que afecte el runtime, que ya carga la fuente "Outfit" vía
   `<link>` en `src/index.html`.

## Estructura

```
src/app/
  core/
    models/        interfaces TS que reflejan los contratos reales de la API/BD
    services/       AuthService, UsuarioService, ServicioService, MaestrosService, CargaMasivaService
    interceptors/   authInterceptor (agrega Bearer token a llamadas a la API)
    guards/         authGuard, sesionIniciadaGuard, soloInvitadoGuard, permisoGuard(...)
    directives/     *appPermiso (oculta elementos sin el permiso indicado)
    utils/          mensajeError (formatea errores HTTP para el usuario)
  shared/
    ui/             Icon, badges, Avatar (átomos del design system)
    layout/         Shell (sidebar + header, único para los 3 roles), PageHeader
  features/
    auth/           Login, Cambiar contraseña inicial
    inicio/         Dashboard / inicio
    usuarios/       Listado, formulario, reset de contraseña
    mi-cuenta/      Perfil propio del usuario logueado
    catalogo/       Servicios (listado + formulario) y Maestros (tabs: proveedores, ciudades, categorías, zonas de recargo, no disponibles)
    carga-masiva/   Carga de Excel + detalle de resultado
```

## Diseño

Los estilos viven en `src/styles/_tokens.scss` (colores, tipografía) y
`_base.scss` (clases utilitarias: `.btn*`, `.campo`, `.tabla`, `.card`,
`.badge`, etc.), extraídos del design system de Docsi
(`DOCSI_DESIGN SYSTEM.pdf`). Las pantallas siguen los mockups
aprobados en el canvas de Claude Design (Login, Inicio, Usuarios,
Catálogo de servicios, Carga masiva).

## Permisos (RBAC)

Los códigos de permiso usados en el frontend son exactamente los del
seed de seguridad del backend (`sql/07_seguridad_seed.sql`):
`catalogo:lectura`, `catalogo:escritura`, `usuarios:lectura`,
`usuarios:crear_empleado`, `usuarios:gestionar`, `citas:crear`,
`citas:lectura`, `citas:actualizar`, `pagos:registrar`,
`pagos:lectura`, `sistema:administrar`.

`AuthService.tienePermiso(...codigos)` decide visibilidad de menú y
acciones; `permisoGuard(...)` protege rutas; la directiva
`*appPermiso="'codigo'"` oculta elementos individuales en las
plantillas.

Regla fina de gestión de usuarios (`seguridad.fn_usuario_verificar_gestion`,
aplicada también en el frontend para no mostrar acciones que el
backend rechazaría): un `EJECUTIVO` puede gestionar cualquier rol
(incluyendo otros `EJECUTIVO`); un `ADMINISTRADOR` solo puede
gestionar cuentas `EMPLEADO_ATC`.

## Limitaciones conocidas / simplificaciones

Documentadas también en comentarios del código donde aplica:

- **Historial de cargas masivas**: la API actual no expone un
  `GET /cargas` para listar cargas pasadas. La pantalla de Carga
  masiva muestra un historial "reciente" guardado en `localStorage`
  del navegador (claramente rotulado como tal en la UI), no un
  historial real del sistema. Si se necesita historial completo,
  hace falta un endpoint nuevo en el backend.
- **Formulario de servicio**: cubre los campos principales del
  catálogo. El detalle de precios por turno y por modalidad no está
  incluido en esta primera versión; queda como siguiente iteración
  si el backend expone esos sub-recursos por separado.
- **Tipo de proveedor**: la lista de opciones está fija en el código
  (`MaestrosService.listarTiposProveedor()`) porque no existe un
  endpoint de catálogo para ese campo.
- **Reseteo de contraseña por un supervisor**: siempre fuerza cambio
  de contraseña en el siguiente login (comportamiento fijo del
  backend), no es una opción configurable desde la UI.

## Autenticación

`AuthService` guarda el JWT devuelto por
`POST /api/v1/auth/login` en `localStorage` y lo adjunta como
`Authorization: Bearer <token>` únicamente en llamadas hacia `apiUrl`
(ver `auth.interceptor.ts`). `authGuard` protege las rutas privadas;
`sesionIniciadaGuard` fuerza el cambio de contraseña inicial cuando el
backend lo indica; `soloInvitadoGuard` evita que un usuario ya logueado
vuelva a `/login`.
