# 02 — البنية التقنية وقاعدة البيانات

**مشروع Laravel واحد.** الباك إند Laravel، الواجهات React + TypeScript عبر Inertia،
التنسيق Tailwind v4. الموقع العام والاستوديو والوحة العميل — كلها داخل نفس المستودع
ونفس عملية البناء.

---

## أ. المكدّس

```
PHP            8.3+
Framework      Laravel 12
الجسر          Inertia.js v2   ← لا REST API منفصل، لا SPA مستقلة
الواجهة        React 19 + TypeScript (strict)
البناء         Vite 6
التنسيق        Tailwind CSS v4 + CSS variables
المكوّنات       shadcn/ui (نسخة React) معدّلة RTL
قاعدة البيانات  PostgreSQL 16  (MySQL 8 يعمل — استخدم JSON casts في الحالتين)
التخزين        Laravel Filesystem → S3 driver → Cloudflare R2
الطوابير       Laravel Queue — database driver (Redis لاحقًا)
الجدولة        Laravel Scheduler
PDF            Gotenberg (حاوية Docker) — Blade → HTML → PDF
الصور          Intervention Image v3
الصلاحيات      spatie/laravel-permission
DTOs           spatie/laravel-data + spatie/typescript-transformer
الروابط في JS  tighten/ziggy
البريد         Resend
الاختبار       Pest 3 (وحدة + feature) + Playwright (E2E)
النشر          VPS + Docker Compose
```

### لماذا Inertia وليس API + SPA منفصلة

مطوّر واحد. Inertia يلغي طبقة الـ API كاملة: لا resources، لا تكرار للتحقق،
لا مزامنة أنواع يدوية، لا مشاكل CORS أو توكنات. تكتب Controller يُرجع
`Inertia::render()` ويستقبله مكوّن React مباشرة. عند الحاجة لتطبيق موبايل
لاحقًا، تُضيف `routes/api.php` فوق نفس الـ Actions.

### حزم مرفوضة عمدًا (لا تُضِفها)

| الحزمة | السبب |
|---|---|
| `spatie/laravel-medialibrary` | نحتاج تحكمًا دقيقًا في bucket خاص + تاريخ انتهاء + سجل وصول لكل صورة طفل. `Asset` model مخصص أوضح. |
| `spatie/laravel-model-states` | 22 حالة = 22 كلاس. Enum + خدمة انتقال أبسط وأسهل اختبارًا. |
| `barryvdh/laravel-dompdf` / `mPDF` | دعم عربي/RTL ضعيف وتخطيط غير دقيق للصفحات الكاملة. |
| أي state manager (Redux/Zustand) | Inertia props هي مصدر الحقيقة. `useForm` و`router.reload` يكفيان. |
| Redis | حجم العمل لا يبرّره في السنة الأولى. أضِفه عند تجاوز 300 طلب/شهر. |

---

## ب. بنية المجلدات

