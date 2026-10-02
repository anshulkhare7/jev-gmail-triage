# Jev Gmail Triage

This project is an experiment with Jev, a decisions model from TypeSafe. It is available on [OpenRouter](https://openrouter.ai).

The experiment uses a real task. The script reads your Gmail inbox. Jev classifies each message. The script then adds a label to the message.

Gmail is only the test data. The goal is to learn how Jev behaves.

## About Jev

Jev is not a chat model. It does not write text. It answers questions about data that you give to it.

| Item | Detail |
|---|---|
| Model slugs | `typesafe/jev-1.13` (fixed version), `~typesafe/jev-latest` (newest version) |
| Endpoint | `POST https://openrouter.ai/api/alpha/decisions` |
| Not supported | `/chat/completions`. This endpoint returns a 400 error for Jev. |
| Input | A `state` object and a `questions` object |
| Output | Probabilities, not text |
| Cost | You pay for input tokens. Output tokens are free. |
| Status | The endpoint is in alpha. It can change. |

### Question types

Each question has a `type`, `instructions`, and `criteria`.

| Type | Use | Answer |
|---|---|---|
| `noul` | Yes or no question | A number from 0 to 1. This number is the probability of "yes". |
| `choice` | Select one option | The selected option, a `confidence` value, and a probability for each option |
| `score` | Select a point on an ordered scale | A weighted `score`, a `confidence` value, and a probability for each point |

Jev answers each question on its own. A question cannot see the answers to other questions in the same request.

### Example request

```json
{
  "model": "typesafe/jev-1.13",
  "state": {
    "from": "billing@example.com",
    "subject": "Your invoice is ready",
    "snippet": "Your invoice for September is attached."
  },
  "questions": {
    "category": {
      "type": "choice",
      "instructions": "How should this email be triaged?",
      "criteria": {
        "Important": "Work, finance, security, or an item that needs action.",
        "Unimportant": "A newsletter, a notification, or a routine receipt.",
        "Can-Delete": "A promotion, junk, or an expired code."
      }
    }
  }
}
```

### Example response

```json
{
  "answers": {
    "category": {
      "type": "choice",
      "choice": "Important",
      "confidence": 0.91,
      "probabilities": { "Important": 0.95, "Unimportant": 0.05, "Can-Delete": 0 }
    }
  },
  "usage": { "input_tokens": 210, "output_tokens": 20, "cost": 0.00001 }
}
```

The numbers in this example are illustrative. They are not real output.

### What this project learns about Jev

- **Probabilities:** Jev gives a probability for each option. You can set a threshold and send low-confidence messages to a person.
- **No reasons:** Jev does not explain its answer. The `reason` column in the CSV contains the confidence value.
- **Cost:** Output is free. Cost depends on the size of the `state` object.
- **Format:** The response always has the same structure. The script does not need to repair broken JSON.

## How this project uses Jev

1. The script sends one request for each email.
2. Each request has one `choice` question with three options: Important, Unimportant, and Can-Delete.
3. The script reads the `choice` value and the `confidence` value from the answer.
4. The script adds the matching Gmail label.

Eight requests run in parallel. The script retries a request with backoff after a 429, a 5xx, or a connection error.

The script also supports other models. If the slug does not contain `jev`, the script uses `/chat/completions`. The model must return a strict JSON list. This path exists for comparison.

## Labels

| Label | Meaning |
|---|---|
| `AI/Important` | Personal, work, finance, security, travel, or an item that needs action |
| `AI/Unimportant` | Low value but harmless. Examples are newsletters, notifications, and receipts. |
| `AI/Can-Delete` | Promotions, junk, and expired codes |

**The script only adds labels.** It never deletes, trashes, archives, or removes a message or a label. It reads only the From, Subject, Date, and snippet fields. It does not read message bodies.

## Setup

1. Create Google OAuth credentials.
   1. Open Google Cloud Console and enable the Gmail API.
   2. Create an OAuth client of type *Desktop app*.
   3. Save the JSON file as `credentials.json` in this directory.
   4. If the consent screen is in "Testing" mode, add your account as a test user.
2. Install the dependencies. This step uses [uv](https://docs.astral.sh/uv/).
   ```
   uv venv
   uv pip install -r requirements.txt
   ```
3. Set the environment variables.
   ```
   export OPENROUTER_API_KEY=...
   export MODEL_SLUG=~typesafe/jev-latest
   ```

The first run opens a browser for Google consent. The scope is `gmail.modify`. The script saves the token in `token.json`.

## Usage

Run a dry run first. It writes a CSV file and does not change Gmail.

```
.venv/bin/python gmail_triage.py --dry-run
```

Read the CSV. Then run the script without `--dry-run` to apply the labels.

```
.venv/bin/python gmail_triage.py --max 200
```

| Flag | Default | Description |
|---|---|---|
| `--max N` | 200 | Maximum number of new messages to process. The newest messages go first. |
| `--batch-size N` | 20 | Number of messages in each batch |
| `--dry-run` | off | Write a CSV file. Do not change Gmail. |
| `--csv PATH` | `dry_run.csv` | Path of the dry-run CSV file |
| `--processed-file PATH` | `processed_ids.txt` | List of processed message IDs. A new run skips these messages. |
| `--credentials PATH` | `credentials.json` | OAuth client file |
| `--token PATH` | `token.json` | Saved OAuth token |
| `-v` | off | Show debug messages |

The CSV file has these columns: `id`, `from`, `subject`, `category`, `reason`.

## State

The script adds message IDs to `processed_ids.txt` after it labels each batch. A dry run does not record any IDs. If Jev fails on a message, the script skips it. The next run tries it again.

## Files

| File | Purpose |
|---|---|
| `gmail_triage.py` | The script |
| `requirements.txt` | The dependencies |
| `credentials.json`, `token.json` | Google secrets. Git ignores them. |
| `processed_ids.txt`, `*.csv` | Personal mail data. Git ignores them. |

## Change the categories

Edit `DECISION_CRITERIA` in `gmail_triage.py`. Jev uses these descriptions to select a category. Better descriptions give better results.

## Sources

- [Jev tutorial on OpenRouter](https://openrouter.ai/docs/guides/community/jev-tutorial)
- [Jev model page](https://openrouter.ai/typesafe/jev-1.13)
