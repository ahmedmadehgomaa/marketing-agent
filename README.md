# Marketing Agent — Jackie AI Assistant

An n8n-based personal AI assistant that you can chat with through Telegram. The workflow accepts text messages and voice notes, uses an LLM to interpret requests, and connects to Google services for email, calendar events, and tasks. It also supports web search and a scheduled morning briefing.

## Features

- **Telegram chat interface** — send the assistant a text message.
- **Voice messages** — download Telegram voice notes and transcribe them before processing.
- **AI-powered responses** — uses an OpenRouter chat model (`cohere/north-mini-code:free` in the exported workflow).
- **Conversation memory** — keeps a rolling window of up to 20 context messages per chat session.
- **Google Calendar**
  - List events for a requested time range.
  - Create, update, and delete events.
- **Gmail**
  - Retrieve unread inbox messages.
  - Search emails and retrieve a message by ID.
  - Send email and reply to existing messages.
  - Mark a message as read.
- **Google Tasks**
  - List tasks, create tasks, complete tasks, and delete tasks.
- **Web search** — uses a Brave Search tool for current information.
- **Daily morning briefing** — scheduled for 08:00, requesting calendar events, unread-email summaries, and open tasks.
- **Long-message handling** — splits long assistant responses into Telegram-sized chunks.
- **Error response** — sends a fallback message when the AI agent reports an error.

## Workflow overview

```text
Telegram Trigger
  ├─ Show Typing
  └─ Voice or Text → If
       ├─ Empty message text → Get Voice File → Transcribe a recording ─┐
       └─ Text message ─────────────────────────────────────────────────┤
                                                                       ▼
                                                        Jackie, AI Assistant
                                                          ├─ OpenRouter model
                                                          ├─ Window Buffer Memory
                                                          ├─ Google Calendar tools
                                                          ├─ Gmail tools
                                                          ├─ Google Tasks tools
                                                          └─ Brave web search
                                                                       ▼
                                                           Split Long Reply
                                                                       ▼
                                                                 Telegram
```

A separate scheduled path runs `Every Morning → Briefing Request → Jackie, AI Assistant`. The briefing is sent to the configured Telegram chat ID.

## Requirements

- An n8n instance (the workflow export uses standard n8n nodes plus community/AI nodes).
- A Telegram bot and Telegram API credentials.
- An OpenRouter API account and a model available to your account.
- Google OAuth credentials with access to the relevant Google Calendar, Gmail, and Google Tasks resources.
- A Brave Search API credential for web search.
- An OpenAI-compatible audio transcription credential for the `Transcribe a recording` node.
- The Brave Search community node installed in n8n: `@brave/n8n-nodes-brave-search`.

> **Note:** The exported workflow references credential names, not portable credentials. Credentials must be created or mapped in your own n8n instance after importing.

## Setup

### 1. Import the workflow

1. Open your n8n instance.
2. Choose **Workflows → Import from File**.
3. Select `Marketing Agent.json`.
4. Open the imported workflow and review all credentials and configuration values before enabling it.

### 2. Configure credentials

Assign valid credentials to these nodes:

| Service | Nodes | What to configure |
|---|---|---|
| Telegram | `Listen for incoming events`, `Show Typing`, `Get Voice File`, `Telegram`, `Notify Error` | Telegram bot API credentials |
| OpenRouter | `OpenRouter` | OpenRouter API key and model access |
| Audio transcription | `Transcribe a recording` | A valid transcription provider credential |
| Google Calendar | `Google Calendar`, `create_event`, `update_event`, `delete_event` | Google OAuth and calendar access |
| Gmail | `Get Email`, `search_emails`, `get_email_by_id`, `reply_to_email`, `mark_email_as_read`, `Send Email` | Google OAuth and Gmail permissions |
| Google Tasks | `Create a task in Google Tasks`, `Get many tasks in Google Tasks`, `complete_task`, `delete_task` | Google OAuth and task-list access |
| Brave Search | `web_search` | Brave Search API credential |

The export contains account-specific calendar and task-list identifiers. Confirm these IDs belong to the intended Google account and task lists.

### 3. Set the morning briefing destination

Open the **Briefing Request** node and replace the `chatId` placeholder with the Telegram chat ID that should receive the daily briefing.

The **Every Morning** node is configured for 08:00. Check the workflow's timezone and your n8n instance timezone so the briefing runs at the intended local time.