```
tuyoof/
├── CLAUDE.md
├── docs/
├── docker-compose.yml
├── app/
│   ├── Actions/                    # منطق الأعمال — كلاس واحد لكل عملية
│   │   ├── Orders/
│   │   │   ├── CreateOrder.php
│   │   │   ├── TransitionOrderStatus.php
│   │   │   └── GenerateOrderTasks.php
│   │   ├── Story/
│   │   ├── Prompts/
│   │   ├── Assets/
│   │   └── Book/
│   ├── Console/Commands/
│   ├── Data/                       # Spatie Data — تتولّد منها أنواع TS
│   │   ├── OrderData.php
│   │   ├── StoryPageData.php
│   │   ├── CharacterBibleData.php
│   │   └── Ai/
│   │       ├── StoryDraftData.php
│   │       ├── SafetyReportData.php
│   │       └── PagePromptData.php
│   ├── Enums/
│   │   ├── OrderStatus.php
│   │   ├── CustomerStage.php
│   │   ├── AgeGroup.php
│   │   ├── ImageStatus.php
│   │   ├── AssetKind.php
│   │   ├── TaskType.php
│   │   └── AiRunKind.php
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Site/               # الموقع العام
│   │   │   ├── Customer/           # لوحة العميل
│   │   │   └── Studio/             # الاستوديو
│   │   ├── Middleware/
│   │   │   ├── HandleInertiaRequests.php
│   │   │   └── EnsureOrderAccess.php
│   │   └── Requests/
│   ├── Jobs/
│   │   ├── GenerateStoryDraft.php
│   │   ├── RunSafetyCheck.php
│   │   ├── GenerateImagePrompts.php
│   │   ├── BuildPreviewPdf.php
│   │   ├── BuildPrintPdf.php
│   │   └── DeleteExpiredChildPhotos.php
│   ├── Models/
│   ├── Notifications/
│   ├── Policies/
│   ├── Services/
│   │   ├── Ai/
│   │   │   ├── AnthropicClient.php
│   │   │   ├── BudgetGuard.php
│   │   │   └── Prompts/            # برومبتات النظام كثوابت
│   │   ├── Book/
│   │   │   ├── GotenbergRenderer.php
│   │   │   └── PreflightChecker.php
│   │   ├── Orders/OrderStateMachine.php
│   │   ├── Prompts/PromptBuilder.php
│   │   └── Storage/AssetManager.php
│   └── Support/
├── database/
│   ├── migrations/
│   ├── factories/
│   └── seeders/
├── lang/ar/                        # رسائل الباك إند
├── resources/
│   ├── css/
│   │   ├── app.css                 # Tailwind v4 + توكنات طيوف
│   │   ├── studio.css              # توكنات Metronic
│   │   └── book.css                # CSS الطباعة
│   ├── js/
│   │   ├── app.tsx
│   │   ├── layouts/
│   │   │   ├── SiteLayout.tsx
│   │   │   ├── StudioLayout.tsx
│   │   │   └── CustomerLayout.tsx
│   │   ├── pages/                  # يطابق Inertia::render()
│   │   │   ├── Site/
│   │   │   │   ├── Home.tsx
│   │   │   │   ├── Create/         # المعالج
│   │   │   │   ├── Packages.tsx
│   │   │   │   └── ...
│   │   │   ├── Customer/
│   │   │   │   ├── Track.tsx
│   │   │   │   └── Order.tsx
│   │   │   └── Studio/
│   │   │       ├── Dashboard.tsx
│   │   │       ├── Orders/
│   │   │       │   ├── Index.tsx
│   │   │       │   └── Show.tsx    # التبويبات السبعة
│   │   │       ├── Tasks/
│   │   │       ├── Library/
│   │   │       └── ...
│   │   ├── components/
│   │   │   ├── site/               # عالم طيوف
│   │   │   ├── studio/             # Metronic
│   │   │   └── ui/                 # shadcn
│   │   ├── hooks/
│   │   ├── lib/
│   │   │   ├── i18n.ts
│   │   │   └── utils.ts
│   │   ├── lang/ar.json            # نصوص الواجهة
│   │   └── types/
│   │       └── generated.d.ts      # مولّدة من app/Data — لا تحرّرها يدويًا
│   └── views/
│       ├── app.blade.php           # جذر Inertia
│       └── book/
│           ├── render.blade.php    # الكتاب كاملًا للطباعة
│           └── templates/          # 6 قوالب صفحات
├── routes/
│   ├── web.php                     # يستدعي الباقي
│   ├── site.php
│   ├── customer.php
│   ├── studio.php
│   ├── internal.php                # مسار عرض الكتاب لـ Gotenberg (signed)
│   └── console.php                 # الجدولة
└── tests/
    ├── Unit/
    ├── Feature/
    └── e2e/                        # Playwright
```

**قواعد ملزمة:**

- **لا منطق أعمال في Controllers.** الـController يتحقق ← يستدعي Action ← يُرجع `Inertia::render`.
- **كل عملية تكتب في قاعدة البيانات = Action كلاس واحد** بمنهج `handle()`.
- **لا استعلامات Eloquent داخل مكوّنات React** بالطبع، ولا داخل Blade.
- **الأنواع تتولّد، لا تُكتب.** غيّر `app/Data/*.php` ثم `php artisan typescript:transform`.

---

## ج. جسر Inertia — القواعد

```php
// Controller
public function show(Order $order): Response
{
    $this->authorize('view', $order);

    return Inertia::render('Studio/Orders/Show', [
        'order'   => OrderData::from($order->load('customer', 'child', 'package')),
        'story'   => Inertia::defer(fn () => StoryData::from($order->story)),
        'pages'   => Inertia::defer(fn () => StoryPageData::collect($order->story->pages)),
        'canLock' => $request->user()->can('lockCharacter', $order),
    ]);
}
```

