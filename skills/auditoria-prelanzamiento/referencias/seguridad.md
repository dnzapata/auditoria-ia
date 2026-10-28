# Paso 2 — Seguridad

Recorre solo las secciones que apliquen al stack. Para cada una: busca, abre el archivo para
confirmar, decide gravedad con los criterios de `informe.md`, y anota evidencia (`ruta:línea` y un
fragmento corto con los secretos ocultos).

Los patrones de búsqueda son expresiones regulares para la herramienta Grep. Ignora las carpetas
listadas en `reconocimiento.md`.

---

## S1. Claves y secretos expuestos

### Qué buscar

Patrones de claves conocidas (usa Grep con `-i` desactivado salvo que se indique):

| Patrón | Qué es | ¿Secreta? |
|---|---|---|
| `sk-ant-[A-Za-z0-9_-]{20,}` | Anthropic (Claude) | Sí |
| `sk-(proj-\|svcacct-\|admin-)?[A-Za-z0-9_-]{20,}` | OpenAI (descarta las que empiezan con `sk-ant-`) | Sí |
| `AIza[0-9A-Za-z_-]{35}` | Google. Puede ser la `apiKey` de Firebase (pública) **o** una clave de Gemini/Maps (ver nota) | Depende |
| `sb_secret_[A-Za-z0-9_-]{10,}` | Supabase, clave secreta nueva | Sí |
| `sb_publishable_[A-Za-z0-9_-]{10,}` | Supabase, clave pública nueva | No |
| `eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}` | JWT. En Supabase puede ser `anon` (pública) o `service_role` (secreta) | Depende |
| `(sk\|rk)_live_[0-9A-Za-z]{10,}` | Stripe, clave secreta real | Sí |
| `whsec_[A-Za-z0-9]{10,}` | Stripe, secreto de webhook | Sí |
| `APP_USR-[0-9]{6,}-[0-9]{6}-[0-9a-f]{32}-[0-9]+` | Mercado Pago, access token | Sí |
| `APP_USR-[0-9a-f]{8}-[0-9a-f]{4}-` | Mercado Pago, public key | No |
| `AKIA[0-9A-Z]{16}` | AWS | Sí |
| `gh[pousr]_[A-Za-z0-9]{36}\|github_pat_[A-Za-z0-9_]{20,}` | GitHub | Sí |
| `gsk_[A-Za-z0-9]{20,}` | Groq | Sí |
| `hf_[A-Za-z0-9]{30,}` | Hugging Face | Sí |
| `r8_[A-Za-z0-9]{20,}` | Replicate | Sí |
| `re_[A-Za-z0-9_]{20,}` | Resend | Sí |
| `SG\.[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}` | SendGrid | Sí |
| `xox[baprs]-[A-Za-z0-9-]{10,}` | Slack | Sí |
| `-----BEGIN [A-Z ]*PRIVATE KEY-----` | Clave privada (certificado, cuenta de servicio, SSH) | Sí |
| `"type"\s*:\s*"service_account"` | Cuenta de servicio de Google/Firebase (acceso de administrador) | Sí |
| `(postgres\|postgresql\|mysql\|mongodb(\+srv)?)://[^:\s/]+:[^@\s]+@` | URL de base de datos con contraseña | Sí |

Patrón genérico (con `-i`), para revisar a mano:
`(secret|password|passwd|token|api_?key|private_?key|service_?role)\s*[:=]\s*["'][^"'\s]{8,}["']`

Variables de entorno que **se publican en el cliente** por su prefijo y no deberían tener secretos
(con `-i`):
`(NEXT_PUBLIC|VITE|EXPO_PUBLIC|REACT_APP|NUXT_PUBLIC|PUBLIC|GATSBY)_[A-Z0-9_]*(SECRET|SERVICE_ROLE|SERVICE_KEY|OPENAI|ANTHROPIC|CLAUDE|GEMINI|GROQ|PRIVATE|ACCESS_TOKEN|STRIPE_SECRET|SK_)`

