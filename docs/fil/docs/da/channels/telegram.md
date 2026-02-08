---
summary: "Status for Telegram-bot, funktioner og konfiguration"
read_when:
  - Arbejder med Telegram-funktioner eller webhooks
title: "Telegram"
x-i18n:
  source_path: channels/telegram.md
  source_hash: 604e2dc12d2b776d
  provider: openai
  model: gpt-5.2-chat-latest
  workflow: v1
  generated_at: 2026-02-08T10:51:08Z
---

# Telegram (Bot API)

Status: produktionsklar for bot-DM’er + grupper via grammY. Long-polling som standard; webhook er valgfri.

## Hurtig opsætning (begynder)

1. Opret en bot med **@BotFather** ([direkte link](https://t.me/BotFather)). Bekræft, at håndtaget er præcis `@BotFather`, og kopiér derefter tokenet.
2. Sæt tokenet:
   - Env: `TELEGRAM_BOT_TOKEN=...`
   - Eller config: `channels.telegram.botToken: "..."`.
   - Hvis begge er sat, har config forrang (env fallback er kun for standardkonto).
3. Start gatewayen.
4. DM-adgang er parring som standard; godkend parringskoden ved første kontakt.

Minimal konfiguration:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
    },
  },
}
```

## Hvad det er

- En Telegram Bot API-kanal ejet af Gateway.
- Deterministisk routing: svar sendes tilbage til Telegram; modellen vælger aldrig kanaler.
- DM’er deler agentens hovedsession; grupper holdes isoleret (`agent:<agentId>:telegram:group:<chatId>`).

## Opsætning (hurtig sti)

### 1. Opret et bot-token (BotFather)

1. Åbn Telegram og chat med **@BotFather** ([direkte link](https://t.me/BotFather)). Bekræft, at håndtaget er præcis `@BotFather`.
2. Kør `/newbot`, og følg derefter vejledningen (navn + brugernavn, der slutter på `bot`).
3. Kopiér tokenet og opbevar det sikkert.

Valgfrie BotFather-indstillinger:

- `/setjoingroups` — tillad/afvis at tilføje botten til grupper.
- `/setprivacy` — styr om botten ser alle gruppebeskeder.

### 2. Konfigurér tokenet (env eller config)

Eksempel:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

Env-mulighed: `TELEGRAM_BOT_TOKEN=...` (virker for standardkontoen).
Hvis både env og config er sat, har config forrang.

Understøttelse af flere konti: brug `channels.telegram.accounts` med tokens pr. konto og valgfri `name`. Se [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts) for det fælles mønster.

3. Start gatewayen. Telegram starter, når et token er løst (først config, derefter env fallback).
4. DM-adgang er som standard parring. Godkend koden, når botten kontaktes første gang.
5. For grupper: tilføj botten, beslut privatliv/admin-adfærd (nedenfor), og sæt derefter `channels.telegram.groups` for at styre mention-gating + tilladelseslister.

## Token + privatliv + tilladelser (Telegram-siden)

### Oprettelse af token (BotFather)

- `/newbot` opretter botten og returnerer tokenet (hold det hemmeligt).
- Hvis et token lækker, tilbagekald/regenerér det via @BotFather og opdatér din konfiguration.

### Synlighed af gruppebeskeder (Privacy Mode)

Telegram-bots er som standard i **Privacy Mode**, som begrænser hvilke gruppebeskeder de modtager.
Hvis din bot skal se _alle_ gruppebeskeder, har du to muligheder:

- Deaktivér privacy mode med `/setprivacy` **eller**
- Tilføj botten som **admin** i gruppen (admin-bots modtager alle beskeder).

**Bemærk:** Når du ændrer privacy mode, kræver Telegram, at botten fjernes og tilføjes igen
i hver gruppe, før ændringen træder i kraft.

### Gruppens tilladelser (admin-rettigheder)

Admin-status sættes inde i gruppen (Telegram UI). Admin-bots modtager altid alle
gruppebeskeder, så brug admin, hvis du har brug for fuld synlighed.

## Sådan virker det (adfærd)

- Indgående beskeder normaliseres til den fælles kanal-konvolut med svar-kontekst og medie-pladsholdere.
- Gruppesvar kræver som standard en mention (native @mention eller `agents.list[].groupChat.mentionPatterns` / `messages.groupChat.mentionPatterns`).
- Multi-agent-override: sæt per-agent-mønstre på `agents.list[].groupChat.mentionPatterns`.
- Svar routes altid tilbage til den samme Telegram-chat.
- Long-polling bruger grammY runner med per-chat-sekvensering; samlet samtidighed begrænses af `agents.defaults.maxConcurrent`.
- Telegram Bot API understøtter ikke læsekvitteringer; der er ingen `sendReadReceipts`-mulighed.

## Udkast-streaming

OpenClaw kan streame delvise svar i Telegram-DM’er ved brug af `sendMessageDraft`.

Krav:

- Threaded Mode aktiveret for botten i @BotFather (forum topic mode).
- Kun private chat-tråde (Telegram inkluderer `message_thread_id` på indgående beskeder).
- `channels.telegram.streamMode` må ikke være sat til `"off"` (standard: `"partial"`; `"block"` aktiverer chunkede udkastsopdateringer).

Udkast-streaming er kun for DM’er; Telegram understøtter det ikke i grupper eller kanaler.

## Formatering (Telegram HTML)

- Udgående Telegram-tekst bruger `parse_mode: "HTML"` (Telegram’s understøttede tag-undergruppe).
- Markdown-lignende input renderes til **Telegram-sikker HTML** (fed/kursiv/gennemstreget/kode/links); blok-elementer flades ud til tekst med linjeskift/punkttegn.
- Rå HTML fra modeller escapes for at undgå Telegram-parsefejl.
- Hvis Telegram afviser HTML-payloaden, forsøger OpenClaw igen med samme besked som ren tekst.

## Kommandoer (native + brugerdefinerede)

OpenClaw registrerer native kommandoer (som `/status`, `/reset`, `/model`) i Telegrams bot-menu ved opstart.
Du kan tilføje brugerdefinerede kommandoer til menuen via config:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
    },
  },
}
```