- **`Inertia::defer()`** لكل ما هو ثقيل (الصفحات، الصور، السجل) — الصفحة تظهر فورًا.
- **`router.reload({ only: ['pages'] })`** لتحديث جزئي بدل إعادة تحميل كاملة.
- **Polling** لحالة الطلب في لوحة العميل: `router.reload({ only: ['stage'] })` كل 30 ثانية.
- **`useForm`** لكل نموذج — أخطاء التحقق تصل من Laravel تلقائيًا إلى `errors`.
- **مشاركة عامة** في `HandleInertiaRequests::share()`: المستخدم، الصلاحيات، الترجمات،
  رسائل flash. لا تشارك شيئًا ثقيلًا هنا.
- **`Ziggy`** لاستخدام `route('studio.orders.show', id)` داخل React.

---

## د. قاعدة البيانات

الأسماء ملزمة. أضف أعمدة عند الحاجة، لا تُعِد تسمية الجداول.

### الجداول

| الجدول | الأعمدة الأساسية |
|---|---|
| `users` | id, name, email, password, is_active, timestamps · (الأدوار عبر spatie/permission) |
| `customers` | id, name, phone(idx), email, city, address, channel, notes, timestamps |
| `child_profiles` | id, customer_id, name, age_group, address_form, traits(json), interests(json), favorite_things(json), favorite_color, fears, avoid_list, notes |
| `consents` | id, order_id(uniq), guardian_name, policy_version, production_use(bool), **marketing_use(bool, default false)**, signed_at, ip_address, user_agent, revoked_at |
| `products` | id, type, slug(uniq), name_ar, name_en, is_active |
| `packages` | id, product_id, slug(uniq), name_ar, description_ar, price_cents, currency, page_count, revisions_allowed, delivery_days_min, delivery_days_max, is_printed, sort_order, is_active |
| `addons` | id, slug(uniq), name_ar, price_cents, is_active |
| `orders` | id, number(uniq), public_token(uniq,64), status, customer_id, child_id, package_id, assignee_id, occasion, educational_goal, world, visual_style_id, dedication, requested_date, subtotal_cents, discount_cents, total_cents, currency, payment_status, payment_method, payment_ref, shipping_name, shipping_phone, shipping_address, tracking_number, revisions_used, photos_delete_at, use_vision_for_character(bool, default false), due_at, timestamps · idx(status), idx(customer_id) |
| `order_addon` | id, order_id, addon_id, qty, price_cents |
| `order_status_history` | id, order_id, from_status, to_status, user_id, note, created_at |
| `stories` | id, order_id(uniq), title, logline, moral, version, status, word_count, timestamps |
| `story_pages` | id, story_id, index, kind, text_ar, scene_desc_ar, image_prompt, negative_prompt, aspect_ratio, layout_template, text_safe_area, prompt_version, image_asset_id, image_status, review_note · uniq(story_id,index) |
| `story_versions` | id, story_id, version, snapshot(json), reason, created_at |
| `character_bibles` | id, order_id(uniq), hero_name, hero_descriptor, hero_outfit, distinctive_feature, supporting(json), style_prompt, negative_block, color_palette(json), do_not_list(json), is_locked, locked_by_id, locked_at |
| `visual_styles` | id, slug(uniq), name_ar, preview_url, style_prompt, negative_block, is_active |
| `story_templates` | id, slug(uniq), title_ar, world, goal, age_groups(json), beats(json), page_count, sample_titles(json), is_active |
| `assets` | id, kind, order_id, child_id, storage_key, disk, mime_type, size_bytes, width, height, checksum, is_private, expires_at, deleted_at, timestamps · idx(order_id) |
| `tasks` | id, order_id, type, status, priority, assignee_id, due_at, completed_at, timestamps |
| `revision_requests` | id, order_id, page_index, note, status, created_at |
| `internal_comments` | id, order_id, user_id, body, created_at |
| `ai_runs` | id, order_id, kind, model, input_tokens, output_tokens, cost_cents, duration_ms, status, prompt_hash, output(json), created_by_id, created_at · idx(kind,created_at) |
| `audit_logs` | id, user_id, action, entity_type, entity_id, metadata(json), ip_address, created_at · idx(entity_type,entity_id) |
| `settings` | key(pk), value(json) |
| `jobs`, `failed_jobs`, `job_batches` | من Laravel |

