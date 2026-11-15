# TRACE chatbot prompt

This folder documents exactly what was sent to the language model in the three
TRACE chatbot conditions.

| File | Contents |
|---|---|
| `system_prompt.txt` | The system prompt, verbatim, as assembled in `pages/api/chat.ts`. |
| `../Knowledge-Base/knowledge.json` | The knowledge base the chatbot answered from. Institution names and the contact email are replaced by placeholders (`[Anonymized Institution A]`, `[Anonymized Institution B]`, `[anonymized contact email]`) for anonymous review; nothing else differs from the deployed file. |

The prompt text is identical across conditions. The only thing that varies is
the `stance` value in the configuration object described below.

## Model settings

| Setting | Value |
|---|---|
| Model | `gpt-4o` (OpenAI Chat Completions API; no dated snapshot was pinned) |
| Temperature | `0.1` |
| Response format | `{ "type": "json_object" }` |

## Conditions

| Study condition | `stance` |
|---|---|
| TRACE-Neutral | `NEUTRAL` |
| TRACE-Collector | `COLLECTOR` |
| TRACE-User | `USER` |

The stance is read from the request's `stance` query parameter and defaults to
`NEUTRAL`.

## Messages sent on every turn

1. **System:** the contents of `system_prompt.txt`.
2. **System:** `KNOWLEDGE_JSON = ` followed by the serialized knowledge base.
3. **System:** `CONFIG_JSON = ` followed by the serialized configuration object.
4. The conversation so far (user and assistant turns).
5. **User:** the latest user message. The handler appends it after the
   conversation history, so it appears a second time when the history already
   ends with it.

### `KNOWLEDGE_JSON`

All entries of the knowledge base with `visible` not set to `false` (61
entries), each reduced to five fields:

| Field | Source in `knowledge.json` |
|---|---|
| `id` | `id` |
| `question` | `suggestedQuestion`, or `title` if absent |
| `answer` | `summary` |
| `tags` | `tags` |
| `parentId` | `parentId` |

Example entry:

```json
{
  "id": "is_participation_voluntary",
  "question": "Is participation voluntary in the study?",
  "answer": "Yes. Sharing your data is completely voluntary, and choosing not to participate will not affect you in any way.",
  "tags": ["voluntary", "participation", "data sharing", "choice", "voluntary participation", "effect"],
  "parentId": "basic_information"
}
```

### `CONFIG_JSON`

| Field | Meaning |
|---|---|
| `stance` | `NEUTRAL`, `COLLECTOR` or `USER`. |
| `askedNodeIds` | Ids of the entries already covered in this session. |
| `targetNodeId` | Id of the entry the participant selected in the overview panel, or `null` for a free-text question. |

Example (illustrative values):

```json
{
  "stance": "COLLECTOR",
  "askedNodeIds": ["study_objectives", "what_data_is_collected"],
  "targetNodeId": "data_storage"
}
```

## Model output

The model returns one JSON object:

```json
{
  "answer": "string",
  "primaryNodeId": "string | null",
  "coveredNodeIds": ["string"]
}
```

## Post-processing

Before the answer is shown, the server:

- removes any trailing meta content from `answer` (for example a "related
  questions" list or JSON fragments);
- adds `primaryNodeId` to `coveredNodeIds` if the model left it out;
- if the model's output is not valid JSON, falls back to the knowledge-base
  entry whose question shares the most words with the user's message (at least
  two), or to a fixed out-of-scope message if there is none.
