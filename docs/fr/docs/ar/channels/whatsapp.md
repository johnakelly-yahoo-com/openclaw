---
summary: "تكامل WhatsApp (قناة الويب): تسجيل الدخول، صندوق الوارد، الردود، الوسائط، والعمليات"
read_when:
  - عند العمل على سلوك قناة WhatsApp/الويب أو توجيه صندوق الوارد
title: "WhatsApp"
x-i18n:
  source_path: channels/whatsapp.md
  source_hash: 9f7acdf2c71819ae
  provider: openai
  model: gpt-5.2-chat-latest
  workflow: v1
  generated_at: 2026-02-08T10:48:42Z
---

# WhatsApp (قناة الويب)

الحالة: WhatsApp Web عبر Baileys فقط. يمتلك Gateway الجلسة (الجلسات).

## البدء السريع (للمبتدئين)

1. استخدم **رقم هاتف منفصل** إن أمكن (موصى به).
2. هيّئ WhatsApp في `~/.openclaw/openclaw.json`.
3. شغّل `openclaw channels login` لمسح رمز QR (الأجهزة المرتبطة).
4. ابدأ تشغيل الـ Gateway.

التهيئة الدنيا:

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

## الأهداف

- عدة حسابات WhatsApp (متعدد الحسابات) ضمن عملية Gateway واحدة.
- توجيه حتمي: تعود الردود إلى WhatsApp، بلا توجيه عبر النموذج.
- يرى النموذج سياقًا كافيًا لفهم الردود المقتبسة.

## عمليات كتابة التهيئة

افتراضيًا، يُسمح لـ WhatsApp بكتابة تحديثات التهيئة الناتجة عن `/config set|unset` (يتطلب `commands.config: true`).

للتعطيل:

```json5
{
  channels: { whatsapp: { configWrites: false } },
}
```

## البنية (من يمتلك ماذا)

- **Gateway** يمتلك مقبس Baileys وحلقة صندوق الوارد.
- **CLI / تطبيق macOS** يتواصلان مع الـ Gateway؛ بلا استخدام مباشر لـ Baileys.
- **مستمع نشط** مطلوب للإرسال الصادر؛ وإلا يفشل الإرسال فورًا.

## الحصول على رقم هاتف (وضعان)

يتطلب WhatsApp رقم هاتف محمول حقيقي للتحقق. عادةً ما يتم حظر أرقام VoIP والأرقام الافتراضية. هناك طريقتان مدعومتان لتشغيل OpenClaw على WhatsApp:

### رقم مخصص (موصى به)

استخدم **رقم هاتف منفصل** لـ OpenClaw. أفضل تجربة مستخدم، توجيه نظيف، بلا مشكلات الدردشة مع الذات. الإعداد المثالي: **هاتف Android احتياطي/قديم + eSIM**. اتركه على Wi‑Fi ومزوّدًا بالطاقة، واربطه عبر QR.

**WhatsApp Business:** يمكنك استخدام WhatsApp Business على الجهاز نفسه برقم مختلف. ممتاز للإبقاء على WhatsApp الشخصي منفصلًا — ثبّت WhatsApp Business وسجّل رقم OpenClaw هناك.

**مثال تهيئة (رقم مخصص، قائمة سماح لمستخدم واحد):**

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

**وضع الإقران (اختياري):**
إذا رغبت في الإقران بدل قائمة السماح، اضبط `channels.whatsapp.dmPolicy` على `pairing`. يحصل المرسلون غير المعروفين على رمز إقران؛ وافق عبر:
`openclaw pairing approve whatsapp <code>`

### الرقم الشخصي (حل احتياطي)

حل سريع: شغّل OpenClaw على **رقمك الشخصي**. راسل نفسك («Message yourself» في WhatsApp) للاختبار كي لا تزعج جهات الاتصال. توقّع قراءة رموز التحقق على هاتفك الرئيسي أثناء الإعداد والتجارب. **يجب تفعيل وضع الدردشة مع الذات.**
عندما يطلب المعالج رقم WhatsApp الشخصي، أدخل الهاتف الذي سترسل منه (المالك/المرسل)، وليس رقم المساعد.

