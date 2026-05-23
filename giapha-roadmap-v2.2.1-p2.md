# GIAPHA-OS — ROADMAP v2.2.1 · PHẦN 2/2
### Media, Source, GEDCOM, UI, Production & Kế hoạch triển khai
> **Phần 2 gồm:** Phase 2–6 · GEDCOM Import/Export · Vietnamese UI · Production · Sprint plan · Kết quả kỳ vọng
> **Phần 1 gồm:** Thay đổi so với v2.2 · Kiến trúc · Phase 0 (Nền móng) · Phase 1 (Data model)

---

# PHASE 2 — MEDIA & SOURCE (Tuần 9–11)

---

## Bước 2.1 — Media System

```sql
-- File: docs/migrations/YYYYMMDD_media_system.sql

CREATE TABLE public.media (
  id                UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type              TEXT NOT NULL DEFAULT 'photo'
    CHECK (type IN ('photo','document','video','audio','other')),
  mime_type         TEXT,
  storage_path      TEXT NOT NULL,
  thumbnail_path    TEXT,
  original_filename TEXT,
  file_size_bytes   INT,
  title             TEXT,
  description       TEXT,
  date_taken        DATE,
  privacy_level     TEXT NOT NULL DEFAULT 'family'
    CHECK (privacy_level IN ('public','family','private')),
  uploaded_by       UUID REFERENCES public.profiles(id),
  deleted_at        TIMESTAMPTZ,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.person_media (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  person_id  UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  media_id   UUID REFERENCES public.media(id)   ON DELETE CASCADE NOT NULL,
  is_primary BOOLEAN DEFAULT FALSE,
  caption    TEXT,
  sort_order INT DEFAULT 0,
  UNIQUE(person_id, media_id)
);

CREATE TABLE public.event_media (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  event_id   UUID REFERENCES public.events(id) ON DELETE CASCADE NOT NULL,
  media_id   UUID REFERENCES public.media(id)  ON DELETE CASCADE NOT NULL,
  caption    TEXT,
  sort_order INT DEFAULT 0,
  UNIQUE(event_id, media_id)
);

CREATE TABLE public.family_media (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  family_id  UUID REFERENCES public.families(id) ON DELETE CASCADE NOT NULL,
  media_id   UUID REFERENCES public.media(id)    ON DELETE CASCADE NOT NULL,
  caption    TEXT,
  sort_order INT DEFAULT 0,
  UNIQUE(family_id, media_id)
);
```

**Migration avatar_url cũ:**

```sql
-- Chuyển avatar_url → media + person_media
INSERT INTO public.media (type, storage_path, title, privacy_level)
SELECT 'photo', avatar_url, 'Ảnh đại diện', 'family'
FROM public.persons WHERE avatar_url IS NOT NULL;
-- Sau đó link person_media.is_primary = true (viết script riêng)
```

---

## Bước 2.2 — Source & Citation System

```sql
-- File: docs/migrations/YYYYMMDD_source_system.sql

CREATE TABLE public.sources (
  id               UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type             TEXT NOT NULL DEFAULT 'document'
    CHECK (type IN ('document','oral','photo','record','book','website','other')),
  title            TEXT NOT NULL,
  author           TEXT,
  publisher        TEXT,
  publication_year INT,
  repository       TEXT,
  url              TEXT,
  description      TEXT,
  media_id         UUID REFERENCES public.media(id),
  privacy_level    TEXT NOT NULL DEFAULT 'family'
    CHECK (privacy_level IN ('public','family','private')),
  deleted_at       TIMESTAMPTZ,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.citations (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  source_id   UUID REFERENCES public.sources(id) ON DELETE CASCADE NOT NULL,
  page        TEXT,
  confidence  TEXT DEFAULT 'medium'
    CHECK (confidence IN ('high','medium','low','unknown')),
  quote       TEXT,
  note        TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.citation_links (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  citation_id UUID REFERENCES public.citations(id) ON DELETE CASCADE NOT NULL,
  entity_type TEXT NOT NULL
    CHECK (entity_type IN ('person','event','family','person_name')),
  entity_id   UUID NOT NULL,
  field_name  TEXT,   -- NULL = nguồn cho cả entity. Ví dụ: 'birth_date', 'occupation'
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Bước 2.3 — Privacy Model và RLS

```
Rule theo privacy_level:
┌───────────────┬──────────┬────────────┬──────────────────┐
│ Privacy       │ public   │ member     │ editor/admin     │
├───────────────┼──────────┼────────────┼──────────────────┤
│ public        │ ✅ Thấy  │ ✅ Thấy    │ ✅ Thấy + sửa   │
│ family        │ ❌ Ẩn    │ ✅ Thấy    │ ✅ Thấy + sửa   │
│ private       │ ❌ Ẩn    │ ❌ Ẩn      │ ✅ Thấy + sửa   │
└───────────────┴──────────┴────────────┴──────────────────┘

Mặc định:
  - Người còn sống (living = true) → privacy = 'family'
  - Tài liệu, giấy tờ             → privacy = 'private'
  - Ảnh chân dung                  → privacy = 'family'
  - Nguồn lời kể miệng (oral)      → privacy = 'private'
```

```sql
CREATE POLICY "Media privacy" ON public.media FOR SELECT USING (
  privacy_level = 'public'
  OR (
    privacy_level = 'family'
    AND auth.uid() IS NOT NULL
    AND EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND is_active = TRUE
    )
  )
  OR public.is_admin()
  OR public.is_editor()
);
```

---

# PHASE 3 — PLACE & AUDIT (Tuần 12–13)

---

## Bước 3.1 — Place System

```sql
-- File: docs/migrations/YYYYMMDD_place_system.sql