### 4. Review the AI model and transcription configuration

- The **OpenRouter** node is configured to use `cohere/north-mini-code:free`. Change it if that model is unavailable or unsuitable for your use case.
- The **Transcribe a recording** node uses an OpenAI audio transcription operation. Verify the credential and provider settings.
- The workflow's **Jackie, AI Assistant** system message specifies concise Telegram-friendly replies and safety checks for destructive actions.

### 5. Verify permissions and activate

1. Test a basic Telegram text message.
2. Test a voice note and confirm transcription succeeds.
3. Test read-only Google Calendar, Gmail, and Google Tasks operations first.
4. Test create/update/delete actions only with disposable or clearly identified test items.
5. Test web search and the morning briefing.
6. Activate the workflow once all tests pass.

## Usage examples

Send a message to the Telegram bot. Example requests:

- `What is on my calendar today?`
- `Summarize my unread emails from today.`
- `Search my emails for the latest invoice.`
- `Create a calendar event called Project Review tomorrow at 10 AM.`
- `Create a task called Prepare the project report.`
- `Show my open tasks.`
- `Search the web for the latest n8n release.`
- Send a Telegram voice note with a request instead of typing.

Use explicit dates and times when possible. The assistant's system instructions say to assume today when a date is omitted, so include a date when you mean another day.

## Important behavior and limitations

- The morning briefing text asks for today's calendar events, a short unread-email summary, and open tasks.
- The configured memory is an in-memory sliding window of 20 context messages. Persistence across restarts is not guaranteed by this node.
- The workflow's AI instructions say to look up an event or task before updating, completing, or deleting it, and to request confirmation before destructive calendar/task actions or sending/replying to email unless the user has clearly asked for the exact action. Review actual agent behavior during testing.
- The workflow uses a Telegram Markdown parse mode. Certain characters in generated replies may need escaping if Telegram rejects a message.
- The response-splitting code targets chunks of about 3,800 characters. Test long messages and formatting in your Telegram client.
- The voice/text routing checks whether `message.text` is empty and then expects `message.voice.file_id`. Telegram updates that are neither text messages nor voice notes (such as photos, documents, or some other update types) may not be handled as intended.
- The workflow's Google Calendar listing requests summary and start time fields. Additional event details may not be available in that tool response.
- The exported JSON includes references to configured accounts and service IDs. Do not publish raw workflow exports without reviewing and removing account-specific or sensitive configuration details.

## Troubleshooting

### Telegram trigger does not receive messages

- Confirm the bot token is valid and the workflow is active.
- Check whether another workflow or service is using the bot's webhook.
- If n8n is running behind a tunnel or reverse proxy, verify the public webhook URL and HTTPS configuration.
- Inspect the failed execution in n8n for the trigger's error details.

### Voice messages fail

- Confirm the incoming update contains `message.voice.file_id`.
- Check the Telegram credential can retrieve the voice file.
- Verify the transcription node has valid credentials and receives binary audio data.

### Google tools fail

- Reconnect the relevant Google OAuth credential.
- Confirm the connected account has access to the target calendar, Gmail mailbox, or task list.
- Verify the configured calendar and task-list IDs.
- Check Google API consent and permissions if authorization errors occur.

### Web search fails

- Verify the Brave Search node is installed and enabled.
- Add or reconnect a valid Brave Search API credential.

### Morning briefing is not delivered

- Confirm the workflow is active and the schedule timezone is correct.
- Replace the placeholder chat ID in `Briefing Request`.
- Check Telegram permissions and inspect the execution history for node errors.

### Telegram rejects a response

- Check whether Markdown characters in the AI response are valid for Telegram's parse mode.
- Try a shorter test response and inspect the `Telegram` node's execution details.

## Security notes

- Treat Telegram bot tokens, API keys, OAuth credentials, and workflow exports as sensitive.
- Use n8n's credential store; do not paste secrets into node parameters or commit them to Git.
- Limit Google OAuth scopes to the access this workflow actually needs.
- Test destructive operations carefully and keep backups of important calendar events, emails, and tasks.
- Review access to the bot and workflow before sharing them with others.

## Project files

- `Marketing Agent.json` — n8n workflow export.
- `System Message.txt — Agent system message.

## License

No license is specified in the provided workflow export. Add a `LICENSE` file if you intend to distribute this project, and choose a license that matches your intentions.
