# مشروع نبراس RAG — ورقة الطريق (Roadmap)

## 1) تعريف المشروع والهدف
**نبراس** هو نظام **RAG (استرجاع معزَّز بالتوليد)** عربي يعمل محليًا 100٪، يستقبل أسئلة بالعربية ويجيب عنها بالاعتماد الحصري على كتب/نصوص يقوم المستخدم برفعها.  
الهدف: أجوبة عربية أكاديمية دقيقة مع **استشهادات واضحة** (اسم الكتاب + نطاق الأسطر + نسبة التشابه + مقتطف حرفي قصير).

---

## 2) المعمارية المختصرة
- **قاعدة البيانات:** PostgreSQL — جداول `book`, `chunk`, `conversation`, `message`.
- **التضمين (Embeddings):** عبر LM Studio (نموذج: `text-embedding-intfloat-multilingual-e5-large-instruct`).
- **نموذج المحادثة:** `Qwen2.5-7B-Instruct-1M-GGUF` (من LM Studio).
- **الواجهة:** Streamlit عربية (RTL).
- **منطق الاسترجاع:** تشابه **Cosine** بين تضمين السؤال وتضمين المقاطع، مع حد قبول افتراضي `MIN_ACCEPT = 0.55` (يمكن رفعه لاحقًا إلى 0.80 لتحسين الدقة).
- **الاقتباس/المراجع:** عرض اسم الكتاب + الأسطر + النسبة + مقتطف (≤ 20 كلمة).

---

## 3) التقنيات والأدوات
- Python 3.x
- PostgreSQL + psycopg2
- Streamlit
- Requests
- LM Studio API (محليًا على `http://127.0.0.1:1234/v1`)
- tqdm (للـ progress في الإدخال)

---

## 4) إعداد البيئة والتشغيل السريع
### 4.1 متطلبات قبل التشغيل
1. PostgreSQL يعمل والاتصال صالح.
2. LM Studio يعمل على `127.0.0.1:1234`، مع تحميل:
   - نموذج محادثة: **Qwen2.5-7B-Instruct-1M-GGUF**
   - نموذج تضمين: **text-embedding-intfloat-multilingual-e5-large-instruct**
3. وضع الكتب النصية داخل المجلد: `./books/` (ملفات .txt بترميز UTF-8).

### 4.2 متغيرات الاتصال بقاعدة البيانات (ضمن السكربتات)
```
host=localhost, port=5432, user=postgres, password=******, dbname=nebras_rag
```

---

## 5) بنية قاعدة البيانات (Schema) — المختصر المفيد
### جدول `book`
- `id` SERIAL PK
- `name` TEXT
- `type` TEXT
- `file_url` TEXT (اختياري)
- `line_count` INT
- `chunk_count` INT
- `size` DOUBLE PRECISION (أُضيف لاحقًا)
- `content` TEXT
- `processing_status` TEXT

### جدول `chunk`
- `id` SERIAL PK
- `book_id` INT (FK -> book.id)
- `book_name` TEXT
- `content` TEXT
- `start_line` INT
- `end_line` INT
- `embedding_vector` VECTOR/JSON (تُخزن قائمة الأرقام)
- `embedding_model` TEXT
- `embedding_dim` INT

### جدول `conversation`
- `id` SERIAL PK
- `title` TEXT
- `message_count` INT
- `last_message_at` TIMESTAMP DEFAULT now()

### جدول `message`
- `id` SERIAL PK
- `conversation_id` INT (FK -> conversation.id)
- `role` TEXT (`user`|`assistant`)
- `content` TEXT
- `references_json` JSONB (اختياري)

> **ملاحظة:** تم لاحقًا إضافة عمود `size` في `book` عبر:
```
ALTER TABLE book ADD COLUMN IF NOT EXISTS size DOUBLE PRECISION DEFAULT 0;
```

---

## 6) الملفات/السكربتات التي أنجزناها وتعمل حاليًا
### 6.1 إنشاء الجداول الأساسية
- `setup_database.py` — إنشاء قاعدة البيانات والجداول الأساسية.
- `setup_chat_schema.py` — يتأكد من وجود `conversation` و `message` أو ينشئهما.

