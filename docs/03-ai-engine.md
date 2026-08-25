# 03 — محرك القصص والذكاء الاصطناعي

> **تذكير:** النظام لا يولّد صورًا. يولّد **برومبتات**. أمير يولّد الصور خارجيًا ويرفعها.

---

## أ. خط الإنتاج الذكي

```
بيانات الطفل
     ↓
[1] Story Writer ──────→ عنوان + عبرة + صفحات (نص عربي + وصف مشهد)
     ↓
[2] Safety & Age Check → تقرير: تحذيرات، مفردات صعبة، طول الجمل
     ↓  (مراجعة بشرية + اعتماد)
[3] Character Bible ───→ كتلة وصف إنجليزية ثابتة للبطل + الأسلوب + negative
     ↓  (قفل بشري)
[4] Prompt Engineer ───→ برومبت لكل صفحة = [style] + [character] + [scene] + [composition]
     ↓
[5] تصدير حزمة البرومبتات (.md) ──→ أمير يولّد الصور خارجيًا
     ↓
[6] رفع الصور دفعةً واحدة → مطابقة بالاسم → مراجعة → تنضيد
```

كل استدعاء يُسجَّل في `AiRun` مع التكلفة. إذا تجاوز الشهر `AI_MONTHLY_BUDGET_USD`
يتوقف التوليد ويظهر تنبيه في الاستوديو.

---

## ب. القواعد المشتركة لكل استدعاءات AI

1. الإخراج **JSON صارم فقط** — بدون مقدمات، بدون ```` ``` ````.
2. التحقق عبر **Spatie Data** (`StoryDraftData::validateAndCreate()`).
   عند الفشل: إعادة محاولة واحدة مع رسالة تصحيح، ثم `AiValidationException`.
3. لا تُرسل: اسم عائلة الطفل، رقم الهاتف، العنوان، البريد. **الاسم الأول فقط.**
4. صورة الطفل لا تُرسل إلا إذا `$order->use_vision_for_character === true`.
5. `temperature`: 1 للقصة، 0.3 للبرومبتات وفحص السلامة.
6. كل برومبت نظام في `app/Services/Ai/Prompts/*.php` كثابت في كلاس — لا نصوص داخل الدوال.
7. **كل استدعاء داخل Job على طابور `ai`** — لا استدعاء متزامن داخل request.
8. `BudgetGuard::assertWithinBudget()` قبل كل استدعاء، و`AiRun::record()` بعده.

```php
// app/Services/Ai/AnthropicClient.php
final class AnthropicClient
{
    public function structured(
        string $system,
        string $userMessage,
        string $dataClass,          // كلاس Spatie Data للتحقق
        float $temperature = 1.0,
        AiRunKind $kind,
        ?Order $order = null,
    ): Data {
        app(BudgetGuard::class)->assertWithinBudget($kind);

        $response = Http::withHeaders([
                'x-api-key'         => config('services.anthropic.key'),
                'anthropic-version' => '2023-06-01',
            ])
            ->timeout(180)
            ->retry(2, fn (int $attempt) => $attempt * 10_000)
            ->post('https://api.anthropic.com/v1/messages', [
                'model'       => config('services.anthropic.model'),
                'max_tokens'  => 8_000,
                'temperature' => $temperature,
                'system'      => $system,
                'messages'    => [['role' => 'user', 'content' => $userMessage]],
            ])
            ->throw();

        // سجّل التكلفة دائمًا — حتى لو فشل التحقق بعدها
        AiRun::record($kind, $response, $order);

        return $dataClass::validateAndCreate(
            $this->decodeJson($response->json('content.0.text'))
        );
    }
}
```

---

## ج. [1] كاتب القصة — Story Writer

### مدخلات

```php
// app/Data/Ai/StoryInputData.php
class StoryInputData extends Data
{
    public function __construct(
        public string $childFirstName,      // الاسم الأول فقط — لا اسم عائلة
        public AgeGroup $ageGroup,
        public AddressForm $addressForm,
        /** @var list<string> */ public array $traits,
        /** @var list<string> */ public array $interests,
        /** @var list<string> */ public array $favoriteThings,
        public ?string $favoriteColor,
        public ?string $fears,
        public ?string $avoidList,
        public string $world,               // فضاء | غابة | بحر | مدينة الحكايات | المدرسة
        public string $goal,                // شجاعة | ثقة | حب المدرسة | صداقة | مغامرة
        public ?string $occasion,
        public int $pageCount,              // 12 | 16 | 20 | 24
        public ?string $dedication,
        /** @var list<array>|null */ public ?array $templateBeats,  // من StoryTemplate
    ) {}

    public static function fromOrder(Order $order): self { /* ... */ }
}
```

### مدى العمر (ملزم في البرومبت وفي الفحص)

| الفئة | كلمات/صفحة | طول الجملة | المفردات | البنية |
|---|---|---|---|---|
| 3–5 | 20–35 | ≤ 8 كلمات | ملموسة جدًا، تكرار إيقاعي | مشكلة بسيطة → مساعدة → فرح |
| 6–8 | 40–70 | ≤ 12 كلمة | فصحى مبسطة | 3 عقبات → قرار → نتيجة |
| 9–10 | 70–110 | ≤ 16 كلمة | فصحى مع صور بلاغية خفيفة | صراع داخلي + حل بمجهود البطل |

### System Prompt

```
أنت كاتب قصص أطفال عربي محترف تكتب لدار "طيوف".
مهمتك: كتابة قصة شخصية يكون الطفل المذكور بطلها الحقيقي.

