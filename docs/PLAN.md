# خطة مشروع WaveDrop — استقبال AirDrop على Android ومعرض Live Photos

> اسم مبدئي. الوثيقة دي بتكمّل ملخص المشروع (القسم 1) وبتحوّله لخطة تنفيذ.

## 1. ملخص المشروع (مثبّت)

- **استقبال من iPhone** بـ AirDrop الأصلي، من غير أي تطبيق على الآيفون.
- **معرض خاص** للصور والفيديوهات والـ Live Photos، مع تشغيل الحركة والصوت.
- **مشاركة على Instagram** بتصدير مناسب للـ Story والـ Reel.

حدود ثابتة:

1. مش كل أجهزة Android تقدر تستقبل AirDrop.
2. الأصل لا يُعدَّل أبدًا؛ أي نسخة للعرض أو المشاركة هي نسخة مشتقة.
3. مفيش تزييف لهوية iPhone عند Instagram.

---

## 2. المعمارية: ثلاث طبقات منفصلة

```
┌────────────────────┐   ┌─────────────────────┐   ┌──────────────────────┐
│  Ingest (الاستقبال) │ → │  Library (المكتبة)    │ → │  Export (التصدير)      │
│  مصادر قابلة للتبديل │   │  تخزين + فهرسة + عرض │   │  Motion Photo / MP4  │
└────────────────────┘   └─────────────────────┘   └──────────────────────┘
```

طبقة الاستقبال بتسلّم **ملفات خام + مصدرها** بس، ومتعرفش أي حاجة عن المعرض. المكتبة متعرفش الملفات جت إزاي. ده اللي بيخلّينا نبدّل طريقة الاستقبال من غير ما نلمس الباقي.

### Gradle modules

| Module | المسؤولية |
|---|---|
| `:core:model` | أنواع البيانات المشتركة (`Asset`, `AssetKind`, `IngestBatch`) |
| `:ingest:api` | الواجهة `IngestSource` وعقد التسليم |
| `:ingest:share` | استقبال من Share sheet (`ACTION_SEND` / `ACTION_SEND_MULTIPLE`) |
| `:ingest:quickshare` | مراقبة مجلد استقبال Quick Share عبر MediaStore |
| `:ingest:airdrop-system` | تكامل مع خدمة AirDrop على مستوى النظام (Tarish أو ما يشبهه) — اختياري |
| `:livephoto` | قراءة الميتاداتا، ربط الصورة بالفيديو، استخراج still-time |
| `:library` | Room DB + تخزين الأصول داخل مساحة التطبيق + كاش المشتقات |
| `:feature:gallery` | واجهة Compose: الشبكة، العارض، تشغيل الحركة |
| `:media:player` | تغليف Media3 ExoPlayer لتشغيل Live/فيديو |
| `:export` | Media3 Transformer، كاتب Motion Photo، تكامل Instagram |
| `:app` | تجميع، DI (Hilt)، التنقل |

```kotlin
interface IngestSource {
    val id: String                       // "share", "quickshare", "airdrop-system"
    val availability: Flow<Availability> // Supported / NeedsSetup / Unsupported(reason)
    val batches: Flow<IngestBatch>       // كل دفعة = ملفات وصلت مع بعض
}

data class IngestBatch(
    val sourceId: String,
    val receivedAt: Instant,
    val files: List<IncomingFile>,       // Uri + displayName + mimeType + size
)
```

---

## 3. طرق الاستقبال حسب الجهاز (Tiers)

| Tier | الطريقة | الأجهزة | دور التطبيق |
|---|---|---|---|
| **A** | Quick Share ↔ AirDrop المدمج في النظام | Pixel 10 حاليًا، وجوجل أعلنت توسّعه لأجهزة أكتر في 2026 | النظام يستقبل؛ إحنا نلتقط الملفات من مجلد الاستقبال أو من Share sheet |
| **B** | خدمة AirDrop على مستوى النظام (Tarish / AWDL مخصص) | ROM مخصص أو صلاحيات نظام + شريحة Wi-Fi متوافقة | نتكامل مع الخدمة عبر Bound Service / Broadcast |
| **C** | مفيش AirDrop | باقي الأجهزة | التطبيق يشتغل كمعرض + أداة تصدير للملفات اللي توصل بأي طريقة؛ ونوضّح للمستخدم إن الاستقبال المباشر مش مدعوم |

قرار: **نبدأ بـ Tier A** لأنه الوحيد اللي يشتغل كـ APK عادي على Play Store. Tier B مسار موازي للمتحمسين وما يعطّلش الإصدار.

### أهم مجهول لازم نحسمه أولًا (Spike 0)