**مثال تهيئة (رقم شخصي، دردشة مع الذات):**

```json
{
  "whatsapp": {
    "selfChatMode": true,
    "dmPolicy": "allowlist",
    "allowFrom": ["+15551234567"]
  }
}
```

تكون بادئة ردود الدردشة مع الذات افتراضيًا `[{identity.name}]` عند الضبط (وإلا `[openclaw]`)
إذا لم يكن `messages.responsePrefix` مضبوطًا. اضبطه صراحةً للتخصيص أو التعطيل
(استخدم `""` لإزالته).

### نصائح لتأمين الأرقام

- **eSIM محلي** من شركة الاتصالات في بلدك (الأكثر موثوقية)
  - النمسا: [hot.at](https://www.hot.at)
  - المملكة المتحدة: [giffgaff](https://www.giffgaff.com) — شريحة مجانية بلا عقد
- **شريحة مسبقة الدفع** — رخيصة، وتحتاج فقط لاستقبال رسالة SMS واحدة للتحقق

**تجنّب:** TextNow، Google Voice، ومعظم خدمات «SMS مجانية» — يحظرها WhatsApp بقوة.

**نصيحة:** يحتاج الرقم فقط لاستقبال رسالة تحقق واحدة. بعد ذلك تستمر جلسات WhatsApp Web عبر `creds.json`.

## لماذا ليس Twilio؟

- دعمت إصدارات OpenClaw المبكرة تكامل WhatsApp Business من Twilio.
- أرقام WhatsApp Business غير مناسبة لمساعد شخصي.
- تفرض Meta نافذة رد مدتها 24 ساعة؛ إذا لم ترد خلال آخر 24 ساعة، لا يمكن لرقم الأعمال بدء رسائل جديدة.
- الاستخدام عالي الحجم أو «الدردشة الكثيفة» يفعّل حظرًا صارمًا، لأن حسابات الأعمال ليست مخصّصة لإرسال عشرات رسائل المساعد الشخصي.
- النتيجة: تسليم غير موثوق وحظر متكرر، لذا أُزيل الدعم.

## تسجيل الدخول + بيانات الاعتماد

- أمر تسجيل الدخول: `openclaw channels login` (QR عبر الأجهزة المرتبطة).
- تسجيل دخول متعدد الحسابات: `openclaw channels login --account <id>` (`<id>` = `accountId`).
- الحساب الافتراضي (عند إغفال `--account`): `default` إن وُجد، وإلا أول معرّف حساب مُهيّأ (مرتّب).
- تُخزَّن بيانات الاعتماد في `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`.
- نسخة احتياطية في `creds.json.bak` (تُستعاد عند التلف).
- التوافق القديم: خزّنت التثبيتات الأقدم ملفات Baileys مباشرةً في `~/.openclaw/credentials/`.
- تسجيل الخروج: `openclaw channels logout` (أو `--account <id>`) يحذف حالة مصادقة WhatsApp (مع الاحتفاظ بالمشترك `oauth.json`).
- مقبس مسجّل الخروج ⇒ خطأ يوجّه لإعادة الربط.

## التدفق الوارد (الرسائل الخاصة + المجموعات)

- تأتي أحداث WhatsApp من `messages.upsert` (Baileys).
- تُفصل مستمعات صندوق الوارد عند الإيقاف لتجنب تراكم معالجات الأحداث في الاختبارات/إعادات التشغيل.
- تُتجاهل محادثات الحالة/البث.
- تستخدم الدردشات المباشرة E.164؛ وتستخدم المجموعات معرّف JID للمجموعة.
- **سياسة الرسائل الخاصة**: يتحكم `channels.whatsapp.dmPolicy` في الوصول إلى الدردشة المباشرة (الافتراضي: `pairing`).
  - الإقران: يحصل المرسلون غير المعروفين على رمز إقران (الموافقة عبر `openclaw pairing approve whatsapp <code>`؛ تنتهي الرموز بعد ساعة).
  - الفتح: يتطلب أن يتضمن `channels.whatsapp.allowFrom` القيمة `"*"`.
  - رقم WhatsApp المرتبط بك موثوق ضمنيًا، لذا تتجاوز رسائل الذات فحوصات `channels.whatsapp.dmPolicy` و `channels.whatsapp.allowFrom`.

### وضع الرقم الشخصي (حل احتياطي)

إذا شغّلت OpenClaw على **رقم WhatsApp الشخصي**، فعّل `channels.whatsapp.selfChatMode` (انظر المثال أعلاه).

السلوك:

- لا تُطلق الرسائل الصادرة الخاصة ردود الإقران مطلقًا (لمنع إزعاج جهات الاتصال).
- ما زال المرسلون غير المعروفين الواردون يتبعون `channels.whatsapp.dmPolicy`.
- وضع الدردشة مع الذات (allowFrom يتضمن رقمك) يتجنب إيصالات القراءة التلقائية ويتجاهل معرّفات الإشارة.
- تُرسل إيصالات القراءة لرسائل DM غير الدردشة مع الذات.

## إيصالات القراءة

افتراضيًا، يعلّم الـ Gateway رسائل WhatsApp الواردة كمقروءة (علامتا الصح الزرقاء) بمجرد قبولها.

تعطيل عام:

```json5
{
  channels: { whatsapp: { sendReadReceipts: false } },
}
```

تعطيل لكل حساب:

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        personal: { sendReadReceipts: false },
      },
    },
  },
}
```

ملاحظات:

- يتجاوز وضع الدردشة مع الذات إيصالات القراءة دائمًا.

## أسئلة WhatsApp الشائعة: الإرسال + الإقران

**هل سيرسل OpenClaw رسائل إلى جهات اتصال عشوائية عند ربط WhatsApp؟**  
لا. سياسة الرسائل الخاصة الافتراضية هي **الإقران**، لذا يحصل المرسلون غير المعروفين فقط على رمز إقران ولا تُعالَج رسائلهم. يرد OpenClaw فقط على الدردشات التي يستقبلها، أو على الإرسالات التي تُطلقها صراحةً (وكيل/CLI).

**كيف يعمل الإقران على WhatsApp؟**  
الإقران بوابة DM للمرسلين غير المعروفين:

- أول رسالة DM من مرسل جديد تعيد رمزًا قصيرًا (ولا تُعالَج الرسالة).
- وافق عبر: `openclaw pairing approve whatsapp <code>` (اعرض القائمة بـ `openclaw pairing list whatsapp`).
- تنتهي الرموز بعد ساعة؛ وتُحدّد الطلبات المعلّقة بثلاثة لكل قناة.

**هل يمكن لعدة أشخاص استخدام مثيلات OpenClaw مختلفة على رقم WhatsApp واحد؟**  
نعم، عبر توجيه كل مرسل إلى وكيل مختلف باستخدام `bindings` (نظير `kind: "dm"`، مرسل E.164 مثل `+15551234567`). تظل الردود صادرة من **حساب WhatsApp نفسه**، وتنهار الدردشات المباشرة إلى الجلسة الرئيسية لكل وكيل، لذا استخدم **وكيلًا واحدًا لكل شخص**. التحكم في وصول DM (`dmPolicy`/`allowFrom`) عالمي لكل حساب WhatsApp. راجع [توجيه متعدد الوكلاء](/concepts/multi-agent).

**لماذا تطلبون رقم هاتفي في المعالج؟**  
يستخدمه المعالج لضبط **قائمة السماح/المالك** بحيث تُسمح رسائل DM الخاصة بك. لا يُستخدم للإرسال التلقائي. إذا شغّلت على رقم WhatsApp الشخصي، استخدم الرقم نفسه وفعّل `channels.whatsapp.selfChatMode`.

## توحيد الرسائل (ما يراه النموذج)

- `Body` هو نص الرسالة الحالية مع الغلاف.

- يُلحَق سياق الرد المقتبس **دائمًا**:

  ```
  [Replying to +1555 id:ABC123]
  <quoted text or <media:...>>
  [/Replying]
  ```

- كما تُضبط بيانات وصفية للرد:
  - `ReplyToId` = stanzaId
  - `ReplyToBody` = نص مقتبس أو عنصر نائب للوسائط
  - `ReplyToSender` = E.164 عند توفره

- تستخدم الرسائل الواردة التي تحتوي على وسائط فقط عناصر نائبة:
  - `<media:image|video|audio|document|sticker>`

## المجموعات

- تُعيَّن المجموعات إلى جلسات `agent:<agentId>:whatsapp:group:<jid>`.
- سياسة المجموعات: `channels.whatsapp.groupPolicy = open|disabled|allowlist` (الافتراضي `allowlist`).
- أوضاع التفعيل:
  - `mention` (الافتراضي): يتطلب @mention أو تطابق regex.
  - `always`: يفعّل دائمًا.
- `/activation mention|always` خاص بالمالك فقط ويجب إرساله كرسالة مستقلة.
- المالك = `channels.whatsapp.allowFrom` (أو E.164 الذاتي إذا لم يُضبط).
- **حقن السجل** (المعلّق فقط):
  - تُدرج الرسائل الأخيرة _غير المعالجة_ (الافتراضي 50) تحت:
    `[Chat messages since your last reply - for context]` (لا يُعاد حقن الرسائل الموجودة بالفعل في الجلسة)
  - الرسالة الحالية تحت:
    `[Current message - respond to this]`
  - يُلحَق لاحقة المرسل: `[from: Name (+E164)]`
- تُخزَّن بيانات المجموعة مؤقتًا لمدة 5 دقائق (العنوان + المشاركون).

## تسليم الردود (التفريع)

- يرسل WhatsApp Web رسائل قياسية (لا يوجد تفريع ردود مقتبسة في الـ Gateway الحالي).
- تُتجاهل وسوم الرد على هذه القناة.

## تفاعلات الإقرار (تفاعل تلقائي عند الاستلام)

يمكن لـ WhatsApp إرسال تفاعلات إيموجي تلقائيًا إلى الرسائل الواردة فور الاستلام، قبل أن يُنشئ البوت ردًا. يوفّر هذا تغذية راجعة فورية للمستخدمين بأن رسالتهم قد استُلِمت.

**التهيئة:**

```json
{
  "whatsapp": {
    "ackReaction": {
      "emoji": "👀",
      "direct": true,
      "group": "mentions"
    }
  }
}
```

**الخيارات:**

- `emoji` (نص): الإيموجي المستخدم للإقرار (مثل "👀"، "✅"، "📨"). فارغ أو محذوف = تعطيل الميزة.
- `direct` (منطقي، الافتراضي: `true`): إرسال التفاعلات في الدردشات المباشرة/DM.
- `group` (نص، الافتراضي: `"mentions"`): سلوك دردشات المجموعات:
  - `"always"`: التفاعل مع كل رسائل المجموعة (حتى دون @mention)
  - `"mentions"`: التفاعل فقط عند @mention للبوت
  - `"never"`: عدم التفاعل مطلقًا في المجموعات

**تجاوز لكل حساب:**

```json
{
  "whatsapp": {
    "accounts": {
      "work": {
        "ackReaction": {
          "emoji": "✅",
          "direct": false,
          "group": "always"
        }
      }
    }
  }
}
```

**ملاحظات السلوك:**

- تُرسل التفاعلات **فورًا** عند استلام الرسالة، قبل مؤشرات الكتابة أو ردود البوت.
- في المجموعات مع `requireMention: false` (التفعيل: دائمًا)، سيتفاعل `group: "mentions"` مع جميع الرسائل (ليس فقط @mentions).
- إرسال بلا انتظار: تُسجَّل إخفاقات التفاعل لكن لا تمنع رد البوت.
- يُضمَّن معرّف المشارك JID تلقائيًا لتفاعلات المجموعات.
- يتجاهل WhatsApp `messages.ackReaction`؛ استخدم `channels.whatsapp.ackReaction` بدلًا منه.

## أداة الوكيل (التفاعلات)

- الأداة: `whatsapp` مع إجراء `react` (`chatJid`، `messageId`، `emoji`، اختياري `remove`).
- اختياري: `participant` (مرسل المجموعة)، `fromMe` (التفاعل مع رسالتك)، `accountId` (تعدد الحسابات).
- دلالات إزالة التفاعل: انظر [/tools/reactions](/tools/reactions).
- تقييد الأداة: `channels.whatsapp.actions.reactions` (الافتراضي: مفعّل).

## الحدود

- يُجزَّأ النص الصادر إلى `channels.whatsapp.textChunkLimit` (الافتراضي 4000).
- تجزئة أسطر جديدة اختيارية: اضبط `channels.whatsapp.chunkMode="newline"` للتقسيم عند الأسطر الفارغة (حدود الفقرات) قبل تجزئة الطول.
- تُحدَّد عمليات حفظ الوسائط الواردة بواسطة `channels.whatsapp.mediaMaxMb` (الافتراضي 50 ميغابايت).
- تُحدَّد عناصر الوسائط الصادرة بواسطة `agents.defaults.mediaMaxMb` (الافتراضي 5 ميغابايت).

## الإرسال الصادر (نص + وسائط)

- يستخدم مستمع الويب النشط؛ خطأ إذا لم يكن الـ Gateway قيد التشغيل.
- تجزئة النص: حد أقصى 4k لكل رسالة (قابل للتهيئة عبر `channels.whatsapp.textChunkLimit`، اختياري `channels.whatsapp.chunkMode`).
- الوسائط:
  - الصور/الفيديو/الصوت/المستندات مدعومة.
  - يُرسل الصوت كـ PTT؛ `audio/ogg` ⇒ `audio/ogg; codecs=opus`.
  - العنوان (Caption) فقط على عنصر الوسائط الأول.
  - يدعم جلب الوسائط HTTP(S) والمسارات المحلية.
  - صور GIF المتحركة: يتوقع WhatsApp ملف MP4 مع `gifPlayback: true` للتشغيل المتكرر المضمّن.
    - CLI: `openclaw message send --media <mp4> --gif-playback`
    - Gateway: تتضمن المعاملات `gifPlayback: true` في `send`

## الملاحظات الصوتية (صوت PTT)

يرسل WhatsApp الصوت كملاحظات صوتية **(فقاعة PTT)**.

- أفضل النتائج: OGG/Opus. يعيد OpenClaw كتابة `audio/ogg` إلى `audio/ogg; codecs=opus`.
- يتم تجاهل `[[audio_as_voice]]` في WhatsApp (الصوت يُرسل بالفعل كملاحظة صوتية).

## حدود الوسائط + التحسين

- الحد الافتراضي الصادر: 5 ميغابايت (لكل عنصر وسائط).
- التجاوز: `agents.defaults.mediaMaxMb`.
- تُحسَّن الصور تلقائيًا إلى JPEG ضمن الحد (تغيير الحجم + مسح الجودة).
- وسائط أكبر من الحد ⇒ خطأ؛ ويعود رد الوسائط إلى تحذير نصي.

## نبضات القلب

- **نبضة Gateway** تسجّل صحة الاتصال (`web.heartbeatSeconds`، الافتراضي 60 ثانية).
- **نبضة الوكيل** يمكن تهيئتها لكل وكيل (`agents.list[].heartbeat`) أو عالميًا
  عبر `agents.defaults.heartbeat` (احتياطي عند عدم وجود إعدادات لكل وكيل).
  - تستخدم مطالبة النبضة المهيّأة (الافتراضي: `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`) + سلوك التخطي `HEARTBEAT_OK`.
  - التسليم افتراضيًا إلى آخر قناة مستخدمة (أو الهدف المهيّأ).

## سلوك إعادة الاتصال

- سياسة التراجع: `web.reconnect`:
  - `initialMs`، `maxMs`، `factor`، `jitter`، `maxAttempts`.
- عند بلوغ maxAttempts، تتوقف مراقبة الويب (وضع متدهور).
- تسجيل الخروج ⇒ التوقف ويتطلب إعادة الربط.

## خريطة سريعة للتهيئة

- `channels.whatsapp.dmPolicy` (سياسة DM: إقران/قائمة سماح/فتح/معطّل).
- `channels.whatsapp.selfChatMode` (إعداد الهاتف نفسه؛ يستخدم البوت رقم WhatsApp الشخصي).
- `channels.whatsapp.allowFrom` (قائمة سماح DM). يستخدم WhatsApp أرقام E.164 (لا أسماء مستخدمين).
- `channels.whatsapp.mediaMaxMb` (حد حفظ الوسائط الواردة).
- `channels.whatsapp.ackReaction` (تفاعل تلقائي عند استلام الرسالة: `{emoji, direct, group}`).
- `channels.whatsapp.accounts.<accountId>.*` (إعدادات لكل حساب + اختياري `authDir`).
- `channels.whatsapp.accounts.<accountId>.mediaMaxMb` (حد وسائط واردة لكل حساب).
- `channels.whatsapp.accounts.<accountId>.ackReaction` (تجاوز تفاعل الإقرار لكل حساب).
- `channels.whatsapp.groupAllowFrom` (قائمة سماح مرسلي المجموعات).
- `channels.whatsapp.groupPolicy` (سياسة المجموعات).
- `channels.whatsapp.historyLimit` / `channels.whatsapp.accounts.<accountId>.historyLimit` (سياق سجل المجموعات؛ يعطّله `0`).
- `channels.whatsapp.dmHistoryLimit` (حد سجل DM بعدد أدوار المستخدم). تجاوزات لكل مستخدم: `channels.whatsapp.dms["<phone>"].historyLimit`.
- `channels.whatsapp.groups` (قائمة سماح المجموعات + افتراضات تقييد الإشارة؛ استخدم `"*"` للسماح للجميع)
- `channels.whatsapp.actions.reactions` (تقييد تفاعلات أداة WhatsApp).
- `agents.list[].groupChat.mentionPatterns` (أو `messages.groupChat.mentionPatterns`)
- `messages.groupChat.historyLimit`
- `channels.whatsapp.messagePrefix` (بادئة واردة؛ لكل حساب: `channels.whatsapp.accounts.<accountId>.messagePrefix`؛ مُهمل: `messages.messagePrefix`)
- `messages.responsePrefix` (بادئة صادرة)
- `agents.defaults.mediaMaxMb`
- `agents.defaults.heartbeat.every`
- `agents.defaults.heartbeat.model` (تجاوز اختياري)
- `agents.defaults.heartbeat.target`
- `agents.defaults.heartbeat.to`
- `agents.defaults.heartbeat.session`
- `agents.list[].heartbeat.*` (تجاوزات لكل وكيل)
- `session.*` (النطاق، الخمول، التخزين، mainKey)
- `web.enabled` (تعطيل بدء القناة عندما تكون false)
- `web.heartbeatSeconds`
- `web.reconnect.*`

## السجلات + استكشاف الأخطاء وإصلاحها

- الأنظمة الفرعية: `whatsapp/inbound`، `whatsapp/outbound`، `web-heartbeat`، `web-reconnect`.
- ملف السجل: `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (قابل للتهيئة).
- دليل الاستكشاف: [استكشاف أخطاء Gateway وإصلاحها](/gateway/troubleshooting).

## استكشاف الأخطاء وإصلاحها (سريع)

**غير مرتبط / مطلوب تسجيل دخول QR**

- العَرَض: يُظهر `channels status` القيمة `linked: false` أو تحذير «Not linked».
- الحل: شغّل `openclaw channels login` على مضيف Gateway وامسح رمز QR (WhatsApp → الإعدادات → الأجهزة المرتبطة).

**مرتبط لكن غير متصل / حلقة إعادة اتصال**

- العَرَض: يُظهر `channels status` القيمة `running, disconnected` أو تحذير «Linked but disconnected».
- الحل: `openclaw doctor` (أو أعد تشغيل الـ Gateway). إذا استمر، أعد الربط عبر `channels login` وتحقق من `openclaw logs --follow`.

**بيئة تشغيل Bun**

- Bun **غير موصى به**. WhatsApp (Baileys) وTelegram غير موثوقين على Bun.
  شغّل الـ Gateway باستخدام **Node**. (انظر ملاحظة بيئة التشغيل في «بدء الاستخدام».)
