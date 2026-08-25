# CLAUDE.md — طيوف | TUYOOF

> هذا الملف يُقرأ في كل جلسة. لا تبدأ أي مهمة قبل قراءته بالكامل.
> المرجع التفصيلي في `docs/`. هذا الملف = القواعد المُلزِمة فقط.

---

## 1. ما هو المشروع

منصة عربية تُنتج **قصص أطفال شخصية** يكون الطفل بطلها — مبنية على اسمه وعمره وصفاته
واهتماماته وصورته — وتُسلَّم رقميًا (PDF) أو مطبوعة.

**مشروع Laravel واحد** فيه ثلاث واجهات، كلها React عبر Inertia:

| الطبقة | المسار | الجمهور |
|---|---|---|
| **Site** | `/` | الأهل والمدارس — موقع عام + معالج إنشاء القصة |
| **Customer** | `/track`, `/orders/{token}` | العميل — تتبع، معاينة، موافقة |
| **Studio** | `/studio` | الفريق — نظام إدارة الإنتاج |
| **Engine** | `app/Services/Ai` | داخلي — محرك القصص والبرومبتات |

**الفريق:** أمير (مدير المشروع + المطوّر) — سجود — بسملة.

---

## 2. القاعدة الذهبية للذكاء الاصطناعي

> **النظام لا يولّد صورًا. النظام يولّد بـرومبتات.**

الذكاء الاصطناعي في هذا المشروع مسؤول عن:

1. كتابة نص القصة بالعربية (مقسّم إلى صفحات).
2. بناء **Character Bible** — وصف بصري ثابت للبطل بالإنجليزية.
3. توليد **image prompt + negative prompt** لكل صفحة وللغلاف.
4. مراجعة آلية أولية للنص (لغة، عمر، سلامة).
5. مساعدات تسويقية وواجهة (اقتراح عوالم، مقطع تشويقي مجاني).

الذكاء الاصطناعي **ليس** مسؤولًا عن:

- توليد الصور نفسها — أمير/الفريق يولّدها خارجيًا ويرفعها للنظام.
- الموافقة النهائية على أي محتوى — دائمًا مراجعة بشرية.
- إرسال أي شيء للعميل تلقائيًا بدون موافقة بشرية.

أي كود يفترض وجود image generation API = **خطأ**. ابنِ دائمًا `PromptStudio` + `ImageUpload`.

---

## 3. الاصطلاحات الملزِمة

### التسمية

- **لا تسمِّ أي شيء `Book`.** استخدم `Product` / `StoryProduct`.
  السبب: المستقبل يحتوي Coloring Book, Poster, Stickers, Audio Story, Gift Box, School Kit.
- رقم الطلب: `TUY-00001` (prefix ثابت + counter من DB، وليس ULID مرئي للعميل).
- **PHP:** Models مفرد PascalCase (`StoryPage`)، الجداول جمع snake_case (`story_pages`)،
  الأعمدة snake_case. Actions أفعال (`GenerateImagePrompts`).
- **React:** مكوّنات `PascalCase.tsx`، hooks `use-*.ts`، مساعدات `kebab-case.ts`.
- **صفحات Inertia:** المسار في `Inertia::render('Studio/Orders/Show')` يطابق
  `resources/js/pages/Studio/Orders/Show.tsx` حرفيًا.
- المسارات العامة بالإنجليزية: `/create`, `/stories`, `/track`, `/studio/orders`.
- أسماء المسارات في Laravel: `studio.orders.show`, `site.create.store` — تُستخدم في
  React عبر Ziggy `route()`. **ممنوع كتابة URL نصيًا في JSX.**

### اللغة

- **اللغة الافتراضية: العربية، RTL.** الإنجليزية لاحقًا — لكن ابنِ البنية متعددة
  اللغات من اليوم الأول (لا تكتب نصوصًا مباشرة في JSX ولا في Blade).
