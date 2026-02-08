---
summary: "فهم الوسائط الواردة (صور/صوت/فيديو) بشكل اختياري مع موفّر + بدائل CLI"
read_when:
  - عند تصميم أو إعادة هيكلة فهم الوسائط
  - عند ضبط المعالجة المسبقة للصوت/الفيديو/الصور الواردة
title: "فهم الوسائط"
x-i18n:
  source_path: nodes/media-understanding.md
  source_hash: 4b275b152060eae3
  provider: openai
  model: gpt-5.2-chat-latest
  workflow: v1
  generated_at: 2026-02-08T10:48:45Z
---

# فهم الوسائط (الواردة) — 2026-01-17

يمكن لـ OpenClaw **تلخيص الوسائط الواردة** (صور/صوت/فيديو) قبل تشغيل خطّ الرد. يكتشف تلقائيًا توفّر الأدوات المحلية أو مفاتيح الموفّرين، ويمكن تعطيله أو تخصيصه. إذا كان الفهم مُعطّلًا، فستظل النماذج تتلقى الملفات/الروابط الأصلية كالمعتاد.

## الأهداف

- اختياري: هضمٌ مسبق للوسائط الواردة إلى نص قصير لتسريع التوجيه وتحسين تحليل الأوامر.
- الحفاظ على تسليم الوسائط الأصلية إلى النموذج (دائمًا).
- دعم **واجهات موفّري الخدمات** و**بدائل CLI**.
- السماح بعدة نماذج مع تراجعٍ مُرتّب (خطأ/حجم/مهلة).

## السلوك عالي المستوى

1. جمع المرفقات الواردة (`MediaPaths`، `MediaUrls`، `MediaTypes`).
2. لكل قدرة مُمكّنة (صورة/صوت/فيديو)، اختيار المرفقات وفق السياسة (الافتراضي: **الأول**).
3. اختيار أول إدخال نموذج مؤهّل (الحجم + القدرة + المصادقة).
4. إذا فشل نموذج أو كان الوسيط كبيرًا جدًا، **يتم التراجع إلى الإدخال التالي**.
5. عند النجاح:
   - تصبح `Body` كتلة `[Image]` أو `[Audio]` أو `[Video]`.
   - يضبط الصوت `{{Transcript}}`؛ ويستخدم تحليل الأوامر نص التسمية عند توفره،
     وإلا فيستخدم التفريغ النصي.
   - تُحفَظ التسميات كـ `User text:` داخل الكتلة.

إذا فشل الفهم أو كان مُعطّلًا، **يتابع تدفّق الرد** باستخدام النص الأصلي + المرفقات.

## نظرة عامة على التهيئة

يدعم `tools.media` **نماذج مشتركة** بالإضافة إلى تجاوزات لكل قدرة:

- `tools.media.models`: قائمة نماذج مشتركة (استخدم `capabilities` للتحكم).
- `tools.media.image` / `tools.media.audio` / `tools.media.video`:
  - القيم الافتراضية (`prompt`، `maxChars`، `maxBytes`، `timeoutSeconds`، `language`)
  - تجاوزات الموفّر (`baseUrl`، `headers`، `providerOptions`)
  - خيارات Deepgram للصوت عبر `tools.media.audio.providerOptions.deepgram`
  - **قائمة `models` اختيارية لكل قدرة** (تُفضَّل قبل النماذج المشتركة)
  - سياسة `attachments` (`mode`، `maxAttachments`، `prefer`)
  - `scope` (تحكم اختياري حسب القناة/نوع الدردشة/مفتاح الجلسة)
- `tools.media.concurrency`: الحد الأقصى للتشغيلات المتزامنة للقدرات (الافتراضي **2**).

```json5
{
  tools: {
    media: {
      models: [
        /* shared list */
      ],
      image: {
        /* optional overrides */
      },
      audio: {
        /* optional overrides */
      },
      video: {
        /* optional overrides */
      },
    },
  },
}
```

### إدخالات النماذج

يمكن أن يكون كل إدخال `models[]` **موفّرًا** أو **CLI**:

```json5
{
  type: "provider", // default if omitted
  provider: "openai",
  model: "gpt-5.2",
  prompt: "Describe the image in <= 500 chars.",
  maxChars: 500,
  maxBytes: 10485760,
  timeoutSeconds: 60,
  capabilities: ["image"], // optional, used for multi‑modal entries
  profile: "vision-profile",
  preferredProfile: "vision-fallback",
}
```