قواعد ملزمة:
1. اللغة عربية فصحى مبسطة، دافئة، بلا وعظ مباشر. العبرة تظهر من الأحداث لا من جملة نصيحة.
2. البطل هو الطفل باسمه الأول، ويُخاطَب/يوصف بصيغة {addressForm}. لا تخطئ في الضمائر أبدًا.
3. صفات الطفل واهتماماته يجب أن تُحرّك الحبكة فعليًا، لا أن تُذكر كزينة.
   إذا كان يحب الديناصورات، فالديناصور جزء من الحل — لا مجرد ذكر عابر.
4. الطفل يحل المشكلة بنفسه في النهاية. لا ينقذه بالغ.
5. ممنوع تمامًا: خوف حقيقي، عنف، موت، فقد، أشباح، تنمّر غير محلول،
   وصف جسدي للطفل، إشارات دينية أو سياسية، علامات تجارية، أسماء شخصيات محمية.
6. إذا ذُكر خوف للطفل: عالجه بلطف ونهاية آمنة تمامًا، ولا تُضخّمه.
7. احترم قائمة التجنب: {avoidList}.
8. عدد الصفحات = {pageCount} بالضبط، والصفحة الأولى (index 0) هي الغلاف.
9. مدى العمر: {ageBand} — {wordsPerPage} كلمة لكل صفحة، والجملة ≤ {maxSentenceWords} كلمة.
10. لكل صفحة اكتب أيضًا "sceneDescAr": وصف بصري للمشهد بالعربية موجّه لفريق الرسم
    (المكان، الزمن، وضعية البطل، المشاعر، العناصر الأساسية). هذا ليس جزءًا من القصة.

أخرج JSON فقط بالمخطط المطلوب. لا تكتب أي نص خارج الـ JSON.
```

### مخطط الإخراج

```php
// app/Data/Ai/StoryDraftData.php
class StoryDraftData extends Data
{
    public function __construct(
        #[Required] public string $title,
        #[Required] public string $logline,
        #[Required] public string $moral,
        /** @var DataCollection<StoryCharacterData> */
        public DataCollection $characters,
        /** @var DataCollection<StoryDraftPageData> */
        public DataCollection $pages,
    ) {}
}

class StoryDraftPageData extends Data
{
    public function __construct(
        public int $index,
        public PageKind $kind,
        public string $textAr,
        public string $sceneDescAr,
        public string $emotion,      // فرح | فضول | تردد | شجاعة | دهشة
        public string $timeOfDay,    // صباح | ظهيرة | غروب | ليل
        public string $location,
    ) {}
}
```

`validateAndCreate()` يرفض تلقائيًا أي استجابة ناقصة أو بعدد صفحات مخالف.
أضِف قاعدة مخصصة تتحقق أن `count($pages) === $input->pageCount` وأن الفهارس متسلسلة.

---

## د. [2] فحص السلامة والعمر

استدعاء منفصل، `temperature: 0.3`. يُرجع تقريرًا لا يعدّل النص.

```php
class SafetyReportData extends Data
{
    public function __construct(
        public SafetyVerdict $verdict,        // pass | warn | fail
        public AgeFitData $ageFit,
        /** @var DataCollection<SafetyIssueData> */
        public DataCollection $issues,
    ) {}
}

