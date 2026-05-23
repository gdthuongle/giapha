# GIAPHA-OS — ROADMAP v2.2.1 · PHẦN 1/2
### Nền móng kỹ thuật & Data model cốt lõi
> **Phần 1 gồm:** Thay đổi so với v2.2 · Kiến trúc · Phase 0 (Nền móng) · Phase 1 (Data model)
> **Phần 2 gồm:** Phase 2–6 · GEDCOM · UI · Production · Sprint plan · Kết quả kỳ vọng

---

---

## NHỮNG GÌ ĐÃ THAY ĐỔI TỪ v2.2 → v2.2.1

| # | Vấn đề | Đã sửa |
|---|--------|--------|
| 1 | **Bug họ/tên nghiêm trọng**: `parts.pop()` lấy từ cuối → export sai "Văn /An/" | Sửa: họ = từ **đầu tiên**, export "Văn An /Nguyễn/" |
| 2 | Bước 4.1b chỉ có hướng dẫn chung, thiếu code copy-paste | Thêm toàn bộ code `exporter.ts` thực tế |
| 3 | Parser chưa đọc được `SURN`, `GIVN`, `_LUNAR` khi import lại | Thêm Bước 4.1e: cập nhật `parseGedcom()` |
| 4 | Bước 4.1 bị gắn vào Phase 4 (tuần 14), nhưng thực ra làm được ngay | Ghi rõ: có thể làm **trước Phase 1** vì chỉ sửa `utils/gedcom.ts` |
| 5 | `ExportButton.tsx` chưa có code tải file đúng encoding | Thêm `downloadGedcomFile()` hoàn chỉnh |
| 6 | Test cases còn thiếu một số case quan trọng | Bổ sung test họ/tên, ký tự đặc biệt, round-trip |

---

## MỤC LỤC