لما الآيفون يبعت Live Photo عبر AirDrop لجهاز Android متوافق، **إيه اللي بيوصل فعلًا؟**

- `HEIC + MOV` كملفين منفصلين؟ (المتوقع لو اختار المستخدم Options → All Photos Data)
- صورة ثابتة بس (JPEG/HEIC)؟
- اسم الملفين متطابق (`IMG_1234.HEIC` / `IMG_1234.MOV`)؟
- الميتاداتا (ContentIdentifier) محفوظة؟

لازم نختبر على جهاز حقيقي بإعدادات مختلفة (Most Compatible / High Efficiency، مع وبدون All Photos Data) ونوثّق النتيجة في `docs/spikes/airdrop-payload.md` قبل ما نبني منطق الربط.

---

## 4. نموذج البيانات

```kotlin
enum class AssetKind { PHOTO, VIDEO, LIVE_PHOTO }

@Entity
data class Asset(
    @PrimaryKey val id: String,          // UUID
    val kind: AssetKind,
    val stillPath: String?,              // الأصل كما وصل — لا يتغير
    val videoPath: String?,              // MOV للـ Live أو الفيديو العادي
    val contentIdentifier: String?,      // Apple asset identifier للربط
    val stillTimeUs: Long?,              // لحظة الصورة الثابتة داخل الفيديو
    val capturedAt: Instant?,
    val width: Int, val height: Int,
    val isHdr: Boolean,
    val sourceId: String,
    val sha256: String,                  // منع التكرار
    val receivedAt: Instant,
)

@Entity
data class Derivative(                   // نسخ مشتقة قابلة للحذف وإعادة الإنشاء
    @PrimaryKey val key: String,         // assetId + preset
    val assetId: String,
    val preset: String,                  // "thumb", "preview-sdr", "motionphoto", "ig-story", ...
    val path: String,
    val createdAt: Instant,
)
```

التخزين: الأصول داخل `filesDir/originals/` (خاصة بالتطبيق) مع خيار "حفظ نسخة في المعرض العام" عبر MediaStore. المشتقات في `cacheDir` ويتمسحوا من غير خسارة.

---

## 5. ربط الـ Live Photo (`:livephoto`)

ترتيب الأولوية:

1. **ContentIdentifier**: من Apple MakerNote في الـ HEIC/JPEG، ومن `com.apple.quicktime.content.identifier` في الـ MOV. لو متطابقين → Live Photo مؤكد.
2. **نفس الاسم الأساسي + نفس الدفعة** (`IMG_1234.HEIC` + `IMG_1234.MOV`) لو الميتاداتا اتشالت.
3. **تقارب وقت الالتقاط** (فرق < 1 ثانية) + مدة فيديو ≤ 4 ثواني — كحل أخير، ويتعلّم كـ "ربط تقديري".

من الـ MOV نقرأ `com.apple.quicktime.still-image-time` عشان نعرف أي فريم هو الصورة الثابتة.

الوحدة دي **Kotlin خالص بدون Android** قدر الإمكان، عشان تتختبر بـ JUnit على عينات حقيقية (`livephoto/src/test/resources/samples/`).

---

## 6. المعرض والعرض

- **الشبكة**: Compose `LazyVerticalGrid` + Coil مع decoder للـ HEIC (Android 9+ عبر `ImageDecoder`). شارة "LIVE" على الـ Live Photos.
- **العارض**: الصورة الثابتة تظهر فورًا؛ **ضغطة مطوّلة** تشغّل الفيديو بالصوت (زي iOS)، وتشغيل قصير صامت عند فتح الصورة (قابل للإيقاف من الإعدادات).
- **الانتقال**: الفيديو ينتهي على فريم `stillTimeUs` عشان الانتقال يبقى سلس.
- **HDR**: لو الجهاز مش بيدعم HEVC 10-bit / Dolby Vision → ننشئ مشتق `preview-sdr` بـ Transformer مرة واحدة ونكاشه.

---

## 7. التصدير والمشاركة (`:export`)

| Preset | الصيغة | الاستخدام |
|---|---|---|
| `original` | الملفات كما هي (HEIC + MOV) | نسخ احتياطي / رجوعها لجهاز Apple |
| `motionphoto` | JPEG + MP4 مدمج بـ XMP (`GCamera:MotionPhoto`, `Container:Directory`) | Google Photos وأي معرض Android يدعم Motion Photos |
| `still-jpeg` | JPEG sRGB | أقصى توافق للصورة الثابتة |
| `ig-story` | MP4 H.264 + AAC، SDR، 1080×1920، 30fps | Instagram Story |
| `ig-reel` | نفس المواصفات، مع خيار Loop أو Bounce (ذهاب وعودة) لمدة 6–9 ثواني | Instagram Reel |