CREATE TABLE public.places (
  id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name            TEXT NOT NULL,
  name_search     TEXT,   -- Không dấu, dùng cho search
  type            TEXT
    CHECK (type IN ('country','province','district','ward',
                    'commune','village','hamlet',
                    'cemetery','temple','church','hospital','other')),
  parent_place_id UUID REFERENCES public.places(id),
  latitude        DECIMAL(10, 8),
  longitude       DECIMAL(11, 8),
  description     TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.place_aliases (
  id           UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  place_id     UUID REFERENCES public.places(id) ON DELETE CASCADE NOT NULL,
  alias        TEXT NOT NULL,         -- "Sài Gòn", "Gia Định"
  alias_search TEXT,                  -- "sai gon"
  period_note  TEXT,                  -- "Tên dùng trước 1975"
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Bước 3.2 — Search Architecture

> **v2.2.1: Thêm migration đầy đủ cho `remove_diacritics`** (thiếu trong v2.1)

```sql
-- File: docs/migrations/YYYYMMDD_search_setup.sql

-- Bước 1: Cài extension (có sẵn trong Supabase)
CREATE EXTENSION IF NOT EXISTS unaccent;

-- Bước 2: Tạo function bỏ dấu
CREATE OR REPLACE FUNCTION public.remove_diacritics(input TEXT)
RETURNS TEXT AS $$
  SELECT lower(unaccent(input));
$$ LANGUAGE sql IMMUTABLE;

-- Test ngay: SELECT public.remove_diacritics('Nguyễn Văn Ánh');
-- Kết quả phải là: 'nguyen van anh'

-- Fallback TypeScript nếu unaccent không dùng được:
-- utils/text/removeDiacritics.ts
-- export function removeDiacritics(text: string): string {
--   return text.normalize('NFD')
--     .replace(/[\u0300-\u036f]/g, '')
--     .replace(/đ/g, 'd').replace(/Đ/g, 'd')
--     .toLowerCase();
-- }
```

**Materialized view tìm kiếm:**

```sql
CREATE MATERIALIZED VIEW public.person_search_index AS
SELECT
  p.id,
  pn.full_text                                    AS primary_name,
  public.remove_diacritics(pn.full_text)          AS primary_name_search,
  STRING_AGG(DISTINCT pn2.full_text, ' ')         AS all_names,
  STRING_AGG(DISTINCT public.remove_diacritics(pn2.full_text), ' ') AS all_names_search,
  EXTRACT(YEAR FROM be.start_date)::INT           AS birth_year,
  EXTRACT(YEAR FROM de.start_date)::INT           AS death_year,
  p.generation, p.gender, p.living
FROM public.persons p
LEFT JOIN public.person_names pn   ON pn.person_id = p.id AND pn.is_primary = TRUE  AND pn.deleted_at IS NULL
LEFT JOIN public.person_names pn2  ON pn2.person_id = p.id AND pn2.deleted_at IS NULL
LEFT JOIN public.person_events pe1 ON pe1.person_id = p.id
LEFT JOIN public.events be         ON be.id = pe1.event_id AND be.type = 'birth'    AND be.deleted_at IS NULL
LEFT JOIN public.person_events pe2 ON pe2.person_id = p.id
LEFT JOIN public.events de         ON de.id = pe2.event_id AND de.type = 'death'    AND de.deleted_at IS NULL
WHERE p.deleted_at IS NULL
GROUP BY p.id, pn.full_text, be.start_date, de.start_date, p.generation, p.gender, p.living;

CREATE INDEX ON public.person_search_index
  USING gin(to_tsvector('simple', all_names_search));
```

---

## Bước 3.3 — Audit Log

```sql
-- File: docs/migrations/YYYYMMDD_audit_log.sql

CREATE TABLE public.audit_logs (
  id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  table_name TEXT NOT NULL,
  record_id  UUID NOT NULL,
  action     TEXT NOT NULL CHECK (action IN ('CREATE','UPDATE','DELETE','RESTORE')),
  changed_by UUID REFERENCES public.profiles(id),
  old_data   JSONB,
  new_data   JSONB,
  changed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_person_changes()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.audit_logs (table_name, record_id, action, changed_by, old_data, new_data)
  VALUES (
    'persons',
    COALESCE(NEW.id, OLD.id),
    CASE TG_OP WHEN 'INSERT' THEN 'CREATE' WHEN 'UPDATE' THEN 'UPDATE' ELSE 'DELETE' END,
    auth.uid(),
    CASE WHEN TG_OP != 'INSERT' THEN TO_JSONB(OLD) ELSE NULL END,
    CASE WHEN TG_OP != 'DELETE' THEN TO_JSONB(NEW) ELSE NULL END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER audit_persons AFTER INSERT OR UPDATE OR DELETE ON public.persons
  FOR EACH ROW EXECUTE FUNCTION log_person_changes();
-- Tương tự cho families và events
```

---

# PHASE 4 — GEDCOM & DEDUP (Tuần 14–16)

> **⚡ Quan trọng:** Bước 4.1 (GEDCOM exporter) **có thể làm sớm hơn** — ngay cả trước Phase 1 — vì chỉ cần sửa `utils/gedcom.ts` hiện tại, không phụ thuộc vào Family Model hay Event System.

---

## Bước 4.1a — Tách GEDCOM module

**Tạo thư mục `utils/gedcom/`:**

```
utils/gedcom/
├── index.ts           ← Export API công khai
├── parser.ts          ← Parse .ged → objects (sẽ cập nhật ở 4.1e)
├── mapper.ts          ← Map GEDCOM → giapha schema
├── exporter.ts        ← ★ File quan trọng nhất — viết mới hoàn toàn ở 4.1b
├── validator.ts       ← Kiểm tra file GEDCOM
└── compat/
    ├── familygem.ts   ← Fix quirks của FamilyGem
    ├── gramps.ts      ← Fix quirks của Gramps
    └── ancestry.ts    ← Fix quirks của Ancestry.com
```

**Mapping GEDCOM ↔ giapha:**
```
GEDCOM tag  → Bảng trong giapha
──────────────────────────────────────────────
INDI        → persons + person_names
FAM         → families + family_parents + family_children
HUSB / WIFE → family_parents (role = husband/wife)
CHIL        → family_children (default biological)
BIRT        → events (type=birth) + person_events
DEAT        → events (type=death) + person_events
MARR        → events (type=marriage, family_id = family.id)
BURI        → events (type=burial)
OCCU        → events (type=occupation)
RESI        → events (type=residence)
PLAC        → places (parse "Làng, Huyện, Tỉnh, VN")
ABT 1945    → date_modifier='about',   date_precision='year'
BEF 1975    → date_modifier='before',  date_precision='year'
AFT 1954    → date_modifier='after',   date_precision='year'
BET X AND Y → date_modifier='between', date_precision='range'
EST 1945    → date_modifier='estimated'
CAL 1902    → date_modifier='calculated'
_LUNAR      → lunar_year, lunar_month, lunar_day
_LUNAR_LEAP → lunar_is_leap_month
```

---

## Bước 4.1b — ★ GEDCOM Exporter mới (Unicode chuẩn + sửa bug họ/tên)

> **Đây là file quan trọng nhất của Phase 4.** Thay thế toàn bộ `exportToGedcom()` trong `utils/gedcom.ts` hiện tại.

### Bug cần sửa trước khi viết code mới

**Code cũ đang sai họ/tên:**
```typescript
// BUG trong utils/gedcom.ts hiện tại:
const parts = person.full_name.trim().split(" ");
const lastName = parts.length > 1 ? parts.pop() : "";  // ← lấy từ CUỐI
const firstName = parts.join(" ");
// Kết quả: "Nguyễn Văn An" → export "Nguyễn Văn /An/" ← SAI!
// Đúng phải là: "Văn An /Nguyễn/"
```

**Quy tắc tên Việt Nam trong GEDCOM:**
- Họ = **từ đầu tiên**: `Nguyễn`, `Trần`, `Lê`
- Phần còn lại = tên + đệm: `Văn An`, `Thị Hoa`
- Format GEDCOM: `GivenName /Surname/` → `Văn An /Nguyễn/`

### Toàn bộ code `utils/gedcom/exporter.ts`

```typescript
// utils/gedcom/exporter.ts
// Thay thế exportToGedcom() trong utils/gedcom.ts

// ════════════════════════════════════════
// TYPES
// ════════════════════════════════════════

export interface ExportPerson {
  id:         string;
  full_name:  string | null;
  surname?:   string | null;     // Họ (nếu đã tách sẵn từ person_names)
  given_name?: string | null;    // Tên + đệm
  gender:     'male' | 'female' | 'other';
  note?:      string | null;

  // Ngày sinh
  birth_year?:          number | null;
  birth_month?:         number | null;
  birth_day?:           number | null;
  birth_date_modifier?: string | null;   // 'about' | 'before' | 'after' | 'exact'
  birth_place_text?:    string | null;
  birth_lunar_year?:    number | null;
  birth_lunar_month?:   number | null;
  birth_lunar_day?:     number | null;
  birth_lunar_is_leap?: boolean;

  // Ngày mất
  is_deceased?:         boolean;
  death_year?:          number | null;
  death_month?:         number | null;
  death_day?:           number | null;
  death_date_modifier?: string | null;
  death_place_text?:    string | null;
  death_lunar_year?:    number | null;
  death_lunar_month?:   number | null;
  death_lunar_day?:     number | null;
  death_lunar_is_leap?: boolean;
}

export interface ExportRelationship {
  type:     string;     // 'marriage' | 'biological_child' | 'adopted_child'
  person_a: string;     // UUID
  person_b: string;     // UUID
}

// ════════════════════════════════════════
// CONSTANTS
// ════════════════════════════════════════

const GEDCOM_MONTHS = [
  'JAN','FEB','MAR','APR','MAY','JUN',
  'JUL','AUG','SEP','OCT','NOV','DEC'
];

const DATE_MODIFIER_MAP: Record<string, string> = {
  'about':       'ABT',
  'approximate': 'ABT',
  'before':      'BEF',
  'after':       'AFT',
  'estimated':   'EST',
  'calculated':  'CAL',
  'exact':       '',
  '':            '',
};

const CRLF = '\r\n';  // GEDCOM 5.5.1 dùng CRLF

// ════════════════════════════════════════
// HELPER FUNCTIONS
// ════════════════════════════════════════

/** BOM UTF-8 — 3 bytes EF BB BF, phải là ký tự đầu tiên của file */
function utf8Bom(): string {
  return '\uFEFF';
}

/** Escape @ trong giá trị GEDCOM thành @@ */
function escapeGedcomValue(value: string): string {
  return value.replace(/@/g, '@@');
}

/**
 * Tách họ và tên theo quy tắc Việt Nam.
 *
 * Quy tắc: Họ = từ ĐẦU TIÊN (không phải cuối cùng!)
 * "Nguyễn Văn An" → surname="Nguyễn", givenName="Văn An"
 * "Lê Thị Hoa"   → surname="Lê",      givenName="Thị Hoa"
 * "Madonna"       → surname="",        givenName="Madonna"
 *
 * GEDCOM format: "GivenName /Surname/"
 * Ví dụ: "Văn An /Nguyễn/"
 */
function splitVietnameseName(
  fullName:   string,
  surname?:   string | null,
  givenName?: string | null
): { surname: string; givenName: string } {
  // Ưu tiên dùng surname/givenName đã tách từ person_names table
  if (surname?.trim()) {
    return {
      surname:   surname.trim(),
      givenName: (givenName ?? '').trim(),
    };
  }

  const parts = fullName.trim().split(/\s+/).filter(Boolean);

  if (parts.length === 0) return { surname: 'Unknown', givenName: 'Unknown' };
  if (parts.length === 1) return { surname: '',         givenName: parts[0] };

  // Họ = từ ĐẦU TIÊN, phần còn lại = tên + đệm
  return {
    surname:   parts[0],
    givenName: parts.slice(1).join(' '),
  };
}

/**
 * Format ngày theo chuẩn GEDCOM.
 *
 * Ví dụ output:
 *   "18 MAR 1902"   ← ngày đầy đủ
 *   "MAR 1902"      ← chỉ tháng/năm
 *   "1945"          ← chỉ năm
 *   "ABT 1945"      ← khoảng năm
 *   "BEF 1975"      ← trước năm
 *   "AFT 1954"      ← sau năm
 */
function formatGedcomDate(
  year?:     number | null,
  month?:    number | null,
  day?:      number | null,
  modifier?: string | null
): string | null {
  if (!year) return null;

  const prefix  = modifier ? (DATE_MODIFIER_MAP[modifier] ?? '') : '';
  const parts: string[] = [];

  if (day  && day  > 0)           parts.push(String(day).padStart(2, '0'));
  if (month && month >= 1 && month <= 12) parts.push(GEDCOM_MONTHS[month - 1]);
  parts.push(String(year));

  const dateStr = parts.join(' ');
  return prefix ? `${prefix} ${dateStr}` : dateStr;
}

/**
 * Format text dài thành nhiều dòng GEDCOM (dùng CONT cho dòng tiếp theo).
 * GEDCOM 5.5.1 giới hạn 255 ký tự/dòng.
 */
function formatGedcomText(level: number, tag: string, value: string): string {
  if (!value?.trim()) return '';
  const lines = value.split('\n');
  let result = `${level} ${tag} ${escapeGedcomValue(lines[0])}${CRLF}`;
  for (let i = 1; i < lines.length; i++) {
    result += `${level + 1} CONT ${escapeGedcomValue(lines[i])}${CRLF}`;
  }
  return result;
}

/** Format ngày hiện tại cho GEDCOM header */
function formatExportDate(): string {
  const now   = new Date();
  const day   = String(now.getDate()).padStart(2, '0');
  const month = GEDCOM_MONTHS[now.getMonth()];
  const year  = now.getFullYear();
  return `${day} ${month} ${year}`;
}

// ════════════════════════════════════════
// MAIN EXPORT FUNCTION
// ════════════════════════════════════════

/**
 * Export gia phả thành file GEDCOM chuẩn UTF-8.
 *
 * Tương thích: FamilyGem, Gramps, webtrees, Ancestry.com, Legacy Family Tree, RootsMagic.
 *
 * Custom tags (data Việt Nam):
 *   _GIAPHA_VER    — Version roadmap
 *   _LUNAR         — Ngày âm lịch (format: DD/MM/YYYY)
 *   _LUNAR_LEAP    — Tháng nhuận (Y/N)
 *
 * @returns string UTF-8 với BOM + CRLF
 */
export function exportToGedcom(data: {
  persons:       ExportPerson[];
  relationships: ExportRelationship[];
  version?:      string;
}): string {
  // Dùng mảng rồi join — nhanh hơn string concatenation
  const lines: string[] = [];

  // Helper: thêm dòng kèm CRLF
  const L = (text: string) => lines.push(text + CRLF);

  // ──────────────────────────────────────────────
  // BOM + HEADER
  // ──────────────────────────────────────────────
  lines.push(utf8Bom());  // BOM phải là ký tự đầu tiên, KHÔNG có CRLF

  L('0 HEAD');
  L('1 GEDC');
  L('2 VERS 5.5.1');          // 5.5.1 — tương thích rộng nhất
  L('1 CHAR UTF-8');          // ← KHAI BÁO ENCODING — quan trọng nhất
  L('1 SOUR GIAPHA_OS');
  L('2 NAME Giapha OS');
  L(`2 VERS ${data.version ?? '2.2.1'}`);
  L('1 LANG Vietnamese');
  L(`1 DATE ${formatExportDate()}`);
  L(`1 _GIAPHA_VER ${data.version ?? '2.2.1'}`);

  // ──────────────────────────────────────────────
  // MAP UUID → GEDCOM XREF ngắn (tối đa 20 ký tự theo spec)
  // UUID dài 36 ký tự, phải chuyển thành I1, I2, I3...
  // ──────────────────────────────────────────────
  const personXrefMap = new Map<string, string>();
  let pCounter = 1;
  for (const p of data.persons) {
    if (p.id) personXrefMap.set(p.id, `I${pCounter++}`);
  }

  const getXref = (id: string) => {
    const x = personXrefMap.get(id);
    return x ? `@${x}@` : '@UNKNOWN@';
  };

  // ──────────────────────────────────────────────
  // XÂY DỰNG FAMILIES từ relationships
  // ──────────────────────────────────────────────
  const personMap = new Map(data.persons.map(p => [p.id, p]));

  interface GedcomFamily {
    id:       string;
    husbId?:  string;
    wifeId?:  string;
    children: string[];
  }

  const families:               GedcomFamily[]         = [];
  const personToFamiliesParent: Map<string, string[]>  = new Map();
  let fCounter = 1;

  // Tạo family từ marriage relationships
  for (const rel of data.relationships) {
    if (rel.type !== 'marriage' || !rel.person_a || !rel.person_b) continue;

    const pA = personMap.get(rel.person_a);
    const pB = personMap.get(rel.person_b);
    if (!pA || !pB) continue;

    const famId = `F${fCounter++}`;
    let husbId: string | undefined;
    let wifeId: string | undefined;

    if      (pA.gender === 'male'   && pB.gender === 'female') { husbId = pA.id; wifeId = pB.id; }
    else if (pA.gender === 'female' && pB.gender === 'male')   { husbId = pB.id; wifeId = pA.id; }
    else                                                         { husbId = pA.id; wifeId = pB.id; }

    families.push({ id: famId, husbId, wifeId, children: [] });

    for (const pid of [rel.person_a, rel.person_b]) {
      const list = personToFamiliesParent.get(pid) ?? [];
      list.push(famId);
      personToFamiliesParent.set(pid, list);
    }
  }

  // Gán children vào family
  const childFamcMap: Map<string, string[]> = new Map();

  for (const rel of data.relationships) {
    if (!['biological_child','adopted_child'].includes(rel.type)) continue;
    if (!rel.person_a || !rel.person_b) continue;

    const parentFamilies = personToFamiliesParent.get(rel.person_a) ?? [];

    if (parentFamilies.length > 0) {
      const fam = families.find(f => f.id === parentFamilies[0]);
      if (fam && !fam.children.includes(rel.person_b)) {
        fam.children.push(rel.person_b);
        const fc = childFamcMap.get(rel.person_b) ?? [];
        fc.push(fam.id);
        childFamcMap.set(rel.person_b, fc);
      }
    } else {
      // Parent chưa có family → tạo gia đình đơn thân
      const pP   = personMap.get(rel.person_a);
      if (!pP) continue;
      const famId = `F${fCounter++}`;
      const fam: GedcomFamily = {
        id:       famId,
        husbId:   pP.gender === 'male'   ? rel.person_a : undefined,
        wifeId:   pP.gender === 'female' ? rel.person_a : undefined,
        children: [rel.person_b],
      };
      families.push(fam);
      const pl = personToFamiliesParent.get(rel.person_a) ?? [];
      pl.push(famId);
      personToFamiliesParent.set(rel.person_a, pl);
      const fc = childFamcMap.get(rel.person_b) ?? [];
      fc.push(famId);
      childFamcMap.set(rel.person_b, fc);
    }
  }

  // ──────────────────────────────────────────────
  // EXPORT INDIVIDUALS (INDI)
  // ──────────────────────────────────────────────
  for (const person of data.persons) {
    if (!person.id) continue;

    const xref = personXrefMap.get(person.id)!;
    L(`0 @${xref}@ INDI`);

    // ── Tên ─────────────────────────────────────
    const displayName = (person.full_name ?? 'Unknown').trim();
    const { surname, givenName } = splitVietnameseName(
      displayName,
      person.surname,
      person.given_name
    );

    // GEDCOM NAME: "GivenName /Surname/"
    // Ví dụ: "Văn An /Nguyễn/"  ← HỌ trong dấu //
    if (surname) {
      L(`1 NAME ${escapeGedcomValue(givenName)} /${escapeGedcomValue(surname)}/`);
    } else {
      L(`1 NAME ${escapeGedcomValue(givenName)} //`);
    }

    // SURN và GIVN riêng để phần mềm parse dễ hơn
    if (surname)   L(`2 SURN ${escapeGedcomValue(surname)}`);
    if (givenName) L(`2 GIVN ${escapeGedcomValue(givenName)}`);

    // ── Giới tính ────────────────────────────────
    if      (person.gender === 'male')   L('1 SEX M');
    else if (person.gender === 'female') L('1 SEX F');
    else                                 L('1 SEX U');

    // ── Ngày sinh ────────────────────────────────
    const birthDate = formatGedcomDate(
      person.birth_year, person.birth_month, person.birth_day,
      person.birth_date_modifier
    );

    const hasBirth = birthDate || person.birth_place_text
      || person.birth_lunar_year || person.birth_lunar_month || person.birth_lunar_day;

    if (hasBirth) {
      L('1 BIRT');
      if (birthDate) L(`2 DATE ${birthDate}`);
      if (person.birth_place_text) L(`2 PLAC ${escapeGedcomValue(person.birth_place_text)}`);

      // Custom tags âm lịch sinh — PHẢI có prefix _
      if (person.birth_lunar_year || person.birth_lunar_month || person.birth_lunar_day) {
        const lp: string[] = [];
        if (person.birth_lunar_day)   lp.push(String(person.birth_lunar_day).padStart(2, '0'));
        if (person.birth_lunar_month) lp.push(String(person.birth_lunar_month).padStart(2, '0'));
        if (person.birth_lunar_year)  lp.push(String(person.birth_lunar_year));
        L(`2 _LUNAR ${lp.join('/')}`);
        if (person.birth_lunar_is_leap) L('2 _LUNAR_LEAP Y');
      }
    }

    // ── Ngày mất ─────────────────────────────────
    if (person.is_deceased) {
      const deathDate = formatGedcomDate(
        person.death_year, person.death_month, person.death_day,
        person.death_date_modifier
      );

      // "1 DEAT Y" nếu biết đã mất nhưng không có ngày
      L(`1 DEAT${deathDate ? '' : ' Y'}`);
      if (deathDate) L(`2 DATE ${deathDate}`);
      if (person.death_place_text) L(`2 PLAC ${escapeGedcomValue(person.death_place_text)}`);

      // Custom tags âm lịch mất / ngày giỗ
      if (person.death_lunar_year || person.death_lunar_month || person.death_lunar_day) {
        const lp: string[] = [];
        if (person.death_lunar_day)   lp.push(String(person.death_lunar_day).padStart(2, '0'));
        if (person.death_lunar_month) lp.push(String(person.death_lunar_month).padStart(2, '0'));
        if (person.death_lunar_year)  lp.push(String(person.death_lunar_year));
        L(`2 _LUNAR ${lp.join('/')}`);
        if (person.death_lunar_is_leap) L('2 _LUNAR_LEAP Y');
      }
    }

    // ── Family links ─────────────────────────────
    // FAMC: gia đình mà người này là con
    for (const famId of childFamcMap.get(person.id) ?? []) {
      L(`1 FAMC @${famId}@`);
    }
    // FAMS: gia đình mà người này là cha/mẹ
    for (const famId of personToFamiliesParent.get(person.id) ?? []) {
      L(`1 FAMS @${famId}@`);
    }

    // ── Ghi chú ──────────────────────────────────
    if (person.note?.trim()) {
      lines.push(formatGedcomText(1, 'NOTE', person.note));
    }
  }

  // ──────────────────────────────────────────────
  // EXPORT FAMILIES (FAM)
  // ──────────────────────────────────────────────
  for (const fam of families) {
    L(`0 @${fam.id}@ FAM`);
    if (fam.husbId) L(`1 HUSB ${getXref(fam.husbId)}`);
    if (fam.wifeId) L(`1 WIFE ${getXref(fam.wifeId)}`);
    for (const childId of fam.children) {
      L(`1 CHIL ${getXref(childId)}`);
    }
  }

  // ──────────────────────────────────────────────
  // TRAILER
  // ──────────────────────────────────────────────
  L('0 TRLR');

  return lines.join('');
}
```

---

## Bước 4.1c — Test cho exporter

```typescript
// tests/gedcom/exporter.test.ts

