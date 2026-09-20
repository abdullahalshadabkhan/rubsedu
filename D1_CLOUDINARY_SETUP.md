# 🚀 Cloudflare D1 + Cloudinary সেটআপ গাইড (Rajshahi Cantonment Public School & College)

এই ভার্সনে অ্যাপটি আর শুধু ব্রাউজারের LocalStorage-এ নয় — **Cloudflare D1** (একটি সত্যিকারের
SQL ডেটাবেস, Cloudflare Pages Functions দিয়ে চালিত) এ ডেটা রাখে, এবং ছবি + ব্যাকআপ
**Cloudinary**-তে জমা হয়। ফলে যেকোনো ডিভাইস/ব্রাউজার থেকে লগইন করলেই একই ডেটা দেখা যাবে,
এবং D1/ইন্টারনেট না থাকলে সাইট স্বয়ংক্রিয়ভাবে সবশেষ লোড হওয়া ডেটার একটি LocalStorage
কপি থেকে পড়া চালিয়ে যায় (তিন স্তরের ব্যাকআপ: D1 প্রধান, Cloudinary ছবি/JSON ব্যাকআপ,
LocalStorage অফলাইন ফলব্যাক)।

কোনো npm install / build লাগবে না — আগের মতোই শুধু ফাইল ডিপ্লয় করলেই চলবে, শুধু নিচের
ধাপগুলো একবার সেটআপ করতে হবে।

কোডে যা যা ইতিমধ্যে প্রস্তুত করা আছে:
```
functions/api/[[path]].js   ← D1 এর সাথে কথা বলা Cloudflare Pages Function (API)
schema.sql                  ← D1 ডেটাবেস টেবিল তৈরির SQL
wrangler.toml                ← D1 ডেটাবেস তৈরি/লোকাল টেস্টের জন্য (ঐচ্ছিক, CLI ব্যবহার করলে)
admin.html                   ← ইতিমধ্যে D1Adapter ব্যবহার করছে, Admin Key জেনারেট করে বসানো আছে
index.html                    ← ইতিমধ্যে D1Adapter ব্যবহার করছে (পাবলিক/read-only মোডে)
```

---

## ধাপ ১ — Cloudflare D1 ডেটাবেস তৈরি করুন