### الـ Enums

```php
// app/Enums/OrderStatus.php
enum OrderStatus: string
{
    case Draft = 'draft';
    case New = 'new';
    case DataReview = 'data_review';
    case OnHold = 'on_hold';
    case Writing = 'writing';
    case StoryReview = 'story_review';
    case CharacterSetup = 'character_setup';
    case PromptsReady = 'prompts_ready';
    case Illustrating = 'illustrating';
    case ImageReview = 'image_review';
    case BookLayout = 'book_layout';
    case Qc = 'qc';
    case PreviewSent = 'preview_sent';
    case RevisionRequested = 'revision_requested';
    case Approved = 'approved';
    case PrintReady = 'print_ready';
    case Printing = 'printing';
    case Packing = 'packing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
    case Refunded = 'refunded';

    public function labelAr(): string { /* من lang/ar/orders.php */ }

    public function customerStage(): CustomerStage
    {
        return match ($this) {
            self::New, self::DataReview, self::OnHold => CustomerStage::Received,
            self::Writing, self::StoryReview          => CustomerStage::Writing,
            self::CharacterSetup, self::PromptsReady,
            self::Illustrating, self::ImageReview     => CustomerStage::Illustrating,
            self::BookLayout, self::Qc                => CustomerStage::Preparing,
            self::PreviewSent, self::RevisionRequested=> CustomerStage::AwaitingYou,
            self::Approved, self::PrintReady,
            self::Printing, self::Packing             => CustomerStage::Printing,
            self::Shipped                             => CustomerStage::OnTheWay,
            default                                   => CustomerStage::Arrived,
        };
    }
}
```

باقي الـ enums: `AgeGroup`, `AddressForm`, `PaymentStatus`, `StoryStatus`, `PageKind`,
`ImageStatus`, `AssetKind`, `TaskType`, `TaskStatus`, `ProductType`, `AiRunKind`, `Role`.

### مثال Migration

```php
// database/migrations/xxxx_create_story_pages_table.php
Schema::create('story_pages', function (Blueprint $table) {
    $table->id();
    $table->foreignId('story_id')->constrained()->cascadeOnDelete();
    $table->unsignedSmallInteger('index');
    $table->string('kind')->default(PageKind::Spread->value);
    $table->text('text_ar')->nullable();
    $table->text('scene_desc_ar')->nullable();
    $table->text('image_prompt')->nullable();
    $table->text('negative_prompt')->nullable();
    $table->string('aspect_ratio')->default('1:1');
    $table->string('layout_template')->default('text-bottom');
    $table->string('text_safe_area')->nullable();
    $table->unsignedSmallInteger('prompt_version')->default(1);
    $table->foreignId('image_asset_id')->nullable()->constrained('assets')->nullOnDelete();
    $table->string('image_status')->default(ImageStatus::Pending->value);
    $table->text('review_note')->nullable();
    $table->timestamps();

    $table->unique(['story_id', 'index']);
});
```

### مثال Model

```php
// app/Models/StoryPage.php
class StoryPage extends Model
{
    protected $fillable = [/* ... */];

    protected function casts(): array
    {
        return [
            'kind'         => PageKind::class,
            'image_status' => ImageStatus::class,
        ];
    }

    public function story(): BelongsTo  { return $this->belongsTo(Story::class); }
    public function image(): BelongsTo  { return $this->belongsTo(Asset::class, 'image_asset_id'); }

    public function scopeAwaitingImage(Builder $q): Builder
    {
        return $q->whereIn('image_status', [ImageStatus::Pending, ImageStatus::PromptReady]);
    }
}
```

**قاعدة:** كل حقل `json` يأخذ cast صريحًا (`'array'` أو Data class). ممنوع `json_decode` يدويًا.

---

## هـ. آلة الحالات

```php
// app/Services/Orders/OrderStateMachine.php
final class OrderStateMachine
{
    /** @var array<string, list<OrderStatus>> */
    private const TRANSITIONS = [
        'new'         => [OrderStatus::DataReview, OrderStatus::Cancelled],
        'data_review' => [OrderStatus::Writing, OrderStatus::OnHold, OrderStatus::Cancelled],
        // ...
    ];

    public function canTransition(Order $order, OrderStatus $to): bool { }

    public function transition(Order $order, OrderStatus $to, ?User $by, ?string $note): void
    {
        // 1. تحقق من الانتقال  2. حدّث  3. اكتب order_status_history
        // 4. أطلق OrderStatusChanged event  5. ولّد المهام إن لزم
    }
}
```

