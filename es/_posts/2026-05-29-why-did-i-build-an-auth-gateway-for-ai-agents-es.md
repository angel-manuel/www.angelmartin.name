---
layout: post
title: "¿Por qué construí una pasarela de autorización para Agentes de IA?"
post_ref: auth_gateway_for_ai_agents
lang: es
categories:
tags:
  - ai
  - ai agents
  - security
  - authorization
  - overslash
---

Como muchos otros últimamente, me interesé por los Agentes de IA (tan a fondo que estoy programando mi propio agente, pero de eso hablaré en un futuro post). Naturalmente, levanté mi OpenClaw en un VPS y empecé a intentar usarlo a cada ocasión. Y aun así, no he conseguido ahorrar mucho tiempo en mi día a día.

Para muchas cosas veo que mi IA puede hacer el 50% del trabajo. Por ejemplo, quería organizar una barbacoa y necesitaba comprar carne. Mi agente de IA fue capaz de:

+ encontrar una tienda cerca de mi ubicación
+ Abrir la página web
+ ver que ofrecían 8 packs distintos de productos para barbacoa (para esto tuvo que cargar y entender imágenes)
+ Estimar cuánta comida come la gente
+ Sugerir "menú 2 + menú 8"

La tienda tiene su número de teléfono publicado y aceptan pedidos por WhatsApp. Pero mi agente no tiene acceso a WhatsApp.