```json5
{
  type: "cli",
  command: "gemini",
  args: [
    "-m",
    "gemini-3-flash",
    "--allowed-tools",
    "read_file",
    "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
  ],
  maxChars: 500,
  maxBytes: 52428800,
  timeoutSeconds: 120,
  capabilities: ["video", "image"],
}
```

يمكن لقوالب CLI أيضًا استخدام:

- `{{MediaDir}}` (الدليل الذي يحتوي ملف الوسائط)
- `{{OutputDir}}` (دليل مؤقت يُنشأ لهذا التشغيل)
- `{{OutputBase}}` (مسار قاعدة الملف المؤقت، بدون امتداد)

## الإعدادات الافتراضية والحدود

الإعدادات الافتراضية الموصى بها:

- `maxChars`: **500** للصور/الفيديو (قصير وملائم للأوامر)
- `maxChars`: **غير مُعيّن** للصوت (تفريغ كامل ما لم تُحدّد حدًا)
- `maxBytes`:
  - الصور: **10MB**
  - الصوت: **20MB**
  - الفيديو: **50MB**

القواعد:

- إذا تجاوز الوسيط `maxBytes`، يتم تخطي ذلك النموذج وتجربة **النموذج التالي**.
- إذا أعاد النموذج أكثر من `maxChars`، يتم اقتطاع الإخراج.
- `prompt` افتراضيًا هو «وصف {media}.» إضافةً إلى إرشادات `maxChars` (للصور/الفيديو فقط).
- إذا كان `<capability>.enabled: true` ولكن لم تُهيَّأ نماذج، يحاول OpenClaw استخدام
  **نموذج الرد النشط** عندما يدعم موفّره القدرة.

### الكشف التلقائي عن فهم الوسائط (الافتراضي)

إذا لم يتم ضبط `tools.media.<capability>.enabled` على `false` ولم تُهيّئ
نماذج، يقوم OpenClaw بالكشف التلقائي بالترتيب التالي و**يتوقف عند أول خيار يعمل**:

1. **أدوات CLI المحلية** (للصوت فقط؛ إذا كانت مُثبّتة)
   - `sherpa-onnx-offline` (يتطلب `SHERPA_ONNX_MODEL_DIR` مع المرمِّز/فك الترميز/الدمج/الرموز)
   - `whisper-cli` (`whisper-cpp`؛ يستخدم `WHISPER_CPP_MODEL` أو النموذج الصغير المُضمّن)
   - `whisper` (CLI بلغة Python؛ يُنزّل النماذج تلقائيًا)
2. **Gemini CLI** (`gemini`) باستخدام `read_many_files`
3. **مفاتيح الموفّرين**
   - الصوت: OpenAI → Groq → Deepgram → Google
   - الصور: OpenAI → Anthropic → Google → MiniMax
   - الفيديو: Google

لتعطيل الكشف التلقائي، اضبط:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

ملاحظة: كشف الملفات التنفيذية هو جهدٌ أفضل عبر macOS/Linux/Windows؛ تأكّد من أن CLI موجود على `PATH` (نقوم بتوسيع `~`)، أو اضبط نموذج CLI صريحًا بمسار أمر كامل.

## القدرات (اختياري)

إذا ضبطت `capabilities`، فلن يعمل الإدخال إلا لأنواع الوسائط تلك. بالنسبة للقوائم المشتركة، يمكن لـ OpenClaw استنتاج القيم الافتراضية:

- `openai`، `anthropic`، `minimax`: **صور**
- `google` (واجهة Gemini API): **صور + صوت + فيديو**
- `groq`: **صوت**
- `deepgram`: **صوت**

بالنسبة لإدخالات CLI، **اضبط `capabilities` صراحةً** لتجنّب تطابقات غير متوقعة.
إذا حذفت `capabilities`، يصبح الإدخال مؤهّلًا للقائمة التي يظهر فيها.

## مصفوفة دعم الموفّرين (تكاملات OpenClaw)

| القدرة  | تكامل الموفّر                                  | ملاحظات                                                                         |
| ------- | ---------------------------------------------- | ------------------------------------------------------------------------------- |
| الصور   | OpenAI / Anthropic / Google / أخرى عبر `pi-ai` | أي نموذج يدعم الصور في السجل يعمل.                              |
| الصوت   | OpenAI، Groq، Deepgram، Google                 | تفريغ صوت الموفّر (Whisper/Deepgram/Gemini). |
| الفيديو | Google (Gemini API)         | فهم الفيديو عبر الموفّر.                                        |