الانتقال **حصريًا** عبر هذه الخدمة. ممنوع `$order->update(['status' => ...])` في أي مكان آخر.
الإشعارات تُطلق من الـ Event listener، لا من داخل الـ Action.

---

## و. التخزين والأصول

### الأقراص

```php
// config/filesystems.php
'r2_private' => [ 'driver' => 's3', 'bucket' => env('R2_BUCKET_PRIVATE'), 'visibility' => 'private', ... ],
'r2_public'  => [ 'driver' => 's3', 'bucket' => env('R2_BUCKET_PUBLIC'),  'visibility' => 'public',  ... ],
```

### المفاتيح

```
orders/{order}/photos/{ulid}.{ext}     خاص — يُحذف بعد 30 يومًا
orders/{order}/pages/p{nn}.png         خاص
orders/{order}/preview/preview.pdf     خاص + علامة مائية
orders/{order}/print/print.pdf         خاص
marketing/{slug}/...                    عام — فقط عند marketing_use = true
```

### الوصول

**ممنوع** وضع رابط R2 مباشر في أي prop يصل إلى React.
كل وصول عبر `GET /assets/{asset}` الذي:

1. `AssetPolicy::view()` — الدور + الإسناد للطلب.
2. إذا `kind = child_photo` → `AuditLog::record('VIEW_CHILD_PHOTO', ...)`.
3. `Storage::disk('r2_private')->temporaryUrl($key, now()->addMinutes(15))`.
4. `redirect()->away($url)`.

### الرفع

`AssetManager::storeChildPhoto()`:
- تحقق MIME حقيقي عبر `finfo` وليس امتداد الملف.
- حد 10MB.
- **إعادة ترميز عبر Intervention Image لإزالة EXIF** — خصوصًا إحداثيات GPS.
- حساب `checksum` لمنع التكرار.
- ضبط `expires_at = null` الآن، ويُضبط عند التسليم.

### الحذف

```php
// routes/console.php
Schedule::job(new DeleteExpiredChildPhotos)->dailyAt('03:00');
```

يحذف من R2، يضع `deleted_at`، يكتب `AuditLog`. `photos_delete_at` = تاريخ التسليم + 30 يومًا.

---

## ز. الطوابير والجدولة

```php
// كل عمل ثقيل = Job على طابور
GenerateStoryDraft::dispatch($order)->onQueue('ai');
GenerateImagePrompts::dispatch($story)->onQueue('ai');
BuildPreviewPdf::dispatch($order)->onQueue('pdf');
BuildPrintPdf::dispatch($order)->onQueue('pdf');
```

- طابور `ai`: `tries = 2`, `backoff = [10, 60]`, `timeout = 180`.
- طابور `pdf`: `tries = 2`, `timeout = 300`, **تزامن 1** (`WithoutOverlapping`).
- حاوية `worker` منفصلة في docker-compose تشغّل `queue:work --queue=ai,pdf,default`.
- تقدّم الـ Job يُعرض في الاستوديو عبر عمود حالة على `Order` + `router.reload`.

---

## ح. توليد PDF عبر Gotenberg

لماذا Gotenberg وليس Dompdf: العربية RTL والخطوط المضمّنة والصور الكاملة
لا تعمل بشكل مقبول إلا عبر Chromium حقيقي.

```
[Job BuildPreviewPdf]
   → يوقّع رابطًا: /internal/book/{order}/render?preview=1&signature=...
   → POST إلى http://gotenberg:3000/forms/chromium/convert/url
       + paperWidth/paperHeight بالبوصة (21cm+bleed)
       + marginTop/Bottom/Left/Right = 0
       + printBackground = true
       + waitForExpression: window.__BOOK_READY__ === true
   → يستقبل الـ PDF ويخزّنه في r2_private
   → يحدّث الطلب ويطلق إشعارًا داخليًا
```

**قواعد:**
- المسار `/internal/book/...` محمي بـ `signed` middleware ومقيّد بشبكة Docker الداخلية.
- الخطوط تُخدم محليًا من `public/fonts` عبر `@font-face` — لا CDN، لأن Gotenberg قد يفشل عليه.
- Blade يضبط `window.__BOOK_READY__ = true` بعد تحميل كل الصور.
- المعاينة: 100dpi + علامة مائية. الطباعة: كامل + bleed 3mm + علامات قص.
- `PreflightChecker` يعمل **قبل** التصيير: دقة كل صورة، تباين النص، النص داخل الهامش الآمن.