import { exportToGedcom } from '@/utils/gedcom/exporter';
import { describe, it, expect } from 'vitest';

describe('exportToGedcom — Unicode & Encoding', () => {

  it('File bắt đầu bằng BOM UTF-8 (\\uFEFF)', () => {
    const result = exportToGedcom({ persons: [], relationships: [] });
    expect(result.charCodeAt(0)).toBe(0xFEFF);
  });

  it('Header có CHAR UTF-8', () => {
    const result = exportToGedcom({ persons: [], relationships: [] });
    expect(result).toContain('1 CHAR UTF-8');
  });

  it('Line ending là CRLF (\\r\\n)', () => {
    const result = exportToGedcom({ persons: [], relationships: [] });
    expect(result).toContain('\r\n');
    // Không có LF đơn (sau BOM)
    const afterBom = result.slice(1);
    expect(afterBom).not.toMatch(/(?<!\r)\n/);
  });

  it('GEDCOM version là 5.5.1', () => {
    const result = exportToGedcom({ persons: [], relationships: [] });
    expect(result).toContain('2 VERS 5.5.1');
  });

  it('Tên VN: họ là từ ĐẦU TIÊN — "Văn An /Nguyễn/"', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'Nguyễn Văn An', gender: 'male' }],
      relationships: [],
    });
    expect(result).toContain('NAME Văn An /Nguyễn/');
    // Đảm bảo KHÔNG có lỗi cũ
    expect(result).not.toContain('NAME Nguyễn Văn /An/');
  });

  it('Tên VN 1 từ: không có họ', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'An', gender: 'male' }],
      relationships: [],
    });
    expect(result).toContain('NAME An //');
  });

  it('Tên có SURN và GIVN riêng', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'Lê Thị Hoa', gender: 'female' }],
      relationships: [],
    });
    expect(result).toContain('2 SURN Lê');
    expect(result).toContain('2 GIVN Thị Hoa');
  });

  it('Tên có dấu tiếng Việt không bị vỡ', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'Phạm Thị Ngọc Ánh', gender: 'female' }],
      relationships: [],
    });
    expect(result).toContain('Phạm');
    expect(result).toContain('Ngọc Ánh');
    // Không có ký tự rác
    expect(result).not.toMatch(/[ÃïÂ]/);
  });

  it('Ngày đầy đủ → "18 MAR 1902"', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Test', gender: 'male',
        birth_year: 1902, birth_month: 3, birth_day: 18,
      }],
      relationships: [],
    });
    expect(result).toContain('2 DATE 18 MAR 1902');
  });

  it('Chỉ năm → "1945" (không có tháng/ngày)', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'Test', gender: 'male', birth_year: 1945 }],
      relationships: [],
    });
    expect(result).toContain('2 DATE 1945');
    expect(result).not.toContain('2 DATE 00');  // Không có "00 JAN 1945"
  });

  it('Modifier about → "ABT 1945"', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Test', gender: 'male',
        birth_year: 1945, birth_date_modifier: 'about',
      }],
      relationships: [],
    });
    expect(result).toContain('2 DATE ABT 1945');
  });

  it('Modifier before → "BEF 1975"', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Test', gender: 'male',
        is_deceased: true, death_year: 1975, death_date_modifier: 'before',
      }],
      relationships: [],
    });
    expect(result).toContain('2 DATE BEF 1975');
  });

  it('Âm lịch → custom tag _LUNAR', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Test', gender: 'male',
        is_deceased: true,
        death_year: 1978, death_month: 1, death_day: 5,
        death_lunar_year: 1977, death_lunar_month: 12, death_lunar_day: 5,
      }],
      relationships: [],
    });
    expect(result).toContain('2 DATE 05 JAN 1978');
    expect(result).toContain('2 _LUNAR 05/12/1977');
  });

  it('Tháng nhuận → _LUNAR_LEAP Y', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Test', gender: 'male',
        is_deceased: true,
        death_lunar_year: 1978, death_lunar_month: 3, death_lunar_day: 10,
        death_lunar_is_leap: true,
      }],
      relationships: [],
    });
    expect(result).toContain('2 _LUNAR 10/03/1978');
    expect(result).toContain('2 _LUNAR_LEAP Y');
  });

  it('Ký tự @ trong tên → escape thành @@', () => {
    const result = exportToGedcom({
      persons: [{ id: 'p1', full_name: 'Tên@Đặc biệt', gender: 'male' }],
      relationships: [],
    });
    expect(result).toContain('@@');
    expect(result).not.toMatch(/@Tên@/);
  });

  it('Export tên đúng khi có surname/given_name sẵn', () => {
    const result = exportToGedcom({
      persons: [{
        id: 'p1', full_name: 'Nguyễn Văn An',
        surname: 'Nguyễn', given_name: 'Văn An',
        gender: 'male',
      }],
      relationships: [],
    });
    expect(result).toContain('NAME Văn An /Nguyễn/');
    expect(result).toContain('2 SURN Nguyễn');
    expect(result).toContain('2 GIVN Văn An');
  });

  it('Round-trip: export rồi verify tên vẫn đúng', () => {
    const result = exportToGedcom({
      persons: [
        { id: 'p1', full_name: 'Nguyễn Văn An',   gender: 'male',   birth_year: 1945 },
        { id: 'p2', full_name: 'Trần Thị Bình',    gender: 'female', birth_year: 1948 },
      ],
      relationships: [{ type: 'marriage', person_a: 'p1', person_b: 'p2' }],
    });
    // Verify tên xuất hiện đúng trong file
    expect(result).toContain('Nguyễn');
    expect(result).toContain('Văn An');
    expect(result).toContain('Trần');
    expect(result).toContain('Thị Bình');
    // Verify family được tạo
    expect(result).toContain('0 @F1@ FAM');
    expect(result).toContain('1 HUSB @I1@');
    expect(result).toContain('1 WIFE @I2@');
  });

});
```

---

## Bước 4.1d — Cập nhật ExportButton.tsx

```typescript
// components/ExportButton.tsx (hoặc nơi đang gọi export)