- [Kiến trúc tổng thể](#phần-1--kiến-trúc-tổng-thể)
- [Phase 0 — Nền móng kỹ thuật](#phase-0--nền-móng-kỹ-thuật-tuần-12)
- [Phase 1 — Data model cốt lõi](#phase-1--data-model-cốt-lõi-tuần-38)
- [Phase 2 — Media & Source](#phase-2--media--source-tuần-911)
- [Phase 3 — Place & Audit](#phase-3--place--audit-tuần-1213)
- [Phase 4 — GEDCOM & Dedup](#phase-4--gedcom--dedup-tuần-1416) ← Bước 4.1 làm được sớm hơn
- [Phase 5 — UI/UX & Graph](#phase-5--uiux--graph-tuần-1720)
- [Phase 6 — Production](#phase-6--production-tuần-2122)
- [Tổng hợp files](#phần-2--tổng-hợp-files)
- [Sprint plan](#phần-3--sprint-plan)
- [Kết quả kỳ vọng](#phần-4--kết-quả-kỳ-vọng)

---

## PHẦN 1 — KIẾN TRÚC TỔNG THỂ

```
PostgreSQL / Supabase
│
├── IDENTITY LAYER
│   ├── persons              ← Danh tính thuần túy (không còn ngày tháng)
│   └── person_names         ← Tên húy, pháp danh, tên tự, tên thánh...
│
├── RELATIONSHIP LAYER
│   ├── families             ← Đơn vị gia đình
│   ├── family_parents       ← Cha/mẹ trong gia đình
│   └── family_children      ← Con cái trong gia đình
│
├── EVENT LAYER
│   ├── events               ← Mọi sự kiện: sinh/mất/cưới/di cư...
│   └── person_events        ← Liên kết người ↔ sự kiện
│
├── PLACE LAYER
│   ├── places               ← Địa danh có cấu trúc phân cấp
│   └── place_aliases        ← Tên lịch sử (Sài Gòn = TP.HCM)
│
├── MEDIA LAYER
│   ├── media                ← Ảnh, tài liệu, video
│   ├── person_media         ← Ảnh của người
│   ├── event_media          ← Ảnh/tài liệu của sự kiện
│   └── family_media         ← Ảnh của gia đình
│
├── EVIDENCE LAYER
│   ├── sources              ← Nguồn tài liệu
│   ├── citations            ← Trích dẫn từ nguồn
│   └── citation_links       ← Trích dẫn gắn vào trường cụ thể
│
└── SYSTEM LAYER
    ├── audit_logs           ← Ai sửa gì lúc nào
    └── person_search_index  ← Materialized view để tìm kiếm nhanh

Next.js App
│
├── services/                ← Business logic + RPC transaction
│   ├── person.service.ts
│   ├── family.service.ts
│   └── event.service.ts
│
├── rules/                   ← Domain rules
│   ├── family.rules.ts
│   ├── event.rules.ts
│   └── person.rules.ts
│
└── utils/
    ├── calendar/            ← Âm lịch, Can Chi, Age Calculation
    ├── date-parser/         ← Parse ngày tiếng Việt
    ├── gedcom/              ← Module GEDCOM (parser + exporter)
    └── graph/               ← Kinship engine
```

---

# PHASE 0 — NỀN MÓNG KỸ THUẬT (Tuần 1–2)
### Mục tiêu: Tạo khung an toàn trước khi thay đổi data

Không có gì thay đổi visually với người dùng ở phase này. Đây là lớp bảo vệ cho toàn bộ dự án.

---

## Bước 0.1 — Service Layer + Transaction qua Supabase RPC

### Tại sao cần?

Khi tạo một gia đình, cần insert vào 3–4 bảng cùng lúc. Nếu bước 2 lỗi mạng, bảng 1 đã có data nhưng bảng 2 không có → **data hỏng một nửa**. Không sửa được.

### ⚠️ Vấn đề quan trọng với Supabase

Supabase JS client **không** hỗ trợ transaction trực tiếp. Mỗi `.insert()` là một request riêng biệt. Phải dùng **PostgreSQL RPC function** để đảm bảo "tất cả thành công hoặc không có gì".

### Cách dùng Supabase RPC

**Bước 1 — Vào Supabase Dashboard → SQL Editor → chạy:**

```sql
-- File: docs/migrations/YYYYMMDD_rpc_create_family.sql

CREATE OR REPLACE FUNCTION public.create_family_unit(payload JSONB)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  new_family_id UUID;
BEGIN
  -- 1. Tạo family
  INSERT INTO public.families (type, status)
  VALUES (
    (payload->>'type')::TEXT,
    'active'
  )
  RETURNING id INTO new_family_id;

  -- 2. Gắn cha (nếu có)
  IF payload->>'parent_a_id' IS NOT NULL THEN
    INSERT INTO public.family_parents (family_id, person_id, role)
    VALUES (
      new_family_id,
      (payload->>'parent_a_id')::UUID,
      (payload->>'parent_a_role')::TEXT
    );
  END IF;

  -- 3. Gắn mẹ (nếu có)
  IF payload->>'parent_b_id' IS NOT NULL THEN
    INSERT INTO public.family_parents (family_id, person_id, role)
    VALUES (
      new_family_id,
      (payload->>'parent_b_id')::UUID,
      (payload->>'parent_b_role')::TEXT
    );
  END IF;

  -- 4. Gắn con cái (nếu có)
  IF payload->'children' IS NOT NULL THEN
    INSERT INTO public.family_children (family_id, person_id, relationship_type)
    SELECT
      new_family_id,
      (child->>'id')::UUID,
      COALESCE(child->>'type', 'biological')
    FROM JSONB_ARRAY_ELEMENTS(payload->'children') AS child;
  END IF;

  -- 5. Ghi audit log
  INSERT INTO public.audit_logs (table_name, record_id, action, changed_by)
  VALUES ('families', new_family_id, 'CREATE', auth.uid());

  -- Nếu bất kỳ bước nào lỗi → PostgreSQL tự rollback toàn bộ
  RETURN JSONB_BUILD_OBJECT('success', true, 'family_id', new_family_id);

EXCEPTION WHEN OTHERS THEN
  RETURN JSONB_BUILD_OBJECT('success', false, 'error', SQLERRM);
END;
$$;
```

**Bước 2 — Gọi từ TypeScript:**

```typescript
// services/family.service.ts

export async function createFamilyUnit(data: CreateFamilyPayload) {
  // Kiểm tra rules TRƯỚC KHI gọi DB
  const ruleErrors = validateFamilyRules(data);
  if (ruleErrors.length > 0) {
    return { success: false, errors: ruleErrors };
  }

  // Gọi RPC — PostgreSQL tự rollback nếu lỗi
  const { data: result, error } = await supabase.rpc('create_family_unit', {
    payload: {
      type: data.type,                      // 'marriage' | 'partnership' | 'unknown'
      parent_a_id: data.parentA?.id ?? null,
      parent_a_role: data.parentA?.role ?? null,  // 'husband' | 'wife' | 'partner'
      parent_b_id: data.parentB?.id ?? null,
      parent_b_role: data.parentB?.role ?? null,
      children: data.children ?? [],        // [{ id, type }]
    }
  });

  if (error || !result?.success) {
    return { success: false, error: result?.error || error?.message };
  }

  return { success: true, familyId: result.family_id };
}
```

> **Quy tắc bắt buộc:** Mọi thao tác ghi vào 2+ bảng đều phải qua RPC function, **không** gọi `.insert()` nhiều lần từ JS.

---

## Bước 0.2 — Soft Delete (không bao giờ xóa vĩnh viễn ngay)

**Vấn đề:** Xóa nhầm 1 người → mất toàn bộ sự kiện, ảnh, quan hệ của họ.

**Giải pháp:** Thêm cột `deleted_at`. Khi "xóa" chỉ là đặt `deleted_at = now()`.

```sql
-- File: docs/migrations/YYYYMMDD_soft_delete.sql
-- Chạy trong Supabase SQL Editor

ALTER TABLE public.persons
  ADD COLUMN IF NOT EXISTS deleted_at  TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS deleted_by  UUID REFERENCES public.profiles(id);

ALTER TABLE public.families
  ADD COLUMN IF NOT EXISTS deleted_at  TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS deleted_by  UUID REFERENCES public.profiles(id);

ALTER TABLE public.events
  ADD COLUMN IF NOT EXISTS deleted_at  TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS deleted_by  UUID REFERENCES public.profiles(id);

ALTER TABLE public.media
  ADD COLUMN IF NOT EXISTS deleted_at  TIMESTAMPTZ;

ALTER TABLE public.sources
  ADD COLUMN IF NOT EXISTS deleted_at  TIMESTAMPTZ;

-- Views: mọi query dùng view này thay vì truy cập bảng trực tiếp
CREATE OR REPLACE VIEW public.persons_active AS
  SELECT * FROM public.persons WHERE deleted_at IS NULL;

CREATE OR REPLACE VIEW public.families_active AS
  SELECT * FROM public.families WHERE deleted_at IS NULL;

CREATE OR REPLACE VIEW public.events_active AS
  SELECT * FROM public.events WHERE deleted_at IS NULL;
```

**Quy tắc purge — quan trọng:**

```
✅ Có thể xóa thật:   media orphan (ảnh không gắn với entity nào)
✅ Có thể xóa thật:   import_temp (dữ liệu tạm khi import GEDCOM thất bại)
❌ KHÔNG xóa thật:    persons — dù đã soft delete 90 ngày
❌ KHÔNG xóa thật:    families, events, sources, citations
```

**Luồng xóa trong UI:**
```
User click "Xóa"
  → gọi service: UPDATE persons SET deleted_at=now(), deleted_by=uid WHERE id=...
  → biến mất khỏi app
  → trang /dashboard/trash: xem danh sách + nút "Khôi phục"
  → sau 90 ngày: admin thấy nút "Xóa vĩnh viễn" (chỉ dùng cho media orphan)
```

---

## Bước 0.3 — Domain Rules Engine

**Tạo file `rules/person.rules.ts`:**

```typescript
// rules/person.rules.ts

export type RuleSeverity = 'error' | 'warning';

export interface RuleResult {
  valid:    boolean;
  severity: RuleSeverity;
  message:  string;
  field?:   string;  // Trường nào bị lỗi (để form highlight màu đỏ)
}

export const personRules = {

  birthBeforeDeath(birthYear: number | null, deathYear: number | null): RuleResult {
    if (!birthYear || !deathYear) return ok();
    if (birthYear > deathYear) return {
      valid: false, severity: 'error',
      message: `Năm sinh (${birthYear}) không thể sau năm mất (${deathYear})`,
      field: 'birth_year'
    };
    return ok();
  },

  reasonableAge(birthYear: number | null, deathYear: number | null): RuleResult {
    if (!birthYear || !deathYear) return ok();
    const age = deathYear - birthYear;
    if (age > 130) return {
      valid: true,  // Warning — vẫn cho lưu, chỉ cảnh báo
      severity: 'warning',
      message: `Tuổi thọ ${age} năm là bất thường. Vui lòng kiểm tra lại.`
    };
    return ok();
  },
};

function ok(): RuleResult {
  return { valid: true, severity: 'warning', message: '' };
}
```

**Tạo file `rules/family.rules.ts`:**

```typescript
// rules/family.rules.ts

export const familyRules = {

  noSelfParent(parentId: string, childId: string): RuleResult {
    if (parentId === childId) return {
      valid: false, severity: 'error',
      message: 'Một người không thể là cha/mẹ của chính mình'
    };
    return ok();
  },

  noAncestorCycle(
    personId: string,
    ancestorIds: Set<string>  // Tập tổ tiên của người được thêm làm parent
  ): RuleResult {
    if (ancestorIds.has(personId)) return {
      valid: false, severity: 'error',
      message: 'Không thể thêm quan hệ này — sẽ tạo vòng lặp trong cây gia phả'
    };
    return ok();
  },

  reasonableParentAge(parentBirthYear: number | null, childBirthYear: number | null): RuleResult {
    if (!parentBirthYear || !childBirthYear) return ok();
    const age = childBirthYear - parentBirthYear;
    if (age < 12 || age > 80) return {
      valid: true,  // Warning — vẫn cho lưu
      severity: 'warning',
      message: `Tuổi cha/mẹ khi sinh con là ${age} tuổi. Vui lòng kiểm tra lại.`
    };
    return ok();
  },
};
```

**Hiển thị lỗi trong UI:**
```
❌ Lỗi — không thể lưu:
   "Năm sinh (1980) không thể sau năm mất (1975)"

⚠️ Cảnh báo — dữ liệu bất thường:
   "Tuổi cha khi sinh con là 78 tuổi. Vui lòng kiểm tra lại."
   [Bỏ qua và lưu]  [Hủy]
```

---

## Bước 0.4 — Optimistic Locking (2 người cùng sửa)

**Vấn đề:** User A và User B cùng mở trang sửa thông tin. Người lưu sau ghi đè người lưu trước mà không báo.

```sql
-- File: docs/migrations/YYYYMMDD_optimistic_lock.sql

ALTER TABLE public.persons  ADD COLUMN IF NOT EXISTS version INT NOT NULL DEFAULT 1;
ALTER TABLE public.families ADD COLUMN IF NOT EXISTS version INT NOT NULL DEFAULT 1;
ALTER TABLE public.events   ADD COLUMN IF NOT EXISTS version INT NOT NULL DEFAULT 1;

-- Trigger tự tăng version mỗi UPDATE
-- Client KHÔNG được tự gửi version mới
CREATE OR REPLACE FUNCTION increment_version()
RETURNS TRIGGER AS $$
BEGIN
  NEW.version = OLD.version + 1;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER persons_version  BEFORE UPDATE ON public.persons
  FOR EACH ROW EXECUTE FUNCTION increment_version();
CREATE TRIGGER families_version BEFORE UPDATE ON public.families
  FOR EACH ROW EXECUTE FUNCTION increment_version();
CREATE TRIGGER events_version   BEFORE UPDATE ON public.events
  FOR EACH ROW EXECUTE FUNCTION increment_version();
```

**Code trong service:**

```typescript
// services/person.service.ts

export async function updatePerson(
  id: string,
  data: UpdatePersonData,
  expectedVersion: number  // Lấy từ lúc đọc dữ liệu — dùng để kiểm tra, không gửi lên
) {
  const { data: result } = await supabase
    .from('persons')
    .update({
      // KHÔNG gửi 'version' trong đây — trigger tự tăng
      preferred_name: data.preferredName,
      gender:         data.gender,
      living:         data.living,
      updated_at:     new Date().toISOString(),
    })
    .eq('id', id)
    .eq('version', expectedVersion)  // Phải khớp version hiện tại
    .select('id, version')
    .single();

  if (!result) {
    return {
      success: false,
      conflict: true,
      message: 'Thông tin này vừa được người khác cập nhật. Vui lòng tải lại trang.'
    };
  }

  return { success: true, newVersion: result.version };
}
```

---

## Bước 0.5 — Test Setup

```bash
# Chạy 1 lần trong terminal dự án
npm install --save-dev vitest @vitest/ui

# Thêm vào package.json → scripts:
# "test": "vitest run",
# "test:watch": "vitest",
# "test:ui": "vitest --ui"
```

**Cấu trúc thư mục test:**
```
tests/
├── calendar/
│   ├── canChi.test.ts
│   └── lunarDate.test.ts
├── date/
│   ├── ageCalculation.test.ts     ← Quan trọng nhất
│   └── dateParser.test.ts
├── rules/
│   ├── familyRules.test.ts
│   └── personRules.test.ts
├── gedcom/
│   ├── parser.test.ts
│   └── exporter.test.ts           ← Bao gồm Unicode + họ/tên tests
└── graph/
    └── kinship.test.ts
```

---

# PHASE 1 — DATA MODEL CỐT LÕI (Tuần 3–8)

> **Thứ tự sprint v2.2.1:** Family Model làm **TRƯỚC** Event System vì marriage event cần gắn với family.

---

## Bước 1.1 — Person Names System

**Vấn đề:** `persons.other_names = "Ông Giáo, Cậu Hai, Tráng Phủ"` — text lộn xộn, không cấu trúc.

```sql
-- File: docs/migrations/YYYYMMDD_person_names.sql

CREATE TYPE name_type_enum AS ENUM (
  'birth',       -- Tên khai sinh
  'courtesy',    -- Tên tự
  'posthumous',  -- Tên húy (kiêng gọi khi còn sống)
  'religious',   -- Pháp danh / Tên thánh
  'married',     -- Tên sau khi lấy chồng/vợ
  'nickname',    -- Biệt danh
  'alias'        -- Tên khác không rõ loại
);

CREATE TABLE public.person_names (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  person_id  UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  type       name_type_enum NOT NULL DEFAULT 'birth',
  full_text  TEXT NOT NULL,        -- Tên đầy đủ
  surname    TEXT,                  -- Họ (Nguyễn, Trần, Lê...)
  given_name TEXT,                  -- Tên + đệm (Văn An, Thị Hoa...)
  language   TEXT DEFAULT 'vi',
  is_primary BOOLEAN DEFAULT FALSE, -- Tên chính hiển thị trong cây
  note       TEXT,
  deleted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Mỗi người chỉ có 1 tên chính
CREATE UNIQUE INDEX idx_person_names_primary
  ON public.person_names(person_id)
  WHERE is_primary = TRUE AND deleted_at IS NULL;

-- Migration: chuyển full_name cũ
INSERT INTO public.person_names (person_id, type, full_text, is_primary)
SELECT id, 'birth', full_name, TRUE
FROM public.persons
WHERE full_name IS NOT NULL AND full_name != '';
```

**UI mới trong MemberForm.tsx:**
```
╔══════════════════════════════════════════╗
║  HỌ VÀ TÊN                               ║
║  Tên khai sinh: [Nguyễn Văn An         ] ║
║                                          ║
║  TÊN KHÁC (nếu có)                       ║
║  [Tên tự    ▼] [Tráng Phủ            ] [×]║
║  [Pháp danh ▼] [Thích Quảng Đức      ] [×]║
║  [+ Thêm tên khác]                       ║
╚══════════════════════════════════════════╝
```

---

## Bước 1.2 — Family Model (làm TRƯỚC Event System)

```sql
-- File: docs/migrations/YYYYMMDD_family_model.sql

CREATE TYPE family_type_enum AS ENUM (
  'marriage', 'partnership', 'unknown'
);

CREATE TYPE family_status_enum AS ENUM (
  'active', 'divorced', 'widowed', 'separated', 'unknown'
);

CREATE TABLE public.families (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type       family_type_enum   NOT NULL DEFAULT 'marriage',
  status     family_status_enum NOT NULL DEFAULT 'active',
  start_year INT,
  end_year   INT,
  note       TEXT,
  version    INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ,
  deleted_by UUID REFERENCES public.profiles(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE parent_role_enum AS ENUM ('husband', 'wife', 'partner');

CREATE TABLE public.family_parents (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  family_id  UUID REFERENCES public.families(id) ON DELETE CASCADE NOT NULL,
  person_id  UUID REFERENCES public.persons(id)  ON DELETE CASCADE NOT NULL,
  role       parent_role_enum NOT NULL,
  sort_order INT DEFAULT 0,
  UNIQUE(family_id, person_id)
);

CREATE TYPE child_type_enum AS ENUM (
  'biological', 'adopted', 'foster', 'stepchild'
);

CREATE TABLE public.family_children (
  id                UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  family_id         UUID REFERENCES public.families(id) ON DELETE CASCADE NOT NULL,
  person_id         UUID REFERENCES public.persons(id)  ON DELETE CASCADE NOT NULL,
  relationship_type child_type_enum NOT NULL DEFAULT 'biological',
  sort_order        INT DEFAULT 0,
  UNIQUE(family_id, person_id)
);
```

**Migration từ relationships cũ:**

```typescript
// scripts/migrate-to-family-model.ts
// Chạy: npx ts-node scripts/migrate-to-family-model.ts

async function migrateToFamilyModel() {
  console.log('Bắt đầu migration...');

  // Lấy tất cả quan hệ hôn nhân
  const { data: marriages } = await supabase
    .from('relationships')
    .select('*')
    .eq('type', 'marriage');

  for (const marriage of marriages ?? []) {
    // Tạo family qua RPC
    const result = await supabase.rpc('create_family_unit', {
      payload: {
        type: 'marriage',
        parent_a_id:   marriage.person_a,
        parent_a_role: 'husband',
        parent_b_id:   marriage.person_b,
        parent_b_role: 'wife',
        children: [],
      }
    });

    if (!result.data?.success) {
      console.error('Lỗi tạo family:', marriage.id, result.data?.error);
      continue;
    }

    const familyId = result.data.family_id;

    // Tìm con cái của cặp này
    const { data: childRels } = await supabase
      .from('relationships')
      .select('*')
      .in('type', ['biological_child', 'adopted_child'])
      .or(`person_a.eq.${marriage.person_a},person_a.eq.${marriage.person_b}`);

    for (const childRel of childRels ?? []) {
      await supabase.from('family_children').insert({
        family_id:         familyId,
        person_id:         childRel.person_b,
        relationship_type: childRel.type === 'adopted_child' ? 'adopted' : 'biological',
      });
    }

    console.log(`✓ Family ${familyId} tạo xong`);
  }

  console.log('Migration xong. Hãy verify trước khi xóa relationships cũ.');
}

migrateToFamilyModel().catch(console.error);
```

---

## Bước 1.3 — Event Date System

### Phần A — Schema

```sql
-- File: docs/migrations/YYYYMMDD_event_system.sql

-- v2.2.1: Thêm 'range' và 'text' (bổ sung từ v2.1)
CREATE TYPE date_precision_enum AS ENUM (
  'day',      -- 12/03/1945
  'month',    -- 03/1945
  'year',     -- 1945
  'decade',   -- khoảng 1940s
  'range',    -- khoảng 1940–1950
  'text',     -- "Thời Pháp thuộc" (không parse được)
  'unknown'
);

CREATE TYPE date_modifier_enum AS ENUM (
  'exact', 'about', 'before', 'after', 'between', 'estimated', 'calculated'
);

-- Enum thay vì TEXT để tránh lỗi chính tả
CREATE TYPE calendar_type_enum AS ENUM (
  'gregorian', 'lunar', 'text', 'unknown'
);

CREATE TYPE event_type_enum AS ENUM (
  'birth', 'death', 'marriage', 'divorce', 'burial',
  'baptism', 'confirmation', 'ordination', 'graduation',
  'occupation', 'residence', 'migration', 'military',
  'award', 'retirement', 'custom'
);

CREATE TABLE public.events (
  id    UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type  event_type_enum NOT NULL,
  title TEXT,

  -- Ngày tháng
  start_date DATE,     -- Ngày bắt đầu (hoặc ngày duy nhất)
  end_date   DATE,     -- Ngày kết thúc (dùng cho khoảng)
  sort_date  DATE,     -- Ngày dùng để sắp xếp timeline (không hiển thị)
                       -- "Khoảng 1945" → 1945-06-30 (giữa năm)
                       -- "Trước 1975"  → 1974-12-31
                       -- "Sau 1954"    → 1955-01-01

  date_precision     date_precision_enum  DEFAULT 'day',
  date_modifier      date_modifier_enum   DEFAULT 'exact',
  canonical_calendar calendar_type_enum   DEFAULT 'gregorian',
  date_original_text TEXT,  -- Lưu nguyên văn: "Mùng 5 tháng Giêng năm Nhâm Dần"

  -- Âm lịch
  lunar_year          INT,
  lunar_month         INT,
  lunar_day           INT,
  lunar_is_leap_month BOOLEAN DEFAULT FALSE,

  -- Nơi diễn ra
  place_id   UUID,       -- FK tới places (Phase 3)
  place_text TEXT,       -- Địa điểm text tự do (dùng trước khi có places)

  description TEXT,
  family_id   UUID REFERENCES public.families(id),  -- Cho marriage/divorce events

  deleted_at TIMESTAMPTZ,
  deleted_by UUID REFERENCES public.profiles(id),
  version    INT NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE event_role_enum AS ENUM (
  'principal', 'child', 'husband', 'wife',
  'witness', 'officiant', 'deceased', 'participant'
);

CREATE TABLE public.person_events (
  id        UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  person_id UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  event_id  UUID REFERENCES public.events(id)  ON DELETE CASCADE NOT NULL,
  role      event_role_enum DEFAULT 'principal',
  UNIQUE(person_id, event_id, role)
);
```

### Phần B — Age Calculation (v2.2.1: sửa bug logic)

> **Bug trong v2.1:** Điều kiện `if` quá rộng — case 'month' bị lọt vào block 'year_only'. Đã sửa bằng cách kiểm tra `date_precision` tường minh.

```typescript
// utils/calendar/ageCalculation.ts

export type AgePrecision = 'exact' | 'year_only' | 'partial' | 'unknown';

export interface AgeResult {
  minAge:       number | null;
  maxAge:       number | null;
  precision:    AgePrecision;
  display:      string;       // "khoảng 55–56 tuổi"
  displayShort: string;       // "~55–56 tuổi"
}

/**
 * Tính tuổi từ start_date/end_date của events (không dùng year/month/day nữa).
 *
 * Sinh 12/03/1945, Mất 05/01/2001 → "55 tuổi"        (chính xác)
 * Sinh 1945,       Mất 2001       → "khoảng 55–56 tuổi" (chỉ biết năm)
 * Sinh 03/1945,    Mất 2001       → "khoảng 55–56 tuổi" (thiếu ngày)
 */
export function calculateAgeFromEvents(
  birthEvent: {
    start_date:     string | null;
    end_date:       string | null;
    date_precision: string;
  } | null,
  deathEvent: {
    start_date:     string | null;
    end_date:       string | null;
    date_precision: string;
  } | null
): AgeResult {
  const unknown: AgeResult = {
    minAge: null, maxAge: null,
    precision: 'unknown', display: '', displayShort: ''
  };

  if (!birthEvent?.start_date) return unknown;

  const birthStart = new Date(birthEvent.start_date);
  const birthEnd   = birthEvent.end_date ? new Date(birthEvent.end_date) : birthStart;

  const now      = new Date();
  const refStart = deathEvent?.start_date ? new Date(deathEvent.start_date) : now;
  const refEnd   = deathEvent?.end_date   ? new Date(deathEvent.end_date)   : now;

  // Tách 3 case TƯỜNG MINH — không overlap nhau
  const birthIsDay   = birthEvent.date_precision === 'day';
  const deathIsDay   = !deathEvent || deathEvent.date_precision === 'day';
  const birthIsYear  = ['year', 'decade', 'unknown'].includes(birthEvent.date_precision);
  const deathIsYear  = deathEvent
    ? ['year', 'decade', 'unknown'].includes(deathEvent.date_precision)
    : false;

  // CASE 1: Cả hai chính xác đến ngày → kết quả chính xác
  if (birthIsDay && deathIsDay) {
    const age = calcAge(birthStart, refStart);
    return {
      minAge: age, maxAge: age,
      precision: 'exact',
      display: `${age} tuổi`,
      displayShort: `${age} tuổi`
    };
  }

  // CASE 2: Ít nhất 1 bên chỉ biết năm → khoảng ±1
  if (birthIsYear || deathIsYear) {
    const minAge = calcAge(birthEnd,   refStart);  // sinh cuối năm, mất đầu năm
    const maxAge = calcAge(birthStart, refEnd);    // sinh đầu năm, mất cuối năm
    const display = minAge === maxAge
      ? `khoảng ${minAge} tuổi`
      : `khoảng ${minAge}–${maxAge} tuổi`;
    return {
      minAge, maxAge, precision: 'year_only',
      display, displayShort: `~${minAge}–${maxAge} tuổi`
    };
  }

  // CASE 3: Biết tháng/năm (precision = 'month') → partial
  const minAge = calcAge(birthEnd,   refStart);
  const maxAge = calcAge(birthStart, refEnd);
  return {
    minAge, maxAge, precision: 'partial',
    display: `khoảng ${minAge === maxAge ? minAge : `${minAge}–${maxAge}`} tuổi`,
    displayShort: `~${Math.round((minAge + maxAge) / 2)} tuổi`
  };
}

function calcAge(birthDate: Date, refDate: Date): number {
  let age = refDate.getFullYear() - birthDate.getFullYear();
  const m = refDate.getMonth() - birthDate.getMonth();
  if (m < 0 || (m === 0 && refDate.getDate() < birthDate.getDate())) age--;
  return Math.max(0, age);
}

/** Hiển thị ngắn: "~1945 – 2001 (~55–56 tuổi)" */
export function formatLifespan(
  birthEvent: Parameters<typeof calculateAgeFromEvents>[0],
  deathEvent: Parameters<typeof calculateAgeFromEvents>[1],
  isLiving: boolean
): string {
  if (!birthEvent?.start_date) return '';
  const bYear = new Date(birthEvent.start_date).getFullYear();
  const bApprox = birthEvent.date_precision !== 'day' ? '~' : '';
  const dStr = isLiving ? 'nay'
    : deathEvent?.start_date
      ? `${deathEvent.date_precision !== 'day' ? '~' : ''}${new Date(deathEvent.start_date).getFullYear()}`
      : '?';
  const age = calculateAgeFromEvents(birthEvent, deathEvent);
  const ageStr = age.precision !== 'unknown' ? ` (${age.displayShort})` : '';
  return `${bApprox}${bYear} – ${dStr}${ageStr}`;
}
```

**Test cases:**

```typescript
// tests/date/ageCalculation.test.ts

import { calculateAgeFromEvents } from '@/utils/calendar/ageCalculation';
import { describe, it, expect } from 'vitest';

describe('calculateAgeFromEvents', () => {

  it('Biết đủ ngày/tháng/năm → chính xác', () => {
    const r = calculateAgeFromEvents(
      { start_date: '1945-03-12', end_date: '1945-03-12', date_precision: 'day' },
      { start_date: '2001-01-05', end_date: '2001-01-05', date_precision: 'day' }
    );
    expect(r.precision).toBe('exact');
    expect(r.minAge).toBe(55);
    expect(r.display).toBe('55 tuổi');
    // Sinh 12/03/1945, mất 05/01/2001 → chưa qua sinh nhật → 55 tuổi
  });

  it('Chỉ biết năm → khoảng 55–56 tuổi', () => {
    const r = calculateAgeFromEvents(
      { start_date: '1945-01-01', end_date: '1945-12-31', date_precision: 'year' },
      { start_date: '2001-01-01', end_date: '2001-12-31', date_precision: 'year' }
    );
    expect(r.precision).toBe('year_only');
    expect(r.minAge).toBe(55);
    expect(r.maxAge).toBe(56);
    expect(r.display).toBe('khoảng 55–56 tuổi');
  });

  it('Biết tháng/năm → partial', () => {
    const r = calculateAgeFromEvents(
      { start_date: '1945-03-01', end_date: '1945-03-31', date_precision: 'month' },
      { start_date: '2001-01-01', end_date: '2001-01-31', date_precision: 'month' }
    );
    expect(r.precision).toBe('partial');
    expect(r.display).toContain('khoảng');
  });

  it('Không có năm sinh → unknown', () => {
    const r = calculateAgeFromEvents(null, null);
    expect(r.precision).toBe('unknown');
    expect(r.display).toBe('');
  });

  it('Còn sống → tuổi so với hiện tại', () => {
    const currentYear = new Date().getFullYear(); // 2026
    const r = calculateAgeFromEvents(
      { start_date: '1945-01-01', end_date: '1945-12-31', date_precision: 'year' },
      null
    );
    expect(r.minAge).toBe(currentYear - 1945 - 1);
    expect(r.maxAge).toBe(currentYear - 1945);
  });

});
```

### Phần C — Date Parser

```typescript
// utils/date-parser/parseVietnameseDate.ts

interface ParseResult {
  success:      boolean;
  year?:        number;
  month?:       number;
  day?:         number;
  precision:    'day' | 'month' | 'year' | 'unknown';
  modifier:     'exact' | 'about' | 'before' | 'after' | 'unknown';
  originalText: string;
  isLunar:      boolean;
}

export function parseVietnameseDate(input: string): ParseResult {
  const text = input.trim();

  // "khoảng 1945", "~1945", "khoảng năm 1945"
  const about = text.match(/^(?:khoảng|~)\s*(?:năm\s+)?(\d{4})$/i);
  if (about) return { success: true, year: +about[1], precision: 'year', modifier: 'about', originalText: text, isLunar: false };

  // "trước 1975", "trước năm 1975"
  const before = text.match(/^trước\s+(?:năm\s+)?(\d{4})$/i);
  if (before) return { success: true, year: +before[1], precision: 'year', modifier: 'before', originalText: text, isLunar: false };

  // "sau 1954", "sau năm 1954"
  const after = text.match(/^sau\s+(?:năm\s+)?(\d{4})$/i);
  if (after) return { success: true, year: +after[1], precision: 'year', modifier: 'after', originalText: text, isLunar: false };

  // Chỉ năm: "1945"
  const yearOnly = text.match(/^(\d{4})$/);
  if (yearOnly) return { success: true, year: +yearOnly[1], precision: 'year', modifier: 'exact', originalText: text, isLunar: false };

  // Tháng/Năm: "03/1945"
  const monthYear = text.match(/^(\d{1,2})\/(\d{4})$/);
  if (monthYear) return { success: true, year: +monthYear[2], month: +monthYear[1], precision: 'month', modifier: 'exact', originalText: text, isLunar: false };

  // Ngày đầy đủ: "12/03/1945"
  const full = text.match(/^(\d{1,2})[\/\-](\d{1,2})[\/\-](\d{4})$/);
  if (full) return { success: true, day: +full[1], month: +full[2], year: +full[3], precision: 'day', modifier: 'exact', originalText: text, isLunar: false };

  // Âm lịch: "Mùng 5 tháng Giêng năm 1945"
  const lunar = text.match(/^(?:mùng|ngày)\s+(\d{1,2})\s+tháng\s+(giêng|chạp|\d{1,2})(?:\s+năm\s+(.+))?$/i);
  if (lunar) {
    const months: Record<string, number> = { giêng: 1, chạp: 12 };
    const month = months[lunar[2].toLowerCase()] ?? +lunar[2];
    return { success: true, day: +lunar[1], month, precision: lunar[3] ? 'day' : 'month', modifier: 'exact', originalText: text, isLunar: true };
  }

  // Không parse được → lưu vào date_original_text với precision='text'
  return { success: false, precision: 'unknown', modifier: 'unknown', originalText: text, isLunar: false };
}
```

### Phần D — Can Chi (tính runtime, không lưu DB)

```typescript
// utils/calendar/canChi.ts

const CAN = ['Giáp','Ất','Bính','Đinh','Mậu','Kỷ','Canh','Tân','Nhâm','Quý'];
const CHI = ['Tý','Sửu','Dần','Mão','Thìn','Tỵ','Ngọ','Mùi','Thân','Dậu','Tuất','Hợi'];

export function getCanChiYear(year: number): string {
  // 1902 → "Nhâm Dần", 2024 → "Giáp Thìn"
  return `${CAN[(year - 4) % 10]} ${CHI[(year - 4) % 12]}`;
}

export function getCanChiMonth(year: number, month: number): string {
  const idx = ((year % 5) * 2 + (month - 1)) % 10;
  return `${CAN[idx]} ${CHI[(month + 1) % 12]}`;
}

export function getCanChiDay(year: number, month: number, day: number): string {
  const a   = Math.floor((14 - month) / 12);
  const y   = year + 4800 - a;
  const m   = month + 12 * a - 3;
  const jdn = day + Math.floor((153*m+2)/5) + 365*y
    + Math.floor(y/4) - Math.floor(y/100) + Math.floor(y/400) - 32045;
  return `${CAN[jdn % 10]} ${CHI[jdn % 12]}`;
}

export function formatLunarFull(day: number, month: number, year: number, isLeap: boolean): string {
  const dayStr  = day <= 10 ? `Mùng ${day}` : `${day}`;
  const mStr    = month === 1 ? 'Giêng' : month === 12 ? 'Chạp' : `${month}`;
  const leapStr = isLeap ? ' nhuận' : '';
  return `${dayStr} tháng ${mStr}${leapStr} năm ${getCanChiYear(year)}`;
}
```

### Phần E — Migration birth/death → events (thứ tự an toàn)

```typescript
// scripts/migrate-dates-to-events.ts

async function migrateDatesToEvents() {
  const { data: persons } = await supabase.from('persons').select('*');

  for (const p of persons ?? []) {

    // --- MIGRATE NGÀY SINH ---
    if (p.birth_year) {
      const precision = p.birth_day ? 'day' : p.birth_month ? 'month' : 'year';
      let startDate: string, endDate: string, sortDate: string;

      if (p.birth_year && p.birth_month && p.birth_day) {
        const d = `${p.birth_year}-${pad(p.birth_month)}-${pad(p.birth_day)}`;
        startDate = endDate = sortDate = d;
      } else if (p.birth_year && p.birth_month) {
        startDate = `${p.birth_year}-${pad(p.birth_month)}-01`;
        endDate   = lastDay(p.birth_year, p.birth_month);
        sortDate  = `${p.birth_year}-${pad(p.birth_month)}-15`;
      } else {
        startDate = `${p.birth_year}-01-01`;
        endDate   = `${p.birth_year}-12-31`;
        sortDate  = `${p.birth_year}-06-30`;
      }

      const { data: evt } = await supabase.from('events').insert({
        type: 'birth', start_date: startDate, end_date: endDate,
        sort_date: sortDate, date_precision: precision,
        date_modifier: 'exact', canonical_calendar: 'gregorian',
      }).select().single();

      if (evt) {
        await supabase.from('person_events').insert({
          person_id: p.id, event_id: evt.id, role: 'child'
        });
      }
    }

    // --- MIGRATE NGÀY MẤT ---
    if (p.is_deceased && p.death_year) {
      const precision = p.death_day ? 'day' : p.death_month ? 'month' : 'year';
      const hasLunar  = !!(p.death_lunar_year || p.death_lunar_month || p.death_lunar_day);

      const startDate = p.death_year && p.death_month && p.death_day
        ? `${p.death_year}-${pad(p.death_month)}-${pad(p.death_day)}`
        : `${p.death_year}-01-01`;
      const endDate   = p.death_year && p.death_month && p.death_day
        ? startDate
        : `${p.death_year}-12-31`;
      const sortDate  = startDate;

      const { data: evt } = await supabase.from('events').insert({
        type: 'death', start_date: startDate, end_date: endDate,
        sort_date: sortDate, date_precision: precision,
        date_modifier: 'exact',
        lunar_year:          p.death_lunar_year   ?? null,
        lunar_month:         p.death_lunar_month  ?? null,
        lunar_day:           p.death_lunar_day    ?? null,
        lunar_is_leap_month: false,
        canonical_calendar:  hasLunar ? 'lunar' : 'gregorian',
      }).select().single();

      if (evt) {
        await supabase.from('person_events').insert({
          person_id: p.id, event_id: evt.id, role: 'deceased'
        });
      }
    }
  }
  console.log('Migration ngày sinh/mất xong.');
}

function pad(n: number) { return String(n).padStart(2, '0'); }
function lastDay(y: number, m: number) {
  return new Date(y, m, 0).toISOString().split('T')[0];
}
```

### Phần F — Person Model Cleanup (thứ tự an toàn)

> **v2.2.1: Không drop `is_deceased` trước khi đã copy sang `living`.**

```sql
-- File: docs/migrations/YYYYMMDD_person_cleanup.sql
-- Chạy SAU KHI migrate và verify đã xong

-- Bước 1: Thêm cột mới trước
ALTER TABLE public.persons ADD COLUMN IF NOT EXISTS living BOOLEAN;

-- Bước 2: Copy data (is_deceased vẫn còn ở đây)
UPDATE public.persons SET living = NOT COALESCE(is_deceased, false);

-- Bước 3: Đặt NOT NULL và default
ALTER TABLE public.persons
  ALTER COLUMN living SET NOT NULL,
  ALTER COLUMN living SET DEFAULT TRUE;

-- Bước 4: VERIFY — phải trả về 0 trước khi đi tiếp
-- SELECT COUNT(*) FROM persons WHERE living IS NULL;

-- Bước 5: Xóa cột cũ (CHỈ sau khi verify bước 4 = 0)
ALTER TABLE public.persons
  DROP COLUMN IF EXISTS is_deceased,
  DROP COLUMN IF EXISTS birth_year, DROP COLUMN IF EXISTS birth_month,
  DROP COLUMN IF EXISTS birth_day,
  DROP COLUMN IF EXISTS death_year, DROP COLUMN IF EXISTS death_month,
  DROP COLUMN IF EXISTS death_day,
  DROP COLUMN IF EXISTS death_lunar_year, DROP COLUMN IF EXISTS death_lunar_month,
  DROP COLUMN IF EXISTS death_lunar_day;
```

---


---

> **⏩ Tiếp theo: Phần 2/2** — Phase 2 (Media & Source), Phase 3 (Place & Audit),
> Phase 4 (GEDCOM & Dedup), Phase 5 (UI/UX), Phase 6 (Production),
> Tổng hợp files, Sprint plan 22 tuần, Kết quả kỳ vọng.
