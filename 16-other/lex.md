# Amazon Lex and Amazon Connect

## Amazon Lex

> **Status (2026):** **Amazon Lex V1 reached end of support on Sept 15, 2025** (V1 console and resources no longer accessible). Only **Lex V2** is current - use the V1-to-V2 migration tool (aliases, Lambda, logs, channels must be recreated).

- Provides text or voice conversational interfaces (Lex for voice, Lex for Alexa)
- Powers the Alexa service
- Lex provides 2 main bits of functionality:
    - Automatic speech recognition (ASR) - speech to text
    - Natural Language Understanding (NLU) - intent
- Lex allows us to build voice and text understanding into our applications
- Key concepts (V2):
    - **Bot** -> one or more **locales** (languages) in a single bot (V2 improvement - V1 needed one bot per language)
    - **Intent**: action the user wants (e.g. `BookHotel`), with **sample utterances** and **fulfillment**
    - **Slots**: parameters of an intent (built-in types like `AMAZON.Date` or custom slot types), with prompts and validation
    - **Lambda code hooks**: dialog code hook (validation, dynamic prompts) and fulfillment code hook (business logic) - one Lambda per bot alias/locale
    - **Versions and aliases**: immutable versions, aliases point to versions (e.g. `prod`)
    - Session attributes/context to carry state between intents
- Channels: web/mobile SDK, Amazon Connect (voice/chat IVR), Slack, Facebook Messenger, Twilio
- Generative AI features (powered by Amazon Bedrock): descriptive bot builder (create a bot from a description), utterance generation, **assisted slot resolution**, **QnAIntent** (answers FAQs from a Bedrock knowledge base / Kendra / OpenSearch index), assisted NLU
- Conversation logs (text to CloudWatch Logs, audio to S3), analytics dashboards, test workbench
- Lex vs Bedrock Agents: Lex = structured intent/slot conversational flows (IVR, form filling, deterministic); Bedrock Agents/LLMs = open-ended generative conversations and tool use - often combined (Lex QnAIntent or Lex fallback to Bedrock)
- It scales well, integrates with other AWS services, it is quick to deploy and it has a pay as you go pricing model
- Use cases:
    - Chatbots
    - Voice Assistants
    - Q&A Bots
    - Info/Enterprise Bots

## Amazon Connect

- It is a contact center as a service
- Requires no infrastructure in on-premises
- It is omnichannel: voice and chat, incoming and outgoing
- Integrates with PSTN networks for traditional voice, allowing us to accept incoming calls and make outgoing calls using the traditional cellular phone networks
- Agents can connect using the internet from anywhere
- AWS Connect can integrate with other services such as Lambda/Lex for additional intelligence and features
- It is quick to provision, provides a pay as you go pricing. It is scalable
- 2026 naming: the contact center product is now marketed as **Amazon Connect Customer** (part of the Amazon Connect family with Connect Decisions, Talent and Health) - same service and APIs
- Key concepts:
    - **Instance**: identity management (Connect-managed, SAML 2.0, or existing directory), phone numbers (claimed DID/toll-free or ported), queues, routing profiles, hours of operation
    - **Flows** (contact flows): drag-and-drop IVR/chat logic - play prompts, invoke Lex bots, Lambda functions (e.g. look up customer in DynamoDB/CRM), set queues, transfer
    - **Contact Control Panel (CCP)** / agent workspace in browser (softphone)
    - Channels: voice, chat, SMS, tasks, email, WhatsApp/Apple Messages, outbound campaigns
- **Contact Lens**: real-time and post-call analytics - transcription, sentiment, PII redaction, categories/rules, conversational analytics, agent performance evaluation, generative post-contact summaries
- **AI agents** (formerly *Amazon Q in Connect*, earlier *Connect Wisdom*): generative AI assistance for human agents (suggested responses, knowledge retrieval) and self-service AI agents for customers, supports MCP tools
- **Customer Profiles** (unified customer view from CRMs/S3), **Cases**, ~~Voice ID~~ (voice authentication - **end of support May 20, 2026**; use partner solutions such as Pindrop)
- Data/streaming: contact trace records (CTRs) and agent events via Kinesis Data Streams/Firehose; call recordings in S3 (KMS); real-time media via **Kinesis Video Streams** (e.g. for custom real-time transcription with Transcribe)
- Exam: "replace on-prem call center quickly, scale with seasonal demand, pay per minute, integrate IVR chatbot" -> Amazon Connect + Lex + Lambda <span style="color: red;">EXAM</span>