class SafetyIssueData extends Data
{
    public function __construct(
        public int $pageIndex,
        public IssueType $type,               // safety|pronoun|age|repetition|moral_preachy|off_brief
        public IssueSeverity $severity,       // low|medium|high
        public string $detail,
        public string $suggestion,
    ) {}
}
```

`fail` يمنع `OrderStateMachine` من الانتقال إلى `CharacterSetup` حتى يتدخّل `reviewer`.
افرض هذا داخل آلة الحالات نفسها، لا في الواجهة.

---

## هـ. [3] Character Bible — سرّ ثبات الشخصية

هذه أهم خطوة تقنيًا. عدم ثبات شكل الطفل بين الصفحات = أكبر خطر تشغيلي في المشروع
(مذكور صراحةً في دراسة الجدوى). الحل: **كتلة وصف إنجليزية واحدة تُكرَّر حرفيًا
في كل برومبت، بدون أي تعديل.**

### وضعان

**الوضع الافتراضي — نموذج منظّم (بدون إرسال صور):**

الأدمن ينظر للصور ويملأ:
`hairColor · hairStyle · skinTone · eyeColor · faceShape · glasses? · outfit ·
distinctiveFeature · buildType`

النظام يحوّلها إلى كتلة إنجليزية موحّدة.

**الوضع الاختياري — Vision:**

يتطلب `useVisionForCharacter = true` + موافقة إنتاج سارية. تُرسل صورة واحدة
فقط إلى Anthropic لاستخراج وصف كرتوني، ثم **تُحذف من السياق** ولا تُخزَّن نتيجة
وسيطة تحتوي الصورة.

### System Prompt

```
You are a character designer for a children's book studio.
Produce a SHORT, REUSABLE English descriptor block for an illustrated cartoon child character.

Rules:
- The character is a friendly, stylized CARTOON child. Never photorealistic.
- Never describe the child as attractive, mature, or in any way that sexualizes.
- No clothing that is revealing. Default to simple, modest, colorful children's clothing.
- Keep it under 45 words. Every word must be visually actionable.
- Include exactly: age impression, hair, skin tone, eyes, outfit with a locked color,
  and ONE distinctive feature that will make the character recognizable across pages.
- The outfit color and the distinctive feature must stay identical across all pages.
- Output JSON only.
```

### الإخراج

```php
class CharacterBibleData extends Data
{
    public function __construct(
        #[Max(400)] public string $heroDescriptor,   // ≤ 45 كلمة — تُكرَّر حرفيًا في كل صفحة
        public string $heroOutfit,
        public string $distinctiveFeature,
        /** @var DataCollection<SupportingCharacterData> */
        public DataCollection $supporting,
        /** @var list<string> */ public array $colorPalette,   // 4–6 hex
        /** @var list<string> */ public array $doNotList,
    ) {}
}
```

مثال على `heroDescriptor` جيد:

```
A cheerful 6-year-old boy, short curly black hair, warm medium-brown skin,
large dark-brown eyes, wearing a mustard-yellow hoodie with a small blue star
on the chest and navy trousers. Always carries a small blue telescope.
```

**بعد القفل (`isLocked`) لا يتغيّر حرف واحد.** أي تعديل يُنشئ نسخة جديدة ويستوجب
إعادة توليد كل الصفحات المتأثرة.

---

## و. [4] مهندس البرومبتات — Prompt Engineer

### بنية البرومبت (تركيب برمجي، لا يُترك للنموذج كاملًا)

```
[STYLE BLOCK]      ← ثابت من VisualStyle
[CHARACTER BLOCK]  ← ثابت من CharacterBible.heroDescriptor
[SCENE BLOCK]      ← متغيّر، يولّده الـ AI من sceneDescAr
[COMPOSITION]      ← نسبة الأبعاد + المنطقة الآمنة للنص + زاوية الكاميرا
[QUALITY TAGS]     ← ثابت
```

النموذج يولّد **SCENE BLOCK فقط**. الباقي تركيب في PHP:

```php
// app/Services/Prompts/PromptBuilder.php
final class PromptBuilder
{
    private const QUALITY = "QUALITY: children's picture book illustration, clean shapes, "
        ."soft lighting, high detail, consistent character design.";

