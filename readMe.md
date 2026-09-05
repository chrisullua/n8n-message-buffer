# N8N MESSAGE BUFFER
Este workflow tiene como objetivo agrupar información de multiples mensajes ingresados consecutivamente por un mismo usuario antres de procesarlos.

# Problema que busca solucionar.
En conversaciones naturales el usuario suele comunicarse mediante multiples mensajes consecutivos.

# Ejemplo:
[mensaje 1: Hola.
mensaje 2: Me llamo Patroclo.
mensaje 3: Quiero consultar por...]

Si luego queremos procesar cada mensaje individualmente (por ejemplo, por medio de un Agente IA), esto causaría respuestas incorrectas o innecesarias.
También dependiendo la estructura del workflow esto podria causar que se envíe una respuesta por cada uno de los mensajes recibidos, lo que restaría naturalidad a la conversación.

# Solución propuesta.

El workflow almacena la información de mensajes ingresados, espera un tiempo determinado (30 segundos, esto puede ajustarse en el unico nodo Wait del WF según conveniencia) y luego de validar que el último mensaje ingresado es efectivamente el ultimo mensaje recibido en el periodo de tiempo establecido, recupera la memoria de los items ingresados y los rejunta para su posterior tratamiento.

# Tecnologias
- n8n.
- Nodo oficial de Whatsapp Business (opcional, se puede adaptar el flujo a distintos triggers de ingreso de mensajes como Telegram).
- Simple Memory de n8n (tambien opcional, se puede tambien usar otras BDs como Postgres).

# Base de datos

Para este workflow uso una única BD nombrada "buffer_mensajes" que sigue la siguiente estructura.

1) id.
2) user_id (String).       -> uso el id del usuario que ingresa en el trigger cuando es disparado.
3) texto_mensaje (String). -> es el mensaje textual ingresado.
4) processed (Boolean).    -> valor que indica si el item ya fué procesado o no.
5) createdAT (Time).       -> se crea por defecto en las tablas de persistencia de n8n, indica cuando fué agregado el item a la tabla.
6) updatedAt (Time).       -> tambien se crea automaticamente, indica cúando el item se actualizó por ultima vez.

# Flujo

1) Whatsapp Trigger.
2) Data table (Guardar data).
3) Wait. (Configurado en 30 segundos).
4) Data table 2 (Recuperar ultimo mensaje guardado).
5) IF (Validación: ¿Este mensaje es el último mensaje ingresado?). Si el resultado es 'false', no hace falta hacer nada, si es 'true', continua en el punto 6.
6) Data table (Recuperar todos los mensajes que aún no han sido procesados).
7) Data table (Actualizar el estado de los mensajes ya reunidos para marcar que ya fueron procesados y evitar que se los vuelva a procesar en ejecuciones futuras).
8) Set (Extraemos los mensajes, obtendremos un número de items igual al número de mensajes ingresados).
9) Aggregate (Juntamos todos los items en una misma colección).

En este punto ya debería haber una colección de datos iterable con la información de todos los mensajes recientes recibidos.
[FIN]