## Opsætningsfejlfinding (kommandoer)

- `setMyCommands failed` i logs betyder typisk, at udgående HTTPS/DNS er blokeret til `api.telegram.org`.
- Hvis du ser `sendMessage`- eller `sendChatAction`-fejl, så tjek IPv6-routing og DNS.

Mere hjælp: [Kanal-fejlfinding](/channels/troubleshooting).

Noter:

- Brugerdefinerede kommandoer er **kun menupunkter**; OpenClaw implementerer dem ikke, medmindre du håndterer dem andetsteds.
- Kommandonavne normaliseres (førende `/` fjernes, gøres til små bogstaver) og skal matche `a-z`, `0-9`, `_` (1–32 tegn).
- Brugerdefinerede kommandoer **kan ikke tilsidesætte native kommandoer**. Konflikter ignoreres og logges.
- Hvis `commands.native` er deaktiveret, registreres kun brugerdefinerede kommandoer (eller ryddes, hvis ingen).

## Grænser

- Udgående tekst chunkes til `channels.telegram.textChunkLimit` (standard 4000).
- Valgfri linjeskifts-chunking: sæt `channels.telegram.chunkMode="newline"` for at splitte på tomme linjer (afsnitsgrænser) før længde-chunking.
- Medie-downloads/uploads er begrænset af `channels.telegram.mediaMaxMb` (standard 5).
- Telegram Bot API-forespørgsler timeouter efter `channels.telegram.timeoutSeconds` (standard 500 via grammY). Sæt lavere for at undgå lange hængninger.
- Gruppehistorik-kontekst bruger `channels.telegram.historyLimit` (eller `channels.telegram.accounts.*.historyLimit`), med fallback til `messages.groupChat.historyLimit`. Sæt `0` for at deaktivere (standard 50).
- DM-historik kan begrænses med `channels.telegram.dmHistoryLimit` (brugeromgange). Per-bruger-overrides: `channels.telegram.dms["<user_id>"].historyLimit`.

## Gruppeaktiveringstilstande

Som standard svarer botten kun på mentions i grupper (`@botname` eller mønstre i `agents.list[].groupChat.mentionPatterns`). For at ændre denne adfærd:

### Via config (anbefalet)

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": { requireMention: false }, // always respond in this group
      },
    },
  },
}
```

**Vigtigt:** At sætte `channels.telegram.groups` opretter en **tilladelsesliste** — kun listede grupper (eller `"*"`) accepteres.
Forum-emner arver deres overordnede gruppekonfiguration (allowFrom, requireMention, skills, prompts), medmindre du tilføjer per-emne-overrides under `channels.telegram.groups.<groupId>.topics.<topicId>`.

For at tillade alle grupper med altid-svar:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false }, // all groups, always respond
      },
    },
  },
}
```

For at bevare mention-only for alle grupper (standardadfærd):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: true }, // or omit groups entirely
      },
    },
  },
}
```

### Via kommando (session-niveau)

Send i gruppen:

- `/activation always` – svar på alle beskeder
- `/activation mention` – kræv mentions (standard)

**Bemærk:** Kommandoer opdaterer kun sessionstilstand. For vedvarende adfærd på tværs af genstarter, brug config.

### Få gruppe-chat-ID’et

Videresend en vilkårlig besked fra gruppen til `@userinfobot` eller `@getidsbot` på Telegram for at se chat-ID’et (negativt tal som `-1001234567890`).

**Tip:** For dit eget bruger-ID kan du DM’e botten, og den svarer med dit bruger-ID (parringsbesked), eller bruge `/whoami`, når kommandoer er aktiveret.

**Privatlivsnote:** `@userinfobot` er en tredjepartsbot. Hvis du foretrækker det, så tilføj botten til gruppen, send en besked, og brug `openclaw logs --follow` til at læse `chat.id`, eller brug Bot API `getUpdates`.

## Konfigurationsskrivninger

Som standard har Telegram tilladelse til at skrive konfigurationsopdateringer, der udløses af kanalhændelser eller `/config set|unset`.

Dette sker, når:

- En gruppe opgraderes til en supergruppe, og Telegram udsender `migrate_to_chat_id` (chat-ID ændres). OpenClaw kan migrere `channels.telegram.groups` automatisk.
- Du kører `/config set` eller `/config unset` i en Telegram-chat (kræver `commands.config: true`).

Deaktiver med:

```json5
{
  channels: { telegram: { configWrites: false } },
}
```

## Emner (forum-supergrupper)

Telegram-forumemner inkluderer en `message_thread_id` pr. besked. OpenClaw:

- Tilføjer `:topic:<threadId>` til Telegram-gruppesessionsnøglen, så hvert emne er isoleret.
- Sender skriveindikatorer og svar med `message_thread_id`, så svar bliver i emnet.
- Generelt emne (thread id `1`) er specielt: beskedafsendelser udelader `message_thread_id` (Telegram afviser det), men skriveindikatorer inkluderer det stadig.
- Eksponerer `MessageThreadId` + `IsForum` i skabelonkontekst for routing/templating.
- Emnespecifik konfiguration er tilgængelig under `channels.telegram.groups.<chatId>.topics.<threadId>` (skills, tilladelseslister, auto-svar, systemprompter, deaktiver).
- Emnekontekster arver gruppeindstillinger (requireMention, tilladelseslister, skills, prompter, aktiveret), medmindre de tilsidesættes pr. emne.

Private chats kan i nogle kanttilfælde inkludere `message_thread_id`. OpenClaw holder DM-sessionsnøglen uændret, men bruger stadig thread-id’et til svar/udkast-streaming, når det er til stede.

## Inline-knapper

Telegram understøtter inline-tastaturer med callback-knapper.

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

For per-konto-konfiguration:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

Scopes:

- `off` — inline-knapper deaktiveret
- `dm` — kun DM’er (gruppe-mål blokeret)
- `group` — kun grupper (DM-mål blokeret)
- `all` — DM’er + grupper
- `allowlist` — DM’er + grupper, men kun afsendere tilladt af `allowFrom`/`groupAllowFrom` (samme regler som kontrolkommandoer)

Standard: `allowlist`.
Legacy: `capabilities: ["inlineButtons"]` = `inlineButtons: "all"`.

### Afsendelse af knapper

Brug message-værktøjet med parameteren `buttons`:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose an option:",
  buttons: [
    [
      { text: "Yes", callback_data: "yes" },
      { text: "No", callback_data: "no" },
    ],
    [{ text: "Cancel", callback_data: "cancel" }],
  ],
}
```

Når en bruger klikker på en knap, sendes callback-data tilbage til agenten som en besked med formatet:
`callback_data: value`

### Konfigurationsmuligheder