## الموفّرون الموصى بهم

**الصور**

- فضّل نموذجك النشط إذا كان يدعم الصور.
- إعدادات افتراضية جيدة: `openai/gpt-5.2`، `anthropic/claude-opus-4-6`، `google/gemini-3-pro-preview`.

**الصوت**

- `openai/gpt-4o-mini-transcribe`، `groq/whisper-large-v3-turbo`، أو `deepgram/nova-3`.
- بديل CLI: `whisper-cli` (whisper-cpp) أو `whisper`.
- إعداد Deepgram: [Deepgram (تفريغ الصوت)](/providers/deepgram).

**الفيديو**

- `google/gemini-3-flash-preview` (سريع)، `google/gemini-3-pro-preview` (أغنى).
- بديل CLI: أداة `gemini` (تدعم `read_file` للفيديو/الصوت).

## سياسة المرفقات

تتحكم سياسة `attachments` لكل قدرة في المرفقات التي تتم معالجتها:

- `mode`: `first` (الافتراضي) أو `all`
- `maxAttachments`: تحديد الحد الأقصى للعدد المعالج (الافتراضي **1**)
- `prefer`: `first`، `last`، `path`، `url`

عند `mode: "all"`، تُوسَم المخرجات بـ `[Image 1/2]`، `[Audio 2/2]`، إلخ.

## أمثلة التهيئة

### 1. قائمة نماذج مشتركة + تجاوزات

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-5.2", capabilities: ["image"] },
        {
          provider: "google",
          model: "gemini-3-flash-preview",
          capabilities: ["image", "audio", "video"],
        },
        {
          type: "cli",
          command: "gemini",
          args: [
            "-m",
            "gemini-3-flash",
            "--allowed-tools",
            "read_file",
            "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
          ],
          capabilities: ["image", "video"],
        },
      ],
      audio: {
        attachments: { mode: "all", maxAttachments: 2 },
      },
      video: {
        maxChars: 500,
      },
    },
  },
}
```

### 2. الصوت + الفيديو فقط (إيقاف الصور)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          {
            type: "cli",
            command: "whisper",
            args: ["--model", "base", "{{MediaPath}}"],
          },
        ],
      },
      video: {
        enabled: true,
        maxChars: 500,
        models: [
          { provider: "google", model: "gemini-3-flash-preview" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
            ],
          },
        ],
      },
    },
  },
}
```

### 3. فهم الصور اختياري

```json5
{
  tools: {
    media: {
      image: {
        enabled: true,
        maxBytes: 10485760,
        maxChars: 500,
        models: [
          { provider: "openai", model: "gpt-5.2" },
          { provider: "anthropic", model: "claude-opus-4-6" },
          {
            type: "cli",
            command: "gemini",
            args: [
              "-m",
              "gemini-3-flash",
              "--allowed-tools",
              "read_file",
              "Read the media at {{MediaPath}} and describe it in <= {{MaxChars}} characters.",
            ],
          },
        ],
      },
    },
  },
}
```

### 4. إدخال واحد متعدد الوسائط (قدرات صريحة)

```json5
{
  tools: {
    media: {
      image: {
        models: [
          {
            provider: "google",
            model: "gemini-3-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      audio: {
        models: [
          {
            provider: "google",
            model: "gemini-3-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
      video: {
        models: [
          {
            provider: "google",
            model: "gemini-3-pro-preview",
            capabilities: ["image", "video", "audio"],
          },
        ],
      },
    },
  },
}
```

## إخراج الحالة

عند تشغيل فهم الوسائط، يتضمن `/status` سطر ملخص قصير:

```
📎 Media: image ok (openai/gpt-5.2) · audio skipped (maxBytes)
```

يوضح ذلك نتائج كل قدرة والموفّر/النموذج المختار عند الاقتضاء.

## ملاحظات

- الفهم **بأفضل جهد**. الأخطاء لا تمنع الردود.
- لا تزال المرفقات تُمرَّر إلى النماذج حتى عند تعطيل الفهم.
- استخدم `scope` لتقييد أماكن تشغيل الفهم (مثل الرسائل الخاصة فقط).

## مستندات ذات صلة

- [التهيئة](/gateway/configuration)
- [دعم الصور والوسائط](/nodes/images)
