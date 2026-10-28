# Paso 3 — Preparación para producción

Estos puntos no son "hackeos" sino cosas que hacen que una app real pierda dinero, datos o usuarios.
Muchas no se pueden ver en el código: en ese caso van a "No verificable" con una pregunta concreta
para el dueño.

---

## P1. Claves de IA y límites de gasto

Aplica si el proyecto usa OpenAI, Anthropic (Claude), Gemini, Groq, Mistral, Replicate, etc.

### Qué buscar

- **Llamadas directas desde el cliente**: `api.openai.com`, `api.anthropic.com`,
  `generativelanguage.googleapis.com`, `api.groq.com` en código de navegador o de la app móvil.
  `dangerouslyAllowBrowser: true` en el SDK de OpenAI o Anthropic es una confesión explícita.
  **Crítico** (ver también S1): cualquiera saca la clave y la usa a tu costo.
- **Proxy sin autenticación**: una función de servidor que reenvía a la IA pero no exige usuario
  logueado. Es lo mismo que publicar la clave. **Crítico**.
- **Sin límite por usuario**: el proxy exige usuario pero no cuenta cuántas peticiones hace cada uno
  (no hay tabla de uso, contador, ni rate limit). Un solo usuario con un script puede generar una
  factura enorme. **Alto**.
- **El cliente decide parámetros caros**: `model`, `max_tokens`, `messages` completos o el prompt de
  sistema vienen del cliente. Pueden elegir el modelo más caro o usar tu cuenta como chat gratis para
  cualquier cosa. **Alto** para `model`/`max_tokens`, **medio** para el prompt.
- **Sin `max_tokens`** o con valores muy altos: **medio**.
- **Sin manejo de errores de cuota** (429, 529, saldo agotado): la app se rompe o reintenta en bucle
  (más gasto). **Medio**.
- **Datos personales enviados a la IA** sin mencionarlo en la política de privacidad: **medio** (ver
  P6).
- Si la IA puede ejecutar acciones (herramientas, consultas a la base, envío de emails) con texto que
  escriben los usuarios: riesgo de "prompt injection". **Alto** si puede tocar datos de otros usuarios.

### No verificable (preguntar siempre)

- ¿Tienes un **límite de gasto mensual** configurado en el panel del proveedor (OpenAI: Limits;
  Anthropic Console: Limits; Google Cloud: presupuestos y alertas)?
- ¿Recibes alertas cuando el gasto sube de golpe?

### Pedírselo a Claude Code (ejemplo)

```text
La función supabase/functions/chat/index.ts llama a OpenAI pero no limita cuánto puede usar cada
usuario. Agrega:
1. Una tabla uso_ia (user_id, fecha, peticiones) con RLS para que solo el servidor escriba.
2. En la función: obtener el usuario del token, rechazar con 429 si superó 50 peticiones en el día,
   y sumar 1 en cada petición exitosa.
3. Modelo y max_tokens fijos en el servidor (ignora los que mande el cliente).
4. Si OpenAI responde 429 o error de cuota, devolver un mensaje claro y no reintentar.
Hazlo en una migración nueva y muéstrame cómo probar el límite.
```

---

## P2. Límites de pedidos por usuario (rate limiting)

### Qué buscar

- Endpoints públicos o caros sin ningún límite: registro, login, recuperación de contraseña,
  formularios de contacto, envío de emails/SMS/WhatsApp, subida de archivos, generación de PDFs,
  llamadas a APIs pagas.
- **Envío de SMS o WhatsApp** sin límite ni captcha: los atacantes lo usan para mandar miles de
  mensajes a números premium a tu costo ("SMS pumping"). **Alto**.
- **Abuso con cuentas gratis o anónimas**: si cualquiera puede crear cuentas sin límite (ver S5),
  revisa qué puede hacer cada una sin tope: insertar filas (políticas `for insert` sin máximo por
  usuario), llamar funciones que mandan push/emails o llaman APIs pagas, reclamar pruebas gratis. Un
  script con dos cuentas propias puede llenar la base (el plan gratuito de Supabase tiene un límite de
  tamaño) o gastar las invocaciones del plan. **Medio** (alto si hay costos directos como SMS o IA).
  Arreglo: captcha en el registro o inicio anónimo, contador por usuario en las funciones, máximo de filas
  por usuario o limpieza periódica de datos viejos.
- Supabase: sección `[auth.rate_limit]` en `config.toml` (los valores de producción se ven en el
  panel). Edge Functions no tienen límite incorporado.
- Firebase: App Check, cuotas en Cloud Functions (`maxInstances`).
- Subidas sin límite de tamaño (`fileSizeLimit` del bucket, validación en el servidor): **medio**.
- Bucles de reintento sin espera creciente (backoff) en el cliente: pueden tumbar tu propio backend si
  hay un problema. **Medio**.

### Arreglo típico

Contador por usuario o por IP en la base o en Redis/Upstash, límites en el proveedor (Supabase Auth,
Cloudflare, Vercel), captcha (Turnstile/hCaptcha) en formularios públicos.