Android: `buildConfigField` y `resValue` en `build.gradle(.kts)` y su origen (normalmente
`local.properties`); `res/values/*.xml`; `assets/`. Todo eso queda dentro del APK y se extrae en
minutos con herramientas gratuitas.

Flutter: `.env` listado en `assets:` de `pubspec.yaml`; `String.fromEnvironment` con secretos
(`--dart-define` también queda en el binario).

### Cómo interpretar

**¿Dónde está?** Esto decide la gravedad más que la clave misma:

- **En el cliente** (todo el código de una SPA de Vite/React, Expo, Flutter, Android, iOS; en Next.js,
  cualquier archivo con `'use client'` o variable `NEXT_PUBLIC_`): cualquiera que use la app puede
  sacarla. Una clave secreta acá es **crítica**.
- **En git con remoto**: cualquiera con acceso al repositorio la ve, y queda en el historial aunque se
  borre. Secreta → **crítica** si el repo podría ser público, **alta** si es privado (pregunta en "No
  verificable").
- **En git sin remoto o solo en el historial**: **alta** (rotarla igual).
- **En un `.env` local ignorado por git y usado solo por el servidor**: es el uso correcto. No es
  hallazgo.

**JWT de Supabase:** decodifica solo la parte central (es información de configuración, no el secreto)
para ver el campo `role`:

- Bash: `printf '%s' '<parte-central>' | tr '_-' '/+' | base64 -d 2>/dev/null; echo`
- PowerShell: `$p='<parte-central>'.Replace('-','+').Replace('_','/'); $p+='='*((4-$p.Length%4)%4); [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))`

`"role":"anon"` es pública por diseño. `"role":"service_role"` salta todas las reglas de seguridad de
la base: en el cliente es **crítico**.

**`AIza…`:** si está en `google-services.json`, `GoogleService-Info.plist` o en el `firebaseConfig`
de la web, es la clave pública de Firebase (no es hallazgo; como mucho recomendar restringirla en
Google Cloud Console). Si se usa para llamar a Gemini (`generativelanguage.googleapis.com`,
`GoogleGenerativeAI(`) o a Maps desde el cliente, es una clave de uso pago expuesta: **alta** o
**crítica** según qué APIs tenga habilitadas.

**Archivos de firma de Android** (`*.jks`, `*.keystore`, `keystore.properties`, `key.properties`, o
contraseñas `storePassword`/`keyPassword` escritas en `build.gradle`): si están versionados, quien los
tenga puede firmar actualizaciones falsas de la app (salvo que uses Play App Signing y esta sea solo la
clave de subida). **Crítico** si están en git con remoto, **alto** en otro caso.

### Arreglo

1. **Rotar primero**: revocar la clave en el panel del proveedor y generar una nueva. Borrarla del
   código no alcanza: ya está en el historial de git o en APKs publicados.
2. Mover la clave al servidor (Edge Function de Supabase, Cloud Function, ruta API de Next.js) y que la
   app llame a ese servidor.
3. Agregar los archivos a `.gitignore` y, si estaba en el historial y el repo es público, limpiar el
   historial (git filter-repo) **después** de rotar.

### Pedírselo a Claude Code (ejemplo)

```text
En este proyecto la clave de OpenAI está en el cliente (src/lib/ai.ts, variable VITE_OPENAI_KEY) y
cualquiera puede sacarla del navegador. Ya la revoqué y tengo una nueva.
1. Crea una Supabase Edge Function llamada "chat" que reciba el mensaje del usuario, verifique que la
   petición trae un usuario logueado válido (Authorization: Bearer), y llame a OpenAI con la clave
   leída de los secretos de Supabase (Deno.env.get("OPENAI_API_KEY")). Fija el modelo y max_tokens en
   el servidor; no los aceptes desde el cliente.
2. Cambia src/lib/ai.ts para que llame a esa función con supabase.functions.invoke en lugar de a
   OpenAI directamente, y elimina VITE_OPENAI_KEY de todo el proyecto.
3. Dime el comando exacto para cargar el secreto con supabase secrets set y para desplegar la función.
No cambies nada más.
```

---

## S2. Supabase

Fuentes: `supabase/migrations/*.sql`, `supabase/seed.sql`, `supabase/config.toml`,
`supabase/functions/**`, y el código cliente que usa `.from(`, `.rpc(`, `.storage`.

Si no hay migraciones en el repo (tablas creadas a mano en el panel), **no puedes ver RLS**: mándalo a
"No verificable" con las consultas de abajo.

### Qué buscar

1. **Tablas sin RLS.** Lista cada `create table` del esquema `public` (o sin esquema) y busca su
   `alter table … enable row level security`. Tabla sin RLS y con datos de usuarios: **crítico**
   (cualquiera con la clave `anon`, que es pública, puede leerla y modificarla entera). Tabla sin RLS
   solo con datos públicos de catálogo: **medio** (igual pueden modificarla).
2. **Políticas abiertas.** Busca `create policy` y lee cada una:
   - `using (true)` o `with check (true)` para `anon`/`public`/`authenticated` en tablas con datos
     personales: **crítico** o **alto**.
   - Políticas para `authenticated` sin condición de dueño (`auth.uid() = user_id` o equivalente):
     cualquier usuario registrado ve/modifica lo de todos. **Alto** (crítico si el registro es libre
     y los datos son sensibles).
   - `for insert … with check (true)`: un usuario puede crear filas a nombre de otro. **Alto**.
   - Políticas que usan `auth.jwt() -> 'user_metadata'` o `raw_user_meta_data`: el usuario puede
     editar su propio `user_metadata` desde la app, así que puede darse permisos. **Crítico** si
     decide roles/admin. Lo correcto es `app_metadata` o una tabla de roles.
   - Nota: una política `for update using (…)` sin `with check` **no** es un error: Postgres usa la
     misma condición para ambos. No lo reportes.
   - RLS activo sin ninguna política = nadie accede salvo el servidor. Es seguro (puede romper la app,
     pero no filtra datos).
3. **Escalada de privilegios por columnas.** Tablas como `profiles`/`usuarios` con columnas `role`,
   `is_admin`, `plan`, `premium`, `credits`, `saldo`, `verificado`, y una política que deja al usuario
   actualizar **su propia fila**: puede ponerse `is_admin = true` o créditos infinitos. **Crítico**.
   Arreglo: sacar esas columnas a otra tabla que solo escribe el servidor, o revocar `update` sobre esas
   columnas, o un trigger que las proteja.
4. **Funciones `security definer`.** Buscan `security definer` (con `-i`). Se ejecutan con permisos de
   dueño, saltando RLS, y si están en `public` se pueden llamar con `.rpc()` desde cualquier cliente.
   Revisa que validen `auth.uid()` y tengan `set search_path = ''` (o un esquema fijo). Sin validación
   y tocando datos de usuarios: **alto**/**crítico**.
5. **Vistas.** `create view` en `public`: por defecto ignoran RLS de las tablas que leen. Si no tienen
   `with (security_invoker = true)` y muestran datos de usuarios (sobre todo si leen `auth.users`):
   **alto**/**crítico**.
6. **`grant` explícitos** a `anon` sobre tablas o funciones sensibles: revisar.
7. **Storage**: ver S4.
8. **`config.toml`**:
   - `[functions.<nombre>] verify_jwt = false`: la función acepta llamadas sin usuario. Correcto solo
     para webhooks que validan una firma o tareas programadas con un secreto propio. Si no valida
     nada: **alto**/**crítico** según qué haga.
   - `[auth] enable_signup = true` en una app que debería ser solo por invitación: **alto**.
   - `enable_anonymous_sign_ins = true`: no es hallazgo por sí solo; activa la revisión de cuentas
     anónimas de S5 y P2.
   - `[auth.email] enable_confirmations = false`: cualquiera se registra con un email ajeno. **Medio**.
   - `minimum_password_length` menor a 8: **bajo**.
   - Ojo: `config.toml` refleja el entorno local; la configuración real de producción está en el panel.
     Si hay dudas, va a "No verificable".
9. **Edge Functions** (`supabase/functions/*/index.ts`):
   - Usan `SUPABASE_SERVICE_ROLE_KEY` (salta RLS) **y** toman el `user_id` del cuerpo de la petición en
     vez de obtenerlo del token (`supabase.auth.getUser(jwt)`): un usuario puede actuar como otro.
     **Crítico**.
   - No verifican el header `Authorization` cuando `verify_jwt = false`. Ver punto 8.
   - Webhooks (pagos, Stripe, Mercado Pago, Google Play RTDN) que no validan firma o secreto: **alto**.
   - Devuelven errores internos completos al cliente (`error.message` de la base, stack): **bajo**.
10. **Cliente**: filtrar con `.eq('user_id', miId)` en la app **no es seguridad**; solo RLS lo es.
    Si la seguridad depende de esos filtros, reportarlo dentro del hallazgo de RLS.
11. **Restos de pruebas que siguen activos.** Busca (con `-i`) `prueba`, `test`, `debug`, `temp`,
    `borrar cuando`, `solo para pruebas`, `TODO` en migraciones, Edge Functions y `seed.sql`. Si una
    migración crea tablas, triggers, funciones o tareas de `pg_cron` "de prueba" y **ninguna migración
    posterior las elimina**, siguen en producción. Lo más común: tablas de registro (logs, historiales)
    que guardan datos de usuarios sin vencimiento y que la función de "borrar mis datos" no limpia.
    **Medio** si guardan datos personales (o contradicen la política de privacidad, ver P6); **alto** si
    son legibles o escribibles desde el cliente; **bajo** si no guardan nada de usuarios. Arreglo: una
    migración nueva que haga `drop` de lo que sobra.

### Consultas para que el usuario verifique en producción (solo lectura)

Si falta información, incluye en "No verificable" estas consultas para pegar en el SQL Editor de
Supabase, y recomienda mirar **Advisors → Security Advisor** en el panel:

```sql
-- Tablas del esquema public y si tienen RLS activo
select c.relname as tabla, c.relrowsecurity as rls_activo
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where c.relkind = 'r' and n.nspname = 'public'
order by rls_activo, tabla;

-- Todas las políticas (revisar las que dicen "true" o no mencionan auth.uid())
select tablename, policyname, roles, cmd, qual, with_check
from pg_policies where schemaname in ('public', 'storage')
order by tablename;

-- Buckets públicos
select id, public from storage.buckets;

-- Funciones SECURITY DEFINER en public
select p.proname from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public' and p.prosecdef;
```

### Pedírselo a Claude Code (ejemplo)

```text
La tabla public.pedidos no tiene RLS activado, así que cualquiera con la clave anon puede leer y
modificar todos los pedidos. Crea una migración NUEVA en supabase/migrations (no edites las existentes)
que:
1. Active RLS en public.pedidos.
2. Agregue políticas para que un usuario autenticado solo pueda ver, crear, modificar y borrar sus
   propios pedidos (auth.uid() = user_id), usando (select auth.uid()) para que sea eficiente.
3. No dé ningún acceso al rol anon.
Revisa que el código de la app siga funcionando con estas reglas (busca todos los usos de
.from('pedidos')) y explícame cómo probarlo con dos usuarios distintos.
```

---

## S3. Firebase

Fuentes: `firestore.rules`, `storage.rules`, `database.rules.json`, `firebase.json`, `functions/`.
Si no están en el repo, la configuración real está solo en la consola: "No verificable".

### Qué buscar

1. **Reglas abiertas** (Firestore y Storage):
   - `allow read, write: if true;` o `allow read, write;` (sin condición = permitido): **crítico**.
   - `if request.time < timestamp.date(` — reglas del "modo de prueba": abiertas a todo el mundo hasta
     esa fecha y después la app deja de funcionar. **Crítico** mientras estén vigentes.
   - `match /{document=**}` con `if request.auth != null`: cualquier usuario registrado lee y escribe
     **toda** la base. **Alto** (crítico si el registro es libre y hay datos personales).
   - Colecciones por usuario sin `request.auth.uid == userId` o `== resource.data.<dueño>`: **alto**.
   - Reglas de `create`/`update` que no validan los campos (`request.resource.data`): un usuario puede
     escribir `role: 'admin'` o cambiar el dueño. **Alto** si hay campos de permisos o saldo.
2. **Realtime Database** (`database.rules.json`): `".read": true` o `".write": true` en la raíz o en
   nodos con datos de usuarios: **crítico**. `"auth != null"` en la raíz: **alto**.
3. **Cloud Functions**:
   - `onRequest` que no verifica el token (`verifyIdToken`) antes de leer o escribir datos: **alto**.
   - `onCall` que no revisa `request.auth` / `context.auth`: **alto**.
   - Funciones que toman el `uid` del cuerpo en lugar del token: **crítico** (actuar como otro).
4. **Cuenta de servicio** (`service_account`) en el repo o en la app: ver S1, **crítico**.
5. **App Check** no configurado: **bajo**/**medio** (ayuda a frenar abuso desde scripts, no reemplaza
   las reglas).

### Pedírselo a Claude Code (ejemplo)

```text
firestore.rules permite leer y escribir todo a cualquier usuario logueado (match /{document=**} con
request.auth != null). Reescribe las reglas para que:
- cada usuario solo lea y escriba los documentos de /usuarios/{uid} y sus subcolecciones cuando
  request.auth.uid == uid;
- en create/update se valide que el campo "dueño" sea request.auth.uid y que no se puedan escribir los
  campos "rol" ni "plan";
- todo lo demás quede denegado por defecto.
Lista las colecciones que usa la app (busca collection( y doc( en el código) para no romper nada, y
dame casos de prueba para el emulador de Firebase.
```

---

## S4. Almacenamiento y buckets públicos

### Qué buscar

- Supabase: `insert into storage.buckets` con `public` en `true`; `createBucket(` con `public: true`;
  políticas sobre `storage.objects` con `using (true)` o sin chequear la carpeta del usuario
  (`(storage.foldername(name))[1] = auth.uid()::text`).
- Firebase: `storage.rules` (ver S3).
- AWS S3 / otros: `ACL: 'public-read'`, `BlockPublicAcls: false`, buckets de Cloudinary con subidas sin
  firma (`upload_preset` unsigned) usadas para documentos privados.
- `getPublicUrl(` usado para archivos que deberían ser privados (documentos, facturas, fotos de
  documentos de identidad, recetas médicas).

### Cómo interpretar

Un bucket público está bien para cosas públicas (logos, fotos de productos, avatares). Para documentos
personales es **crítico** aunque los nombres sean "difíciles": los enlaces se comparten, se filtran y
quedan en historiales. Si cualquiera puede **subir** archivos sin límite de tamaño ni de tipo: **medio**
(te pueden usar el almacenamiento para alojar cualquier cosa y pagarás tú).

### Arreglo

Bucket privado + políticas por carpeta de usuario + URLs firmadas con vencimiento
(`createSignedUrl`). Límite de tamaño y tipos permitidos en el bucket.

---

## S5. Login que deja pasar a cualquiera

### Qué buscar

- **Protección solo en la pantalla**: rutas o pantallas que redirigen si no hay usuario
  (`if (!user) navigate('/login')`, guards de React Router, middleware de Next.js) pero los datos
  vienen de una API o tabla que no verifica nada. Saltarse la pantalla es trivial. Confirma revisando
  el lado del servidor/RLS. **Crítico** si los datos quedan abiertos.
- **Roles en el cliente**: `isAdmin`, `role === 'admin'`, listas de emails de administradores en el
  código del cliente, o paneles de admin que solo se ocultan en la interfaz. **Alto**/**crítico** si
  el servidor no vuelve a verificar.
- **Middleware de Next.js** con `matcher` que deja afuera rutas de `/api` o páginas privadas.
- **JWT mal verificado**: `jwt.decode(` usado para autorizar (no verifica la firma), `algorithms`
  con `none`, secretos de JWT débiles o escritos en el código. **Crítico**.
- **Usuarios o contraseñas fijos**: `password === '`, `admin123`, usuarios de prueba que siguen
  activos, "modo demo" que entra sin credenciales. **Crítico** si llega a producción.
- **Recuperación de contraseña / enlaces mágicos** que redirigen a URLs armadas con datos del usuario
  (`redirectTo` con parámetros de la URL). **Medio**.
- **Registro abierto** en apps que deberían ser solo por invitación (ver S2 punto 8).
- **Tokens guardados de forma insegura** en apps móviles (`SharedPreferences` sin cifrar,
  `AsyncStorage` para tokens de larga duración): **bajo**/**medio**.
- **Códigos de invitación o enlaces para compartir**: ¿son aleatorios y largos (UUID o 16+ caracteres
  de un generador seguro), vencen, se pueden usar una sola vez? Códigos secuenciales o predecibles:
  **alto**.
- **Códigos cortos con límite de intentos "por usuario"** (PIN de 4 a 8 dígitos para vincular,
  verificar o invitar): el límite no sirve si crear usuarios es gratis (registro libre, cuentas
  anónimas, emails sin confirmar). Calcula cuántos intentos por hora logra alguien que crea cuentas
  nuevas (límite de registros por IP × intentos por cuenta) y compáralo con la cantidad de códigos
  posibles y su duración. Revisa también si el dueño de los datos **se entera o tiene que aprobar** cuando
  alguien usa su código. Si acertar da acceso a datos personales: **medio**; a datos sensibles (salud,
  menores, finanzas) sin aviso ni aprobación del dueño: **alto**. Arreglo: límite global además del
  límite por usuario, captcha en el registro, códigos más largos, aprobación o aviso al dueño.
- **Cuentas anónimas** (`enable_anonymous_sign_ins = true` en Supabase, `signInAnonymously` en
  Firebase, `/auth/v1/signup` sin email desde la app): cualquiera crea cuentas ilimitadas con un script
  y cada una pasa por `authenticated` en las políticas. No es un error en sí (muchas apps lo usan bien),
  pero multiplica todo lo que dependa de "un límite por usuario": intentos, cuotas, pruebas gratis,
  filas que cada usuario puede insertar. Revisa esos puntos con esa mirada y recomienda captcha en los
  inicios de sesión anónimos. Ver también P2.

---

## S6. Permisos: un usuario ve datos de otro

Es el error más común en apps hechas con IA y el que más daño hace. Conecta con S2 y S3.

### Qué buscar

- Endpoints o funciones que reciben un id (`params.id`, `req.query.id`, `body.userId`,
  `searchParams.get(`) y buscan ese registro **sin comprobar que pertenece a quien pregunta**.
- Funciones de servidor que usan credenciales de administrador (service role, Admin SDK) — esas saltan
  las reglas, así que toda la seguridad depende del código.
- IDs secuenciales (`serial`, `autoincrement`) expuestos en URLs: facilitan probar ids ajenos. Por sí
  solo es **bajo**; combinado con falta de chequeo es **crítico**.
- Funciones de "compartir" o "familia/equipo" (alguien ve datos de otro **a propósito**): revisar que
  la relación se valide en la base (tabla de vínculos + política que la consulte), no en la app.
- Búsquedas o listados que devuelven columnas de más (`select('*')` de `profiles` con email, teléfono,
  dirección de todos los usuarios).

### Cómo explicarlo en el informe

Describe el ataque en una frase concreta: "un usuario logueado puede cambiar el número en
`/api/pedidos/123` y ver el pedido de otra persona". Si no lo pudiste confirmar leyendo el servidor o
las reglas, confianza media y recomienda la prueba con dos cuentas (está en el informe como
verificación manual).

---

## S7. Pagos y compras

Solo si la app cobra algo.

- **El precio o el producto lo decide el cliente** (monto enviado desde la app al crear el pago):
  **crítico**.
- **La compra se valida solo en la app** (Google Play Billing / StoreKit sin verificar el token en el
  servidor; "premium" que se activa con un `update` desde el cliente): **crítico**.
- **Webhooks** sin validar firma (`stripe.webhooks.constructEvent`, `x-signature` de Mercado Pago):
  **alto**.
- Google Play: compras no reconocidas (`acknowledge`) en 3 días se reembolsan solas. Revisar que se
  haga. **Medio**.

---

## S8. Dependencias vulnerables

### Qué buscar

- ¿Hay archivo de bloqueo (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock`,
  `pubspec.lock`, `gradle.lockfile`, `Podfile.lock`) y está en git? Sin bloqueo, cada instalación puede
  traer versiones distintas: **bajo**.
- Si `osv-scanner` está instalado, úsalo en modo sin red (ver `herramientas.md`).
- Si no, revisa a ojo versiones claramente viejas de paquetes críticos (frameworks web, librerías de
  autenticación, parsers, clientes HTTP) **con confianza baja** y aclarando que tu conocimiento de
  vulnerabilidades puede estar desactualizado.
- **Paquetes inventados por la IA**: nombres de dependencias raros, mal escritos o que no reconoces
  (las IAs a veces inventan nombres y alguien registra ese nombre con código malicioso). Lístalos para
  que el usuario los verifique en el registro oficial. **Medio** hasta que se verifiquen.
- Scripts de instalación (`postinstall`) de paquetes poco conocidos.

### Lo que el usuario puede correr (requiere internet, lo hace él)

Ponlo en "No verificable":

- JavaScript: `npm audit --omit=dev` (envía la lista de paquetes al registro de npm).
- Flutter: `flutter pub outdated`.
- Android: revisar avisos de Android Studio en `libs.versions.toml` / `build.gradle`.
- Cualquier proyecto en GitHub: activar **Dependabot alerts** (gratis).

---

## S9. Endurecimiento de la app

### Android (`AndroidManifest.xml`, `build.gradle`, `res/xml/`)

- `android:debuggable="true"` o `isDebuggable = true` en release: **alto**.
- `android:usesCleartextTraffic="true"` o `network_security_config` con `cleartextTrafficPermitted`
  o que confía en certificados de usuario: **medio**.
- Componentes con `android:exported="true"` (activities, receivers, services, providers) sin
  `android:permission`, que hacen acciones o devuelven datos: **medio**/**alto** según qué hagan.
- `android:allowBackup="true"` (o ausente, que vale `true`) sin `dataExtractionRules`/`fullBackupContent`:
  Android sube la base local y las preferencias (incluidas las sesiones guardadas) a la cuenta de Google
  del usuario y las restaura al reinstalar. La copia está cifrada, así que no es una filtración: el
  riesgo real es que contradiga la política de privacidad ("solo en tu teléfono", "si desinstalas se
  borra") y que restaure tokens de sesión. **Medio** si hay datos sensibles o tokens y la política dice
  otra cosa (ver P6); **bajo** en otro caso. No recomiendes desactivarla sin más: en apps donde perder
  los datos es grave, lo correcto suele ser mantenerla, excluir las sesiones y explicarlo en la política.
- `isMinifyEnabled = false` en release: **bajo** (facilita leer el código, no es una vulnerabilidad).
- `Log.d`/`Log.i`/`println` con tokens, emails o datos personales: **medio**.
- WebView con `addJavascriptInterface` o `setAllowFileAccess(true)` cargando contenido remoto: **alto**.
- SQL armado concatenando texto (`rawQuery("… " + x)`, `execSQL` con variables): **medio**/**alto**.

### Web

- `dangerouslySetInnerHTML`, `innerHTML =`, `v-html` con datos que vienen de usuarios: **alto** (XSS).
- `eval(`, `new Function(` con datos externos: **alto**.
- Consultas SQL armadas con plantillas de texto (`` `SELECT … ${x}` ``) en el servidor: **crítico**.
- CORS `Access-Control-Allow-Origin: *` junto con cookies/credenciales: **alto**. Solo `*` en una API
  que exige token: **bajo**.
- Falta de cabeceras de seguridad (CSP, `X-Frame-Options`) en `vercel.json`, `netlify.toml`,
  `firebase.json`: **bajo**.
- Mapas de código fuente (`*.map`) publicados con comentarios o rutas internas: **bajo**.
