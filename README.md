# FootwearPoint

Plataforma web y móvil para la gestión de distribuidoras de calzado: catálogo, ciclos de compra, pedidos, pagos, vales, stock y venta directa, con aislamiento de datos por distribuidora.

## El problema que atiende

Las distribuidoras de calzado trabajan sus pedidos por WhatsApp, llamadas y libretas. Eso provoca pedidos duplicados o perdidos, precios inconsistentes entre revendedores, saldos y anticipos que nadie tiene claros, y nula trazabilidad de quién autorizó qué. FootwearPoint centraliza ese proceso: el catálogo es uno solo, los pedidos siguen estados verificables, los pagos y vales quedan registrados contra el pedido y cada operación sensible se audita.

## Usuarios principales


- Administrador de la plataforma: Da de alta distribuidoras y planes de suscripción
- Distribuidora (staff y sucursales): Administra catálogo, campañas, ciclos de compra, stock, pedidos y venta directa
- Revendedor: Consulta el catálogo, arma pedidos, gestiona a sus propios clientes privados y sus vales
- Cliente directo: Consulta el catálogo y levanta sus pedidos

## Arquitectura

El sistema son tres piezas sobre una misma base de datos:

- **Panel web** — Laravel + Livewire. Lo usan la distribuidora y el administrador.
- **API REST** — Laravel + Sanctum. Es lo que consume la app móvil. El contrato está documentado en [`docs/contrato-api.md`](docs/contrato-api.md).
- **App móvil** — Flutter, en [`mobile/`](mobile/). La usan revendedores y clientes directos.

Dos reglas atraviesan todo el sistema:

1. **Todo se filtra por distribuidora.** El cliente nunca manda un `distribuidora_id`; el servidor lo deduce del usuario autenticado. No hay forma de pedir datos de otra distribuidora.
2. **Cada quien ve solo lo suyo.** Si un revendedor o cliente directo pide un pedido o un vale ajeno, la respuesta es **404**, no 403 — a propósito, para no confirmarle siquiera que ese registro existe.

## Tecnologías

**Backend**

- PHP 8.2, Laravel 12
- Livewire 4 (panel web)
- Laravel Sanctum 4 (autenticación por token para la app)
- spatie/laravel-permission 6 (roles y permisos)
- barryvdh/laravel-dompdf 3 (comprobantes de venta en PDF)
- kreait/laravel-firebase 6 (notificaciones push FCM)
- league/flysystem-aws-s3-v3 (almacenamiento de imágenes de catálogo)
- MySQL

**Frontend**

- Tailwind CSS 4, Vite 7

**Móvil**

- Flutter

**Calidad y herramientas**

- PHPUnit 11 (pruebas)
- Laravel Pint (estilo de código)
- Laravel Pail (logs en desarrollo)

## Requisitos

- PHP 8.2 o superior
- Composer 2
- Node.js 20 o superior y npm
- MySQL 8
- Flutter (solo si vas a trabajar la app móvil)

## Instalación

```bash
git clone https://github.com/Kevin-VR-99/footwearlink.git
cd footwearlink
composer run setup
```

El script `setup` hace todo lo necesario: instala dependencias de PHP, crea el `.env` a partir de `.env.example`, genera la llave de aplicación, corre las migraciones, instala dependencias de Node y compila los assets.

Antes de ejecutarlo, crea la base de datos y ajusta las credenciales en tu `.env`.

Para levantar el entorno de desarrollo completo (servidor, cola de trabajos, logs y Vite, todo en una terminal):

```bash
composer run dev
```

## Variables de entorno

Los nombres de cada variable están en [`.env.example`](.env.example). **El archivo `.env` nunca se versiona** — está en `.gitignore` y así debe quedarse.

Los grupos que debes configurar:

- Aplicación: `APP_NAME`, `APP_ENV`, `APP_KEY`, `APP_DEBUG`, `APP_URL`
- Base de datos: `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`
- Correo: `MAIL_MAILER` y sus credenciales
- Almacenamiento S3: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BUCKET`, `AWS_DEFAULT_REGION`
- Firebase: la ruta al archivo de credenciales de servicio

## Pruebas

```bash
composer run test
```

Las pruebas corren con PHPUnit sobre una base de datos aparte, `footwearpoint_testing`, que debe existir antes de ejecutarlas. La configuración está en [`phpunit.xml`](phpunit.xml): fuerza `APP_ENV=testing`, cache y sesión en memoria, correo capturado en arreglo y colas síncronas, para que ninguna prueba toque servicios reales.

Hay dos suites:

- `tests/Unit` — pruebas unitarias
- `tests/Feature` — pruebas de integración sobre el panel, la API y las reglas de negocio, organizadas por dominio (`Pedido`, `Vale`, `Stock`, `Catalogo`, `Seguridad`, …)

Para correr solo una suite o un archivo:

```bash
php artisan test --testsuite=Feature
php artisan test tests/Feature/Seguridad/AislamientoMultiTenantTest.php
```

## Estructura del proyecto

```
app/
  Http/Controllers/Api/   Controladores de la API que consume la app móvil
  Livewire/               Componentes del panel web
  Models/                 Modelos de dominio
config/                   Configuración de Laravel y paquetes
database/migrations/      Esquema de la base de datos
docs/contrato-api.md      Contrato de la API móvil
mobile/                   App Flutter
resources/views/          Vistas Blade del panel
routes/                   Definición de rutas web y api
tests/                    Pruebas unitarias y de integración
```

## Flujo de trabajo

La rama `main` está protegida: no se le hace push directo. Todo cambio entra por Pull Request con revisión aprobada.

1. Crea una rama desde `main`:
   - `feature/<descripción>` para funcionalidad nueva
   - `fix/<descripción>` para correcciones
2. Haz commits con la convención del equipo: clave de la historia o tarea, descripción en minúsculas y la referencia del ticket.

   ```
   TG-165: pantallas de la app con el diseño del panel web
   E19-08: forzar deploy HEAD en Railway TG-107
   Correccion: el vale aplicado cuenta como pago del pedido TG-167
   Documentacion: README del proyecto
   ```

   Cuando el cambio no corresponde a un ticket, se usa un prefijo descriptivo (`Correccion:`, `Documentacion:`) y se omite la referencia.
3. Sube la rama y abre un Pull Request hacia `main`.
4. Otro integrante revisa y aprueba.
5. Se integra con merge commit, lo que conserva la trazabilidad de qué PR trajo cada cambio.

Los conflictos se resuelven en la rama de trabajo, nunca en `main`: se trae `main` a la rama, se resuelven ahí y se actualiza el PR.

## Despliegue

El panel web se despliega en Railway a partir de la rama `main`. Cada integración a `main` dispara un nuevo despliegue, detrás de un proxy que termina TLS, con la aplicación forzando HTTPS y una política de seguridad de contenido configurada por capa.

## Documentación

- [Contrato de la API móvil](docs/contrato-api.md)

## Equipo

Universidad Tecnológica de la Selva — Ingeniería en Desarrollo y Gestión de Software.

- Alcázar Pavón Ángel Gabriel
- Liévano Santiesteban Ailton Guadalupe
- Morales Maldonado Francisco Rigoberto
- Paniagua Méndez Aurelio Fabián
- Velázquez Ríos Kevin Arturo