    public function build(
        VisualStyle $style,
        CharacterBible $bible,
        string $scene,
        string $aspect,
        string $safeArea,
    ): string {
        return collect([
            $style->style_prompt,
            "CHARACTER (must stay identical in every image): {$bible->hero_descriptor}",
            "SCENE: {$scene}",
            "COMPOSITION: {$aspect} aspect ratio, leave clear empty space at the {$safeArea} "
                ."for text overlay, eye-level friendly camera, full scene visible.",
            self::QUALITY,
        ])->implode("\n\n");
    }

    public function negative(VisualStyle $style, ?string $extra = null): string
    {
        return trim(self::BASE_NEGATIVE.($extra ? ', '.$extra : '').', '.$style->negative_block, ', ');
    }
}
```

**اختبار Pest إلزامي:** نفس المدخلات = نفس المخرجات حرفيًا (byte-for-byte).
ثبات شكل الطفل يعتمد كليًا على هذا.

```php
it('builds a byte-identical prompt for identical inputs', function () {
    $a = app(PromptBuilder::class)->build(...$args);
    $b = app(PromptBuilder::class)->build(...$args);
    expect($a)->toBe($b)->and($a)->toContain($bible->hero_descriptor);
});
```

### System Prompt لتوليد SCENE BLOCK

```
You write image-generation SCENE descriptions for a children's picture book.
For each page you receive the Arabic scene description and the page text.

Rules:
- Output ENGLISH only.
- 25–45 words per scene. Concrete and visual: setting, action, props, light, mood.
- Do NOT describe the main character's appearance — that is handled separately.
  Refer to them only as "the child".
- Do NOT include any text, letters, words, numbers, or signage in the image.
- Keep continuity: same location details across pages that share a location.
- Never depict danger, injury, weapons, darkness that reads as scary, or adults
  touching the child.
- Also output a per-page negative prompt addition if the scene has a specific risk.
- Output JSON only.
```

### الإخراج

```php
class PagePromptsData extends Data
{
    public function __construct(
        /** @var DataCollection<PagePromptData> */
        public DataCollection $pages,
        public string $coverConcept,
    ) {}
}

class PagePromptData extends Data
{
    public function __construct(
        public int $index,
        public string $scene,
        public ?string $extraNegative,
        public TextSafeArea $suggestedSafeArea,  // top|bottom|left|right|none
        public AspectRatio $aspectRatio,         // 1:1|4:3|3:2|16:9
    ) {}
}
```

### Negative Prompt الأساسي (ثابت، يُدمج مع `extraNegative`)

```
text, letters, words, watermark, signature, logo, extra fingers, deformed hands,
extra limbs, distorted face, inconsistent character, multiple different children,
photorealistic, 3d render, uncanny, adult features, makeup, revealing clothing,
scary, blood, weapons, dark horror, low quality, blurry, cropped, out of frame
```

---

## ز. [5] Prompt Studio — الواجهة

هذه الشاشة هي التي يعمل عليها أمير يوميًا. اجعلها سريعة قبل أن تكون جميلة.

### التخطيط

```
┌──────────────────────────────────────────────────────────────┐
│ TUY-00123 · لِيان · 6-8 · عالم: الفضاء · أسلوب: مائي        │
│ [نسخ كتلة الشخصية] [تصدير الحزمة .md] [تنزيل ZIP فارغ]      │
│ [رفع مجلد صور]                     الصور: 9/17 ✓            │
├──────────────────────────────────────────────────────────────┤
│ ┌────────────┐ ┌────────────┐ ┌────────────┐                │
│ │ الغلاف     │ │ صفحة 01    │ │ صفحة 02    │                │
│ │ [صورة/فراغ]│ │ [صورة]     │ │ [فراغ]     │                │
│ │ النص العربي│ │ ...        │ │ ...        │                │
│ │ ▸ البرومبت │ │            │ │            │                │
│ │ [نسخ][رفع] │ │ [✓ معتمدة] │ │ [نسخ][رفع] │                │
│ └────────────┘ └────────────┘ └────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

### السلوكيات المطلوبة