import { exportToGedcom } from '@/utils/gedcom/exporter';

/**
 * Tải file .ged với encoding UTF-8 đúng chuẩn.
 * BOM đã được thêm trong exportToGedcom() — không thêm ở đây nữa.
 */
function downloadGedcomFile(gedcomContent: string, filename: string) {
  const blob = new Blob(
    [gedcomContent],
    { type: 'text/plain;charset=utf-8' }
    // BOM đã có trong gedcomContent (ký tự \uFEFF đầu tiên)
  );

  const url  = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href     = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
}

// Dùng trong component:
async function handleExport() {
  // Fetch data từ Supabase
  const { data: persons }       = await supabase.from('persons').select('*');
  const { data: relationships } = await supabase.from('relationships').select('*');

  const gedcomContent = exportToGedcom({
    persons:       persons ?? [],
    relationships: relationships ?? [],
    version:       '2.2.1',
  });

  const date     = new Date().toISOString().split('T')[0];  // "2026-05-21"
  const filename = `gia-pha-${date}.ged`;

  downloadGedcomFile(gedcomContent, filename);
}
```

> **Nếu dùng Next.js API route thay vì client-side download:**

```typescript
// app/api/export/gedcom/route.ts

import { exportToGedcom } from '@/utils/gedcom/exporter';