### 6.2 إدخال الكتب + حساب الأسطر + التضمين
- `ingest_books.py` (الإصدار المُحسّن):
  - تطبيع بسيط للنص العربي.
  - تجزئة على 400 كلمة مع تداخل 10% (OVERLAP = 40).
  - حساب **start_line/end_line** لكل مقطع.
  - توليد التضمين عبر LM Studio.
  - إدخال `book` و `chunk` بالقيم الصحيحة.

### 6.3 البحث والتوليد عبر واجهة رسومية
- `chat_ui.py`:
  - واجهة Streamlit عربية (RTL).
  - البحث الدلالي (Cosine) + ترتيب أفضل 5 نتائج (TOP_K = 5).
  - **Prompt صارم**: “لا تستخدم أي معرفة خارجية… ضع أرقام المراجع داخل النص (مرجع 1)…”. 
  - توليد الإجابة عبر `/completions` (LM Studio).
  - عرض المراجع المنسقة مع: **اسم الكتاب + الأسطر + نسبة التشابه + مقتطف**.
  - حفظ المحادثة والرسائل في PostgreSQL.

---

## 7) سلوك النظام الحالي (المعايير العملية)
- **اللغة:** عربية فصحى أكاديمية.
- **الاستشهاد:** حصريًا من المقاطع المسترجعة.
- **عند عدم كفاية الأدلة:** “المقاطع لا تحتوي على إجابة واضحة”.
- **العرض:** قسم “📖 المراجع المستعملة” أسفل كل إجابة.
- **التضمين:** `intfloat-multilingual-e5-large-instruct` (قوي جدًا للعربية).

> يمكن ضبط الدقة برفع `MIN_ACCEPT` إلى `0.80` للحصول على تقاطع أدق بين السؤال والمقاطع.

---

## 8) أوامر التشغيل السريعة
1) تشغيل إدخال الكتب:
```
python ingest_books.py
```
2) تشغيل الواجهة:
```
streamlit run chat_ui.py
```
3) إصلاح عمود الحجم (إذا لزم):
```
ALTER TABLE book ADD COLUMN IF NOT EXISTS size DOUBLE PRECISION DEFAULT 0;
```

---

## 9) ما تبقّى (الخطوات القادمة المقترحة)
- رفع حدّ التشابه إلى 0.80 وتقويم الأثر.
- إضافة فلتر حسب `book_name` عند وجود كلمات مفتاحية في السؤال.
- استخراج **اقتباس حرفي ≤ 20 كلمة** من نص المقطع (بخوارزمية جمل).
- سكربت تقييم آلي `evaluate_model.py` (5 أسئلة وتقرير CSV).

---

## 10) هيكل المجلد المقترح للتسليم/الأرشفة
```
rag_gpt_project/
├─ books/                         # ملفات .txt (المدخلات)
├─ chat_ui.py                     # واجهة Streamlit العربية
├─ ingest_books.py                # إدخال الكتب + التضمين + الأسطر
├─ setup_database.py              # إنشاء DB والجداول الأساسية
├─ setup_chat_schema.py           # جداول المحادثة/الرسائل
├─ list_and_view_conversations.py # استعراض المحادثات (اختياري)
├─ README_NEBRAS_RAG_ROADMAP.md   # (هذا الملف)
└─ PROJECT_PACKLIST_NEBRAS_RAG.txt# قائمة الملفات الواجب حفظها
```

---

## 11) ملاحظات تشغيلية
- تأكد أن LM Studio محمّل عليه النموذجين (Chat + Embedding) قبل التشغيل.
- إذا تغير الـ PORT أو الـ HOST لخادم LM Studio، حدّث القيم في السكربتات.
- الترميز الموصى به للكتب: UTF-8.
- يفضّل نصوص نظيفة (بدون تشكيل مفرط) لتحقيق دقة أعلى في الاسترجاع.

---

**تم إعداد هذه الورقة لتكون مرجعًا جاهزًا لإعادة التشغيل أو التسليم أو الرفع لاحقًا.**  
بمجرد رفع هذا الملف مع السكربتات المذكورة، سأتعرف مباشرة على سياق مشروع “نبراس RAG” وأكمل من حيث توقفت.