- نصوص الواجهة في `resources/js/lang/ar.json`. رسائل الباك إند في `lang/ar/`.
- الترجمات تُشارك عبر `HandleInertiaRequests::share()` ويقرأها helper `t()`.
- **كود، متغيرات، تعليقات، commits: إنجليزي.** محتوى المستخدم: عربي.
- في CSS استخدم `ms-*` / `me-*` / `ps-*` / `pe-*` — ممنوع `ml-*` / `mr-*`.

### الحالات

`App\Enums\OrderStatus` (backed enum) هو المصدر الوحيد، مع `customerStage()`
التي تُحوّلها إلى 7 مراحل بلغة طيوف. لا تخترع سلاسل نصية للحالات في أي مكان،
لا في PHP ولا في TS — الأنواع تصل إلى الواجهة عبر `typescript-transformer`.

**الانتقال بين الحالات حصريًا عبر `OrderStateMachine::transition()`.**
ممنوع `$order->update(['status' => ...])` في أي مكان آخر.

---

## 4. الحدود التقنية

```
Backend     Laravel 12 (PHP 8.3+)
Frontend    React 19 + TypeScript strict — داخل نفس مشروع Laravel
الجسر       Inertia.js v2  ← لا REST API منفصل، لا SPA مستقلة
البناء      Vite 6
Styling     Tailwind CSS v4 + CSS variables (بدون hex مباشر في JSX)
UI          shadcn/ui (React) معدّل RTL
DB          PostgreSQL 16 + Eloquent (MySQL 8 يعمل — JSON casts في الحالتين)
Auth        Laravel starter kit (React) + spatie/laravel-permission
Storage     Laravel Filesystem → S3 driver → Cloudflare R2 (خاص + temporaryUrl)
Queue       Laravel Queue — database driver + حاوية worker
AI          خدمة AnthropicClient على Laravel HTTP client — claude-sonnet-4-6
PDF         Gotenberg (حاوية) — Blade → HTML → PDF (bleed 3mm)
DTOs        spatie/laravel-data + spatie/typescript-transformer
Routes JS   tighten/ziggy
Email       Resend
Test        Pest 3 + Playwright (E2E)
Deploy      VPS + Docker Compose
```

**ممنوع بدون سؤالي أولًا:** Redis · أي microservice · تغيير ORM · مكتبة UI ثانية ·
state manager عام (Redux/Zustand — Inertia props هي مصدر الحقيقة) ·
`laravel-medialibrary` · `laravel-model-states` · أي مكتبة PDF غير Gotenberg.
الأسباب مشروحة في `docs/02` قسم «حزم مرفوضة عمدًا».

**قواعد بنيوية ملزمة:**
- لا منطق أعمال في Controllers. Controller = يتحقق ← يستدعي Action ← `Inertia::render`.
- كل عملية كتابة = Action كلاس واحد بمنهج `handle()`.
- الأنواع تتولّد من `app/Data/*.php`، لا تُكتب يدويًا في `resources/js/types`.
- كل شيء ثقيل عبر `Inertia::defer()` وكل تحديث جزئي عبر `router.reload({ only: [...] })`.

---

## 5. الأمان وخصوصية الطفل — غير قابل للتفاوض

هذه ليست بنودًا قانونية، هي **قلب البراند**. أي PR يكسر واحدة منها مرفوض.

1. صور الأطفال في bucket **خاص** فقط. لا URL عام أبدًا. signed URL صلاحيته ≤ 15 دقيقة.
2. لا يُرفع أي شيء بدون سجل `Consent` مرتبط (نسخة النص + الوقت + IP).
3. موافقة التسويق **منفصلة تمامًا** عن موافقة الإنتاج. `marketingConsent` افتراضيًا `false`.
4. حذف تلقائي لصور الطفل بعد **30 يومًا** من التسليم — job مجدول + سجل حذف.
5. كل وصول لصورة طفل يُسجَّل في `AuditLog` (من، متى، أي طلب).
6. RBAC صارم: من لا يعمل على الطلب لا يرى صوره.
7. لا تُرسل صورة طفل لأي API خارجي إلا Anthropic، وفقط إذا فعّل الأدمن
   `useVisionForCharacter` صراحةً لهذا الطلب. الافتراضي: **مغلق** (وصف يدوي منظّم).