export async function GET() {
  // ... fetch data từ DB

  const gedcomContent = exportToGedcom({ persons, relationships, version: '2.2.1' });
  const BOM    = '\uFEFF';
  const buffer = Buffer.from(BOM + gedcomContent.slice(1), 'utf-8');
  // Slice(1) vì gedcomContent đã có BOM ở đầu — tránh double BOM

  // Thực ra exportToGedcom đã push BOM vào lines[0]
  // Dùng trực tiếp:
  const buf = Buffer.from(gedcomContent, 'utf-8');

  return new Response(buf, {
    headers: {
      'Content-Type':        'text/plain; charset=utf-8',
      'Content-Disposition': `attachment; filename="gia-pha-${new Date().toISOString().split('T')[0]}.ged"`,
    }
  });
}
```

---

## Bước 4.1e — Cập nhật parseGedcom() đọc SURN/GIVN/_LUNAR

> **Mới trong v2.2.1:** Parser cần đọc được file do chính giapha export ra, kể cả các tag SURN, GIVN, _LUNAR.

**Cập nhật trong `utils/gedcom/parser.ts` (hoặc `utils/gedcom.ts`):**

```typescript
// utils/gedcom/parser.ts
// Thêm xử lý SURN, GIVN, _LUNAR, _LUNAR_LEAP vào parseGedcom()

