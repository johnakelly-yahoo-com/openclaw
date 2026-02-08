---
summary: "حالة دعم بوت Telegram، والإمكانات، والتهيئة"
read_when:
  - العمل على ميزات Telegram أو webhooks
title: "Telegram"
x-i18n:
  source_path: channels/telegram.md
  source_hash: 604e2dc12d2b776d
  provider: openai
  model: gpt-5.2-chat-latest
  workflow: v1
  generated_at: 2026-02-08T10:49:22Z
---

# Telegram (Bot API)

الحالة: جاهز للإنتاج للمحادثات الخاصة مع البوت + المجموعات عبر grammY. يعتمد الاستطلاع الطويل افتراضيًا؛ webhook اختياري.

## البدء السريع (للمبتدئين)

1. أنشئ بوتًا باستخدام **@BotFather** ([رابط مباشر](https://t.me/BotFather)). تأكّد من أن المعرّف هو بالضبط `@BotFather`، ثم انسخ الرمز المميّز.
2. عيّن الرمز المميّز:
   - متغيرات البيئة: `TELEGRAM_BOT_TOKEN=...`
   - أو في التهيئة: `channels.telegram.botToken: "..."`.
   - إذا تم تعيين الاثنين، تكون أولوية التهيئة أعلى (الرجوع إلى متغيرات البيئة يكون فقط للحساب الافتراضي).
3. شغّل Gateway.
4. الوصول عبر الرسائل الخاصة يتم عبر الإقران افتراضيًا؛ وافق على رمز الإقران عند أول تواصل.

التهيئة الدنيا:

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

## ما هو

- قناة Telegram Bot API مملوكة لـ Gateway.
- توجيه حتمي: الردود تعود إلى Telegram؛ ولا يختار النموذج القنوات.
- تشترك الرسائل الخاصة في الجلسة الرئيسية للوكيل؛ بينما تبقى المجموعات معزولة (`agent:<agentId>:telegram:group:<chatId>`).

## الإعداد (المسار السريع)

### 1. إنشاء رمز بوت (BotFather)

1. افتح Telegram وابدأ محادثة مع **@BotFather** ([رابط مباشر](https://t.me/BotFather)). تأكّد من أن المعرّف هو بالضبط `@BotFather`.
2. نفّذ `/newbot`، ثم اتبع التعليمات (الاسم + اسم مستخدم ينتهي بـ `bot`).
3. انسخ الرمز المميّز واحفظه بأمان.

إعدادات BotFather الاختيارية:

- `/setjoingroups` — السماح/المنع لإضافة البوت إلى المجموعات.
- `/setprivacy` — التحكّم فيما إذا كان البوت يرى جميع رسائل المجموعة.

### 2. تهيئة الرمز المميّز (بيئة أو تهيئة)

مثال:

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

خيار متغيرات البيئة: `TELEGRAM_BOT_TOKEN=...` (يعمل للحساب الافتراضي).
إذا تم تعيين متغيرات البيئة والتهيئة معًا، تكون أولوية التهيئة أعلى.

دعم تعدد الحسابات: استخدم `channels.telegram.accounts` مع رموز مميّزة لكل حساب و `name` اختياريًا. انظر [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts) للنمط المشترك.

3. شغّل Gateway. يبدأ Telegram عند حلّ رمز مميّز (التهيئة أولًا، ثم الرجوع إلى متغيرات البيئة).
4. الوصول عبر الرسائل الخاصة افتراضيه الإقران. وافق على الرمز عند أول تواصل مع البوت.
5. للمجموعات: أضف البوت، قرّر سلوك الخصوصية/الإدارة (أدناه)، ثم عيّن `channels.telegram.groups` للتحكّم ببوابة الذِكر + قوائم السماح.

## الرمز المميّز + الخصوصية + الصلاحيات (جانب Telegram)

### إنشاء الرمز المميّز (BotFather)

- `/newbot` ينشئ البوت ويعيد الرمز المميّز (احتفظ به سرًا).
- إذا تسرّب الرمز، ألغِه/أعد توليده عبر @BotFather وحدّث التهيئة.

### رؤية رسائل المجموعات (وضع الخصوصية)

تعمل بوتات Telegram افتراضيًا في **وضع الخصوصية**، ما يقيّد الرسائل الجماعية التي تتلقّاها.
إذا كان يجب أن يرى البوت _جميع_ رسائل المجموعة، فلديك خياران:

- تعطيل وضع الخصوصية باستخدام `/setprivacy` **أو**
- إضافة البوت كـ **مشرف** في المجموعة (البوتات المشرفة تتلقّى جميع الرسائل).

**ملاحظة:** عند تبديل وضع الخصوصية، يتطلب Telegram إزالة البوت وإعادة إضافته
إلى كل مجموعة حتى يسري التغيير.

### صلاحيات المجموعات (حقوق المشرف)

تُحدَّد حالة المشرف داخل المجموعة (واجهة Telegram). تتلقّى البوتات المشرفة دائمًا جميع
رسائل المجموعة، لذا استخدم حالة المشرف إذا احتجت رؤية كاملة.

## كيف يعمل (السلوك)

- تُطبَّع الرسائل الواردة إلى غلاف القناة المشترك مع سياق الردّ ومواضع وسائط.
- تتطلب ردود المجموعات ذكرًا افتراضيًا (ذكر @ الأصلي أو `agents.list[].groupChat.mentionPatterns` / `messages.groupChat.mentionPatterns`).
- تجاوز متعدد الوكلاء: عيّن أنماطًا لكل وكيل على `agents.list[].groupChat.mentionPatterns`.
- تعود الردود دائمًا إلى نفس دردشة Telegram.
- يستخدم الاستطلاع الطويل مُشغّل grammY مع تسلسل لكل دردشة؛ ويُقيَّد التوازي العام بواسطة `agents.defaults.maxConcurrent`.
- لا يدعم Telegram Bot API إيصالات القراءة؛ ولا يوجد خيار `sendReadReceipts`.

## بثّ المسودات

يمكن لـ OpenClaw بث ردود جزئية في الرسائل الخاصة على Telegram باستخدام `sendMessageDraft`.

المتطلبات:

- تمكين وضع الخيوط للبوت في @BotFather (وضع مواضيع المنتدى).
- خيوط الدردشة الخاصة فقط (يتضمن Telegram `message_thread_id` في الرسائل الواردة).
- عدم تعيين `channels.telegram.streamMode` إلى `"off"` (الافتراضي: `"partial"`، ويُمكّن `"block"` تحديثات مسودة مجزّأة).

بثّ المسودات مخصّص للرسائل الخاصة فقط؛ ولا يدعمه Telegram في المجموعات أو القنوات.

## التنسيق (Telegram HTML)

- يستخدم نص Telegram الصادر `parse_mode: "HTML"` (مجموعة وسوم Telegram المدعومة).
- تُحوَّل المدخلات الشبيهة بـ Markdown إلى **HTML آمن لـ Telegram** (غامق/مائل/شطب/شفرة/روابط)؛ وتُسطَّح عناصر الكتل إلى نص مع أسطر جديدة/نقاط.
- يُهَرَّب HTML الخام من النماذج لتجنّب أخطاء التحليل في Telegram.
- إذا رفض Telegram حمولة HTML، يعيد OpenClaw إرسال الرسالة نفسها كنص عادي.

## الأوامر (الأصلية + المخصّصة)

يسجّل OpenClaw أوامر أصلية (مثل `/status`، `/reset`، `/model`) مع قائمة بوت Telegram عند بدء التشغيل.
يمكنك إضافة أوامر مخصّصة إلى القائمة عبر التهيئة:

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

## استكشاف أخطاء الإعداد (الأوامر)

- ظهور `setMyCommands failed` في السجلات يعني عادةً أن HTTPS/DNS الصادر محجوب إلى `api.telegram.org`.
- إذا رأيت إخفاقات `sendMessage` أو `sendChatAction`، تحقّق من توجيه IPv6 وDNS.

مزيد من المساعدة: [استكشاف أخطاء القنوات](/channels/troubleshooting).

ملاحظات:

- الأوامر المخصّصة هي **عناصر قائمة فقط**؛ لا ينفّذها OpenClaw ما لم تتعامل معها في مكان آخر.
- تُطبَّع أسماء الأوامر (إزالة بادئة `/`، والتحويل إلى أحرف صغيرة) ويجب أن تطابق `a-z`، `0-9`، `_` (1–32 حرفًا).
- لا يمكن للأوامر المخصّصة **تجاوز الأوامر الأصلية**. يتم تجاهل التعارضات وتسجيلها.
- إذا كان `commands.native` معطّلًا، تُسجَّل الأوامر المخصّصة فقط (أو تُزال إن لم توجد).

## الحدود

- يُجزَّأ النص الصادر إلى `channels.telegram.textChunkLimit` (الافتراضي 4000).
- تجزئة اختيارية على الأسطر الجديدة: عيّن `channels.telegram.chunkMode="newline"` للتقسيم على الأسطر الفارغة (حدود الفقرات) قبل تجزئة الطول.
- تُقيَّد تنزيلات/تحميلات الوسائط بـ `channels.telegram.mediaMaxMb` (الافتراضي 5).
- تنتهي مهلة طلبات Telegram Bot API بعد `channels.telegram.timeoutSeconds` (الافتراضي 500 عبر grammY). خفّضها لتجنّب التعليق الطويل.
- يستخدم سياق سجلّ المجموعة `channels.telegram.historyLimit` (أو `channels.telegram.accounts.*.historyLimit`)، مع الرجوع إلى `messages.groupChat.historyLimit`. عيّن `0` لتعطيله (الافتراضي 50).
- يمكن تقييد سجلّ الرسائل الخاصة بـ `channels.telegram.dmHistoryLimit` (عدد أدوار المستخدم). تجاوزات لكل مستخدم: `channels.telegram.dms["<user_id>"].historyLimit`.

## أوضاع تفعيل المجموعات

افتراضيًا، يردّ البوت فقط على الذِكر في المجموعات (`@botname` أو أنماط في `agents.list[].groupChat.mentionPatterns`). لتغيير هذا السلوك:

### عبر التهيئة (مُوصى به)

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

**مهم:** تعيين `channels.telegram.groups` ينشئ **قائمة سماح** — لن تُقبَل إلا المجموعات المدرجة (أو `"*"`).
ترث مواضيع المنتدى إعدادات المجموعة الأصلية (allowFrom، requireMention، skills، prompts) ما لم تضف تجاوزات لكل موضوع ضمن `channels.telegram.groups.<groupId>.topics.<topicId>`.

للسماح لجميع المجموعات مع الرد الدائم:

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

للاحتفاظ بالرد على الذِكر فقط لكل المجموعات (السلوك الافتراضي):

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

### عبر الأمر (على مستوى الجلسة)

أرسل في المجموعة:

- `/activation always` — الرد على جميع الرسائل
- `/activation mention` — يتطلّب الذِكر (الافتراضي)

**ملاحظة:** تحدّث الأوامر حالة الجلسة فقط. للسلوك الدائم عبر إعادة التشغيل، استخدم التهيئة.

### الحصول على معرّف دردشة المجموعة

أعد توجيه أي رسالة من المجموعة إلى `@userinfobot` أو `@getidsbot` على Telegram لرؤية معرّف الدردشة (رقم سالب مثل `-1001234567890`).

**نصيحة:** للحصول على معرّف المستخدم الخاص بك، أرسل رسالة خاصة للبوت وسيجيب بمعرّفك (رسالة الإقران)، أو استخدم `/whoami` بعد تمكين الأوامر.

**ملاحظة الخصوصية:** `@userinfobot` بوت تابع لجهة خارجية. إن فضّلت، أضف البوت إلى المجموعة، أرسل رسالة، واستخدم `openclaw logs --follow` لقراءة `chat.id`، أو استخدم Bot API `getUpdates`.

## كتابة التهيئة

افتراضيًا، يُسمح لـ Telegram بكتابة تحديثات التهيئة المُشغَّلة بأحداث القناة أو `/config set|unset`.

يحدث هذا عندما:

- تتم ترقية مجموعة إلى supergroup ويُصدر Telegram `migrate_to_chat_id` (يتغيّر معرّف الدردشة). يمكن لـ OpenClaw ترحيل `channels.telegram.groups` تلقائيًا.
- تُنفّذ `/config set` أو `/config unset` في دردشة Telegram (يتطلب `commands.config: true`).

عطّل ذلك باستخدام:

```json5
{
  channels: { telegram: { configWrites: false } },
}
```

## المواضيع (مجموعات المنتدى)

تتضمن مواضيع منتدى Telegram `message_thread_id` لكل رسالة. يقوم OpenClaw بما يلي:

- يضيف `:topic:<threadId>` إلى مفتاح جلسة مجموعة Telegram بحيث يكون كل موضوع معزولًا.
- يرسل مؤشرات الكتابة والردود مع `message_thread_id` لتبقى الردود ضمن الموضوع.
- الموضوع العام (معرّف الخيط `1`) خاص: ترسل الرسائل دون `message_thread_id` (يرفضه Telegram)، لكن مؤشرات الكتابة ما زالت تتضمنه.
- يعرِض `MessageThreadId` + `IsForum` في سياق القالب للتوجيه/القوالب.
- تتوفر تهيئة خاصة بالموضوع تحت `channels.telegram.groups.<chatId>.topics.<threadId>` (skills، قوائم السماح، الردّ التلقائي، مطالبات النظام، التعطيل).
- ترث تهيئات الموضوع إعدادات المجموعة (requireMention، قوائم السماح، skills، prompts، التمكين) ما لم يتم تجاوزها لكل موضوع.

قد تتضمن الدردشات الخاصة `message_thread_id` في بعض الحالات الحدّية. يُبقي OpenClaw مفتاح جلسة الرسائل الخاصة دون تغيير، لكنه ما زال يستخدم معرّف الخيط للردود/بثّ المسودات عند وجوده.

## الأزرار المضمّنة

يدعم Telegram لوحات مفاتيح مضمّنة بأزرار ردّ.

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

لتهيئة لكل حساب:

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

النطاقات:

- `off` — تعطيل الأزرار المضمّنة
- `dm` — الرسائل الخاصة فقط (حظر أهداف المجموعات)
- `group` — المجموعات فقط (حظر أهداف الرسائل الخاصة)
- `all` — الرسائل الخاصة + المجموعات
- `allowlist` — الرسائل الخاصة + المجموعات، لكن فقط للمرسلين المسموح لهم بواسطة `allowFrom`/`groupAllowFrom` (نفس قواعد أوامر التحكّم)

الافتراضي: `allowlist`.
قديم: `capabilities: ["inlineButtons"]` = `inlineButtons: "all"`.

### إرسال الأزرار

استخدم أداة الرسائل مع معامل `buttons`:

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

عند نقر المستخدم زرًا، تُرسل بيانات الاستدعاء إلى الوكيل كرسالة بالتنسيق:
`callback_data: value`

### خيارات التهيئة

يمكن تهيئة إمكانات Telegram على مستويين (يُعرض شكل الكائن أعلاه؛ لا تزال مصفوفات السلاسل القديمة مدعومة):

- `channels.telegram.capabilities`: تهيئة إمكانات افتراضية عامة تُطبّق على جميع حسابات Telegram ما لم يتم تجاوزها.
- `channels.telegram.accounts.<account>.capabilities`: إمكانات لكل حساب تتجاوز الافتراضيات العامة لذلك الحساب تحديدًا.

استخدم الإعداد العام عندما يجب أن تتصرف جميع بوتات/حسابات Telegram بالطريقة نفسها. استخدم تهيئة لكل حساب عندما تحتاج بوتات مختلفة إلى سلوكيات مختلفة (مثلًا، حساب يتعامل مع الرسائل الخاصة فقط وآخر مسموح له في المجموعات).

## التحكّم في الوصول (الرسائل الخاصة + المجموعات)

### الوصول عبر الرسائل الخاصة

- الافتراضي: `channels.telegram.dmPolicy = "pairing"`. يتلقى المرسلون المجهولون رمز إقران؛ وتُتجاهل الرسائل حتى تتم الموافقة (تنتهي صلاحية الرموز بعد ساعة).
- الموافقة عبر:
  - `openclaw pairing list telegram`
  - `openclaw pairing approve telegram <CODE>`
- الإقران هو تبادل الرموز الافتراضي المستخدم لرسائل Telegram الخاصة. التفاصيل: [الإقران](/channels/pairing)
- يقبل `channels.telegram.allowFrom` معرّفات مستخدمين رقمية (مُوصى به) أو إدخالات `@username`. **ليس** اسم مستخدم البوت؛ استخدم معرّف المُرسل البشري. يقبل المعالج `@username` ويحوّله إلى المعرّف الرقمي عند الإمكان.

#### العثور على معرّف مستخدم Telegram الخاص بك

أكثر أمانًا (دون بوت طرف ثالث):

1. شغّل Gateway وأرسل رسالة خاصة إلى البوت.
2. نفّذ `openclaw logs --follow` وابحث عن `from.id`.

بديل (Bot API الرسمي):

1. أرسل رسالة خاصة إلى البوت.
2. اجلب التحديثات باستخدام رمز البوت واقرأ `message.from.id`:

   ```bash
   curl "https://api.telegram.org/bot<bot_token>/getUpdates"
   ```

طرف ثالث (أقل خصوصية):

- أرسل رسالة خاصة إلى `@userinfobot` أو `@getidsbot` واستخدم معرّف المستخدم المُعاد.

### الوصول إلى المجموعات

تحكّمان مستقلان:

**1. أي المجموعات مسموح بها** (قائمة سماح المجموعات عبر `channels.telegram.groups`):

- عدم وجود تهيئة `groups` = جميع المجموعات مسموح بها
- مع تهيئة `groups` = فقط المجموعات المدرجة أو `"*"` مسموح بها
- مثال: `"groups": { "-1001234567890": {}, "*": {} }` يسمح بكل المجموعات

**2. أي المرسلين مسموح لهم** (تصفية المرسلين عبر `channels.telegram.groupPolicy`):

- `"open"` = جميع المرسلين في المجموعات المسموح بها يمكنهم الإرسال
- `"allowlist"` = فقط المرسلون في `channels.telegram.groupAllowFrom` يمكنهم الإرسال
- `"disabled"` = لا تُقبل أي رسائل مجموعات إطلاقًا
  الافتراضي هو `groupPolicy: "allowlist"` (محجوب ما لم تُضف `groupAllowFrom`).

يريد معظم المستخدمين: `groupPolicy: "allowlist"` + `groupAllowFrom` + مجموعات محددة مدرجة في `channels.telegram.groups`

للسماح **لأي عضو في المجموعة** بالتحدث في مجموعة محددة (مع إبقاء أوامر التحكّم مقيّدة بالمرسلين المصرّح لهم)، عيّن تجاوزًا لكل مجموعة:

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

## الاستطلاع الطويل مقابل webhook

- الافتراضي: الاستطلاع الطويل (لا يتطلب عنوان URL عامًا).
- وضع webhook: عيّن `channels.telegram.webhookUrl` و `channels.telegram.webhookSecret` (اختياريًا `channels.telegram.webhookPath`).
  - يرتبط المستمع المحلي بـ `0.0.0.0:8787` ويخدم `POST /telegram-webhook` افتراضيًا.
  - إذا كان عنوان URL العام مختلفًا، استخدم وكيلًا عكسيًا ووجّه `channels.telegram.webhookUrl` إلى نقطة النهاية العامة.

## تفرّع الردود

يدعم Telegram ردودًا مترابطة اختيارية عبر الوسوم:

- `[[reply_to_current]]` — الرد على الرسالة المُحفِّزة.
- `[[reply_to:<id>]]` — الرد على معرّف رسالة محدد.

يُتحكَّم بها عبر `channels.telegram.replyToMode`:

- `first` (الافتراضي)، `all`، `off`.

## الرسائل الصوتية (ملاحظة صوتية مقابل ملف)

يميّز Telegram بين **الملاحظات الصوتية** (فقاعة دائرية) و**ملفات الصوت** (بطاقة بيانات).
يعتمد OpenClaw افتراضيًا ملفات الصوت للتوافق الخلفي.

لفرض فقاعة ملاحظة صوتية في ردود الوكيل، أدرج هذا الوسم في أي مكان في الرد:

- `[[audio_as_voice]]` — إرسال الصوت كملاحظة صوتية بدل ملف.

يُزال الوسم من النص المُسلَّم. تتجاهل القنوات الأخرى هذا الوسم.

لإرسال أداة الرسائل، عيّن `asVoice: true` مع عنوان URL متوافق مع الصوت `media`
(`message` اختياري عند وجود وسائط):

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

## الملصقات

يدعم OpenClaw استقبال وإرسال ملصقات Telegram مع تخزين مؤقّت ذكي.

### استقبال الملصقات

عند إرسال المستخدم ملصقًا، يتعامل OpenClaw معه حسب نوع الملصق:

- **ملصقات ثابتة (WEBP):** تُنزَّل وتُعالَج عبر الرؤية. يظهر الملصق كموضع `<media:sticker>` في محتوى الرسالة.
- **ملصقات متحركة (TGS):** يتم تجاوزها (تنسيق Lottie غير مدعوم للمعالجة).
- **ملصقات فيديو (WEBM):** يتم تجاوزها (تنسيق الفيديو غير مدعوم للمعالجة).

حقل سياق القالب المتاح عند استقبال الملصقات:

- `Sticker` — كائن يحتوي على:
  - `emoji` — الرمز التعبيري المرتبط بالملصق
  - `setName` — اسم مجموعة الملصقات
  - `fileId` — معرّف ملف Telegram (لإرسال الملصق نفسه مرة أخرى)
  - `fileUniqueId` — معرّف ثابت للبحث في التخزين المؤقّت
  - `cachedDescription` — وصف الرؤية المخزّن مؤقتًا عند توفره

### التخزين المؤقّت للملصقات

تُعالَج الملصقات عبر قدرات الرؤية للذكاء الاصطناعي لتوليد أوصاف. وبما أن الملصقات نفسها تُرسل كثيرًا، يخزّن OpenClaw هذه الأوصاف لتجنّب استدعاءات API المتكررة.

**كيف يعمل:**

1. **المرة الأولى:** تُرسل صورة الملصق إلى الذكاء الاصطناعي لتحليل الرؤية. يُنشئ الذكاء الاصطناعي وصفًا (مثل: «قطة كرتونية تلوّح بحماس»).
2. **التخزين:** يُحفظ الوصف مع معرّف الملف، والرمز التعبيري، واسم المجموعة.
3. **المرات اللاحقة:** عند رؤية الملصق نفسه مرة أخرى، يُستخدم الوصف المخزّن مباشرةً دون إرسال الصورة للذكاء الاصطناعي.

**موقع التخزين المؤقّت:** `~/.openclaw/telegram/sticker-cache.json`

**تنسيق إدخال التخزين المؤقّت:**

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

**الفوائد:**

- تقليل تكاليف API بتجنّب مكالمات الرؤية المتكررة للملصق نفسه
- زمن استجابة أسرع للملصقات المخزّنة (لا تأخير لمعالجة الرؤية)
- تمكين البحث عن الملصقات بناءً على الأوصاف المخزّنة

يتم ملء التخزين المؤقّت تلقائيًا عند استقبال الملصقات. لا يلزم أي إدارة يدوية.

### إرسال الملصقات

يمكن للوكيل إرسال والبحث عن الملصقات باستخدام الإجراءين `sticker` و `sticker-search`. تكون معطّلة افتراضيًا ويجب تمكينها في التهيئة:

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

**إرسال ملصق:**

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

المعاملات:

- `fileId` (مطلوب) — معرّف ملف Telegram للملصق. احصل عليه من `Sticker.fileId` عند استقبال ملصق، أو من نتيجة `sticker-search`.
- `replyTo` (اختياري) — معرّف الرسالة للرد.
- `threadId` (اختياري) — معرّف خيط الرسالة لمواضيع المنتدى.

**البحث عن ملصقات:**

يمكن للوكيل البحث في الملصقات المخزّنة حسب الوصف أو الرمز التعبيري أو اسم المجموعة:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

يعيد ملصقات مطابقة من التخزين المؤقّت:

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

يستخدم البحث مطابقة تقريبية عبر نص الوصف، وأحرف الرموز التعبيرية، وأسماء المجموعات.

**مثال مع التفرّع:**

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

## البثّ (المسودات)

يمكن لـ Telegram بث **فقاعات مسودات** أثناء توليد الوكيل للرد.
يستخدم OpenClaw Bot API `sendMessageDraft` (ليست رسائل حقيقية) ثم يرسل
الرد النهائي كرسالة عادية.

المتطلبات (Telegram Bot API 9.3+):

- **دردشات خاصة مع تمكين المواضيع** (وضع موضوع المنتدى للبوت).
- يجب أن تتضمن الرسائل الواردة `message_thread_id` (خيط موضوع خاص).
- يتم تجاهل البث للمجموعات/supergroups/القنوات.

التهيئة:

- `channels.telegram.streamMode: "off" | "partial" | "block"` (الافتراضي: `partial`)
  - `partial`: تحديث فقاعة المسودة بأحدث نص متدفّق.
  - `block`: تحديث فقاعة المسودة في كتل أكبر (مجزّأة).
  - `off`: تعطيل بثّ المسودات.
- اختياري (فقط لـ `streamMode: "block"`):
  - `channels.telegram.draftChunk: { minChars?, maxChars?, breakPreference? }`
    - الافتراضيات: `minChars: 200`، `maxChars: 800`، `breakPreference: "paragraph"` (مقيّدة بـ `channels.telegram.textChunkLimit`).

ملاحظة: بثّ المسودات منفصل عن **بثّ الكتل** (رسائل القناة).
بثّ الكتل معطّل افتراضيًا ويتطلب `channels.telegram.blockStreaming: true`
إذا أردت رسائل Telegram مبكرة بدل تحديثات المسودة.

بثّ الاستدلال (Telegram فقط):

- `/reasoning stream` يبث الاستدلال داخل فقاعة المسودة أثناء توليد الرد،
  ثم يرسل الإجابة النهائية دون الاستدلال.
- إذا كان `channels.telegram.streamMode` هو `off`، يتم تعطيل بثّ الاستدلال.
  مزيد من السياق: [البث + التجزئة](/concepts/streaming).

## سياسة إعادة المحاولة

تعيد استدعاءات Telegram API الصادرة المحاولة عند أخطاء الشبكة العابرة/429 مع تراجع أُسّي وإزاحة عشوائية. يمكن التهيئة عبر `channels.telegram.retry`. انظر [سياسة إعادة المحاولة](/concepts/retry).

## أداة الوكيل (الرسائل + التفاعلات)

- الأداة: `telegram` مع إجراء `sendMessage` (`to`، `content`، `mediaUrl` اختياري، `replyToMessageId`، `messageThreadId`).
- الأداة: `telegram` مع إجراء `react` (`chatId`، `messageId`، `emoji`).
- الأداة: `telegram` مع إجراء `deleteMessage` (`chatId`، `messageId`).
- دلالات إزالة التفاعل: انظر [/tools/reactions](/tools/reactions).
- بوابات الأدوات: `channels.telegram.actions.reactions`، `channels.telegram.actions.sendMessage`، `channels.telegram.actions.deleteMessage` (الافتراضي: مُمكّن)، و `channels.telegram.actions.sticker` (الافتراضي: معطّل).

## إشعارات التفاعلات

**كيف تعمل التفاعلات:**
تصل تفاعلات Telegram كـ **أحداث `message_reaction` منفصلة**، وليست خصائص ضمن حمولة الرسالة. عندما يضيف المستخدم تفاعلًا، يقوم OpenClaw بما يلي:

1. يتلقى تحديث `message_reaction` من Telegram API
2. يحوّله إلى **حدث نظام** بالتنسيق: `"Telegram reaction added: {emoji} by {user} on msg {id}"`
3. يضع حدث النظام في الطابور باستخدام **مفتاح الجلسة نفسه** للرسائل العادية
4. عند وصول الرسالة التالية في تلك المحادثة، تُفرَّغ أحداث النظام وتُسبق إلى سياق الوكيل

يرى الوكيل التفاعلات كـ **إشعارات نظام** في سجل المحادثة، وليس كبيانات وصفية للرسائل.

**التهيئة:**

- `channels.telegram.reactionNotifications`: يتحكّم في أي التفاعلات تُطلق إشعارات
  - `"off"` — تجاهل جميع التفاعلات
  - `"own"` — الإخطار عند تفاعل المستخدمين مع رسائل البوت (أفضل جهد؛ في الذاكرة) (الافتراضي)
  - `"all"` — الإخطار لجميع التفاعلات

- `channels.telegram.reactionLevel`: يتحكّم في قدرة الوكيل على التفاعل
  - `"off"` — لا يمكن للوكيل التفاعل مع الرسائل
  - `"ack"` — يرسل البوت تفاعلات إقرار (👀 أثناء المعالجة) (الافتراضي)
  - `"minimal"` — يمكن للوكيل التفاعل باعتدال (إرشاد: 1 لكل 5–10 تبادلات)
  - `"extensive"` — يمكن للوكيل التفاعل بسخاء عند الملاءمة

**مجموعات المنتدى:** تتضمن التفاعلات في مجموعات المنتدى `message_thread_id` وتستخدم مفاتيح جلسة مثل `agent:main:telegram:group:{chatId}:topic:{threadId}`. يضمن ذلك بقاء التفاعلات والرسائل في الموضوع نفسه معًا.

**مثال تهيئة:**

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

**المتطلبات:**

- يجب على بوتات Telegram طلب `message_reaction` صراحةً في `allowed_updates` (يُهيَّأ تلقائيًا بواسطة OpenClaw)
- في وضع webhook، تُضمَّن التفاعلات في webhook `allowed_updates`
- في وضع الاستطلاع، تُضمَّن التفاعلات في `getUpdates` `allowed_updates`

## أهداف التسليم (CLI/cron)

- استخدم معرّف دردشة (`123456789`) أو اسم مستخدم (`@name`) كهدف.
- مثال: `openclaw message send --channel telegram --target 123456789 --message "hi"`.

## استكشاف الأخطاء وإصلاحها

**البوت لا يرد على رسائل غير مذكورة في مجموعة:**

- إذا عيّنت `channels.telegram.groups.*.requireMention=false`، فيجب تعطيل **وضع الخصوصية** في Bot API الخاص بـ Telegram.
  - BotFather: `/setprivacy` → **تعطيل** (ثم إزالة البوت وإعادة إضافته إلى المجموعة)
- يعرض `openclaw channels status` تحذيرًا عندما تتوقع التهيئة رسائل مجموعات دون ذكر.
- يمكن لـ `openclaw channels status --probe` أيضًا التحقق من العضوية لمعرّفات المجموعات الرقمية الصريحة (لا يمكنه تدقيق قواعد البدل `"*"`).
- اختبار سريع: `/activation always` (على مستوى الجلسة؛ استخدم التهيئة للاستمرارية)

**البوت لا يرى رسائل المجموعات إطلاقًا:**

- إذا كان `channels.telegram.groups` معيّنًا، فيجب إدراج المجموعة أو استخدام `"*"`
- تحقّق من إعدادات الخصوصية في @BotFather → «خصوصية المجموعات» يجب أن تكون **OFF**
- تأكّد من أن البوت عضو فعليًا (وليس مجرد مشرف دون وصول قراءة)
- تحقّق من سجلات Gateway: `openclaw logs --follow` (ابحث عن «skipping group message»)

**البوت يرد على الذِكر ولكن ليس على `/activation always`:**

- يُحدّث أمر `/activation` حالة الجلسة لكنه لا يستمر في التهيئة
- للسلوك الدائم، أضف المجموعة إلى `channels.telegram.groups` مع `requireMention: false`

**أوامر مثل `/status` لا تعمل:**

- تأكّد من أن معرّف مستخدم Telegram الخاص بك مخوّل (عبر الإقران أو `channels.telegram.allowFrom`)
- تتطلب الأوامر التفويض حتى في المجموعات مع `groupPolicy: "open"`

**الاستطلاع الطويل يُجهَض فورًا على Node 22+ (غالبًا مع الوكلاء/جلب مخصّص):**

- أصبح Node 22+ أكثر صرامة بخصوص مثيلات `AbortSignal`؛ يمكن لإشارات خارجية إجهاض استدعاءات `fetch` فورًا.
- حدّث إلى إصدار OpenClaw يطبّع إشارات الإجهاض، أو شغّل Gateway على Node 20 حتى تتمكّن من التحديث.

**يبدأ البوت ثم يتوقف عن الاستجابة بصمت (أو يسجّل `HttpError: Network request ... failed`):**

- تقوم بعض المضيفات بحلّ `api.telegram.org` إلى IPv6 أولًا. إذا لم يكن لخادمك خروج IPv6 عامل، قد يعلق grammY على طلبات IPv6 فقط.
- أصلح ذلك بتمكين خروج IPv6 **أو** فرض حل IPv4 لـ `api.telegram.org` (مثلًا، أضف إدخال `/etc/hosts` باستخدام سجل A لـ IPv4، أو فضّل IPv4 في مكدس DNS لنظام التشغيل)، ثم أعد تشغيل Gateway.
- فحص سريع: `dig +short api.telegram.org A` و `dig +short api.telegram.org AAAA` لتأكيد ما يعيده DNS.

## مرجع التهيئة (Telegram)

التهيئة الكاملة: [التهيئة](/gateway/configuration)

خيارات الموفّر:

- `channels.telegram.enabled`: تمكين/تعطيل بدء القناة.
- `channels.telegram.botToken`: رمز البوت (BotFather).
- `channels.telegram.tokenFile`: قراءة الرمز من مسار ملف.
- `channels.telegram.dmPolicy`: `pairing | allowlist | open | disabled` (الافتراضي: الإقران).
- `channels.telegram.allowFrom`: قائمة سماح الرسائل الخاصة (معرّفات/أسماء مستخدمين). يتطلب `open` وجود `"*"`.
- `channels.telegram.groupPolicy`: `open | allowlist | disabled` (الافتراضي: قائمة سماح).
- `channels.telegram.groupAllowFrom`: قائمة سماح مرسلي المجموعات (معرّفات/أسماء مستخدمين).
- `channels.telegram.groups`: افتراضيات لكل مجموعة + قائمة سماح (استخدم `"*"` للافتراضيات العامة).
  - `channels.telegram.groups.<id>.groupPolicy`: تجاوز لكل مجموعة لـ groupPolicy (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.requireMention`: افتراضي بوابة الذِكر.
  - `channels.telegram.groups.<id>.skills`: مُرشِّح skills (الحذف = كل skills، الفارغ = لا شيء).
  - `channels.telegram.groups.<id>.allowFrom`: تجاوز قائمة سماح المرسلين لكل مجموعة.
  - `channels.telegram.groups.<id>.systemPrompt`: مطالبة نظام إضافية للمجموعة.
  - `channels.telegram.groups.<id>.enabled`: تعطيل المجموعة عندما `false`.
  - `channels.telegram.groups.<id>.topics.<threadId>.*`: تجاوزات لكل موضوع (نفس حقول المجموعة).
  - `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`: تجاوز لكل موضوع لـ groupPolicy (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.topics.<threadId>.requireMention`: تجاوز بوابة الذِكر لكل موضوع.
- `channels.telegram.capabilities.inlineButtons`: `off | dm | group | all | allowlist` (الافتراضي: قائمة سماح).
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`: تجاوز لكل حساب.
- `channels.telegram.replyToMode`: `off | first | all` (الافتراضي: `first`).
- `channels.telegram.textChunkLimit`: حجم تجزئة الصادر (أحرف).
- `channels.telegram.chunkMode`: `length` (افتراضي) أو `newline` للتقسيم على الأسطر الفارغة (حدود الفقرات) قبل تجزئة الطول.
- `channels.telegram.linkPreview`: تبديل معاينات الروابط للرسائل الصادرة (الافتراضي: true).
- `channels.telegram.streamMode`: `off | partial | block` (بثّ المسودات).
- `channels.telegram.mediaMaxMb`: حد الوسائط الواردة/الصادرة (MB).
- `channels.telegram.retry`: سياسة إعادة المحاولة لاستدعاءات Telegram API الصادرة (عدد المحاولات، minDelayMs، maxDelayMs، jitter).
- `channels.telegram.network.autoSelectFamily`: تجاوز Node autoSelectFamily (true=تمكين، false=تعطيل). الافتراضي معطّل على Node 22 لتجنّب مهلات Happy Eyeballs.
- `channels.telegram.proxy`: عنوان URL للوكيل لاستدعاءات Bot API (SOCKS/HTTP).
- `channels.telegram.webhookUrl`: تمكين وضع webhook (يتطلب `channels.telegram.webhookSecret`).
- `channels.telegram.webhookSecret`: سرّ webhook (مطلوب عند تعيين webhookUrl).
- `channels.telegram.webhookPath`: مسار webhook المحلي (الافتراضي `/telegram-webhook`).
- `channels.telegram.actions.reactions`: بوابة تفاعلات أداة Telegram.
- `channels.telegram.actions.sendMessage`: بوابة إرسال رسائل أداة Telegram.
- `channels.telegram.actions.deleteMessage`: بوابة حذف رسائل أداة Telegram.
- `channels.telegram.actions.sticker`: بوابة إجراءات ملصقات Telegram — الإرسال والبحث (الافتراضي: false).
- `channels.telegram.reactionNotifications`: `off | own | all` — التحكّم في أي التفاعلات تُطلق أحداث النظام (الافتراضي: `own` عند عدم التعيين).
- `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` — التحكّم في قدرة الوكيل على التفاعل (الافتراضي: `minimal` عند عدم التعيين).

خيارات عامة ذات صلة:

- `agents.list[].groupChat.mentionPatterns` (أنماط بوابة الذِكر).
- `messages.groupChat.mentionPatterns` (الرجوع العام).
- `commands.native` (الافتراضي `"auto"` → مفعّل لـ Telegram/Discord، معطّل لـ Slack)، `commands.text`، `commands.useAccessGroups` (سلوك الأوامر). تجاوز باستخدام `channels.telegram.commands.native`.
- `messages.responsePrefix`، `messages.ackReaction`، `messages.ackReactionScope`، `messages.removeAckAfterReply`.