### Instagram

- Instagram على Android **ما بيعرفش الـ Live Photo كنوع**؛ أفضل حل صادق هو تصديرها كفيديو قصير.
- Story: `Intent("com.instagram.share.ADD_TO_STORY")` مع `content://` من `FileProvider` و`source_application` (Facebook App ID — لازم نسجّل تطبيق في Meta for Developers).
- Reel / Feed: `ACTION_SEND` بـ `video/mp4` و`setPackage("com.instagram.android")`، مع fallback للـ Share sheet العادي لو Instagram مش متثبت.
- التحويل بـ **Media3 Transformer**: tone-mapping من HDR لـ SDR، قص/حشو لـ 9:16 مع خلفية مموّهة، وتكرار المقطع للـ Loop/Bounce.
- منسيبش ملفات التصدير للأبد: تتمسح من الكاش بعد 24 ساعة.

---

## 8. التقنيات

- Kotlin، Jetpack Compose، Material 3
- `minSdk 29` (scoped storage + HEIC مستقر)، `targetSdk` = أحدث إصدار
- Hilt، Room، Coroutines/Flow، WorkManager (للتحويلات الطويلة)
- Media3: ExoPlayer + Transformer
- Coil 3 للصور
- `androidx.exifinterface` + parser بسيط للـ ISO-BMFF/QuickTime atoms للميتاداتا اللي ExifInterface مش بيقراها

---

## 9. خارطة الطريق

| المرحلة | المحتوى | معيار الانتهاء |
|---|---|---|
| **0 — Spikes** | توثيق payload الـ AirDrop على Pixel 10؛ تجميع 20+ عينة Live Photo حقيقية؛ تجربة Instagram Story intent | ملف نتائج مكتوب + عينات في المستودع (بعد إزالة بيانات الموقع) |
| **1 — الأساس** | هيكل الـ modules، `:livephoto` بالاختبارات، Room، استقبال عبر Share sheet | مشاركة HEIC+MOV للتطبيق ⇒ يظهروا كـ Live Photo واحدة |
| **2 — المعرض** | الشبكة، العارض، ضغطة مطوّلة للتشغيل، مشتقات SDR | تشغيل سلس على جهاز متوسط، بدون تقطيع في الشبكة |
| **3 — Tier A** | مراقبة مجلد Quick Share واستيراد تلقائي (اختياري بموافقة المستخدم) | AirDrop من iPhone ⇒ الصورة في المعرض خلال ثواني من غير خطوات يدوية |
| **4 — التصدير** | Motion Photo، presets الـ Instagram، Loop/Bounce | الملف المصدَّر يتنشر Story وReel بنجاح ويتعرض كـ Motion Photo في Google Photos |
| **5 — النشر** | كشف التوافق وشاشة onboarding حسب الـ Tier، سياسة خصوصية، Play Store | إصدار Beta مغلق |
| **6 — Tier B** | تكامل مع خدمة AirDrop على مستوى النظام | يعمل على جهاز/ROM مرجعي واحد على الأقل |

---

## 10. المخاطر

| الخطر | الأثر | التخفيف |
|---|---|---|
| AirDrop الوارد يوصّل صورة ثابتة بس | يضيع الـ Live Photo | Spike 0؛ إرشاد المستخدم لاختيار All Photos Data؛ التطبيق يفضل مفيد كمعرض/مصدِّر |
| Quick Share interop محصور في أجهزة قليلة | قاعدة مستخدمين صغيرة | المعمارية القابلة للتبديل؛ Tier C كمعرض مستقل |
| Tier B يحتاج صلاحيات نظام | مستحيل كـ APK عادي على Play | توزيعه كإضافة منفصلة خارج Play |
| تغيير Instagram لواجهة المشاركة | كسر ميزة المشاركة | عزلها في class واحد + fallback للـ Share sheet |
| HEVC/HDR تقيل على أجهزة ضعيفة | بطء أو فشل في العرض | مشتقات SDR مكاشة + تحويل في WorkManager |
| ميتاداتا حساسة (GPS) | خصوصية | خيار "إزالة الموقع" افتراضيًا في كل presets المشاركة |

---

## 11. الخطوة الجاية

1. تنفيذ Spike 0 على جهاز Pixel 10 (أو أي جهاز وصله تحديث Quick Share ↔ AirDrop).
2. بالتوازي: إنشاء هيكل المشروع (Gradle multi-module) ووحدة `:livephoto` باختباراتها على العينات.