// Trong hàm parseGedcom(), phần xử lý INDI record:

// --- XỬ LÝ TAG NAME (sửa lại) ---
if (tag === 'NAME') {
  const nameVal = val || '';

  // GEDCOM NAME format: "GivenName /Surname/"
  // Ví dụ: "Văn An /Nguyễn/"   ← giapha export
  // Ví dụ: "Nguyễn Văn An"     ← format cũ không có //
  const surnameMatch = nameVal.match(/^(.*?)\s*\/([^\/]*)\/\s*(.*)$/);

  if (surnameMatch) {
    const givenPart   = (surnameMatch[1] + ' ' + surnameMatch[3])
      .replace(/@@/g, '@').trim();  // Unescape @@
    const surnamePart = surnameMatch[2].replace(/@@/g, '@').trim();

    // Ghép lại theo thứ tự Việt Nam: Họ + Tên
    if (surnamePart && givenPart) {
      fullName         = `${surnamePart} ${givenPart}`;
      parsedSurname    = surnamePart;
      parsedGivenName  = givenPart;
    } else if (surnamePart) {
      fullName = surnamePart;
    } else {
      fullName = givenPart;
    }
  } else {
    // Format cũ không có //, dùng nguyên
    fullName = nameVal.replace(/@@/g, '@').trim();
  }
}

// --- TAG SURN (họ riêng) ---
if (tag === 'SURN') {
  parsedSurname = (val ?? '').trim();
  // Nếu surname từ SURN đầy đủ hơn từ NAME, cập nhật lại
}

// --- TAG GIVN (tên riêng) ---
if (tag === 'GIVN') {
  parsedGivenName = (val ?? '').trim();
}

// --- XỬ LÝ _LUNAR (âm lịch) ---
// Chạy trong context của BIRT hoặc DEAT record
if (tag === '_LUNAR') {
  // Format: "DD/MM/YYYY" hoặc "MM/YYYY" hoặc "YYYY"
  const parts = (val ?? '').split('/');

  if (parts.length === 3) {
    const [d, m, y] = parts.map(Number);
    if (currentEventType === 'BIRT') {
      birth_lunar_day   = d || null;
      birth_lunar_month = m || null;
      birth_lunar_year  = y || null;
    } else if (currentEventType === 'DEAT') {
      death_lunar_day   = d || null;
      death_lunar_month = m || null;
      death_lunar_year  = y || null;
    }
  } else if (parts.length === 2) {
    const [m, y] = parts.map(Number);
    if (currentEventType === 'BIRT') { birth_lunar_month = m; birth_lunar_year = y; }
    if (currentEventType === 'DEAT') { death_lunar_month = m; death_lunar_year = y; }
  } else if (parts.length === 1) {
    const y = Number(parts[0]);
    if (currentEventType === 'BIRT') birth_lunar_year = y;
    if (currentEventType === 'DEAT') death_lunar_year = y;
  }
}

// --- XỬ LÝ _LUNAR_LEAP ---
if (tag === '_LUNAR_LEAP') {
  const isLeap = (val ?? '').trim().toUpperCase() === 'Y';
  if (currentEventType === 'BIRT') birth_lunar_is_leap = isLeap;
  if (currentEventType === 'DEAT') death_lunar_is_leap = isLeap;
}
```

---

## Bước 4.2 — Import Pipeline với Preview

**Luồng import (KHÔNG import thẳng vào DB):**

```
Bước 1: User upload file .ged
Bước 2: Parse + Validate (không insert gì cả)
Bước 3: Hiển thị Preview Report:
  ┌────────────────────────────────────────────────┐
  │  📋 KẾT QUẢ PHÂN TÍCH FILE GEDCOM             │
  │  File: HoNguyen.ged (128KB)                   │
  │                                               │
  │  ✅ 523 người                                  │
  │  ✅ 147 gia đình                               │
  │  ✅ 921 sự kiện                                │
  │  ✅ 32 ảnh (cần upload riêng)                 │
  │                                               │
  │  ⚠️ 18 người có thể bị trùng                  │
  │     [Xem danh sách]                          │
  │                                               │
  │  ❌ 12 sự kiện có ngày không hợp lệ          │
  │     [Xem chi tiết]                           │
  │                                               │
  │  [Hủy bỏ]          [Tiến hành Import]        │
  └────────────────────────────────────────────────┘
Bước 4: User xem xét và xác nhận
Bước 5: Import qua RPC (tất cả hoặc không có gì)
Bước 6: Import Report
```

---

## Bước 4.3 — Fix GEDCOM Import Bug (mất 1 parent)

```typescript
// Bug cũ: const parentA = husb || wife;
// → Chỉ lấy 1 người, mất người còn lại