1. [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Workers & Pages** → বাম পাশে **D1 SQL Database** → **Create Database**।
2. নাম দিন `rcpsc-db` → **Create**।
3. ডেটাবেস খুলে **Console** ট্যাবে যান, `schema.sql` ফাইলের ভেতরের পুরো SQL কপি করে পেস্ট করে **Execute** করুন
   (অথবা CLI থাকলে: `npx wrangler d1 execute rcpsc-db --remote --file=./schema.sql`)।

## ধাপ ২ — GitHub-এ কোড পুশ করুন

`index.html`, `admin.html` এর পাশাপাশি নতুন `functions/` ফোল্ডার ও `schema.sql`,
`wrangler.toml` ফাইলগুলোও একসাথে GitHub রিপোতে রাখুন (পুরো ফোল্ডারটাই পুশ করুন)।

## ধাপ ৩ — Cloudflare Pages এ ডিপ্লয় ও D1 বাইন্ড করুন

1. Cloudflare Dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git** → রিপো সিলেক্ট করুন।
2. Build সেটিংস: Framework `None`, Build command খালি, Build output directory `/`।
3. ডিপ্লয় হওয়ার পর প্রজেক্টে যান → **Settings → Functions → D1 database bindings** → **Add binding**:
   - Variable name: `DB`  *(হুবহু এই নামেই দিতে হবে)*
   - D1 database: ধাপ ১-এ তৈরি করা `rcpsc-db` সিলেক্ট করুন
4. একই Settings পেজে **Environment Variables** → **Add variable**:
   - নাম: `ADMIN_API_KEY`
   - মান: নিচের মানটাই হুবহু বসান (এটা ইতিমধ্যে `admin.html`-এ বসানো আছে, তাই এখানেও ঠিক এটাই দিতে হবে):
     ```
     5b393033d77a3d2819973b170fc3f0fecccbc7058c196e7b4a9cf8ae772a34c6
     ```
   - **Encrypt** করে রাখুন
5. **Save** করার পর প্রজেক্ট রিডিপ্লয় করুন (Deployments ট্যাব → সর্বশেষ deployment → **Retry deployment**), যাতে নতুন binding/variable কার্যকর হয়।

## ধাপ ৪ — Admin Key যাচাই (ঐচ্ছিক পরিবর্তন)

`admin.html` ফাইলে `APP_CONFIG` অংশে (ফাইলের শুরুর দিকে) Admin Key ইতিমধ্যে বসানো আছে:

```js
const APP_CONFIG = {
  apiBase: '/api',
  adminKey: '5b393033d77a3d2819973b170fc3f0fecccbc7058c196e7b4a9cf8ae772a34c6',
  cloudinary: { ... }
};
```

চাইলে নিজের একটি নতুন গোপন কোড দিয়ে এটা বদলে দিতে পারেন — শুধু মনে রাখবেন, যা-ই বসান,
ঠিক সেই একই মান ধাপ ৩-এর `ADMIN_API_KEY`-তেও বসাতে হবে, দুই জায়গায় মান না মিললে Admin
Panel D1-এ কিছু লিখতে/পড়তে পারবে না।

⚠️ **`index.html` এ কখনো `adminKey` বসাবেন না** — এই ফাইল পাবলিক, যেকেউ সোর্স-কোড দেখতে
পারে। `index.html` এর `adminKey` খালি (`''`) রাখাই সঠিক এবং ইচ্ছাকৃত (এখনই তাই আছে)।

## ধাপ ৫ — Cloudinary সেটআপ (ছবি + ব্যাকআপ)

1. [cloudinary.com](https://cloudinary.com/users/register/free) এ বিনামূল্যে একাউন্ট খুলুন।
2. Dashboard এর উপরে **Cloud name** কপি করুন।
3. **Settings (⚙️) → Upload → Upload presets → Add upload preset**:
   - Signing Mode: **Unsigned** করুন
   - Preset name যা খুশি দিন (যেমন `rcpsc_uploads`), **Save**।
4. `admin.html` ও `index.html` — **দুটো ফাইলেই** `APP_CONFIG.cloudinary` অংশে বসান
   (এখন দুটোতেই `CHANGE_ME_CLOUD_NAME` / `CHANGE_ME_UPLOAD_PRESET` প্লেসহোল্ডার আছে):

```js
cloudinary: {
  cloudName: 'আপনার-cloud-name',
  uploadPreset: 'rcpsc_uploads',
  folder: 'rcpsc'
}
```

5. Commit/push করুন। এখন থেকে ছবি (student/teacher photo, logo, principal photo/signature,
   gallery, event/banner image) সরাসরি Cloudinary-তে আপলোড হবে এবং শুধু তার URL D1-এ
   সংরক্ষিত হবে — ফলে ডেটাবেস হালকা থাকবে এবং ছবিগুলো D1 থেকে আলাদা, স্বতন্ত্রভাবে
   Cloudinary-তে ব্যাকআপ থাকবে।
6. Cloudinary কনফিগার না করলেও অ্যাপ ভেঙে যাবে না — স্বয়ংক্রিয়ভাবে আগের মতো ছবি
   base64 হিসেবে সরাসরি D1-এ সংরক্ষিত হবে (fallback)। তবে প্রোডাকশনে Cloudinary
   কনফিগার করাই ভালো।

### JSON ব্যাকআপও Cloudinary তে
Admin Panel → **Settings → Backup / Restore** এ **☁️ Cloudinary তে ব্যাকআপ নিন**
বাটন আছে — পুরো ডেটাবেসের JSON dump Cloudinary তে (raw ফাইল হিসেবে) আপলোড হয়ে যাবে,
স্থানীয় "Download Backup (JSON)" এর পাশাপাশি একটি অতিরিক্ত ক্লাউড কপি হিসেবে।

## ধাপ ৬ — প্রথমবার সেটআপ

1. `https://YOUR-PROJECT.pages.dev/admin.html` খুলুন → ডিফল্ট লগইন দিয়ে লগইন করুন
   (প্রথমবার লগইন করার সময় ডেমো ডেটা স্বয়ংক্রিয়ভাবে D1-এ তৈরি হবে — এতে কিছুটা সময়
   লাগতে পারে, একাধিক নেটওয়ার্ক রিকোয়েস্ট যায়)।
2. লগইনের পরপরই **Settings → Password** থেকে পাসওয়ার্ড বদলান।
3. `https://YOUR-PROJECT.pages.dev/` খুলে পাবলিক সাইট চেক করুন — সব ডেটা এখন D1 থেকে আসছে।

---

## ⚠️ গুরুত্বপূর্ণ নিরাপত্তা নোট (অবশ্যই পড়ুন)

- **`ADMIN_API_KEY`** ই একমাত্র জিনিস যা `admin.html` কে D1-এ লেখার (create/update/delete)
  অনুমতি দেয়। এই key কাউকে শেয়ার করবেন না, এবং `index.html`-এ কখনো বসাবেন না।
- পাবলিক সাইট (`index.html`, কোনো key ছাড়াই) থেকে যেগুলো **পড়া** (read) যায়:
  school info, teachers (active), classes/sections/subjects, published notices,
  events, gallery, banners, published results, exams, syllabus, holidays, routines,
  **students**, counters। এবং শুধু **admissions** কালেকশনে নতুন আবেদন **জমা** (insert) দেওয়া যায়।
- **`students` কালেকশনটি ইচ্ছাকৃতভাবে পাবলিক-রিডেবল রাখা হয়েছে** — কারণ ওয়েবসাইটের
  Online Result Search, Result Sheet প্রিন্ট এবং Parent Login ফিচারগুলো (বাবা/মায়ের নাম,
  জন্মতারিখসহ) সরাসরি এই ডেটার উপর নির্ভর করে। এর মানে, `/api/students` URL-এ যে কারো
  অ্যাক্সেস থাকলে সে পুরো শিক্ষার্থী তালিকা দেখতে পারবে।
  **payments, feeHeads, users, attendance, messages, routines, examRoutine** — এগুলো
  সবসময় `ADMIN_API_KEY` ছাড়া সম্পূর্ণ বন্ধ থাকে।
- সাইট প্রথমবার খোলার সময় ডেমো ডেটা তৈরি (seed) শুধু `admin.html` থেকেই হয় — প্রথমেই
  অ্যাডমিন প্যানেলে লগইন করা আবশ্যক।

## 🔁 অফলাইন/নেটওয়ার্ক-সমস্যায় ব্যাকআপ আচরণ
D1 API রিচ করা না গেলে (ইন্টারনেট বিচ্ছিন্ন হলে), অ্যাপ স্বয়ংক্রিয়ভাবে শেষবার সফলভাবে
লোড হওয়া ডেটার একটি স্থানীয় (localStorage) কপি ব্যবহার করে পড়ার (read) জন্য — যাতে
সাইট পুরোপুরি বন্ধ না হয়ে যায়। এই সময় নতুন কোনো তথ্য লেখা (save) যাবে না; নেটওয়ার্ক
ফিরে এলে আবার স্বাভাবিকভাবে D1-এ লেখা শুরু হবে।