¿Por qué no tiene acceso? Bueno, no es por falta de capacidad. Existen algunas librerías e incluso una [nueva](https://x.com/steipete/status/1999499414419734831) [CLI](https://github.com/steipete/wacli). Implica enredar un poco con la API no documentada de los dispositivos vinculados de WhatsApp, pero funciona. El resultado es que el agente puede leer y enviar mensajes haciéndose pasar por el usuario.

El problema es que no quiero darle al agente acceso directo a mi WhatsApp; quiero revisar los mensajes que el agente envía antes de aprobarlos.

Así que empecé a pensar en construir mi propio sistema de autorización/aprobación para Agentes de IA.

## ¿Cómo hacer buenas aprobaciones?

Un buen sistema de aprobación para Agentes de IA tendría lo siguiente:
+ **Aprobaciones legibles por humanos**: descripciones y contexto en lenguaje natural, sin bash, sin scripts en línea, sin HTTP/JSON
+ **Alrededor del Agente**: el agente no debería poder saltárselo ni "olvidarse" de usarlo
+ **Flexible**: p. ej. que no me vuelva a preguntar por cada email a este mismo destinatario
+ **Auditable**: qué se hizo, cuándo, quién lo aprobó...
+ **General**: que funcione igual para todos los servicios y acciones
+ **No bloqueante**: el agente no se detiene hasta que apruebas; puede intentar otras acciones o seguir trabajando en otras cosas hasta que llegue la aprobación
+ **Delegable**: si yo puedo aprobar las acciones de mi agente, ¿puede mi agente aprobar selectivamente las acciones de un subagente?

Los distintos intentos actuales fallan en uno o más de estos puntos:
+ Las aprobaciones de los agentes de programación bloquean la ejecución y no son legibles por humanos (un bloque grande de bash en línea es difícil de revisar incluso para programadores).
+ El modo Auto de Claude Code ayuda el 90% de las veces, y aporta el comportamiento no bloqueante rechazando aprobaciones directamente a veces (a veces dejando al agente incapaz de completar su tarea). Está "Alrededor del Agente" pero sigue basándose en un LLM, es decir, no es infinitamente fiable.
+ OpenClaw directamente ni se molesta. Lo crítico es que el Agente puede acceder a los secretos necesarios para actuar en la mayoría de configuraciones normales. Y acabarán en su contexto (si no de forma intermedia, tarde o temprano lo harán).
+ La IETF, OAuth y otros vienen con una avalancha de borradores de especificación: he leído algunos, ojeado otros. Aunque hay valor en ellos, no creo que haga falta la mitad de las cosas.
+ Y aún otros ofrecen soluciones propietarias cerradas.

## Mi intento de resolverlo

¿Y si el agente tuviera una herramienta llamada `curl` que por debajo llama al curl normal PERO además es capaz de inyectar algunos secretos, como API Keys?

También podemos crear otra herramienta `request_secret` que pide los secretos al usuario cuando hacen falta, de manera que el secreto no acabe en el contexto del LLM sino que vaya al harness o a la herramienta `curl`.

Haciendo esto con cuidado podemos **separar al Agente de los secretos**.

Luego podemos ampliar esto poniendo restricciones (gating) a la herramienta `curl`.
Si el agente quiere hacer `curl("POST" "api.com", auth_header="secret_api_key")` entonces necesita:
+ el permiso para hacer `POST` a `api.com`: `"curl:POST:api.com"`
+ y el permiso para usar el secreto `secret_api_key` en `api.com`: `curl:secret:api.com:secret_api_key`
	+ es importante limitar el uso de los secretos a un host, o incluso atar cada secreto a un host, de lo contrario el Agente podría exfiltrar u obtener el contenido del secreto usando un servidor web que haga eco

Con esto ya tenemos una capa de autorización **Alrededor del Agente** y, añadiendo registro (logging), podemos hacerla **Auditable**. Implementé una versión de esto durante un tiempo en mi agente de IA; es una mejora. Anthropic ha lanzado algo parecido pero distinto con los Vaults en Managed Agents.

También funcionará para cualquier API REST con API key, pero las aprobaciones no serán legibles por humanos (a los usuarios normales no les importan las peticiones HTTP, y el meollo de muchas acciones está en el payload HTTP) ni flexibles (no se pueden aprobar solo acciones concretas de un endpoint).

Ahora, supongamos que escribimos un YAML describiendo una API así (parece OpenAPI y lo es, pero con un par de claves personalizadas...):

```yaml
openapi: 3.1.0
info:
  title: Google Calendar
servers:
  - url: https://www.googleapis.com
components:
  securitySchemes:
    oauth:
      type: oauth2
      x-overslash-provider: google
      # ...
paths:
  /calendar/v3/calendars/{calendarId}/events/{eventId}: 
    parameters:
      # ...
      - name: eventId
        in: path
        required: true
        description: Event identifier
        schema:
          type: string
        x-overslash-resolve:
          get: /calendar/v3/calendars/{calendarId}/events/{eventId}
          pick: summary
    get:
      # ...
    patch:
	  # ...
    delete:
      operationId: delete_event
      summary: "Delete event {eventId} on calendar {calendarId}"
      x-overslash-risk: delete
      x-overslash-scope_param: calendarId
```

Entonces sabríamos que existe una API (Google Calendar), que en ese dominio permite ciertos métodos.
Estos métodos operan sobre `eventId`, que son ids UUID feos como `abce6476-44b3-4e16-89f7-1add4d6986de`, pero por suerte la clave `x-overslash-resolve` en los YAML nos dice que podemos convertirlos en otra cosa (en este caso el nombre/resumen del evento de calendario) usando `/calendar/v3/calendars/{calendarId}/events/{eventId}`.

Con esto podemos convertir un:
"¿Apruebas `http:DELETE:https://www.googleapis.com/calendar/v3/calendars/primary/events/abce6476-44b3-4e16-89f7-1add4d6986de` con el secreto `secret_gcal_oauth_key`? \[Sí]\[No]"
en
"¿Eliminar el evento 'Crazy Party on the Moon' del calendario 'primary'? Conexión: user@gmail.com 
\[Sí]\[No]"
"\[Recordar para la conexión user@gmail.com]
\[Recordar para el calendario: primary y la conexión user@gmail.com]"

Lo conseguimos usando el resumen (summary) de la acción de OpenAPI, convirtiendo IDs en información legible. Y rellenamos previamente algunas claves de permiso útiles para "recordar" usando la extensión `x-overslash-scope_param` (la usamos para indicar que parte del payload puede ser importante a la hora de acotar permisos).

Ahora nuestro sistema de aprobación ha ganado **Aprobaciones legibles por humanos** y un acotado **Flexible** de permisos.

También podemos hacer el sistema **No bloqueante**: basta con dejar que el agente intente la acción (o pregunte si la acción está permitida) y mostrar al usuario una aprobación pendiente, diciéndole al agente que la aprobación está en espera. El Agente puede consultar otra herramienta en su heartbeat o tras una espera razonable o, con algo de control sobre el harness, podemos inyectar la aprobación/rechazo en cuanto el usuario actúe.

Si el sistema sabe que un agente es subagente de otro agente, podemos hacer subir la petición por la cadena, hasta un humano o un agente padre. Si el agente padre ya tiene el permiso, le damos la capacidad de aprobar o no la acción de su subagente; también puede optar por seguir escalando la decisión hacia arriba hasta llegar al humano. Así las aprobaciones también pueden ser **Delegadas**.

También podemos dar soporte a MCP de una forma muy parecida. La especificación OpenAPI se puede extender para etiquetar herramientas MCP adecuadamente. Ya tenía [un servidor MCP para WhatsApp fácil de desplegar](https://github.com/angel-manuel/whatsapp-mcp-docker) y esta es la cabecera de un YAML describiendo sus herramientas:

```yaml
openapi: 3.1.0
info:
  title: WhatsApp
  # ...
x-overslash-runtime: mcp
servers: []
paths: {}
x-overslash-mcp:
  auth:
    kind: bearer
  # autodiscover tells us to use tools/list to get MCP tools even if not described here
  autodiscover: true
  tools:
    - name: pairing_start
      risk: write
      description: "Open a WhatsApp pair flow and return the first event (rotating QR payload, plus a phone-link code if {phone} is given)"
      # ...

```

## Cerrando el círculo

Implementé todas las ideas anteriores (y algunas más) en una app llamada Overslash. ¡Esta herramienta hace de proxy de todos los usos de herramientas de servicios externos y se puede consumir como un único MCP!

Este MCP se puede consumir desde Claude Code, claude.ai, OpenClaw y otros. Aquí tienes una demo del flujo completo, terminando con el envío de un mensaje de WhatsApp:

<video controls muted playsinline loop preload="metadata" poster="{{ '/assets/video/overslash-whatsapp-claude-poster.jpg' | relative_url }}" width="100%" style="max-width:100%;border-radius:8px;">
  <source src="{{ '/assets/video/overslash-whatsapp-claude.mp4' | relative_url }}" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

Puedes ver el anuncio completo de Overslash en www.overspiral.com/blog/introducing-overslash/ y empezar a usarlo gratis en www.overslash.com

## Overslash

Implementé lo anterior, y mucho más:
+ Organizaciones/Equipos
+ Jerarquía de agentes
+ Panel para conectar y definir servicios
+ Registros de auditoría

Puedes alojarlo tú mismo o usarlo gratis en la nube en [overslash.com](https://www.overslash.com). O simplemente copia esto en tu OpenClaw, Claude o Codex:

```
Your Human wants to give you access to external services via Overslash.
To connect, follow the instructions at: https://www.overslash.com/SKILL.md
```