---

## ط. الصلاحيات

`spatie/laravel-permission` بـ 8 أدوار من `docs/01`.

```php
// app/Policies/OrderPolicy.php
public function view(User $user, Order $order): bool
{
    return $user->hasAnyRole(['owner', 'admin', 'support', 'reviewer'])
        || $order->assignee_id === $user->id
        || $order->tasks()->where('assignee_id', $user->id)->exists();
}

public function viewChildPhotos(User $user, Order $order): bool
{
    return $this->view($user, $order) && ! $order->consent?->revoked_at;
}
```

- كل Controller في `Studio` يبدأ بـ `$this->authorize(...)`.
- Middleware `role:` على مجموعات المسارات.
- لوحة العميل عبر `EnsureOrderAccess` الذي يقرأ `public_token` — **قراءة وموافقة
  وطلب تعديل فقط**، لا تعديل لبيانات الطفل.

---

## ي. متغيرات البيئة

```env
APP_NAME=Tuyoof
APP_URL=https://tuyoof.com
APP_LOCALE=ar

DB_CONNECTION=pgsql
DB_DATABASE=tuyoof

QUEUE_CONNECTION=database
SESSION_DRIVER=database
CACHE_STORE=database

ANTHROPIC_API_KEY=
AI_MODEL=claude-sonnet-4-6
AI_MONTHLY_BUDGET_USD=150
AI_TEASER_DAILY_BUDGET_USD=5

FILESYSTEM_DISK=r2_private
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_PRIVATE=tuyoof-private
R2_BUCKET_PUBLIC=tuyoof-public
R2_ENDPOINT=

GOTENBERG_URL=http://gotenberg:3000

RESEND_API_KEY=
MAIL_FROM_ADDRESS=hello@tuyoof.com

PHOTO_RETENTION_DAYS=30
BOOK_SIZE_MM=210
BOOK_BLEED_MM=3
```

---

## ك. docker-compose

```yaml
services:
  app:        # PHP-FPM 8.3 + Nginx (أو FrankenPHP)
  worker:     # نفس الصورة، الأمر: php artisan queue:work --queue=ai,pdf,default
  scheduler:  # نفس الصورة، الأمر: php artisan schedule:work
  postgres:
  gotenberg:  # gotenberg/gotenberg:8
```

Vite يعمل محليًا فقط (`npm run dev`)؛ في الإنتاج `npm run build` ثم أصول ثابتة.

---

## ل. الأمان — قائمة فحص لكل PR

- [ ] كل Controller في `Studio` يستدعي `authorize()`.
- [ ] كل مدخل يمر على Form Request بقواعد صريحة.
- [ ] `public_token` = `Str::random(64)` وفريد ومفهرس.
- [ ] الرفع: MIME حقيقي + حد حجم + إعادة ترميز لإزالة EXIF.
- [ ] `RateLimiter` على المعالج والـ teaser و`/track`.
- [ ] **لا اسم طفل ولا صورة في أي `Log::` أبدًا.**
- [ ] لا `$fillable` مفتوح على أعمدة حساسة (`status`, `total_cents`, `public_token`).
- [ ] التسعير يُحسب على الخادم دائمًا — لا تثق بأي سعر قادم من الواجهة.

---

## م. الاختبارات

| النوع | الأداة | ما يُغطّى |
|---|---|---|
| وحدة | Pest | `OrderStateMachine`، `PromptBuilder`، التسعير، `customerStage()` |
| Feature | Pest + `RefreshDatabase` | كل Controller، الصلاحيات، الطوابير (`Queue::fake`) |
| AI | Pest + `Http::fake` | تحقق مخططات الاستجابة وإعادة المحاولة وتسجيل التكلفة |
| E2E | Playwright | المعالج كاملًا، الرفع الدفعي، إرسال المعاينة، موافقة العميل |
| بصري | Playwright screenshots | صفحات الكتاب PDF — تحقق العربية والاتجاه |

الحد الأدنى قبل الإطلاق: مسار E2E كامل
`create → pay → studio order → story → prompts → upload → preview → approve`.
