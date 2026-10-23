# WhatsApp Message Buffer

🌐 [Español](readMe.es.md)

An n8n workflow that groups consecutive WhatsApp messages from the same user into a single batch before processing them, for example with an AI agent.

## The problem

In natural conversations, people rarely send one complete message. They send several in a row:

```
Hi.
My name is Chris.
I'd like to ask about...
```

If every message triggers its own execution, an AI agent will answer each fragment separately. The result is wrong or unnecessary replies, no shared context between messages, and several answers instead of one, which makes the conversation feel unnatural.

## The solution

This workflow uses a **debounce pattern** backed by an n8n Data Table:

1. Every incoming message is saved to a table, flagged as unprocessed.
2. The execution waits a few seconds (15 by default) for more messages.
3. After the wait, each execution checks whether its message is still the **latest unprocessed message** of that user.
4. Only the execution that holds the latest message continues. It retrieves all unprocessed messages of that user, marks them as processed, and outputs them as a single array. The other executions stop silently.

The result is one clean batch, ready to be passed to the next step of your workflow.

## How it works

1. **WhatsApp Trigger** receives the incoming message.
2. **Switch** routes by message type (`text`, `image` or `audio`). Any other type goes to **Stop and Error**.
3. A **Set** node per type extracts `user_id`, `message_type` and `value`, because each type has a different payload structure.
4. **Save message** stores the message in the `message_buffer` table with `processed = false`.
5. **Wait** pauses the execution (15 seconds by default).
6. **Retrieve last message** fetches the latest unprocessed message of the user.
7. **Is this the latest message?** compares it with the message that started this execution. If it is not the same, nothing else happens.
8. **Retrieve all saved messages** gets every unprocessed message of the user, in order of arrival.
9. **Mark as processed** sets `processed = true` on each of them, so they are not processed again.
10. **Extract messages in buffer** and **Unify** (plus a final Set that adds the `user_id`) build the final output.

## Output

The last node outputs a single item:

```json
{
  "messages_array": [
    { "value": "1677174607261743", "message_type": "audio" },
    { "value": "1772159854070922", "message_type": "image" },
    { "value": "And this is a text message", "message_type": "text" }
  ],
  "user_id": "AR.1054574977234806"
}
```

- `messages_array`: all buffered messages, in the order they were received.
- `user_id`: the WhatsApp user ID (`contacts[0].user_id`), falling back to `contacts[0].wa_id` when the first is not present in the payload.
- `value`: the text of the message when `message_type` is `text`, or the WhatsApp **media ID** when it is `image` or `audio`.

Connect your own logic (an AI agent, a CRM update, a reply) to the end of the workflow.

## Supported message types

| Type | `value` contains |
|------|------------------|
| `text` | The message text |
| `image` | The WhatsApp media ID of the image |
| `audio` | The WhatsApp media ID of the audio |
| Anything else (stickers, reactions, locations...) | The workflow stops with an error |

## Data table

The workflow uses a single n8n Data Table named `message_buffer`:

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | String | ID of the user who sent the message |
| `message_type` | String | `text`, `image` or `audio` |
| `value` | String | Text, or media ID for images and audio |
| `processed` | Boolean | Whether the message was already delivered in a batch |
| `createdAt`, `updatedAt` | Date | Created automatically by n8n |

## Setup

1. **Create the Data Table** `message_buffer` with the columns above (n8n adds `id`, `createdAt` and `updatedAt` automatically).
2. **Import** `whatsapp-message-buffer.json` into n8n.
3. **Reselect the table** in the four Data Table nodes: *Save message*, *Retrieve last message*, *Retrieve all saved messages* and *Mark as processed*. IDs of tables and credentials are specific to each n8n instance, so they are removed from the exported file and must be reconnected after importing.
4. **Assign your WhatsApp credential** in the WhatsApp Trigger node.
5. **Test it:** send two or three messages in a row, wait for the pause to end, and check that the final output contains all of them. Check also that they are marked as `processed = true` in the table.

> The workflow is exported as inactive. Activate it once the setup is finished.

## Configuration

- **Wait time:** change the value in the **Wait** node (15 seconds by default). A shorter wait answers faster but may split a burst into two batches. A longer wait groups more messages but delays the response.
- **Other channels:** the WhatsApp Trigger and the Set nodes that read its payload can be replaced to adapt the pattern to Telegram or any other source. The rest of the flow stays the same.
- **Other storage:** the Data Table can be replaced with Postgres or another database.

## Limitations

- **Message types:** only text, image and audio are supported. Other types make the execution end with an error, which shows as a failed execution in n8n. Replace the **Stop and Error** node with a **No Operation** node if you prefer to ignore them silently.
- **Media:** images and audio are stored only as media IDs. To use the files, exchange the ID for a download URL with the WhatsApp Cloud API. These URLs expire quickly, so download the media right after receiving the batch. Image captions are not stored.
- **Batch size:** **Retrieve all saved messages** returns up to 50 messages per execution, which is enough for normal bursts.
- **No retry:** messages are marked as processed before the next step runs. If your downstream logic fails, those messages are not retried automatically.
- **Table growth:** processed rows are never deleted. Add a periodic cleanup if you expect high volume.
- **Scale:** every incoming message opens an execution that stays waiting. For very high volumes, consider a different approach, such as Redis.
- **Privacy:** the table stores user messages. Define a retention policy that fits your use case.

## Tech

- n8n (a version with Data Tables support)
- n8n WhatsApp Business Cloud trigger (Meta)
- n8n Data Tables
