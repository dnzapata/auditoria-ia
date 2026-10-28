# Auditoría pre-lanzamiento para apps hechas con IA

Una skill **gratuita y en español** para [Claude Code](https://claude.com/claude-code) que revisa tu
proyecto antes de salir a producción y te dice, ordenado por gravedad, qué arreglar primero.

Pensada para apps hechas con "vibe coding" (Lovable, Bolt, Cursor, Claude Code, v0…), donde la app
funciona pero nadie revisó si cualquiera puede leer la base de datos o usar tu clave de OpenAI.

> ⚠️ **Es una revisión de mejor esfuerzo, no una certificación.** Lee el código y señala problemas
> probables. No prueba la app funcionando ni revisa la configuración real en la nube. Puede tener
> falsos positivos y no encuentra todo.

## Qué revisa

**Seguridad**

- Claves secretas expuestas en la app, en el navegador o en git (OpenAI, Claude, Supabase
  `service_role`, Stripe, Mercado Pago, cuentas de servicio de Google, firmas de Android…)
- Supabase: tablas sin RLS, políticas abiertas, usuarios que pueden hacerse admin, funciones
  `security definer`, vistas, Edge Functions sin verificación
- Reglas de Firebase (Firestore, Storage, Realtime Database) y Cloud Functions
- Buckets y archivos públicos que deberían ser privados
- Login que se puede saltar, roles decididos en el cliente, códigos de invitación adivinables
- Permisos: un usuario que ve o modifica datos de otro
- Pagos validados solo en la app
- Dependencias vulnerables o inventadas por la IA
- Endurecimiento de Android y web (debuggable, tráfico sin cifrar, XSS, inyección SQL)

**Producción**

- Claves de IA en el cliente, proxies sin login, sin límite de uso por usuario o sin tope de gasto
- Límites de pedidos (rate limiting), abuso de SMS/email
- Backups y recuperación
- Registro de errores y errores silenciados
- Qué pasa si algo se cae (timeouts, pantallas de error, funciones sin conexión)
- Política de privacidad y borrado de cuenta
- Requisitos de Google Play (targetSdk, permisos restringidos, Data safety, prueba cerrada)

**Stacks:** web (React, Vite, Next.js), React Native / Expo, Flutter, Android nativo, iOS (básico),
con Supabase, Firebase o backend propio.

## Qué entrega

Un informe en el chat con:

- veredicto general (🔴 no publicar todavía · 🟠 publicar después de corregir · 🟢 sin bloqueos
  evidentes);
- hallazgos **crítico / alto / medio / bajo**, cada uno con archivo y línea, evidencia (con los
  secretos ocultos), por qué importa en palabras simples, cómo arreglarlo y **el texto listo para
  pedírselo a Claude Code**;
- lo que no se puede ver en el código (backups, límites de gasto, configuración del panel) con cómo
  comprobarlo;
- pruebas manuales simples y orden sugerido de trabajo.

## Qué NO hace (privacidad)

- **No modifica nada.** Solo lee. No instala dependencias, no compila, no hace commits.
- **No envía tu código ni tus datos a ningún servicio de terceros.** No corre `npm audit` ni
  escáneres en línea, no consulta tu Supabase o Firebase en vivo. Lo que necesita internet te lo deja
  indicado para que lo corras tú.
- **Aclaración honesta:** como cualquier uso de Claude Code, el contenido de los archivos que se leen
  pasa por el modelo de Anthropic para poder analizarlo, según las condiciones de tu cuenta de Claude.
  La skill no agrega ningún otro destino.
- No muestra claves completas en el informe.

Las únicas herramientas que usa sin pedir permiso son de lectura (`Read`, `Grep`, `Glob`). Cualquier
comando (por ejemplo `git log`) te pide autorización y explica qué hace.

## Instalación

### Opción 1: como plugin (recomendada)

Dentro de Claude Code:

```text
/plugin marketplace add dnzapata/auditoria-ia
/plugin install auditoria-ia@auditoria-ia
```

O desde la terminal:

```bash
claude plugin marketplace add dnzapata/auditoria-ia
claude plugin install auditoria-ia@auditoria-ia
```

Para recibir versiones nuevas: `claude plugin update auditoria-ia@auditoria-ia`.

### Opción 2: copiar la skill a mano

Windows (PowerShell):

```powershell
git clone https://github.com/dnzapata/auditoria-ia.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse auditoria-ia\skills\auditoria-prelanzamiento "$HOME\.claude\skills\"
```

macOS / Linux:

```bash
git clone https://github.com/dnzapata/auditoria-ia.git
mkdir -p ~/.claude/skills && cp -r auditoria-ia/skills/auditoria-prelanzamiento ~/.claude/skills/
```

Reinicia Claude Code después de instalar.

## Uso

Abre Claude Code en la carpeta de tu proyecto y escribe:

```text
/auditoria-prelanzamiento
```

(si la instalaste como plugin, el comando es `/auditoria-ia:auditoria-prelanzamiento`)

o simplemente: *"revisa si mi app está lista para publicar"*. También puedes indicar otra carpeta:
`/auditoria-prelanzamiento C:\proyectos\mi-app`.

Una revisión completa tarda unos minutos según el tamaño del proyecto.

> El informe describe vulnerabilidades de tu app: **no lo subas a un repositorio público.**

## Herramientas opcionales

Si tienes instaladas [gitleaks](https://github.com/gitleaks/gitleaks),
[osv-scanner](https://github.com/google/osv-scanner) o [semgrep](https://semgrep.dev), la skill las
usa **en modo sin red** para confirmar hallazgos (por ejemplo, claves en commits viejos). Puedes
tenerlas en el `PATH` o en una carpeta indicada con la variable de entorno `AUDITORIA_HERRAMIENTAS`.
No son necesarias y la skill nunca las instala.

## ¿Necesitas una auditoría completa?

Esta skill es un primer filtro. Una auditoría hecha por una persona prueba la app funcionando con
varias cuentas, revisa la configuración en la nube y el APK o sitio publicado, y te acompaña hasta
corregir.

**Daniel Zapata** — más de 30 años en IT · dnzapata@zynaptic.tech

## Licencia

[MIT](LICENSE). Úsala, modifícala y compártela. Sin garantías: ver la licencia.