// Fix: Gán CẢ HAI vào family_parents
async function importFamRecord(fam: GedcomFam) {
  const result = await createFamilyUnit({
    type:    'marriage',
    parentA: fam.husb ? { id: personIdMap[fam.husb], role: 'husband' } : undefined,
    parentB: fam.wife ? { id: personIdMap[fam.wife], role: 'wife'    } : undefined,
    children: fam.chil?.map(c => ({ id: personIdMap[c], type: 'biological' })) ?? [],
  });
  return result;
}
```

---

## Bước 4.4 — Merge/Dedup Tool

```
Giao diện DuplicateFinder:
┌─────────────────────────────────────────────────┐
│  🔍 NGƯỜI CÓ THỂ BỊ TRÙNG                       │
│                                                 │
│  Nguyễn Văn An (1902–1978)                      │
│    ↔ Nguyễn Văn An (1902–?)                     │
│  Độ tương đồng: 87%                             │
│  [Xem chi tiết]  [Gộp]  [Không phải trùng]     │
└─────────────────────────────────────────────────┘
```

Gộp có thể undo trong 30 phút (dùng audit_log để revert).

---

# PHASE 5 — UI/UX & GRAPH ENGINE (Tuần 17–20)

---

## Bước 5.1 — GenealogyDatePicker Component

```
components/GenealogyDatePicker.tsx

┌──────────────────────────────────────────────────┐
│  NGÀY SINH                                       │
│                                                  │
│  Dương lịch:  [12] / [ 3] / [1945]              │
│  ↕ Tự động convert                               │
│  Âm lịch:    [12] / [ 2] / [1945] □ Tháng nhuận │
│  Can Chi:    Ất Dậu (tự động, không sửa được)    │
│                                                  │
│  Độ chính xác:                                   │
│  ● Chính xác  ○ Khoảng  ○ Trước  ○ Sau           │
│                                                  │
│  Ghi chú ngày: [Khoảng năm Ất Dậu...       ]    │
└──────────────────────────────────────────────────┘
```

---

## Bước 5.2 — PersonTimeline Component

```
components/PersonTimeline.tsx

1902  ●── Sinh
           18/03/1902 · Mùng 12/2 năm Nhâm Dần · Nhâm Dần
           Làng Đình Bảng, Từ Sơn, Bắc Ninh
           [📄 Giấy khai sinh — Độ tin cậy: Cao]

1920  ●── Kết hôn với Lê Thị Dung

1945  ●── Di cư vào Nam
           [Khoảng tháng 9, 1945]

1978  ●── Qua đời
           Mùng 5/1 năm Đinh Tỵ · TP. Hồ Chí Minh
           Thọ khoảng 75–76 tuổi ← "khoảng" vì chỉ biết năm sinh
```

---

## Bước 5.3 — Vietnamese Genealogy Mode

```typescript
// utils/calendar/deathAnniversary.ts

export function getUpcomingDeathAnniversaries(
  persons: PersonWithDeathEvent[],
  daysAhead: number = 30
): Anniversary[] {
  // Tìm ngày giỗ âm lịch sắp tới trong daysAhead ngày
}
```

Vietnamese kinship labels:
```typescript
// utils/graph/kinship.ts

const vietnameseKinship: Record<string, string> = {
  'parent:male':                   'Cha',
  'parent:female':                 'Mẹ',
  'parent.parent:male':            'Ông nội',
  'parent.parent:female':          'Bà nội',
  'parent.parent:male:maternal':   'Ông ngoại',
  'parent.parent:female:maternal': 'Bà ngoại',
  'parent.sibling:male':           'Bác/Chú',
  'parent.sibling:female':         'Bác/Cô',
  'parent.sibling:male:maternal':  'Cậu',
  'parent.sibling:female:maternal':'Dì',
  'child.spouse:male':             'Con rể',
  'child.spouse:female':           'Con dâu',
};
```

---

## Bước 5.4 — Graph Engine Refactor

```
utils/graph/
├── index.ts
├── build.ts      ← Xây graph từ families
├── kinship.ts    ← Tính quan hệ + Vietnamese labels
└── traverse.ts   ← BFS/DFS để phát hiện vòng lặp
```

---

# PHASE 6 — PRODUCTION (Tuần 21–22)

---

## Bước 6.1 — Data Quality Dashboard

```
Trang /dashboard/data-quality (admin):

╔══════════════════════════════════════════════════════╗
║  📊 KIỂM TRA CHẤT LƯỢNG DỮ LIỆU                      ║
╠══════════════════════════════════════════════════════╣
║  ✅ 523 người · ✅ 147 gia đình · ✅ 891 sự kiện      ║
╠══════════════════════════════════════════════════════╣
║  ⚠️ CẦN XEM XÉT (45)                                 ║
║  • 12 người không có cha/mẹ trong hệ thống  [Xem]   ║
║  • 8 người không có ngày sinh               [Xem]   ║
║  • 3 người có ngày mất trước ngày sinh      [Sửa]   ║
║  • 5 gia đình không có cha/mẹ               [Xem]   ║
║  • 7 sự kiện không liên kết với ai          [Xem]   ║
║  • 10 ảnh không liên kết với entity nào     [Xóa]   ║
╚══════════════════════════════════════════════════════╝
```

---

## Bước 6.2 — Backup & Restore

```bash
#!/bin/bash
# scripts/backup-db.sh

DATE=$(date +%Y-%m-%d)
BACKUP_DIR="./backups/$DATE"
mkdir -p "$BACKUP_DIR"

# Backup database
pg_dump "$DATABASE_URL" > "$BACKUP_DIR/database.sql"

echo "✓ Backup xong: $BACKUP_DIR"

# Thêm vào crontab: chạy tự động 2 giờ sáng
# 0 2 * * * /path/to/backup-db.sh >> /var/log/giapha-backup.log 2>&1
```

---

## Bước 6.3 — Docker Self-host

```yaml
# docker-compose.yml

version: '3.8'
services:
  app:
    build: .
    ports: ['3000:3000']
    environment:
      - NEXT_PUBLIC_SUPABASE_URL=${SUPABASE_URL}
      - NEXT_PUBLIC_SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
      - DATABASE_URL=${DATABASE_URL}
    restart: unless-stopped

  db:
    image: supabase/postgres:15
    volumes: ['pgdata:/var/lib/postgresql/data']
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    restart: unless-stopped

volumes:
  pgdata:
```

---

## PHẦN 2 — TỔNG HỢP FILES

```
services/
  person.service.ts
  family.service.ts
  event.service.ts

rules/
  person.rules.ts
  family.rules.ts
  event.rules.ts

utils/
  calendar/
    canChi.ts                ← Không lưu DB, tính runtime
    lunarDate.ts             ← Convert âm/dương lịch
    ageCalculation.ts        ← ⭐ Tính tuổi với khoảng (v2.2.1: fix bug)
    deathAnniversary.ts      ← Ngày giỗ sắp tới
  date-parser/
    parseVietnameseDate.ts
    parseGedcomDate.ts
    normalizeDate.ts
    formatGenealogyDate.ts
  gedcom/
    index.ts
    parser.ts                ← v2.2.1: cập nhật đọc SURN/GIVN/_LUNAR
    mapper.ts
    exporter.ts              ← ⭐ v2.2.1: viết lại hoàn toàn (BOM+CRLF+tên đúng)
    validator.ts
    compat/familygem.ts
    compat/gramps.ts
    compat/ancestry.ts
  graph/
    build.ts
    kinship.ts
    traverse.ts
  text/
    removeDiacritics.ts      ← Fallback nếu unaccent không dùng được

