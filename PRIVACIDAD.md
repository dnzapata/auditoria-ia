# Política de privacidad — Auditoría pre-lanzamiento (auditoria-ia)

Última actualización: 30 de octubre de 2026

Esta política explica qué pasa con tus datos cuando usas el plugin **auditoria-ia** (la skill
"Auditoría pre-lanzamiento") en Claude Code, Cowork o las apps de Claude. Responsable: Daniel Zapata,
dnzapata@zynaptic.tech.

## Resumen

- **No recibimos ningún dato.** El plugin es un conjunto de instrucciones de texto: no tiene
  servidores, scripts, servidores MCP, analítica ni llamadas de red propias. Nada de lo que revisa
  llega al autor.
- **No guarda nada.** El informe se muestra en tu conversación con Claude; solo se guarda en un
  archivo si tú lo pides, en el lugar que tú elijas.

## Qué lee y para qué

Cuando le pides una revisión, Claude lee archivos **del proyecto que le indicas** (código,
configuración, archivos como `.env` o `local.properties`) para detectar problemas de seguridad y de
preparación para producción. Las claves que encuentre solo se clasifican (por ejemplo, pública o
secreta); no se usan para conectarse a nada y en el informe se muestran ocultas.

El plugin no lee credenciales de tu usuario o de tu sistema fuera de la carpeta del proyecto.

## Quién procesa los datos

Como en cualquier uso de Claude, el contenido que Claude lee se procesa en los servidores de
Anthropic para generar las respuestas, según las condiciones y la política de privacidad de tu
cuenta de Claude. El plugin no agrega ningún otro destino.

## Menores de edad

El plugin está pensado para personas adultas que desarrollan software.

## Contacto

Para cualquier consulta sobre esta política: dnzapata@zynaptic.tech.
