# Paso 4 — Herramientas locales opcionales

La skill funciona sin ninguna herramienta. Si están instaladas, sirven para confirmar hallazgos y
encontrar cosas que se escapan leyendo (por ejemplo, claves en commits viejos).

**Nunca instales nada.** Si no están, dilo en el alcance y sigue.

## Dónde buscarlas

1. Si existe la variable de entorno `AUDITORIA_HERRAMIENTAS`, es la carpeta donde el usuario guarda
   las herramientas (ejecutables y, opcionalmente, una subcarpeta `semgrep-reglas/`).
2. Si no, en el `PATH`: `command -v gitleaks osv-scanner semgrep` (Bash) o
   `Get-Command gitleaks, osv-scanner, semgrep -ErrorAction SilentlyContinue` (PowerShell).

Anota la versión de cada una (`--version`) en el alcance del informe.

## Reglas para correrlas

- **Sin red** y **sin escribir en el proyecto**. Si una herramienta necesita escribir un reporte,
  mándalo a la salida estándar o a una carpeta temporal fuera del proyecto.
- Usa siempre las opciones que ocultan secretos en la salida.
- Lo que encuentren se **verifica** abriendo el archivo antes de reportarlo, igual que el resto.

## gitleaks (claves en archivos y en el historial de git)

No usa la red.

- Versión 8.19 o superior:
  - historial de git: `gitleaks git <ruta> --redact --no-banner -v`
  - archivos actuales (incluye los no versionados): `gitleaks dir <ruta> --redact --no-banner -v`
- Versiones anteriores: `gitleaks detect --source <ruta> --redact --no-banner -v`
  (`--no-git` para revisar solo archivos).

Un hallazgo solo en el historial (el archivo ya no existe) igual requiere **rotar la clave**.

## osv-scanner (dependencias con vulnerabilidades conocidas)

Por defecto **consulta internet** enviando la lista de paquetes y versiones a osv.dev. Solo se permite
el modo sin red:

- Si ya hay una base descargada:
  `osv-scanner scan source -r <ruta> --offline-vulnerabilities --no-resolve`
- Si no hay base local, para descargarla hace falta internet (descarga la base pública, no envía datos
  del proyecto). **Pide permiso al usuario** antes de agregar `--download-offline-databases`.
- Si la versión instalada usa otras opciones, revisa `osv-scanner --help` y aplica el mismo criterio.

## semgrep (patrones de código inseguro)

`--config auto` y los paquetes `p/…` descargan reglas del registro y `auto` exige enviar métricas:
**no los uses**. Solo reglas locales y métricas apagadas:

`semgrep scan --metrics=off --config <carpeta-de-reglas> <ruta>`

Donde `<carpeta-de-reglas>` es `$AUDITORIA_HERRAMIENTAS/semgrep-reglas` si existe. Si no hay reglas
locales, no lo corras.

## MobSF y análisis del APK

MobSF analiza el APK compilado (lo que realmente se publica) y corre como servidor aparte. Está fuera
del alcance de esta revisión de código: si el usuario lo pide, recomiéndalo como parte de una
auditoría completa.