8. النص المولّد يمر على `contentSafetyCheck` قبل أن يراه أي إنسان أو يُطبع.
9. لا يظهر وجه طفل في أي مادة تسويقية داخل النظام إلا بعلم `marketingConsent = true`.

---

## 6. قواعد التصميم

### الموقع العام = عالم طيوف

اقرأ `docs/04-design-system.md` قبل أي مكوّن واجهة.
لا تستخدم الجماليات الافتراضية للـ AI (خلفية كريمية + serif + accent طيني).
طيوف = **سماء ليل حكايات**: نيلي عميق + ذهب دافئ + ورق فاتح.

### الاستوديو الإداري = Metronic

استخدم skill `metronic-dashboard` حرفيًا:
- زر primary **أسود** وليس أزرق.
- خلفية الصفحة `#F2F2F4`، البطاقات بيضاء `rounded-lg` (8px).
- نص الجسم 14px، خط Inter (وللعربي IBM Plex Sans Arabic).
- بدون ظلال ثقيلة. `dir="rtl"` على الجذر.

**لا تخلط بين النظامين.** الموقع العام دافئ وسردي؛ الاستوديو بارد وعملي وسريع.

الفصل تقني وليس اتفاقًا فقط:
- `SiteLayout.tsx` يستورد `resources/css/app.css` (توكنات طيوف).
- `StudioLayout.tsx` يستورد `resources/css/studio.css` (توكنات Metronic).
- مكوّنات `components/site/` و`components/studio/` **لا تستورد من بعضها إطلاقًا**.
- `components/ui/` (shadcn) مشتركة، لكنها تقرأ ألوانها من متغيرات CSS —
  فتأخذ شكل النظام الذي تعيش داخله تلقائيًا.

---

## 7. Definition of Done

قبل أن تقول "خلصت":

- [ ] `./vendor/bin/pint` نظيف · `php artisan test` أخضر · `npm run build` ينجح.
- [ ] TypeScript بدون `any` وبدون `@ts-ignore`.
- [ ] الصفحة تعمل على 375px عرض.
- [ ] RTL صحيح (لا انزياح، لا أيقونات معكوسة خطأً).
- [ ] كل نص من `resources/js/lang/ar.json` أو `lang/ar/`.
- [ ] حالات فارغة + تحميل + خطأ مغطّاة (بلغة طيوف، لا "Something went wrong").
- [ ] كل Controller يستدعي `authorize()` وكل مدخل عبر Form Request.
- [ ] لا N+1: استخدم `with()` وتحقق بـ `Model::preventLazyLoading()` في التطوير.
- [ ] اختبار Pest واحد على الأقل للمنطق الأساسي، و Feature test للمسار الجديد.

---

## 8. طريقة عملك معي

- **اقرأ قبل أن تكتب.** لو المهمة تلمس ملفًا موجودًا، اقرأه كاملًا أولًا.
- **مرحلة واحدة في كل مرة.** لا تقفز لمرحلة لاحقة من `docs/05-roadmap.md`.
- **اسألني عند الغموض في المنطق التجاري** (سعر، سياسة تعديل، مدة تنفيذ). لا تخترع.
- **لا تخترع محتوى تسويقيًا نهائيًا** — استخدم النصوص في `docs/04` أو ضع `TODO(copy)`.
- بعد كل مهمة: لخّص في 5 أسطر ما بُني، وما القرارات التي اتخذتها نيابة عني، وما التالي.
- حدّث `docs/CHANGELOG.md` بعد كل مرحلة.

---

## 9. خريطة الوثائق

| الملف | متى تقرأه |
|---|---|
| `docs/01-product-spec.md` | قبل أي feature — يحدد السلوك المطلوب |
| `docs/02-architecture.md` | قبل أي كود backend أو DB |
| `docs/03-ai-engine.md` | قبل أي شيء يخص القصة أو البرومبتات |
| `docs/04-design-system.md` | قبل أي مكوّن واجهة |
| `docs/05-roadmap.md` | في بداية كل مرحلة — خذ منها المهمة التالية فقط |
