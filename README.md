# Exportación de IP bloqueadas desde Cloudflare

Este repositorio contiene una automatización que consulta periódicamente los **eventos de seguridad de Cloudflare** de un dominio concreto y publica, de forma actualizada, las direcciones IP que Cloudflare ha bloqueado o desafiado.

El objetivo es disponer de un histórico operativo y auditable de las IP detectadas por las protecciones de Cloudflare —incluidas las reglas personalizadas y los mecanismos automáticos de detección de Cloudflare—, sin necesidad de acceder manualmente al panel de Cloudflare.

## Qué hace

Un GitHub Action se ejecuta de forma programada y realiza estos pasos:

1. Consulta la API GraphQL de Cloudflare para obtener los eventos de seguridad recientes de la zona configurada.
2. Filtra los eventos con acciones de bloqueo o desafío, como `block`, `challenge`, `managed_challenge` y `js_challenge`.
3. Extrae las direcciones IP implicadas y elimina duplicados.
4. Genera o actualiza dos archivos:
   - `blocked-ips.txt`: una IP por línea.
   - `blocked-ips.json`: detalle de los eventos, con fecha, acción, país, ASN, origen de la regla y user agent, si están disponibles.
5. Si hay cambios, realiza un commit y lo publica en este repositorio.

Cloudflare expone estos datos mediante el dataset `firewallEventsAdaptive` de su API GraphQL, el mismo que alimenta la vista **Security Events** del panel. [Cloudflare Docs: Security Events y GraphQL Analytics]

## Archivos generados

| Archivo | Contenido |
|---|---|
| `blocked-ips.txt` | Lista deduplicada de direcciones IP, una por línea. |
| `blocked-ips.json` | Registro detallado de los eventos de seguridad de las últimas 24 horas. |
| `.github/workflows/cloudflare-blocked-ips.yml` | Definición del workflow de GitHub Actions. |

El archivo `.txt` está pensado para usos operativos sencillos, por ejemplo como base para listas de bloqueo en un cortafuegos. El `.json` conserva el contexto de cada evento y facilita su análisis posterior.

## Configuración previa

### 1. Token de Cloudflare

Crea un token personalizado en Cloudflare con estos permisos:

```text
Analytics & Logs > Zone Analytics > Read
```

En **Zone Resources**, limita el acceso a:

```text
Incluir > Dominio específico > tudominio.com
```

No es necesario conceder permisos de edición de DNS, WAF, reglas de firewall, certificados ni configuración de Cloudflare. El token debe tener el menor alcance posible.

### 2. Secretos de GitHub

En el repositorio, ve a:

```text
Settings > Secrets and variables > Actions > New repository secret
```

Crea estos dos secretos:

| Nombre del secreto | Valor |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Token de API de Cloudflare |
| `CLOUDFLARE_ZONE_ID` | Zone ID del dominio |

Los secretos de GitHub Actions son variables cifradas que solo pueden leer los workflows que las referencian explícitamente. No deben incluirse en el código, en archivos de configuración ni en mensajes de commit. [GitHub Docs: Using secrets in GitHub Actions]

## Ejecución

El workflow está configurado para ejecutarse:

- Automáticamente cada día a las 06:00 UTC.
- Manualmente desde la pestaña **Actions**, mediante **Run workflow**.

También puede modificarse la programación en la sección `schedule` del workflow:

```yaml
schedule:
  - cron: "0 6 * * *"
```

## Ejemplo de salida

`blocked-ips.txt`:

```text
203.0.113.45
198.51.100.23
192.0.2.87
```

`blocked-ips.json`:

```json
[
  {
    "action": "block",
    "clientIP": "203.0.113.45",
    "clientCountryName": "US",
    "clientAsn": 64512,
    "source": "custom_rules",
    "datetime": "2026-11-21T18:30:00Z"
  }
]
```

## Interpretación de los datos

Las IP incluidas no son necesariamente direcciones bloqueadas de forma permanente. Representan IP que Cloudflare registró en un evento de seguridad con una acción de bloqueo o desafío durante el periodo consultado.

Por ello, antes de usar la lista para bloquear direcciones en otro sistema, conviene revisar:

- La acción aplicada por Cloudflare.
- La antigüedad del evento.
- El origen del bloqueo: regla personalizada, nivel de seguridad, límite de peticiones, etc.
- La posibilidad de falsos positivos, especialmente si la IP corresponde a un proveedor de VPN, proxy, CDN o servicio compartido.

## Privacidad y seguridad

- Mantén el repositorio en **privado** si no deseas publicar IP, países, ASN, rutas solicitadas ni user agents.
- Si el repositorio es público, considera publicar únicamente `blocked-ips.txt`, sin metadatos adicionales.
- No subas nunca el token de Cloudflare ni el Zone ID a un archivo del repositorio.
- Si sospechas que el token se ha filtrado, revócalo desde Cloudflare y genera uno nuevo.
- Revisa periódicamente los permisos del token y elimínalo si deja de ser necesario.

## Limitaciones

- La exportación depende de la retención y disponibilidad de los datos de analítica asociados al plan de Cloudflare.
- Los eventos de seguridad pueden estar muestreados o limitados según el plan y el intervalo consultado.
- El listado es histórico: una IP puede aparecer por un evento antiguo aunque ya no esté siendo bloqueada actualmente.
- Si necesitas una lista estable de IP bloqueadas de forma explícita, es preferible mantener una **Custom List** en Cloudflare y exportarla mediante su API.

## Licencia

Este proyecto puede utilizarse y adaptarse libremente para fines personales o internos. Añade aquí la licencia que prefieras si planeas compartirlo públicamente.