---

## P3. Backups y recuperación

Casi nunca se ve en el código. Revisa lo que sí se ve y pregunta lo demás.

### Qué buscar

- Scripts o tareas programadas de exportación (`pg_dump`, `gcloud firestore export`, GitHub Actions
  con backups).
- Si la app guarda datos **solo en el teléfono** (Room, SQLite, AsyncStorage, Hive): ¿hay forma de
  exportar/restaurar o se pierde todo al cambiar de teléfono? Revisa `allowBackup` y
  `dataExtractionRules` (ver también S9: es un equilibrio entre no perder datos y no filtrarlos).
- Borrados físicos (`delete from`) de datos importantes sin papelera o borrado lógico: **bajo**.
- Migraciones destructivas (`drop table`, `drop column`, `truncate`) en migraciones recientes:
  **medio**, avisar que se haga backup antes de aplicarlas.

### No verificable (preguntar siempre)

- **Supabase**: ¿en qué plan está? El plan gratuito no incluye backups descargables desde el panel; los
  backups diarios y la recuperación a un punto en el tiempo (PITR) dependen del plan. Además, **los
  proyectos gratuitos se pausan solos tras unos días sin actividad**: para una app en producción eso
  es una caída.
- **Firebase**: Firestore no hace backups automáticos salvo que configures exportaciones programadas
  o backups programados.
- ¿Alguna vez **probaste restaurar** un backup? Un backup que nunca se restauró no está comprobado.

---

## P4. Registro de errores

### Qué buscar

- ¿Hay alguna herramienta? `@sentry/*`, `sentry-android`, `firebase-crashlytics`, `bugsnag`,
  `logrocket`, `datadog`, `posthog` (con captura de errores). Sin ninguna en una app con usuarios:
  **medio** — te vas a enterar de los errores por las reseñas de una estrella.
- **Errores tragados**: `catch (e) {}`, `catch (_) {}`, `.catch(() => {})`, `catch (e: Exception) { }`,
  `runCatching { … }` sin mirar el resultado, en flujos importantes (pagos, guardado, login):
  **medio**.
- **Errores que muestran demasiado al usuario**: stack traces, mensajes de la base de datos, rutas
  internas en pantalla o en respuestas de la API: **bajo**/**medio**.
- **Logs con datos sensibles**: tokens, contraseñas, emails, datos de salud en `console.log`, `Log.d`,
  `print`, o enviados a la herramienta de errores sin filtrar: **medio**.
- `console.log` de depuración por todos lados en producción: **bajo**.

---

## P5. ¿Qué pasa si algo se cae?

### Qué buscar

- **Timeouts**: llamadas de red sin tiempo límite (`fetch` sin `AbortController`/`signal`, clientes
  HTTP sin `timeout`) en flujos donde el usuario espera: **bajo**/**medio**.
- **Pantallas de error**: ¿la app muestra algo útil si el backend no responde, o se queda en blanco /
  cargando para siempre / se cierra? Busca manejo de estados de error en las pantallas principales.
  **Medio** si las pantallas principales no lo manejan.
- **Funciones críticas que dependen de la red**: si la app tiene una función que el usuario necesita
  aunque no haya internet (por ejemplo alarmas o recordatorios), ¿funciona sin conexión?
- **Dependencia de un solo servicio externo** sin alternativa ni mensaje claro (un proveedor de IA,
  de pagos, de mapas). Mencionarlo en el informe como riesgo, **bajo**.
- **Tareas programadas** (cron, `pg_cron`, Cloud Scheduler, WorkManager): ¿qué pasa si una corrida
  falla? ¿Se reintenta? ¿Alguien se entera?
- **Forzar actualización**: si publicas una versión con un error grave, ¿tienes forma de pedirle al
  usuario que actualice? (In-App Updates de Google Play, versión mínima en el backend): **bajo**.

### No verificable (preguntar)

- ¿Tienes un monitor externo que te avise si la app o la API dejan de responder (UptimeRobot,
  Better Stack, etc.)?
- ¿Quién recibe las alertas y qué hace? ¿Está escrito en algún lado?

---

## P6. Política de privacidad y datos personales

No das asesoramiento legal: señalas lo que falta o no coincide.

### Qué buscar

- ¿Existe una política de privacidad (archivo en el repo, pantalla en la app, URL en la ficha de la
  tienda)? Sin política, Google Play rechaza apps que manejan datos personales. **Alto** si hay cuentas
  o datos personales.