Telegram-funktioner kan konfigureres på to niveauer (objektform vist ovenfor; legacy streng-arrays understøttes stadig):

- `channels.telegram.capabilities`: Global standardfunktionskonfiguration anvendt på alle Telegram-konti, medmindre den tilsidesættes.
- `channels.telegram.accounts.<account>.capabilities`: Per-konto-funktioner, der tilsidesætter de globale standarder for den specifikke konto.

Brug den globale indstilling, når alle Telegram-bots/konti skal opføre sig ens. Brug per-konto-konfiguration, når forskellige bots har brug for forskellig adfærd (f.eks. håndterer én konto kun DM’er, mens en anden er tilladt i grupper).

## Adgangskontrol (DM’er + grupper)

### DM-adgang

- Standard: `channels.telegram.dmPolicy = "pairing"`. Ukendte afsendere modtager en parringskode; beskeder ignoreres, indtil de er godkendt (koder udløber efter 1 time).
- Godkend via:
  - `openclaw pairing list telegram`
  - `openclaw pairing approve telegram <CODE>`
- Parring er standard token-udveksling for Telegram-DM’er. Detaljer: [Parring](/channels/pairing)
- `channels.telegram.allowFrom` accepterer numeriske bruger-ID’er (anbefalet) eller `@username`-poster. Det er **ikke** bot-brugernavnet; brug den menneskelige afsenders ID. Opsætningsguiden accepterer `@username` og løser det til det numeriske ID, når det er muligt.

#### Find dit Telegram-bruger-ID

Sikrere (ingen tredjepartsbot):

1. Start gatewayen og DM din bot.
2. Kør `openclaw logs --follow` og kig efter `from.id`.

Alternativ (officiel Bot API):

1. DM din bot.
2. Hent opdateringer med dit bot-token og læs `message.from.id`:

   ```bash
   curl "https://api.telegram.org/bot<bot_token>/getUpdates"
   ```

Tredjepart (mindre privat):

- DM `@userinfobot` eller `@getidsbot` og brug det returnerede bruger-ID.

### Gruppeadgang

To uafhængige kontroller:

**1. Hvilke grupper er tilladt** (gruppe-tilladelsesliste via `channels.telegram.groups`):

- Ingen `groups`-konfiguration = alle grupper tilladt
- Med `groups`-konfiguration = kun listede grupper eller `"*"` er tilladt
- Eksempel: `"groups": { "-1001234567890": {}, "*": {} }` tillader alle grupper

**2. Hvilke afsendere er tilladt** (afsenderfiltrering via `channels.telegram.groupPolicy`):

- `"open"` = alle afsendere i tilladte grupper kan skrive
- `"allowlist"` = kun afsendere i `channels.telegram.groupAllowFrom` kan skrive
- `"disabled"` = ingen gruppebeskeder accepteres overhovedet
  Standard er `groupPolicy: "allowlist"` (blokeret, medmindre du tilføjer `groupAllowFrom`).

De fleste brugere ønsker: `groupPolicy: "allowlist"` + `groupAllowFrom` + specifikke grupper listet i `channels.telegram.groups`

For at tillade **ethvert gruppemedlem** at tale i en specifik gruppe (mens kontrolkommandoer stadig er begrænset til autoriserede afsendere), sæt en per-gruppe-override:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

## Long-polling vs webhook

- Standard: long-polling (ingen offentlig URL påkrævet).
- Webhook-tilstand: sæt `channels.telegram.webhookUrl` og `channels.telegram.webhookSecret` (valgfrit `channels.telegram.webhookPath`).
  - Den lokale lytter binder til `0.0.0.0:8787` og serverer `POST /telegram-webhook` som standard.
  - Hvis din offentlige URL er anderledes, brug en reverse proxy og peg `channels.telegram.webhookUrl` på det offentlige endpoint.

## Svar-trådning

Telegram understøtter valgfri trådede svar via tags:

- `[[reply_to_current]]` -- svar på den udløsende besked.
- `[[reply_to:<id>]]` -- svar på et specifikt besked-ID.

Styres af `channels.telegram.replyToMode`:

- `first` (standard), `all`, `off`.

## Lydbeskeder (stemme vs fil)

Telegram skelner mellem **talebeskeder** (rund boble) og **lydfiler** (metadata-kort).
OpenClaw bruger som standard lydfiler af hensyn til bagudkompatibilitet.

