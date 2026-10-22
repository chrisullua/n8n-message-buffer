# WhatsApp Message Buffer

🌐 [English](README.md)

Un workflow de n8n que agrupa los mensajes consecutivos de WhatsApp de un mismo usuario en un único lote antes de procesarlos, por ejemplo con un agente de IA.

![Vista general del workflow](docs/workflow-overview.png)

## El problema

En una conversación natural, la gente rara vez envía un único mensaje completo. Envía varios seguidos:

```
Hola.
Me llamo Patroclo.
Quiero consultar por...
```

Si cada mensaje dispara su propia ejecución, un agente de IA responderá cada fragmento por separado. El resultado son respuestas incorrectas o innecesarias, sin contexto compartido entre los mensajes, y varias respuestas en lugar de una, lo que le quita naturalidad a la conversación.

## La solución

Este workflow usa un **patrón de debounce** apoyado en una Data Table de n8n:

1. Cada mensaje entrante se guarda en una tabla, marcado como no procesado.
2. La ejecución espera unos segundos (15 por defecto) por si llegan más mensajes.
3. Terminada la espera, cada ejecución verifica si su mensaje sigue siendo el **último mensaje no procesado** de ese usuario.
4. Solo continúa la ejecución que tiene el último mensaje. Recupera todos los mensajes no procesados del usuario, los marca como procesados y los entrega en un único array. Las demás ejecuciones terminan en silencio.

El resultado es un lote limpio, listo para pasarlo al siguiente paso de tu workflow.

## Cómo funciona

1. **WhatsApp Trigger** recibe el mensaje entrante.
2. **Switch** deriva según el tipo de mensaje (`text`, `image` o `audio`). Cualquier otro tipo va a **Stop and Error**.
3. Un nodo **Set** por tipo extrae `user_id`, `message_type` y `value`, porque cada tipo tiene una estructura de payload distinta.
4. **Save message** guarda el mensaje en la tabla `message_buffer` con `processed = false`.
5. **Wait** pausa la ejecución (15 segundos por defecto).
6. **Retrieve last message** obtiene el último mensaje no procesado del usuario.
7. **Is this the latest message?** lo compara con el mensaje que inició esta ejecución. Si no es el mismo, no pasa nada más.
8. **Retrieve all saved messages** obtiene todos los mensajes no procesados del usuario, en orden de llegada.
9. **Mark as processed** marca cada uno con `processed = true`, para que no se procesen otra vez.
10. **Extract messages in buffer** y **Unify** (más un Set final que agrega el `user_id`) arman la salida final.

## Salida

El último nodo entrega un único item:

```json
{
  "messages_array": [
    { "value": "1677174607261743", "message_type": "audio" },
    { "value": "1772159854070922", "message_type": "image" },
    { "value": "Y esto es un mensaje", "message_type": "text" }
  ],
  "user_id": "AR.1054574977234806"
}
```

- `messages_array`: todos los mensajes del buffer, en el orden en que se recibieron.
- `user_id`: el ID de usuario de WhatsApp (`contacts[0].user_id`), con `contacts[0].wa_id` como respaldo cuando el primero no viene en el payload.
- `value`: el texto del mensaje cuando `message_type` es `text`, o el **ID de media** de WhatsApp cuando es `image` o `audio`.

Conectá tu propia lógica (un agente de IA, una actualización de CRM, una respuesta) al final del workflow.

## Tipos de mensaje soportados

| Tipo | Contenido de `value` |
|------|----------------------|
| `text` | El texto del mensaje |
| `image` | El ID de media de WhatsApp de la imagen |
| `audio` | El ID de media de WhatsApp del audio |
| Cualquier otro (stickers, reacciones, ubicaciones...) | El workflow se detiene con un error |

## Tabla de datos

El workflow usa una única Data Table de n8n llamada `message_buffer`:

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `user_id` | String | ID del usuario que envió el mensaje |
| `message_type` | String | `text`, `image` o `audio` |
| `value` | String | Texto, o ID de media para imágenes y audios |
| `processed` | Boolean | Indica si el mensaje ya fue entregado en un lote |
| `createdAt`, `updatedAt` | Date | Los crea n8n automáticamente |

## Instalación

1. **Creá la Data Table** `message_buffer` con las columnas de arriba (n8n agrega `id`, `createdAt` y `updatedAt` automáticamente).
2. **Importá** `whatsapp-message-buffer.json` en n8n.
3. **Reseleccioná la tabla** en los cuatro nodos Data Table: *Save message*, *Retrieve last message*, *Retrieve all saved messages* y *Mark as processed*. Los IDs de tablas y credenciales son propios de cada instancia de n8n, por eso se eliminan del archivo exportado y hay que reconectarlos después de importar.
4. **Asigná tu credencial de WhatsApp** en el nodo WhatsApp Trigger.
5. **Probalo:** enviá dos o tres mensajes seguidos, esperá a que termine la pausa y verificá que la salida final los contiene a todos. Comprobá también que en la tabla queden con `processed = true`.

> El workflow se exporta inactivo. Activalo una vez terminada la configuración.

## Configuración

- **Tiempo de espera:** cambiá el valor en el nodo **Wait** (15 segundos por defecto). Una espera más corta responde más rápido, pero puede dividir una ráfaga en dos lotes. Una espera más larga agrupa más mensajes pero demora la respuesta.
- **Otros canales:** el WhatsApp Trigger y los nodos Set que leen su payload pueden reemplazarse para adaptar el patrón a Telegram o a cualquier otra fuente. El resto del flujo no cambia.
- **Otro almacenamiento:** la Data Table puede reemplazarse por Postgres u otra base de datos.

## Limitaciones

- **Tipos de mensaje:** solo se soportan texto, imagen y audio. Otros tipos hacen que la ejecución termine con un error, que se ve como ejecución fallida en n8n. Reemplazá el nodo **Stop and Error** por un nodo **No Operation** si preferís ignorarlos en silencio.
- **Media:** las imágenes y los audios se guardan solo como IDs de media. Para usar los archivos, canjeá el ID por una URL de descarga con la API de WhatsApp Cloud. Esas URLs expiran rápido, por lo que conviene descargar el archivo apenas recibís el lote. Los textos que acompañan a una imagen (caption) no se guardan.
- **Tamaño del lote:** **Retrieve all saved messages** devuelve hasta 50 mensajes por ejecución, suficiente para ráfagas normales.
- **Sin reintentos:** los mensajes se marcan como procesados antes de que corra el siguiente paso. Si tu lógica posterior falla, esos mensajes no se reintentan automáticamente.
- **Crecimiento de la tabla:** las filas procesadas nunca se borran. Agregá una limpieza periódica si esperás mucho volumen.
- **Escala:** cada mensaje entrante abre una ejecución que queda en espera. Para volúmenes muy altos, considerá otro enfoque, como Redis.
- **Privacidad:** la tabla almacena mensajes de usuarios. Definí una política de retención acorde a tu caso de uso.

## Tecnologías

- n8n (una versión con soporte de Data Tables)
- Trigger de WhatsApp Business Cloud de n8n (Meta)
- Data Tables de n8n