- **¿Coincide con lo que hace el código?** Compara la lista de SDKs y servicios encontrados en el
  reconocimiento (analítica, publicidad, crash reporting, IA, pagos, notificaciones) con lo que dice la
  política. Si la política dice "no compartimos datos con terceros" y la app manda datos a Firebase,
  OpenAI o Sentry: **medio**/**alto**.
- **Promesas concretas de la política contra el código.** Busca en la política frases verificables y
  contrástalas una por una:
  - "solo en tu teléfono", "si desinstalas se borra" → ¿`allowBackup` o iCloud suben esos datos? (S9)
  - "borramos todo lo que hay sobre ti" → lee la función de borrado (`borrar_mis_datos`, `deleteUser`,
    endpoint de borrado) y compárala con **todas** las tablas que guardan el id del usuario, incluidas
    las de registro, historial, pruebas, suscripciones y el propio usuario de autenticación. Lo que
    quede sin borrar y no esté explicado en la política: **medio**.
  - "guardamos los últimos N días" → ¿hay algo que borre lo más viejo (tarea programada, trigger)?
  - "no recogemos X" → ¿algún SDK o tabla lo recoge?
  Cada diferencia es un hallazgo: o se cambia el código o se corrige la política, y además el
  formulario de Seguridad de los datos de Google Play.
- **Varias copias de la política** (por ejemplo una en `legal/` y otra en `web/`) con contenido
  distinto o campos sin completar (`[FECHA]`, `[NOMBRE]`): **bajo**; riesgo de publicar la equivocada.
- **Borrado de cuenta**: si la app permite crear cuenta, ¿hay forma de borrarla **desde la app** y
  también un enlace web para pedirlo? Google Play lo exige. Sin esto: **alto** (bloquea la publicación).
  Revisa que el borrado borre de verdad los datos en el backend (no solo cierre sesión).
- **Datos sensibles**: salud, menores, ubicación precisa, documentos, biometría. Requieren más cuidado
  y suelen tener declaraciones especiales en las tiendas. Señálalos.
- **Contacto del responsable** y forma de ejercer derechos (acceso, rectificación, borrado) en la
  política.
- Menciona como referencia, sin opinar sobre su cumplimiento, que según el país pueden aplicar leyes de
  protección de datos (por ejemplo GDPR en Europa, LGPD en Brasil, Ley 25.326 en Argentina, LFPDPPP en
  México) y que conviene que un profesional lo revise.

---

## P7. Requisitos de Google Play

Solo si hay app Android. Los requisitos de Google cambian todos los años: indica en el informe que
deben confirmarse en la documentación oficial vigente.

### Qué buscar en el código

- **`targetSdk`** en `build.gradle(.kts)`: Google exige apuntar a una versión reciente de Android (sube
  cada año, normalmente en agosto; desde agosto de 2025 el mínimo para apps nuevas y actualizaciones
  es API 35). Por debajo del mínimo vigente no se puede publicar: **alto**.
- **Firma de release**: contraseñas en `build.gradle` (ver S1). Recomendar Play App Signing.
- **Permisos sensibles** en `AndroidManifest.xml` que requieren declaración o justificación en Play
  Console, o están restringidos a ciertos tipos de apps:
  - `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` (`USE_EXACT_ALARM` solo para apps de alarma/calendario),
  - `USE_FULL_SCREEN_INTENT` (solo alarmas y llamadas, desde Android 14),
  - `FOREGROUND_SERVICE_*` (cada tipo se declara y justifica),
  - `ACCESS_BACKGROUND_LOCATION`, `READ_SMS`, `READ_CALL_LOG`, `QUERY_ALL_PACKAGES`,
    `MANAGE_EXTERNAL_STORAGE`, `REQUEST_INSTALL_PACKAGES`, `ACCESSIBILITY_SERVICE`.
  Por cada uno: ¿se usa de verdad en el código? Un permiso pedido y no usado: **medio** (motivo común
  de rechazo). Uno restringido sin caso de uso válido: **alto**.
- **`POST_NOTIFICATIONS`**: si la app manda notificaciones y apunta a Android 13+, debe pedir el
  permiso en tiempo de ejecución.
- **Librerías nativas** (`.so`, NDK, o dependencias que las traen) y soporte de páginas de 16 KB
  (exigido por Play para apps que apuntan a Android 15+): **medio**, verificar con Android Studio.
- **Build de release**: `isDebuggable` falso, `versionCode` que aumenta, `applicationId` definitivo
  (no `com.example`). `com.example` en el `applicationId`: **alto** (Play lo rechaza).
- **Actualizaciones fuera de Play**: código que descarga e instala APKs por su cuenta
  (`REQUEST_INSTALL_PACKAGES`, `PackageInstaller`, descargas de `.apk`): Play lo prohíbe para apps
  distribuidas en la tienda. **Alto**.

### No verificable (preguntar o recordar)

- **Formulario de Seguridad de los datos** (Data safety) en Play Console: debe coincidir con lo que
  hace la app y los SDKs (ver P6).
- **Cuentas de desarrollador personales nuevas**: Google exige una prueba cerrada con un mínimo de
  testers durante un período (al momento de escribir esto, 12 testers durante 14 días) antes de poder
  publicar en producción.
- **Declaraciones de contenido**: público objetivo (si incluye menores hay requisitos extra), anuncios,
  apps de salud, funciones financieras, acceso para revisión (usuario y contraseña de prueba si la app
  requiere login).
- **URL de la política de privacidad** pública y accesible sin login.