For at tvinge en talebesked-boble i agentens svar, inkludér dette tag et vilkårligt sted i svaret:

- `[[audio_as_voice]]` — send lyd som en talebesked i stedet for en fil.

Tagget fjernes fra den leverede tekst. Andre kanaler ignorerer dette tag.

For message-værktøjsafsendelser, sæt `asVoice: true` med en stemme-kompatibel lyd-`media`-URL
(`message` er valgfri, når medie er til stede):

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

## Klistermærker

OpenClaw understøtter modtagelse og afsendelse af Telegram-klistermærker med intelligent caching.

### Modtagelse af klistermærker

Når en bruger sender et klistermærke, håndterer OpenClaw det baseret på klistermærketypen:

- **Statiske klistermærker (WEBP):** Downloades og behandles via vision. Klistermærket vises som en `<media:sticker>`-pladsholder i beskedindholdet.
- **Animerede klistermærker (TGS):** Springes over (Lottie-format understøttes ikke til behandling).
- **Video-klistermærker (WEBM):** Springes over (videoformat understøttes ikke til behandling).

Skabelonkontekstfelt tilgængeligt ved modtagelse af klistermærker:

- `Sticker` — objekt med:
  - `emoji` — emoji knyttet til klistermærket
  - `setName` — navn på klistermærkesættet
  - `fileId` — Telegram-fil-ID (send samme klistermærke tilbage)
  - `fileUniqueId` — stabilt ID til cache-opslag
  - `cachedDescription` — cachet vision-beskrivelse, når tilgængelig

### Klistermærke-cache

Klistermærker behandles via AI’ens vision-funktioner for at generere beskrivelser. Da de samme klistermærker ofte sendes gentagne gange, cacher OpenClaw disse beskrivelser for at undgå redundante API-kald.

**Sådan virker det:**

1. **Første møde:** Klistermærkebilledet sendes til AI’en for vision-analyse. AI’en genererer en beskrivelse (f.eks. "En tegneseriekat, der vinker entusiastisk").
2. **Cache-lagring:** Beskrivelsen gemmes sammen med klistermærkets fil-ID, emoji og sætnavn.
3. **Efterfølgende møder:** Når det samme klistermærke ses igen, bruges den cachede beskrivelse direkte. Billedet sendes ikke til AI’en.

**Cache-placering:** `~/.openclaw/telegram/sticker-cache.json`

**Cache-indgangsformat:**

```json
{
  "fileId": "CAACAgIAAxkBAAI...",
  "fileUniqueId": "AgADBAADb6cxG2Y",
  "emoji": "👋",
  "setName": "CoolCats",
  "description": "A cartoon cat waving enthusiastically",
  "cachedAt": "2026-01-15T10:30:00.000Z"
}
```

**Fordele:**

- Reducerer API-omkostninger ved at undgå gentagne vision-kald for det samme klistermærke
- Hurtigere svartider for cachede klistermærker (ingen vision-behandlingsforsinkelse)
- Muliggør klistermærkesøgning baseret på cachede beskrivelser

Cachen udfyldes automatisk, efterhånden som klistermærker modtages. Der kræves ingen manuel cachehåndtering.

### Afsendelse af klistermærker

Agenten kan sende og søge klistermærker ved hjælp af handlingerne `sticker` og `sticker-search`. Disse er deaktiveret som standard og skal aktiveres i config:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

**Send et klistermærke:**

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

Parametre:

- `fileId` (påkrævet) — Telegram-fil-ID’et for klistermærket. Få dette fra `Sticker.fileId` ved modtagelse af et klistermærke, eller fra et `sticker-search`-resultat.
- `replyTo` (valgfrit) — besked-ID at svare på.
- `threadId` (valgfrit) — besked-tråd-ID for forumemner.

**Søg efter klistermærker:**

Agenten kan søge i cachede klistermærker efter beskrivelse, emoji eller sætnavn:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

Returnerer matchende klistermærker fra cachen:

```json5
{
  ok: true,
  count: 2,
  stickers: [
    {
      fileId: "CAACAgIAAxkBAAI...",
      emoji: "👋",
      description: "A cartoon cat waving enthusiastically",
      setName: "CoolCats",
    },
  ],
}
```

Søgningen bruger fuzzy matching på tværs af beskrivelsestekst, emoji-tegn og sætnavne.