types/
  person.ts
  family.ts
  event.ts
  media.ts
  source.ts
  place.ts

components/
  GenealogyDatePicker.tsx
  PersonTimeline.tsx
  MediaGallery.tsx
  MediaUploader.tsx
  MediaPicker.tsx
  PlaceAutocomplete.tsx
  SourceBadge.tsx
  CitationForm.tsx
  SourceManager.tsx
  DuplicateFinder.tsx
  FamilyCard.tsx
  DataQualityDashboard.tsx

tests/
  calendar/canChi.test.ts
  calendar/lunarDate.test.ts
  date/ageCalculation.test.ts
  date/dateParser.test.ts
  rules/familyRules.test.ts
  rules/personRules.test.ts
  gedcom/parser.test.ts
  gedcom/exporter.test.ts           ← ⭐ v2.2.1: test họ/tên + âm lịch + Unicode
  graph/kinship.test.ts

scripts/
  migrate-to-family-model.ts
  migrate-dates-to-events.ts
  migrate-avatars-to-media.ts
  seed-vietnam-places.ts
  backup-db.sh
  restore-db.sh

docs/migrations/
  YYYYMMDD_rpc_create_family.sql
  YYYYMMDD_soft_delete.sql
  YYYYMMDD_optimistic_lock.sql
  YYYYMMDD_person_names.sql
  YYYYMMDD_family_model.sql
  YYYYMMDD_event_system.sql
  YYYYMMDD_person_cleanup.sql
  YYYYMMDD_media_system.sql
  YYYYMMDD_source_system.sql
  YYYYMMDD_place_system.sql
  YYYYMMDD_search_setup.sql
  YYYYMMDD_audit_log.sql
  seed-vietnam-places.sql
```

---

## PHẦN 3 — SPRINT PLAN (22 tuần)

| Sprint | Tuần | Làm gì | Kết quả kiểm tra |
|--------|------|--------|-----------------|
| 0a | 1 | Service Layer + RPC + Soft Delete + Optimistic Lock | Code sạch, không thay đổi UI |
| 0b | 2 | Domain Rules Engine + Test Setup (Vitest) | Rules ngăn data vô lý |
| **⚡** | **Có thể làm sớm** | **GEDCOM exporter.ts mới (Bước 4.1b–4.1e)** | **Export đúng font, tên đúng họ** |
| 1a | 3 | Person Names: SQL + migration + UI | Tên húy, pháp danh trong PersonDetail |
| 1b | 4 | Family Model: SQL + migration dry-run | FamilyTree vẫn chạy |
| 1c | 5 | Family review UI + kinship/tree update | FamilyTree dùng Family model |
| 1d | 6 | Date Parser + Can Chi Calendar | Parse "Khoảng 1945", hiện Can Chi |
| 1e | 7 | Age Calculation v2.2.1 + Tests | "khoảng 55–56 tuổi" khi chỉ biết năm |
| 1f | 8 | Event System: SQL + migration script | Verify ngày sinh/mất chuyển đúng |
| 1g | — | GenealogyDatePicker + Person Cleanup | Nhập ngày có âm lịch tự động |
| 2a | 9 | Media System + MediaGallery + Privacy | Upload ảnh, gallery, privacy rule |
| 2b | 10 | Source & Citation + SourceBadge | Thêm nguồn, xem badge |
| 2c | 11 | RLS hoàn thiện theo privacy_level | member/editor/admin thấy đúng |
| 3a | 12 | Place System + seed 63 tỉnh | Nhập địa danh có gợi ý |
| 3b | 13 | Search (unaccent + remove_diacritics) + Audit Log | Tìm không dấu hoạt động |
| 4a | 14 | GEDCOM module hoàn thiện + Import Pipeline | Xem preview trước import |
| 4b | 15 | Fix import bug + Merge/Dedup | Import FamilyGem không mất parent |
| 4c | 16 | Test GEDCOM đầy đủ + compat layer | Roundtrip pass |
| 5a | 17 | PersonTimeline UI | Timeline events đẹp |
| 5b | 18 | PersonDetail redesign + FamilyCard | UI mới |
| 5c | 19 | Vietnamese Mode + Death Anniversary | Đời/chi, ngày giỗ sắp tới |
| 5d | 20 | Graph Engine refactor | Kinship module riêng |
| 6a | 21 | Data Quality Dashboard + Backup | Admin dọn data |
| 6b | 22 | Docker hoàn chỉnh + Production | Deploy sẵn sàng |

---

## PHẦN 4 — CHECKLIST TRƯỚC KHI DEPLOY GEDCOM

```
□ exporter.ts mới đã thay thế exportToGedcom() cũ
□ ExportButton.tsx dùng Blob với charset=utf-8
□ Mở file .ged bằng Notepad++ → thấy "UTF-8 with BOM" ở góc phải
□ Mở file .ged bằng FamilyGem → tên tiếng Việt hiển thị đúng
□ Mở file .ged bằng Gramps → không có lỗi encoding
□ Test: "Nguyễn Văn An" → export "Văn An /Nguyễn/" → import lại "Nguyễn Văn An"
□ Test: năm mờ 1945 với modifier 'about' → export "ABT 1945"
□ Test: âm lịch 05/12 Đinh Tỵ → export "_LUNAR 05/12/1977"
□ Test: tháng nhuận → export "_LUNAR_LEAP Y"
□ Tất cả tests trong exporter.test.ts pass (npx vitest run)
```

---

## PHẦN 5 — KẾT QUẢ KỲ VỌNG

| Tính năng | Trước | Sau v2.2.1 |
|-----------|-------|-----------|
| Tên họ VN trong GEDCOM | ❌ "Nguyễn Văn /An/" (sai) | ✅ **"Văn An /Nguyễn/" (đúng)** |
| Font GEDCOM trên Windows | ❌ "NguyÃªn Vò¨n An" | ✅ **"Nguyễn Văn An"** |
| Âm lịch trong GEDCOM | ❌ Bị mất | ✅ **Custom tag _LUNAR** |
| Tháng nhuận | ❌ Bị mất | ✅ **_LUNAR_LEAP Y** |
| Import lại file vừa export | ❌ Tên bị sai | ✅ **Round-trip đúng** |
| Tên húy, pháp danh | ❌ Text lộn xộn | ✅ Hệ thống tên có cấu trúc |
| Đa hôn nhân, con riêng | ❌ Không hỗ trợ | ✅ Family model chuẩn |
| Tuổi khi chỉ biết năm | ❌ "56 tuổi" (sai) | ✅ "khoảng 55–56 tuổi" |
| Ngày mờ | ❌ NULL | ✅ "Khoảng 1945", "Trước 1954" |
| Transaction an toàn | ❌ Hỏng giữa chừng | ✅ RPC function |
| Xóa nhầm dữ liệu | ❌ Mất vĩnh viễn | ✅ Soft delete 90 ngày |
| 2 người cùng sửa | ❌ Ghi đè âm thầm | ✅ Thông báo xung đột |
| Tìm kiếm không dấu | ❌ Không có | ✅ "nguyen van a" tìm được |
| Backup | ❌ Không có | ✅ Tự động hàng ngày |
| Ngày giỗ | ❌ Không có | ✅ Hiển thị sắp tới |