- **نسخ فوري**: زر نسخ على كل برومبت + اختصار لوحة مفاتيح `C` عند التركيز على البطاقة.
- **تحرير يدوي**: أي برومبت قابل للتحرير؛ التحرير يرفع `promptVersion` ويُعلَّم «مُعدّل يدويًا».
- **إعادة توليد صفحة واحدة** بدون المساس بالباقي.
- **تصدير الحزمة** — ملف `TUY-00123-prompts.md`:

```md
# TUY-00123 — لِيان — 16 صفحة
## تعليمات التوليد
- استخدم نفس الأداة ونفس الإعدادات لكل الصفحات.
- استخدم صورة الغلاف المُنتَجة كـ reference/style لباقي الصفحات.
- سمِّ الملفات: cover.png, p01.png ... p16.png, back.png
- الدقة المطلوبة: 2048×2048 على الأقل.

## كتلة الشخصية (لا تُعدّل)
A cheerful 6-year-old girl, ...

## Negative (لكل الصفحات)
text, letters, words, watermark, ...

---
### cover.png — 1:1
<البرومبت الكامل>

### p01.png — 1:1 — مساحة نص: أسفل
<البرومبت الكامل>
...
```

- **الرفع الدفعي**: سحب مجلد → مطابقة بالاسم → معاينة المطابقة قبل التأكيد →
  رفض أي ملف لا يطابق النمط أو أقل من 2048px.
- **مقارنة الثبات**: عرض كل صور الطفل جنبًا إلى جنب في شريط واحد لكشف اختلاف
  الشكل بسرعة قبل التنضيد.
- **حالة الصفحة**: `PENDING → PROMPT_READY → UPLOADED → APPROVED | REDO`.
  زر «إعادة» يكتب سبب الإعادة في `reviewNote`.

---

## ح. AI في الموقع العام (موجّه للعميل)

### 1. اقتراح العالم والهدف

في الخطوة 3–4 من المعالج: بناءً على الصفات والاهتمامات، يقترح النظام عالمين وهدفين
مع سبب قصير بالعربية. `temperature: 0.7`، مخزّن بالـ cache حسب توليفة المدخلات.

### 2. المقطع التشويقي المجاني (Teaser) — أقوى محرّك تحويل

```
System: اكتب 3 جمل فقط بالعربية الفصحى المبسطة، تفتتح قصة أطفال بطلها {name}.
اجعلها مشوّقة وتتوقف عند لحظة تشويق. لا تُنهِ الحبكة. لا تذكر أي شيء مخيف.
أخرج نصًا عاديًا بلا JSON وبلا مقدمات.
```

**ضوابط إلزامية:**
- بدون صور، بدون بيانات حساسة، الاسم الأول فقط.
- 3 محاولات لكل جلسة + `RateLimiter::for('teaser')` بـ 10/ساعة لكل IP.
- `Cache::remember(md5($name.$age.$world.$goal), now()->addDays(7), ...)` —
  نفس التوليفة لا تُستدعى مرتين.
- `AI_TEASER_DAILY_BUDGET_USD` منفصل عن ميزانية الإنتاج؛ عند تجاوزه يُعرض نص
  جاهز من `story_templates.sample_titles` بدل الاستدعاء.
- هذا الاستدعاء **متزامن** استثناءً (العميل ينتظره) — لكن بـ `timeout(20)` وfallback فوري.

### 3. مساعد الأسئلة

يجيب **فقط** من `/faq` و`/packages` عبر context محقون. عند عدم المعرفة:
«ما عندي جواب دقيق لهذا — بحوّلك لفريق طيوف على واتساب.»
ممنوع أن يعد بسعر أو مدة تسليم غير موجودة في قاعدة البيانات.

---

## ط. مراقبة التكلفة

كل استدعاء يكتب `AiRun` بالتكلفة الفعلية. في `/studio/analytics`:

- تكلفة AI لكل طلب (يجب أن تبقى تحت **$1.5** للطلب الواحد).
- التكلفة الشهرية مقابل الميزانية.
- توزيع التكلفة حسب `AiRunKind` — إذا كان `TEASER` يتصدّر فهذا تسويق لا إنتاج،
  قيّمه مقابل معدل التحويل.
- متوسط عدد إعادات التوليد لكل طلب (مؤشر جودة البرومبتات).
