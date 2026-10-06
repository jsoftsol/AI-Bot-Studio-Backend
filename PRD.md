# AI Bot Studio Backend: Product Requirements Document (reverse-engineered)

This spec was reconstructed from the shipped backend code, not written before it. It describes what the system does today. Where something is inferred from names or structure, that is said. The web front ends were not reviewed here. Credentials, hostnames and security internals are left out on purpose.

## 1. Problem statement

Small businesses want a chat bot that answers customer questions from their own material, such as their website, documents and FAQs, and they want it on the channels customers already use. Building that means connecting a language model, a way to store and search the business's content, a chat interface, and each messaging platform's own API.

AI Bot Studio does that work. A business uploads its content, trains a bot and publishes it. An agency can do the same for many clients.

## 2. Target users

| User | What they need |
|---|---|
| Business owner | A bot that answers from their content, and the leads and questions it collects |
| Agency | Many client accounts, each with its own bots, managed from one login |
| Website visitor or customer | Quick, relevant answers on the site, in a messaging app or by phone |
| Support team | Bot-created tickets and bookings in Freshdesk, Calendly or Square |
| Platform admin | Provisioning of users, custom domains and API keys |

## 3. Scope

### 3.1 Implemented

- **Accounts.** Register, login, password reset, change password, per-user API token, learning progress, agency sub-clients with login-as-client.
- **Bring your own key.** Users store their own OpenAI key (encrypted) and attach it to bots.
- **Bots.** Create, clone, edit, delete, groups, tags, custom domains with URL slugs, page and popup styling, welcome gate, call-to-action, quick answers.
- **Knowledge.** File upload (PDF, Word, text and images), web page scraping with summaries, free text, question and answer pairs.
- **Training.** Queued job that splits content, embeds it and writes it to a vector index, with progress reported over sockets.
- **Answering.** Retrieval from the vector index, streaming answers on an OpenAI thread, conversation history, function calls to Freshdesk, Calendly and Square.
- **Instruction generator** and an **assistant library**.
- **Channels.** Website embed (page and widget), WhatsApp Business, Facebook Messenger, Instagram messages, Instagram and Messenger comment and message automations, Gmail auto-replies, phone calls with a realtime voice bot.
- **Leads, conversations, queries, statistics, notifications.**
- **Custom domains** with SSL status polling.
- **Integrations.** Global Control for email and tag firing, SaaSOnboard for user provisioning.

### 3.2 Scaffolded or partial

- **Usage tracking.** Questions are logged, but token counts, cost and quotas are not.
- **Global Control module.** Two routes only.
- **Old conversation module** next to the newer conversation manager.
- **Voice.** Several overlapping handler files, including a tracked backup.
- **Beta endpoint pairs** for loading a bot, querying and saving the welcome gate, some wired to the wrong handler.
- **Real-time chat gateway** that is commented out.
- **A few installed packages** that are no longer used.

### 3.3 Out of scope

- Billing and payments (handled outside this service)
- Model providers other than OpenAI
- Role-based permissions
- Generic outbound webhooks
- A user interface (separate front ends)
- Automated tests

## 4. Key flows

### 4.1 Build and train a bot

1. The user creates a bot and writes or generates its instructions.
2. The user adds knowledge: uploads files, adds web links, pastes text or enters question and answer pairs.
3. File and page jobs run on queues. They extract text, scrape pages and summarize long pages.
4. Training splits the text into chunks, drops tiny ones, embeds each chunk and stores it in the vector index. Progress shows as a percentage in the app.

### 4.2 A visitor asks a question

1. The embed loads the bot, either by ID or by the custom domain and URL slug of the page.
2. The visitor's question is embedded and compared with the bot's chunks. The best matches are added to the OpenAI thread as context.
3. The answer streams back to the visitor. The thread keeps the conversation memory.
4. The question, answer, timing, location and outcome are logged. Lead details from the welcome gate are saved.
5. When the bot decides to, it calls a function: create a Freshdesk ticket, or look up or book an appointment.

### 4.3 Connect a messaging channel

1. The user starts a connection for WhatsApp, Messenger, Instagram or Gmail, and the platform's login flow returns a code.
2. The server stores the connection on the bot and subscribes to the platform's incoming messages.
3. Incoming messages arrive on a webhook, go through the same answering path and the reply goes back through the platform's API.
4. Gmail watches are renewed daily. Instagram and Messenger can also reply to comments by rule.

### 4.4 A phone call

1. Twilio calls the incoming-call webhook.
2. A WebSocket server bridges the call audio to the OpenAI Realtime API.
3. The model answers in voice using the bot's instructions.

### 4.5 An agency manages a client

1. The agency creates a client account under its own.
2. The agency logs in as the client using a second token header and manages the client's bots.

## 5. Non-functional characteristics

- **Concurrency.** Queue workers run 20 jobs at a time. The dev server runs with a raised memory limit.
- **Streaming.** Answers stream over sockets or the HTTP response.
- **Observability.** Console logging only, with query and conversation records as an audit trail.
- **Deployment.** pm2 with separate dev and production processes. The build copies the environment file into the output folder.
- **Testing.** None automated.

## 6. Risks

| Risk | Detail | Effect |
|---|---|---|
| Security review overdue | Credential handling, authentication coverage, input validation and webhook checks need an audit | Exposure of client and customer data |
| No usage or cost control | No token counts, cost records, quotas or per-bot limits | Hard to bill, limit or cap AI spend |
| Dependence on one vendor | OpenAI Assistants, Realtime and an older embedding model | Migration work when those change |
| Scraping at scale | Headless browser jobs are heavy on memory and CPU | Resource pressure on the server |
| No tests or CI | Manual build and reload | Regressions go unnoticed |
| Circular modules and very large files | Forward references, services over 1,500 lines | Hard to change safely |
| Local disk writes | Files and extracted text land on local folders | Disk growth and portability limits |

## 7. Open questions

- Where do billing and plan limits live, and should usage be recorded here?
- Which of the legacy conversation, voice and beta paths can be removed?
- What is the plan for moving off the OpenAI Assistants API?
- Should each bot have its own usage limit?
