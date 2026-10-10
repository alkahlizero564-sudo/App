# AI Builder

تطبيق موبايل لبناء المواقع والتطبيقات بالذكاء الاصطناعي، مربوط مع Claude وGemini وSupabase.

## القوائم

1. **المفاتيح**: مفتاح Claude، مفتاح Gemini، رابط ومفتاح Supabase (تُحفظ مشفرة على الجهاز).
2. **الكود**: محرر كود، وزر تشغيل لعرض الناتج، وحفظ المشاريع في Supabase، وتصدير الموقع كملف HTML.
3. **التعديل**: تختار Claude أو Gemini وتكتب المطلوب، فيُعدّل الكود ويضعه في قائمة الكود.

## الملفات

- `App.js`: كود التطبيق كامل.
- `.github/workflows/build-app-apk.yml`: يبني ملف APK تلقائياً.

## الحصول على ملف APK

1. افتح تبويب **Actions**.
2. اختر **Build App APK** ثم **Run workflow**.
3. بعد 10 إلى 15 دقيقة افتح تبويب **Releases** ونزّل `app.apk`.
4. ثبّته على هاتفك (فعّل التثبيت من مصادر غير معروفة).

## إعداد Supabase

شغّل هذا الكود مرة واحدة في Supabase داخل SQL Editor:

```sql
create table if not exists public.projects (
  name text primary key,
  code text not null default '',
  updated_at timestamptz not null default now()
);

alter table public.projects enable row level security;

drop policy if exists "personal use" on public.projects;
create policy "personal use" on public.projects
  for all to anon
  using (true)
  with check (true);
```

ثم انسخ **Project URL** و **anon public key** من Project Settings ثم API، وضعهما في قائمة المفاتيح داخل التطبيق.
