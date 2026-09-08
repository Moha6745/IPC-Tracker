# نظام تتبّع مراحل المستخلصات — IPC Tracker

صفحة ويب واحدة (`index.html`) لتتبّع مراحل المستخلصات: الاستلام ← المراجعة ← الترجيع للاستشاري ← التسجيل بموارد ← التسجيل باعتماد، مع ملخص وتصدير Excel و PDF وطباعة.

## أوضاع التخزين

| الوضع | متى يعمل | أين تُحفظ البيانات |
|---|---|---|
| **سحابي (Firestore)** | عند وجود إعدادات Firebase صحيحة | Firestore — مشتركة بين كل الأجهزة وتتحدّث لحظياً |
| **سحابي (Claude)** | عند فتح الصفحة داخل بيئة Claude | تخزين Claude |
| **محلي** | عند غياب الإعدادات أو تعذّر الاتصال | `localStorage` في هذا المتصفح فقط |

يظهر الوضع الحالي في شارة أعلى الصفحة، وتفاصيله في بطاقة **المزامنة** في العمود الجانبي.
في الوضع السحابي يُحتفظ بنسخة محلية احتياطية، فإن انقطع الاتصال أو رفضت قواعد الأمان الوصول يتابع النظام العمل محلياً بدل أن يتوقف.

## خطوات الربط بـ Firebase

1. **أنشئ مشروعاً** في [وحدة تحكم Firebase](https://console.firebase.google.com/).
2. **أنشئ قاعدة بيانات Firestore**: Build ← Firestore Database ← Create database (اختر الموقع الأقرب، مثل `eur3` أو `nam5`).
3. **فعّل الدخول المجهول**: Build ← Authentication ← Sign-in method ← Anonymous ← Enable.
   هذا مطلوب لأن قواعد الأمان في `firestore.rules` تشترط `request.auth != null`.
4. **أضف تطبيق ويب**: إعدادات المشروع (⚙) ← Your apps ← Web (`</>`)، ثم انسخ مقطع `firebaseConfig`.
5. **أدخل الإعدادات** بإحدى طريقتين:
   - املأ ملف `firebase-config.js` في المستودع — يصلح للجميع عند نشر الصفحة.
   - أو من داخل النظام: بطاقة **المزامنة** ← إعدادات الاتصال ← الصق المقطع ← **اتصال** (يُحفظ في هذا المتصفح فقط ويتجاوز الملف).
6. **انشر قواعد الأمان** — من المتصفح، بلا تثبيت أي أداة:
   Build ← Firestore Database ← تبويب **Rules** ← امسح المحتوى ← الصق محتوى ملف `firestore.rules` ← **Publish**.

> مفاتيح `firebaseConfig` عامة بطبيعتها ويراها أي متصفح يفتح الصفحة — ليست أسراراً.
> الحماية الفعلية تأتي من قواعد الأمان، فلا تنشر الصفحة بقواعد مفتوحة.

<details>
<summary>بديل: نشر القواعد بسطر أوامر (اختياري، لمن يفضّله)</summary>

```bash
npm install -g firebase-tools
firebase login
firebase use --add            # اختر مشروعك
firebase deploy --only firestore:rules
```
الإعداد جاهز في `firebase.json`. اللصق من وحدة التحكم يؤدي الغرض نفسه تماماً.
</details>

## نقل البيانات المحلية إلى السحابة

عند أول اتصال، إذا كانت في هذا الجهاز مستخلصات محفوظة محلياً، يظهر زر **«رفع بيانات هذا الجهاز (n) إلى السحابة»** في بطاقة المزامنة. يرفع الزر كل المستخلصات المحلية دفعة واحدة، ويدمج المشاريع والأشخاص المحليين مع الموجودين في السحابة دون تكرار.

## بنية البيانات في Firestore

```
app/config          { projects: [{name, company, type}], people: [string] }
ipcs/{id}           { id, project, company, ipcNo, updatedAt,
                      events: [{id, status, round?, date, person, note}] }
```
`status` من: `receive` | `review` | `returned` | `mawared` | `etimad`.

## النشر

- **GitHub Pages** — مفعّل على `main`، والرابط: <https://moha6745.github.io/IPC-Tracker/>
- **Firebase Hosting** — ينشر تلقائياً مع كل دفعة إلى `main` عبر `.github/workflows/firebase-hosting.yml`،
  والرابط: <https://ipc-tracker-daa8e.web.app> (وكذلك `ipc-tracker-daa8e.firebaseapp.com`).
  يحتاج إعداداً لمرة واحدة، انظر أدناه.

- **محلياً**: `python3 -m http.server` ثم افتح العنوان في المتصفح (فتح الملف مباشرة بـ `file://` يعطّل تحميل `firebase-config.js` في بعض المتصفحات).

### إعداد نشر Firebase Hosting (مرة واحدة)

النشر يتم من GitHub Actions، فلا حاجة لتثبيت `firebase-tools` على أي جهاز. يلزم سرّ واحد:

1. **فعّل Hosting**: وحدة تحكم Firebase ← Build ← Hosting ← **Get started** ← تجاوز خطوات الـ CLI بـ Next حتى Continue to console.
2. **أنشئ مفتاح حساب خدمة**: إعدادات المشروع (⚙) ← تبويب **Service accounts** ← **Generate new private key** ← يُنزَّل ملف JSON.
3. **أضف السرّ في GitHub**: المستودع ← Settings ← Secrets and variables ← Actions ← **New repository secret**
   - الاسم: `FIREBASE_SERVICE_ACCOUNT`
   - القيمة: محتوى ملف الـ JSON كاملاً.
4. شغّل المسار من تبويب **Actions** ← «نشر على Firebase Hosting» ← **Run workflow** (أو ادفع أي تعديل إلى `main`).

> ملف الـ JSON مفتاح خاص حقيقي — لا تضعه في المستودع ولا ترسله في محادثة. مكانه أسرار GitHub فقط.
> إن أردت حذفه لاحقاً: Google Cloud Console ← IAM ← Service accounts.

## الملفات

| الملف | الغرض |
|---|---|
| `index.html` | التطبيق كاملاً — الواجهة والمنطق والتصدير |
| `firebase-config.js` | مفاتيح مشروع Firebase (عامة) |
| `firestore.rules` | قواعد أمان Firestore — انسخ محتواه والصقه في تبويب Rules بوحدة التحكم |
| `firestore.indexes.json` | فهارس Firestore (لا يحتاج النظام فهارس مركّبة حالياً) — يلزم فقط لمسار سطر الأوامر |
| `firebase.json` | إعداد استضافة Firebase وقواعد Firestore |
| `.firebaserc` | ربط المستودع بمشروع `ipc-tracker-daa8e` |
| `.github/workflows/firebase-hosting.yml` | نشر تلقائي على Firebase Hosting مع كل دفعة إلى `main` |