**Eksempel med trådning:**

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "-1001234567890",
  fileId: "CAACAgIAAxkBAAI...",
  replyTo: 42,
  threadId: 123,
}
```

## Streaming (udkast)

Telegram kan streame **udkastbobler**, mens agenten genererer et svar.
OpenClaw bruger Bot API `sendMessageDraft` (ikke rigtige beskeder) og sender derefter
det endelige svar som en normal besked.

Krav (Telegram Bot API 9.3+):

- **Private chats med emner aktiveret** (forum topic mode for botten).
- Indgående beskeder skal inkludere `message_thread_id` (privat emne-tråd).
- Streaming ignoreres for grupper/supergrupper/kanaler.

Konfiguration:

- `channels.telegram.streamMode: "off" | "partial" | "block"` (standard: `partial`)
  - `partial`: opdatér udkastboblen med den seneste streamingtekst.
  - `block`: opdatér udkastboblen i større blokke (chunket).
  - `off`: deaktivér udkast-streaming.
- Valgfrit (kun for `streamMode: "block"`):
  - `channels.telegram.draftChunk: { minChars?, maxChars?, breakPreference? }`
    - standarder: `minChars: 200`, `maxChars: 800`, `breakPreference: "paragraph"` (clamped til `channels.telegram.textChunkLimit`).

Bemærk: udkast-streaming er adskilt fra **blokstreaming** (kanalbeskeder).
Blokstreaming er slået fra som standard og kræver `channels.telegram.blockStreaming: true`,
hvis du ønsker tidlige Telegram-beskeder i stedet for udkastsopdateringer.

Begrundelses-stream (kun Telegram):

- `/reasoning stream` streamer begrundelse ind i udkastboblen, mens svaret
  genereres, og sender derefter det endelige svar uden begrundelse.
- Hvis `channels.telegram.streamMode` er `off`, er begrundelses-streaming deaktiveret.
  Mere kontekst: [Streaming + chunking](/concepts/streaming).

## Retry-politik

Udgående Telegram API-kald genforsøges ved forbigående netværks-/429-fejl med eksponentiel backoff og jitter. Konfigurér via `channels.telegram.retry`. Se [Retry-politik](/concepts/retry).

## Agent-værktøj (beskeder + reaktioner)

- Værktøj: `telegram` med handlingen `sendMessage` (`to`, `content`, valgfrit `mediaUrl`, `replyToMessageId`, `messageThreadId`).
- Værktøj: `telegram` med handlingen `react` (`chatId`, `messageId`, `emoji`).
- Værktøj: `telegram` med handlingen `deleteMessage` (`chatId`, `messageId`).
- Semantik for fjernelse af reaktioner: se [/tools/reactions](/tools/reactions).
- Værktøjsgating: `channels.telegram.actions.reactions`, `channels.telegram.actions.sendMessage`, `channels.telegram.actions.deleteMessage` (standard: aktiveret) og `channels.telegram.actions.sticker` (standard: deaktiveret).

## Reaktionsnotifikationer

**Sådan fungerer reaktioner:**
Telegram-reaktioner ankommer som **separate `message_reaction`-events**, ikke som egenskaber i besked-payloads. Når en bruger tilføjer en reaktion, gør OpenClaw følgende:

1. Modtager `message_reaction`-opdateringen fra Telegram API
2. Konverterer den til et **systemevent** med formatet: `"Telegram reaction added: {emoji} by {user} on msg {id}"`
3. Enqueuer systemeventet ved brug af **samme sessionsnøgle** som almindelige beskeder
4. Når den næste besked ankommer i den samtale, drænes systemevents og foranstilles i agentens kontekst

Agenten ser reaktioner som **systemnotifikationer** i samtalehistorikken, ikke som beskedmetadata.

**Konfiguration:**

- `channels.telegram.reactionNotifications`: Styrer hvilke reaktioner der udløser notifikationer
  - `"off"` — ignorér alle reaktioner
  - `"own"` — notificér, når brugere reagerer på bot-beskeder (best-effort; i hukommelsen) (standard)
  - `"all"` — notificér for alle reaktioner

- `channels.telegram.reactionLevel`: Styrer agentens reaktionskapacitet
  - `"off"` — agenten kan ikke reagere på beskeder
  - `"ack"` — botten sender bekræftelsesreaktioner (👀 under behandling) (standard)
  - `"minimal"` — agenten kan reagere sparsomt (retningslinje: 1 pr. 5–10 udvekslinger)
  - `"extensive"` — agenten kan reagere liberalt, når det er passende

**Forumgrupper:** Reaktioner i forumgrupper inkluderer `message_thread_id` og bruger sessionsnøgler som `agent:main:telegram:group:{chatId}:topic:{threadId}`. Dette sikrer, at reaktioner og beskeder i samme emne holdes sammen.

**Eksempelkonfiguration:**

```json5
{
  channels: {
    telegram: {
      reactionNotifications: "all", // See all reactions
      reactionLevel: "minimal", // Agent can react sparingly
    },
  },
}
```

**Krav:**

- Telegram-bots skal eksplicit anmode om `message_reaction` i `allowed_updates` (konfigureres automatisk af OpenClaw)
- For webhook-tilstand er reaktioner inkluderet i webhook-`allowed_updates`
- For polling-tilstand er reaktioner inkluderet i `getUpdates` `allowed_updates`

## Leveringsmål (CLI/cron)

- Brug et chat-id (`123456789`) eller et brugernavn (`@name`) som mål.
- Eksempel: `openclaw message send --channel telegram --target 123456789 --message "hi"`.

## Fejlfinding

**Botten svarer ikke på ikke-mention-beskeder i en gruppe:**

- Hvis du har sat `channels.telegram.groups.*.requireMention=false`, skal Telegrams Bot API **privacy mode** være deaktiveret.
  - BotFather: `/setprivacy` → **Disable** (fjern derefter botten og tilføj den igen til gruppen)
- `openclaw channels status` viser en advarsel, når konfigurationen forventer umarkerede gruppebeskeder.
- `openclaw channels status --probe` kan yderligere tjekke medlemskab for eksplicitte numeriske gruppe-ID’er (den kan ikke auditere wildcard `"*"`-regler).
- Hurtig test: `/activation always` (kun session; brug config for persistens)

**Botten ser slet ikke gruppebeskeder:**

- Hvis `channels.telegram.groups` er sat, skal gruppen være listet eller bruge `"*"`
- Tjek Privacy Settings i @BotFather → "Group Privacy" skal være **OFF**
- Verificér, at botten faktisk er medlem (ikke kun admin uden læseadgang)
- Tjek gateway-logs: `openclaw logs --follow` (se efter "skipping group message")

**Botten svarer på mentions men ikke `/activation always`:**

- Kommandoen `/activation` opdaterer sessionstilstand, men persisterer ikke til config
- For vedvarende adfærd, tilføj gruppen til `channels.telegram.groups` med `requireMention: false`

**Kommandoer som `/status` virker ikke:**

- Sørg for, at dit Telegram-bruger-ID er autoriseret (via parring eller `channels.telegram.allowFrom`)
- Kommandoer kræver autorisation selv i grupper med `groupPolicy: "open"`

**Long-polling afbrydes straks på Node 22+ (ofte med proxies/custom fetch):**

- Node 22+ er strengere med `AbortSignal`-instanser; fremmede signaler kan afbryde `fetch`-kald med det samme.
- Opgradér til en OpenClaw-build, der normaliserer abort-signaler, eller kør gatewayen på Node 20, indtil du kan opgradere.

**Botten starter og stopper derefter stille med at svare (eller logger `HttpError: Network request ... failed`):**

- Nogle hosts opløser `api.telegram.org` til IPv6 først. Hvis din server ikke har fungerende IPv6-egress, kan grammY hænge på IPv6-only-forespørgsler.
- Løs ved at aktivere IPv6-egress **eller** tving IPv4-opløsning for `api.telegram.org` (f.eks. ved at tilføje en `/etc/hosts`-post med IPv4 A-recorden, eller foretræk IPv4 i dit OS’ DNS-stack), og genstart derefter gatewayen.
- Hurtig kontrol: `dig +short api.telegram.org A` og `dig +short api.telegram.org AAAA` for at bekræfte, hvad DNS returnerer.

## Konfigurationsreference (Telegram)

Fuld konfiguration: [Konfiguration](/gateway/configuration)

Udbyderindstillinger:

- `channels.telegram.enabled`: aktiver/deaktiver kanalopstart.
- `channels.telegram.botToken`: bot-token (BotFather).
- `channels.telegram.tokenFile`: læs token fra filsti.
- `channels.telegram.dmPolicy`: `pairing | allowlist | open | disabled` (standard: parring).
- `channels.telegram.allowFrom`: DM-tilladelsesliste (id’er/brugernavne). `open` kræver `"*"`.
- `channels.telegram.groupPolicy`: `open | allowlist | disabled` (standard: tilladelsesliste).
- `channels.telegram.groupAllowFrom`: gruppe-afsender-tilladelsesliste (id’er/brugernavne).
- `channels.telegram.groups`: per-gruppe-standarder + tilladelsesliste (brug `"*"` for globale standarder).
  - `channels.telegram.groups.<id>.groupPolicy`: per-gruppe-override for groupPolicy (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.requireMention`: standard for mention-gating.
  - `channels.telegram.groups.<id>.skills`: skill-filter (udeladt = alle skills, tom = ingen).
  - `channels.telegram.groups.<id>.allowFrom`: per-gruppe-afsender-tilladelsesliste-override.
  - `channels.telegram.groups.<id>.systemPrompt`: ekstra systemprompt for gruppen.
  - `channels.telegram.groups.<id>.enabled`: deaktivér gruppen, når `false`.
  - `channels.telegram.groups.<id>.topics.<threadId>.*`: per-emne-overrides (samme felter som gruppe).
  - `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`: per-emne-override for groupPolicy (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.topics.<threadId>.requireMention`: per-emne mention-gating-override.
- `channels.telegram.capabilities.inlineButtons`: `off | dm | group | all | allowlist` (standard: tilladelsesliste).
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`: per-konto-override.
- `channels.telegram.replyToMode`: `off | first | all` (standard: `first`).
- `channels.telegram.textChunkLimit`: udgående chunk-størrelse (tegn).
- `channels.telegram.chunkMode`: `length` (standard) eller `newline` for at splitte på tomme linjer (afsnitsgrænser) før længde-chunking.
- `channels.telegram.linkPreview`: slå link-forhåndsvisninger til/fra for udgående beskeder (standard: true).
- `channels.telegram.streamMode`: `off | partial | block` (udkast-streaming).
- `channels.telegram.mediaMaxMb`: grænse for indgående/udgående medier (MB).
- `channels.telegram.retry`: retry-politik for udgående Telegram API-kald (forsøg, minDelayMs, maxDelayMs, jitter).
- `channels.telegram.network.autoSelectFamily`: tilsidesæt Node autoSelectFamily (true=aktiver, false=deaktiver). Standard er deaktiveret på Node 22 for at undgå Happy Eyeballs-timeouts.
- `channels.telegram.proxy`: proxy-URL for Bot API-kald (SOCKS/HTTP).
- `channels.telegram.webhookUrl`: aktivér webhook-tilstand (kræver `channels.telegram.webhookSecret`).
- `channels.telegram.webhookSecret`: webhook-hemmelighed (påkrævet, når webhookUrl er sat).
- `channels.telegram.webhookPath`: lokal webhook-sti (standard `/telegram-webhook`).
- `channels.telegram.actions.reactions`: gate Telegram-værktøjsreaktioner.
- `channels.telegram.actions.sendMessage`: gate Telegram-værktøjs-beskedafsendelser.
- `channels.telegram.actions.deleteMessage`: gate Telegram-værktøjs-beskedsletninger.
- `channels.telegram.actions.sticker`: gate Telegram-klistermærkehandlinger — send og søg (standard: false).
- `channels.telegram.reactionNotifications`: `off | own | all` — styr hvilke reaktioner der udløser systemevents (standard: `own`, når ikke sat).
- `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` — styr agentens reaktionskapacitet (standard: `minimal`, når ikke sat).

Relaterede globale indstillinger:

- `agents.list[].groupChat.mentionPatterns` (mention-gating-mønstre).
- `messages.groupChat.mentionPatterns` (global fallback).
- `commands.native` (standard er `"auto"` → til for Telegram/Discord, fra for Slack), `commands.text`, `commands.useAccessGroups` (kommandoadfærd). Tilsidesæt med `channels.telegram.commands.native`.
- `messages.responsePrefix`, `messages.ackReaction`, `messages.ackReactionScope`, `messages.removeAckAfterReply`.
