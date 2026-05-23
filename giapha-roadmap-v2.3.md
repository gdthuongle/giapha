# GIAPHA-OS — ROADMAP v2.3
## Triển khai an toàn từ code hiện tại

> Mục tiêu của bản v2.3: không chỉ thiết kế schema mới, mà chỉ rõ cách nâng cấp **từ code Giapha hiện tại** sang mô hình genealogy chuẩn hơn mà không làm mất dữ liệu, không làm gãy UI, và có đường rollback rõ ràng ở từng bước.
>
> Bản này kế thừa các điểm tốt của v2.2/v2.2.1, đặc biệt là GEDCOM Unicode, sửa họ/tên Việt Nam, Family Model, Event Date System, Soft Delete, RPC transaction, Age Calculation và Test Setup. Đồng thời bản này sửa các rủi ro còn lại: migration gán nhầm con khi tái hôn, import GEDCOM chưa có staging, hard delete chưa bị chặn, `_LUNAR` chưa có prefix riêng, date migration death chưa đủ case, search index thiếu unique index, và rollback chưa cụ thể.

---

## 0. Nguyên tắc triển khai bắt buộc

### 0.1. Không phá code hiện tại trong một lần

Code hiện tại vẫn đang dựa trên:

```text
persons
relationships
custom_events
person_details_private
utils/gedcom.ts
utils/treeHelpers.ts
components/FamilyTree.tsx
components/KinshipFinder.tsx
app/actions/data.ts
```

Vì vậy không được đổi ngay sang schema mới rồi sửa toàn bộ app cùng lúc. Mọi migration lớn phải chạy kiểu **song song**:

```text
Schema cũ vẫn còn
Schema mới được thêm vào
Adapter đọc cả cũ và mới
UI ưu tiên schema mới nhưng fallback schema cũ
Verify đủ test/report
Sau đó mới cleanup cột/bảng cũ
```

### 0.2. Không hard delete dữ liệu genealogy core

Các bảng sau **không được xóa thật** bằng `.delete()` trong code thường:

```text
persons
relationships
families
family_parents
family_children
events
person_events
sources
citations
citation_links
```

Chỉ dùng soft delete. Hard delete chỉ cho:

```text
import staging/temp
media orphan đã xác nhận
log/temporary cache
```

### 0.3. GEDCOM sửa trước, migration schema làm sau

Phần GEDCOM export hiện tại có thể sửa ngay vì chỉ phụ thuộc `persons` và `relationships`. Thứ tự ưu tiên:

```text
1. Sửa GEDCOM export/import cơ bản hiện tại.
2. Thêm test round-trip.
3. Thêm backup và safety layer.
4. Sau đó mới migrate Family/Event.
```

### 0.4. Mọi bước lớn phải có 4 thứ

Mỗi phase phải có:

```text
- Files cần sửa/tạo
- Migration/runbook cụ thể
- Test bắt buộc
- Rollback nếu lỗi
```

Nếu thiếu 1 trong 4 thứ này thì chưa được triển khai production.

---

## 1. Tóm tắt thay đổi v2.3 so với v2.2.1

| # | Vấn đề còn lại ở v2.2.1 | Cách sửa trong v2.3 |
|---|---|---|
| 1 | Migration `relationships → families` có thể gán nhầm con khi tái hôn | Thêm dry-run, review queue, chỉ auto-gán khi đủ bằng chứng |
| 2 | RPC cast enum chưa chuẩn | Cast đúng enum, thêm `SET search_path`, check quyền trong RPC |
| 3 | Soft delete thêm cột nhưng chưa chặn `.delete()` cũ | Thêm service soft delete, grep toàn bộ `.delete()`, optional trigger chặn hard delete |
| 4 | GEDCOM dùng `_LUNAR` quá chung | Export `_GIAPHA_LUNAR`, parser đọc cả `_LUNAR` và `_GIAPHA_LUNAR` |
| 5 | `formatGedcomText()` chưa wrap 255 bytes | Thêm helper wrap theo UTF-8 byte, dùng `CONT/CONC` đúng |
| 6 | Event migration death chỉ có tháng/năm bị xử lý sai | Dùng chung helper normalize date cho birth/death |
| 7 | Search materialized view thiếu unique index | Thêm unique index trước khi dùng `REFRESH CONCURRENTLY` |
| 8 | GEDCOM import chưa có staging | Thêm `import_sessions`, `import_xrefs`, staging tables, preview, commit RPC |
| 9 | Backup để quá muộn | Đưa backup/restore lên Phase 0 trước mọi migration |
| 10 | Graph/UI có thể gãy khi đổi schema | Thêm compatibility adapter đọc cả `relationships` và `families` |
| 11 | Cleanup drop cột cũ quá sớm | Chỉ cleanup sau 2 vòng verify + backup |
| 12 | Rollback chưa rõ | Mỗi phase có rollback theo DB, code, feature flag |

---

## 2. Kiến trúc đích sau v2.3

```text
PostgreSQL / Supabase
│
├── LEGACY COMPAT LAYER
│   ├── persons                  ← vẫn giữ trong giai đoạn chuyển đổi
│   ├── relationships            ← legacy, không drop sớm
│   └── compatibility views/RPC  ← adapter cho UI cũ/mới
│
├── IDENTITY LAYER
│   ├── persons
│   └── person_names
│
├── FAMILY LAYER
│   ├── families
│   ├── family_parents
│   ├── family_children
│   └── migration_family_review
│
├── EVENT LAYER
│   ├── events
│   └── person_events
│
├── GEDCOM IMPORT LAYER
│   ├── import_sessions
│   ├── import_xrefs
│   ├── import_warnings
│   ├── import_staging_persons
│   ├── import_staging_families
│   └── import_staging_events
│
├── MEDIA/SOURCE/PLACE LAYER
│   ├── media, person_media, event_media, family_media
│   ├── sources, citations, citation_links
│   └── places, place_aliases
│
└── SYSTEM LAYER
    ├── audit_logs
    ├── feature_flags
    └── person_search_index
```

Next.js app:

```text
services/
  person.service.ts
  family.service.ts
  event.service.ts
  gedcom-import.service.ts
  migration.service.ts

compat/
  relationships.compat.ts
  family.compat.ts
  date.compat.ts

utils/
  gedcom/
    tokenizer.ts
    parser.ts
    normalizer.ts
    mapper.ts
    exporter.ts
    validator.ts
    writer.ts
    compat/familygem.ts
    compat/gramps.ts
    compat/legacy551.ts
  graph/
    buildFromLegacy.ts
    buildFromFamilies.ts
    buildUnifiedGraph.ts
  date-parser/
  calendar/
```

---

## 3. Feature flags bắt buộc

Tạo file:

```text
lib/featureFlags.ts
```

Nội dung đề xuất:

```typescript
export const featureFlags = {
  gedcomExporterV23: process.env.NEXT_PUBLIC_FF_GEDCOM_EXPORTER_V23 === 'true',
  readPersonNames: process.env.NEXT_PUBLIC_FF_READ_PERSON_NAMES === 'true',
  readFamilies: process.env.NEXT_PUBLIC_FF_READ_FAMILIES === 'true',
  writeFamilies: process.env.NEXT_PUBLIC_FF_WRITE_FAMILIES === 'true',
  readEvents: process.env.NEXT_PUBLIC_FF_READ_EVENTS === 'true',
  writeEvents: process.env.NEXT_PUBLIC_FF_WRITE_EVENTS === 'true',
  gedcomImportStaging: process.env.NEXT_PUBLIC_FF_GEDCOM_IMPORT_STAGING === 'true',
};
```

`.env.local` ban đầu:

```env
NEXT_PUBLIC_FF_GEDCOM_EXPORTER_V23=false
NEXT_PUBLIC_FF_READ_PERSON_NAMES=false
NEXT_PUBLIC_FF_READ_FAMILIES=false
NEXT_PUBLIC_FF_WRITE_FAMILIES=false
NEXT_PUBLIC_FF_READ_EVENTS=false
NEXT_PUBLIC_FF_WRITE_EVENTS=false
NEXT_PUBLIC_FF_GEDCOM_IMPORT_STAGING=false
```

Quy tắc:

```text
- Bật flag từng cái.
- Có lỗi thì tắt flag để quay về code cũ.
- Không dùng flag để che lỗi migration; flag chỉ giúp rollback UI/logic.
```

---

# PHASE 0 — Backup, test, safety baseline

## Mục tiêu

Tạo lớp an toàn trước khi đụng schema lớn.

## Files cần tạo/sửa

```text
scripts/backup-db.sh
scripts/backup-json.ts
scripts/backup-gedcom.ts
scripts/restore-db.sh
scripts/verify-current-data.ts
lib/featureFlags.ts
package.json
tests/smoke/currentData.test.ts
```

## Bước 0.1 — Backup bắt buộc

### `scripts/backup-db.sh`

```bash
#!/bin/bash
set -euo pipefail

DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR="./backups/$DATE"
mkdir -p "$BACKUP_DIR"

pg_dump "$DATABASE_URL" > "$BACKUP_DIR/database.sql"

echo "Backup database xong: $BACKUP_DIR/database.sql"
```

### `scripts/backup-json.ts`

Export các bảng legacy quan trọng:

```text
persons
relationships
custom_events
person_details_private
profiles
```

Output:

```text
backups/YYYY-MM-DD_HH-MM-SS/persons.json
backups/YYYY-MM-DD_HH-MM-SS/relationships.json
...
```

### `scripts/backup-gedcom.ts`

Dùng exporter hiện tại hoặc exporter v2.3 sau khi sửa để xuất một file GEDCOM toàn bộ gia phả.

Output:

```text
backups/YYYY-MM-DD_HH-MM-SS/current-export.ged
```

## Bước 0.2 — Verify dữ liệu hiện tại

`scripts/verify-current-data.ts` cần kiểm tra:

```text
- Số persons
- Số relationships
- Số marriage relationships
- Số biological/adopted child relationships
- Người thiếu full_name
- Quan hệ trỏ tới person không tồn tại
- Self relationship
- Vòng lặp parent-child đơn giản
- Người có birth_year > death_year
```

Output:

```text
reports/current-data-quality.json
reports/current-data-quality.md
```

## Test bắt buộc

```bash
npm run test
npm run lint
npm run typecheck
bash scripts/backup-db.sh
npx ts-node scripts/backup-json.ts
npx ts-node scripts/verify-current-data.ts
```

Pass khi:

```text
- Có file backup database.sql
- Có JSON backup
- Có report current-data-quality
- App vẫn chạy bình thường
- Không bật feature flag mới
```

## Rollback

Phase 0 không thay đổi DB core ngoài thêm file/script. Rollback:

```text
- Revert commit code Phase 0
- Xóa feature flags khỏi .env nếu cần
```

---

# PHASE 1 — GEDCOM hotfix trên code hiện tại

## Mục tiêu

Sửa GEDCOM export/import cơ bản ngay trên schema hiện tại trước khi migration lớn.

## Files cần tạo/sửa

```text
utils/gedcom.ts                       ← tạm giữ wrapper compatibility
utils/gedcom/index.ts                 ← mới
utils/gedcom/exporter.ts              ← mới
utils/gedcom/parser.ts                ← mới hoặc tách từ file cũ
utils/gedcom/writer.ts                ← mới: BOM, CRLF, wrap byte
utils/gedcom/date.ts                  ← mới
utils/gedcom/name.ts                  ← mới
components/ExportButton.tsx           ← sửa nếu đang dùng client download
app/api/export/gedcom/route.ts        ← sửa nếu đang export qua API
tests/gedcom/exporter.test.ts
tests/gedcom/parser.test.ts
tests/gedcom/roundtrip.test.ts
```

## Bước 1.1 — Tách module nhưng giữ wrapper cũ

`utils/gedcom.ts` không xóa ngay. Chuyển thành wrapper:

```typescript
export { exportToGedcom } from './gedcom/exporter';
export { parseGedcom } from './gedcom/parser';
```

Lý do: các file đang import từ `utils/gedcom.ts` không bị gãy.

## Bước 1.2 — Sửa tên Việt Nam

`utils/gedcom/name.ts`:

```typescript
export function splitVietnameseName(fullName: string, surname?: string | null, givenName?: string | null) {
  if (surname?.trim()) {
    return { surname: surname.trim(), givenName: (givenName ?? '').trim() };
  }
  const parts = fullName.trim().split(/\s+/).filter(Boolean);
  if (parts.length === 0) return { surname: 'Unknown', givenName: 'Unknown' };
  if (parts.length === 1) return { surname: '', givenName: parts[0] };
  return { surname: parts[0], givenName: parts.slice(1).join(' ') };
}

export function formatGedcomName(fullName: string, surname?: string | null, givenName?: string | null) {
  const n = splitVietnameseName(fullName, surname, givenName);
  return n.surname ? `${n.givenName} /${n.surname}/` : `${n.givenName} //`;
}

export function parseGedcomName(nameVal: string) {
  const m = nameVal.match(/^(.*?)\s*\/([^\/]*)\/\s*(.*)$/);
  if (!m) return { fullName: nameVal.replace(/@@/g, '@').trim() };
  const given = `${m[1]} ${m[3]}`.replace(/@@/g, '@').trim();
  const surname = m[2].replace(/@@/g, '@').trim();
  const fullName = surname && given ? `${surname} ${given}` : (surname || given);
  return { fullName, surname, givenName: given };
}
```

## Bước 1.3 — Writer chuẩn UTF-8/BOM/CRLF/wrap byte

`utils/gedcom/writer.ts`:

```typescript
const CRLF = '\r\n';
const BOM = '\uFEFF';
const MAX_BYTES = 248;

export class GedcomWriter {
  private lines: string[] = [];

  addRaw(line: string) {
    this.lines.push(line);
  }

  add(level: number, tag: string, value?: string | null) {
    if (value == null || value === '') this.lines.push(`${level} ${tag}`);
    else this.addWrapped(level, tag, sanitizeGedcomValue(value));
  }

  addWrapped(level: number, tag: string, value: string) {
    const paragraphs = value.split(/\r?\n/);
    paragraphs.forEach((paragraph, idx) => {
      const firstTag = idx === 0 ? tag : 'CONT';
      const firstLevel = idx === 0 ? level : level + 1;
      emitWrappedLine(this.lines, firstLevel, firstTag, paragraph);
    });
  }

  toString() {
    return BOM + this.lines.join(CRLF) + CRLF;
  }
}

function emitWrappedLine(lines: string[], level: number, tag: string, value: string) {
  let remaining = value;
  let currentTag = tag;
  let currentLevel = level;

  while (true) {
    const prefix = `${currentLevel} ${currentTag} `;
    const chunk = takeUtf8Chunk(remaining, MAX_BYTES - byteLen(prefix));
    lines.push(prefix + chunk);
    remaining = remaining.slice(chunk.length);
    if (!remaining) break;
    currentTag = 'CONC';
    currentLevel = level + 1;
  }
}

function takeUtf8Chunk(text: string, maxBytes: number) {
  let out = '';
  for (const ch of text) {
    if (byteLen(out + ch) > maxBytes) break;
    out += ch;
  }
  return out;
}

function byteLen(s: string) {
  return Buffer.byteLength(s, 'utf8');
}

export function sanitizeGedcomValue(value: string) {
  return value.trim().replace(/@/g, '@@');
}
```

## Bước 1.4 — Custom tag âm lịch thống nhất

Exporter v2.3 chỉ xuất:

```gedcom
2 _GIAPHA_LUNAR 05/12/1977
2 _GIAPHA_LUNAR_LEAP Y
2 _GIAPHA_CALENDAR lunar
2 _GIAPHA_DATE_TEXT Mùng 5 tháng Chạp năm Đinh Tỵ
```

Parser v2.3 đọc cả cũ và mới:

```typescript
const isLunarTag = tag === '_GIAPHA_LUNAR' || tag === '_LUNAR';
const isLeapTag = tag === '_GIAPHA_LUNAR_LEAP' || tag === '_LUNAR_LEAP';
```

## Bước 1.5 — ExportButton/API không double BOM

Quy tắc:

```text
- `exportToGedcom()` trả về string đã có BOM.
- Client Blob không thêm BOM.
- API route không thêm BOM.
```

API route:

```typescript
const gedcomContent = exportToGedcom({ persons, relationships, version: '2.3' });
const buffer = Buffer.from(gedcomContent, 'utf-8');
return new Response(buffer, {
  headers: {
    'Content-Type': 'text/plain; charset=utf-8',
    'Content-Disposition': `attachment; filename="gia-pha-${date}.ged"`,
  }
});
```

## Test bắt buộc

```text
GEDCOM export:
✅ Bắt đầu bằng BOM U+FEFF
✅ Header có `2 VERS 5.5.1`
✅ Header có `1 CHAR UTF-8`
✅ Line ending CRLF
✅ Không có LF đơn
✅ Tên Nguyễn Văn An → Văn An /Nguyễn/
✅ Có SURN/GIVN
✅ Dấu tiếng Việt không vỡ
✅ @ trong value thành @@
✅ NOTE dài tiếng Việt không dòng nào > 255 bytes
✅ newline dùng CONT
✅ line dài dùng CONC
✅ birth/death chỉ năm không sinh ra ngày 00
✅ ABT/BEF/AFT đúng
✅ _GIAPHA_LUNAR và _GIAPHA_LUNAR_LEAP đúng
✅ Parser đọc lại cả _LUNAR legacy và _GIAPHA_LUNAR
✅ Round-trip export → parse giữ đúng tên và âm lịch
```

Lệnh:

```bash
npx vitest run tests/gedcom
npm run typecheck
npm run lint
```

## Rollback

Nếu export lỗi:

```text
1. Tắt `NEXT_PUBLIC_FF_GEDCOM_EXPORTER_V23=false`.
2. Revert các file `utils/gedcom/*` nếu cần.
3. Giữ wrapper `utils/gedcom.ts` trỏ lại code cũ.
4. Không cần restore DB vì Phase 1 chưa thay đổi DB.
```

---

# PHASE 2 — Safety layer DB và service layer

## Mục tiêu

Thêm transaction RPC, soft delete, audit, optimistic locking, domain rules trước migration.

## Files cần tạo/sửa

```text
docs/migrations/2026xxxx_001_audit_log.sql
docs/migrations/2026xxxx_002_soft_delete.sql
docs/migrations/2026xxxx_003_prevent_hard_delete.sql
docs/migrations/2026xxxx_004_optimistic_lock.sql
docs/migrations/2026xxxx_005_rpc_security_helpers.sql
services/person.service.ts
services/family.service.ts
services/event.service.ts
rules/person.rules.ts
rules/family.rules.ts
rules/event.rules.ts
app/actions/data.ts
```

## Bước 2.1 — Audit log kéo lên sớm

`docs/migrations/2026xxxx_001_audit_log.sql`:

```sql
CREATE TABLE IF NOT EXISTS public.audit_logs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  table_name TEXT NOT NULL,
  record_id UUID NOT NULL,
  action TEXT NOT NULL CHECK (action IN ('CREATE','UPDATE','DELETE','RESTORE','MIGRATE')),
  changed_by UUID,
  old_data JSONB,
  new_data JSONB,
  changed_at TIMESTAMPTZ DEFAULT NOW()
);
```

Tạo trigger audit cho `persons`, `relationships` trước. Sau này thêm `families`, `events`.

## Bước 2.2 — Soft delete

Thêm cột:

```sql
ALTER TABLE public.persons ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
ALTER TABLE public.persons ADD COLUMN IF NOT EXISTS deleted_by UUID;
ALTER TABLE public.relationships ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
ALTER TABLE public.relationships ADD COLUMN IF NOT EXISTS deleted_by UUID;
```

Sau khi tạo `families/events` sẽ thêm tương tự.

## Bước 2.3 — Chặn hard delete core

```sql
CREATE OR REPLACE FUNCTION public.prevent_hard_delete_core()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  RAISE EXCEPTION 'Hard delete is disabled for genealogy core table %. Use soft delete.', TG_TABLE_NAME;
END;
$$;

DROP TRIGGER IF EXISTS prevent_delete_persons ON public.persons;
CREATE TRIGGER prevent_delete_persons
BEFORE DELETE ON public.persons
FOR EACH ROW EXECUTE FUNCTION public.prevent_hard_delete_core();

DROP TRIGGER IF EXISTS prevent_delete_relationships ON public.relationships;
CREATE TRIGGER prevent_delete_relationships
BEFORE DELETE ON public.relationships
FOR EACH ROW EXECUTE FUNCTION public.prevent_hard_delete_core();
```

Chỉ bật trigger này sau khi đã sửa hết code `.delete()`.

## Bước 2.4 — Grep và thay `.delete()`

Lệnh rà soát:

```bash
grep -R "\.delete()" -n app components lib services utils || true
grep -R "from('persons').delete\|from(\"persons\").delete" -n . || true
grep -R "from('relationships').delete\|from(\"relationships\").delete" -n . || true
```

Thay bằng:

```typescript
await personService.softDeletePerson(id, userId);
await familyService.softDeleteRelationshipLegacy(id, userId);
```

## Bước 2.5 — Optimistic locking

Thêm `version` cho bảng legacy trước:

```sql
ALTER TABLE public.persons ADD COLUMN IF NOT EXISTS version INT NOT NULL DEFAULT 1;
ALTER TABLE public.relationships ADD COLUMN IF NOT EXISTS version INT NOT NULL DEFAULT 1;
```

Trigger:

```sql
CREATE OR REPLACE FUNCTION public.increment_version()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.version = OLD.version + 1;
  RETURN NEW;
END;
$$;
```

Tạo trigger cho `persons`, `relationships`; sau này thêm `families/events`.

## Test bắt buộc

```text
✅ Tạo/sửa person vẫn chạy
✅ Xóa person trong UI thực chất update deleted_at
✅ `.delete()` cũ không còn trong code core
✅ Hard delete bị chặn nếu gọi nhầm
✅ Update với expectedVersion cũ báo conflict
✅ Audit log ghi CREATE/UPDATE/DELETE
```

## Rollback

Nếu lỗi:

```sql
DROP TRIGGER IF EXISTS prevent_delete_persons ON public.persons;
DROP TRIGGER IF EXISTS prevent_delete_relationships ON public.relationships;
DROP TRIGGER IF EXISTS persons_version ON public.persons;
DROP TRIGGER IF EXISTS relationships_version ON public.relationships;
```

Không drop cột `deleted_at/version/audit_logs` ngay. Có thể để lại vì không ảnh hưởng dữ liệu.

---

# PHASE 3 — Person Names chạy song song

## Mục tiêu

Tách tên có cấu trúc nhưng vẫn giữ `persons.full_name` làm nguồn fallback.

## Files cần tạo/sửa

```text
docs/migrations/2026xxxx_010_person_names.sql
scripts/migrate-person-names.ts
compat/personName.compat.ts
services/person.service.ts
components/MemberForm.tsx
components/PersonDetail.tsx
tests/personNames/personNames.test.ts
```

## Migration

```sql
CREATE TYPE public.name_type_enum AS ENUM (
  'birth','courtesy','posthumous','religious','married','nickname','alias'
);

CREATE TABLE IF NOT EXISTS public.person_names (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  person_id UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  type public.name_type_enum NOT NULL DEFAULT 'birth',
  full_text TEXT NOT NULL,
  surname TEXT,
  given_name TEXT,
  language TEXT DEFAULT 'vi',
  is_primary BOOLEAN DEFAULT FALSE,
  note TEXT,
  deleted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_person_names_primary
ON public.person_names(person_id)
WHERE is_primary = TRUE AND deleted_at IS NULL;
```

## Script migrate

```text
persons.full_name → person_names.full_text, type='birth', is_primary=true
Không xóa persons.full_name
Không sửa UI ngay nếu flag chưa bật
```

## Compatibility helper

`compat/personName.compat.ts`:

```typescript
export function getDisplayName(person: Person, names?: PersonName[]) {
  const primary = names?.find(n => n.is_primary && !n.deleted_at);
  return primary?.full_text || person.full_name || 'Chưa rõ tên';
}
```

## Test bắt buộc

```text
✅ Mỗi person có tối đa 1 primary name
✅ Người có full_name được migrate sang person_names
✅ UI tắt flag vẫn hiển thị full_name
✅ UI bật flag hiển thị primary person_name
✅ Sửa tên chính có thể sync ngược `persons.full_name` trong giai đoạn chuyển đổi
```

## Rollback

```text
- Tắt NEXT_PUBLIC_FF_READ_PERSON_NAMES=false
- UI quay về persons.full_name
- Không drop person_names
```

---

# PHASE 4 — Family Model chạy song song, không gán nhầm con

## Mục tiêu

Tạo `families`, `family_parents`, `family_children` nhưng không bỏ `relationships` cũ. Migration phải tránh lỗi tái hôn/con riêng.

## Files cần tạo/sửa

```text
docs/migrations/2026xxxx_020_family_model.sql
docs/migrations/2026xxxx_021_rpc_create_family_unit.sql
docs/migrations/2026xxxx_022_migration_family_review.sql
scripts/family-migration-dry-run.ts
scripts/migrate-to-family-model-safe.ts
scripts/verify-family-migration.ts
compat/family.compat.ts
compat/relationships.compat.ts
utils/graph/buildFromLegacy.ts
utils/graph/buildFromFamilies.ts
utils/graph/buildUnifiedGraph.ts
components/FamilyMigrationReview.tsx
components/FamilyTree.tsx
components/KinshipFinder.tsx
tests/family/familyMigration.test.ts
tests/graph/graphCompatibility.test.ts
```

## Migration schema

```sql
CREATE TYPE public.family_type_enum AS ENUM ('marriage','partnership','unknown');
CREATE TYPE public.family_status_enum AS ENUM ('active','divorced','widowed','separated','ended','unknown');
CREATE TYPE public.parent_role_enum AS ENUM ('husband','wife','partner','parent');
CREATE TYPE public.child_type_enum AS ENUM ('biological','adopted','foster','stepchild','unknown');

CREATE TABLE IF NOT EXISTS public.families (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type public.family_type_enum NOT NULL DEFAULT 'marriage',
  status public.family_status_enum NOT NULL DEFAULT 'active',
  start_year INT,
  end_year INT,
  note TEXT,
  legacy_relationship_id UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ,
  deleted_by UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.family_parents (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  family_id UUID REFERENCES public.families(id) ON DELETE CASCADE NOT NULL,
  person_id UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  role public.parent_role_enum NOT NULL,
  sort_order INT DEFAULT 0,
  UNIQUE(family_id, person_id)
);

CREATE TABLE IF NOT EXISTS public.family_children (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  family_id UUID REFERENCES public.families(id) ON DELETE CASCADE NOT NULL,
  person_id UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  relationship_type public.child_type_enum NOT NULL DEFAULT 'biological',
  sort_order INT DEFAULT 0,
  legacy_relationship_id UUID,
  migration_confidence TEXT CHECK (migration_confidence IN ('certain','review','manual')) DEFAULT 'review',
  UNIQUE(family_id, person_id)
);
```

## Review queue

```sql
CREATE TABLE IF NOT EXISTS public.migration_family_review (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  child_id UUID NOT NULL REFERENCES public.persons(id),
  parent_ids UUID[] NOT NULL,
  candidate_family_ids UUID[] DEFAULT '{}',
  suggested_family_id UUID,
  reason TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','resolved','ignored')),
  resolved_family_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  resolved_at TIMESTAMPTZ
);
```

## RPC create family an toàn

```sql
CREATE OR REPLACE FUNCTION public.create_family_unit(payload JSONB)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  new_family_id UUID;
  child JSONB;
BEGIN
  IF NOT (public.is_admin() OR public.is_editor()) THEN
    RAISE EXCEPTION 'Permission denied';
  END IF;

  INSERT INTO public.families (type, status, note, legacy_relationship_id)
  VALUES (
    COALESCE(payload->>'type','marriage')::public.family_type_enum,
    COALESCE(payload->>'status','active')::public.family_status_enum,
    payload->>'note',
    NULLIF(payload->>'legacy_relationship_id','')::UUID
  ) RETURNING id INTO new_family_id;

  IF NULLIF(payload->>'parent_a_id','') IS NOT NULL THEN
    INSERT INTO public.family_parents (family_id, person_id, role)
    VALUES (
      new_family_id,
      (payload->>'parent_a_id')::UUID,
      COALESCE(payload->>'parent_a_role','parent')::public.parent_role_enum
    );
  END IF;

  IF NULLIF(payload->>'parent_b_id','') IS NOT NULL THEN
    INSERT INTO public.family_parents (family_id, person_id, role)
    VALUES (
      new_family_id,
      (payload->>'parent_b_id')::UUID,
      COALESCE(payload->>'parent_b_role','parent')::public.parent_role_enum
    );
  END IF;

  FOR child IN SELECT * FROM jsonb_array_elements(COALESCE(payload->'children','[]'::jsonb)) LOOP
    INSERT INTO public.family_children (
      family_id, person_id, relationship_type, migration_confidence, legacy_relationship_id
    ) VALUES (
      new_family_id,
      (child->>'id')::UUID,
      COALESCE(child->>'type','biological')::public.child_type_enum,
      COALESCE(child->>'migration_confidence','manual'),
      NULLIF(child->>'legacy_relationship_id','')::UUID
    ) ON CONFLICT (family_id, person_id) DO NOTHING;
  END LOOP;

  INSERT INTO public.audit_logs (table_name, record_id, action, changed_by, new_data)
  VALUES ('families', new_family_id, 'CREATE', auth.uid(), payload);

  RETURN jsonb_build_object('success', true, 'family_id', new_family_id);
EXCEPTION WHEN OTHERS THEN
  RETURN jsonb_build_object('success', false, 'error', SQLERRM);
END;
$$;
```

## Dry-run migration logic

Không dùng logic cũ `.or(parent_a, parent_b)` để gán con. Dùng thuật toán:

```text
1. Đọc toàn bộ marriage relationships.
2. Tạo candidate family cho từng marriage.
3. Đọc toàn bộ child relationships: parent_id → child_id.
4. Với mỗi child:
   a. Lấy tất cả parent_ids của child.
   b. Tìm family có parent set khớp.
   c. Nếu khớp duy nhất → auto assign, confidence='certain'.
   d. Nếu chỉ có 1 parent và parent có đúng 1 family → assign nhưng confidence='review'.
   e. Nếu parent có nhiều family hoặc không có family → đưa vào migration_family_review.
5. Không tự đoán con riêng/tái hôn.
```

## Adapter đọc song song

`compat/family.compat.ts`:

```typescript
export async function getUnifiedFamilies() {
  if (featureFlags.readFamilies) {
    const families = await getFamiliesFromNewSchema();
    if (families.length > 0) return families;
  }
  return getFamiliesFromLegacyRelationships();
}
```

## Test bắt buộc

```text
✅ Cặp A+B có con C → family A+B chứa C
✅ A+B có C, A+D có E → C không bị gán vào A+D, E không bị gán vào A+B
✅ Chỉ mẹ + con → tạo/review family đơn thân đúng
✅ Người có nhiều hôn nhân → không auto-gán child nếu mơ hồ
✅ adopted_child giữ type adopted
✅ relationships cũ không bị xóa
✅ FamilyTree tắt flag vẫn chạy schema cũ
✅ FamilyTree bật flag đọc được schema mới
✅ KinshipFinder kết quả không giảm so với schema cũ
✅ migration_family_review có record cho case mơ hồ
```

## Rollback

Nếu migration family lỗi:

```text
1. Tắt NEXT_PUBLIC_FF_READ_FAMILIES=false và NEXT_PUBLIC_FF_WRITE_FAMILIES=false.
2. UI quay lại relationships cũ.
3. Không drop relationships.
4. Có thể truncate bảng mới nếu chưa dùng production write:
   TRUNCATE family_children, family_parents, families, migration_family_review RESTART IDENTITY CASCADE;
5. Restore DB từ backup nếu đã ảnh hưởng bảng legacy.
```

---

# PHASE 5 — Event Date System chạy song song

## Mục tiêu

Chuyển birth/death/marriage/custom events sang `events`, nhưng không drop cột ngày cũ cho đến khi verify xong.

## Files cần tạo/sửa

```text
docs/migrations/2026xxxx_030_event_system.sql
scripts/migrate-dates-to-events-safe.ts
scripts/verify-event-migration.ts
compat/date.compat.ts
services/event.service.ts
utils/date-parser/normalizeDate.ts
utils/date-parser/parseVietnameseDate.ts
utils/date-parser/parseGedcomDate.ts
utils/calendar/ageCalculation.ts
components/GenealogyDatePicker.tsx
components/PersonTimeline.tsx
tests/date/normalizeDate.test.ts
tests/date/ageCalculation.test.ts
tests/events/eventMigration.test.ts
```

## Schema

```sql
CREATE TYPE public.date_precision_enum AS ENUM ('day','month','year','decade','range','text','unknown');
CREATE TYPE public.date_modifier_enum AS ENUM ('exact','about','before','after','between','from_to','estimated','calculated','interpreted','phrase','unknown');
CREATE TYPE public.calendar_type_enum AS ENUM ('gregorian','lunar','text','unknown');
CREATE TYPE public.event_type_enum AS ENUM (
  'birth','death','marriage','divorce','burial','baptism','confirmation','ordination','graduation',
  'occupation','residence','migration','military','award','retirement','custom'
);

CREATE TABLE IF NOT EXISTS public.events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  type public.event_type_enum NOT NULL,
  title TEXT,
  start_date DATE,
  end_date DATE,
  sort_date DATE,
  date_precision public.date_precision_enum DEFAULT 'unknown',
  date_modifier public.date_modifier_enum DEFAULT 'unknown',
  canonical_calendar public.calendar_type_enum DEFAULT 'unknown',
  date_original_text TEXT,
  date_phrase TEXT,
  lunar_year INT,
  lunar_month INT,
  lunar_day INT,
  lunar_is_leap_month BOOLEAN DEFAULT FALSE,
  place_id UUID,
  place_text TEXT,
  description TEXT,
  family_id UUID REFERENCES public.families(id),
  legacy_person_id UUID,
  legacy_source TEXT,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ,
  deleted_by UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE public.event_role_enum AS ENUM ('principal','child','husband','wife','witness','officiant','deceased','participant');

CREATE TABLE IF NOT EXISTS public.person_events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  person_id UUID REFERENCES public.persons(id) ON DELETE CASCADE NOT NULL,
  event_id UUID REFERENCES public.events(id) ON DELETE CASCADE NOT NULL,
  role public.event_role_enum DEFAULT 'principal',
  UNIQUE(person_id, event_id, role)
);
```

## Helper normalize date dùng chung

`utils/date-parser/normalizeDate.ts`:

```typescript
export function normalizePartialDate(year?: number | null, month?: number | null, day?: number | null) {
  if (!year) return null;

  const pad = (n: number) => String(n).padStart(2, '0');
  const lastDay = (y: number, m: number) => new Date(Date.UTC(y, m, 0)).getUTCDate();

  if (month && day) {
    const d = `${year}-${pad(month)}-${pad(day)}`;
    return { start_date: d, end_date: d, sort_date: d, date_precision: 'day' as const };
  }

  if (month) {
    const start = `${year}-${pad(month)}-01`;
    const end = `${year}-${pad(month)}-${pad(lastDay(year, month))}`;
    const sort = `${year}-${pad(month)}-15`;
    return { start_date: start, end_date: end, sort_date: sort, date_precision: 'month' as const };
  }

  return {
    start_date: `${year}-01-01`,
    end_date: `${year}-12-31`,
    sort_date: `${year}-06-30`,
    date_precision: 'year' as const,
  };
}
```

Dùng helper này cho cả birth và death, tránh lỗi death month/year.

## Migration dates

Quy tắc:

```text
- Nếu đã có event legacy_source='persons.birth' cho person thì không insert lại.
- Nếu birth_year có thì tạo birth event.
- Nếu death_year có thì tạo death event.
- Nếu is_deceased=true nhưng không có death_year thì tạo DEAT fact không date? Tùy UI; khuyên chỉ set living=false, không tạo event date rỗng.
- death_lunar_* chuyển vào death event.
- Không drop birth_year/death_year/death_lunar_* trong phase này.
```

## Compatibility read

`compat/date.compat.ts`:

```typescript
export function getBirthEventCompat(person: Person, events: Event[]) {
  const e = events.find(e => e.type === 'birth');
  if (e) return e;
  return legacyBirthEventFromPerson(person);
}

export function getDeathEventCompat(person: Person, events: Event[]) {
  const e = events.find(e => e.type === 'death');
  if (e) return e;
  return legacyDeathEventFromPerson(person);
}
```

## Age calculation rule

```text
- Người sống: tính tuổi đến ngày hiện tại.
- Người đã mất có death event: tính tuổi đến death event.
- Người đã mất nhưng thiếu death event/date: không tính tuổi; hiển thị `năm sinh – ?`.
- Chỉ biết năm: hiển thị khoảng.
- Chỉ biết tháng/năm: hiển thị khoảng nhưng hẹp hơn chỉ biết năm.
```

## Test bắt buộc

```text
✅ birth 1945 → start 1945-01-01, end 1945-12-31, sort 1945-06-30, precision year
✅ birth 03/1945 → start 1945-03-01, end 1945-03-31, sort 1945-03-15, precision month
✅ death 03/2001 → start/end/sort tháng 3, không thành cả năm
✅ death_lunar_* giữ nguyên trong event
✅ Sinh 1945 mất 2001 → khoảng 55–56 tuổi
✅ Sinh 03/1945 mất 01/2001 → partial đúng
✅ Người sống sinh 1945 → tuổi hiện tại dạng khoảng nếu chỉ biết năm
✅ Người đã mất nhưng thiếu death date → không tính tuổi hiện tại
✅ UI tắt flag vẫn đọc cột cũ
✅ UI bật flag đọc events, fallback cột cũ nếu thiếu event
```

## Rollback

```text
1. Tắt NEXT_PUBLIC_FF_READ_EVENTS=false và NEXT_PUBLIC_FF_WRITE_EVENTS=false.
2. UI quay lại cột birth/death cũ trong persons.
3. Không drop cột cũ.
4. Có thể truncate events/person_events nếu chưa ghi dữ liệu mới production:
   TRUNCATE person_events, events RESTART IDENTITY CASCADE;
5. Nếu đã có dữ liệu mới do user nhập, export events ra JSON trước khi rollback.
```

---

# PHASE 6 — Graph/UI compatibility

## Mục tiêu

FamilyTree và KinshipFinder hoạt động ổn trong cả 2 mô hình.

## Files cần sửa/tạo

```text
utils/graph/buildFromLegacy.ts
utils/graph/buildFromFamilies.ts
utils/graph/buildUnifiedGraph.ts
utils/graph/traverse.ts
utils/graph/kinship.ts
utils/treeHelpers.ts
components/FamilyTree.tsx
components/KinshipFinder.tsx
components/FamilyCard.tsx
tests/graph/legacyGraph.test.ts
tests/graph/familyGraph.test.ts
tests/graph/kinshipRegression.test.ts
```

## Quy tắc

```text
- `utils/treeHelpers.ts` không query trực tiếp relationships/families nữa.
- Nó nhận `UnifiedGraph` từ adapter.
- `FamilyTree.tsx` không cần biết nguồn là legacy hay family model.
```

Unified model:

```typescript
export interface UnifiedFamily {
  id: string;
  parents: { personId: string; role: string }[];
  children: { personId: string; type: string }[];
  source: 'legacy' | 'family_model';
}
```

## Test bắt buộc

```text
✅ Cây legacy trước migration và cây unified sau migration có cùng số person
✅ Không mất spouse
✅ Không mất child
✅ Multi-spouse hiển thị đúng family center
✅ Kinship parent/child/spouse/sibling không sai so với trước
✅ Case con riêng không bị nối nhầm nếu migration review chưa resolve
```

## Rollback

```text
- Tắt NEXT_PUBLIC_FF_READ_FAMILIES=false.
- Graph dùng buildFromLegacy.
- Không rollback DB.
```

---

# PHASE 7 — GEDCOM Import Staging

## Mục tiêu

Import GEDCOM không ghi thẳng vào bảng chính. Luôn parse → staging → preview → user xác nhận → commit.

## Files cần tạo/sửa

```text
docs/migrations/2026xxxx_040_gedcom_import_staging.sql
docs/migrations/2026xxxx_041_rpc_commit_gedcom_import.sql
utils/gedcom/tokenizer.ts
utils/gedcom/parser.ts
utils/gedcom/normalizer.ts
utils/gedcom/mapper.ts
utils/gedcom/validator.ts
utils/gedcom/compat/familygem.ts
services/gedcom-import.service.ts
components/GedcomImportPreview.tsx
app/dashboard/import/page.tsx
app/api/import/gedcom/route.ts
tests/gedcom/importParser.test.ts
tests/gedcom/importStaging.test.ts
tests/gedcom/importRoundtrip.test.ts
```

## Schema staging

```sql
CREATE TABLE IF NOT EXISTS public.import_sessions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  filename TEXT NOT NULL,
  gedcom_version TEXT,
  source_app TEXT,
  status TEXT NOT NULL DEFAULT 'parsed' CHECK (status IN ('parsed','validated','committed','failed','cancelled')),
  summary JSONB DEFAULT '{}'::jsonb,
  created_by UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  committed_at TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS public.import_xrefs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  import_session_id UUID REFERENCES public.import_sessions(id) ON DELETE CASCADE NOT NULL,
  gedcom_xref TEXT NOT NULL,
  gedcom_type TEXT NOT NULL,
  local_table TEXT,
  local_id UUID,
  UNIQUE(import_session_id, gedcom_xref)
);

CREATE TABLE IF NOT EXISTS public.import_warnings (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  import_session_id UUID REFERENCES public.import_sessions(id) ON DELETE CASCADE NOT NULL,
  severity TEXT NOT NULL CHECK (severity IN ('info','warning','error')),
  code TEXT NOT NULL,
  message TEXT NOT NULL,
  gedcom_xref TEXT,
  payload JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.import_staging_persons (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  import_session_id UUID REFERENCES public.import_sessions(id) ON DELETE CASCADE NOT NULL,
  gedcom_xref TEXT NOT NULL,
  payload JSONB NOT NULL,
  candidate_person_id UUID,
  action TEXT DEFAULT 'create' CHECK (action IN ('create','merge','skip')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(import_session_id, gedcom_xref)
);

CREATE TABLE IF NOT EXISTS public.import_staging_families (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  import_session_id UUID REFERENCES public.import_sessions(id) ON DELETE CASCADE NOT NULL,
  gedcom_xref TEXT NOT NULL,
  payload JSONB NOT NULL,
  action TEXT DEFAULT 'create' CHECK (action IN ('create','merge','skip','review')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(import_session_id, gedcom_xref)
);

CREATE TABLE IF NOT EXISTS public.import_staging_events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  import_session_id UUID REFERENCES public.import_sessions(id) ON DELETE CASCADE NOT NULL,
  gedcom_xref TEXT,
  owner_xref TEXT,
  payload JSONB NOT NULL,
  action TEXT DEFAULT 'create' CHECK (action IN ('create','merge','skip','review')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Import pipeline

```text
1. User upload .ged
2. tokenizer/parser tạo GedcomDocument AST
3. normalizer xử lý NAME, DATE, NOTE/CONT/CONC, SOUR, OBJE, FAM/FAMC/FAMS
4. validator tạo warnings/errors
5. mapper ghi staging tables, không ghi bảng chính
6. UI preview hiển thị:
   - số INDI/FAM/events/media/source
   - warning/error
   - duplicate candidates
   - FAM/FAMC/FAMS conflict
7. User xác nhận
8. RPC commit staging → bảng chính trong transaction
9. import_xrefs ghi map @I1@/@F1@ → UUID
10. Report kết quả
```

## GEDCOM parser bắt buộc xử lý

```text
✅ HEAD/CHAR UTF-8/BOM
✅ INDI/NAME/SURN/GIVN/SEX
✅ BIRT/DEAT/BURI/OCCU/RESI/MARR/DIV
✅ FAM/HUSB/WIFE/CHIL
✅ INDI.FAMC và INDI.FAMS
✅ DATE: exact, ABT, BEF, AFT, BET...AND, FROM...TO, EST, CAL, INT, phrase
✅ PLAC giữ place_text trước, normalize places sau
✅ NOTE inline và NOTE xref
✅ CONT = newline
✅ CONC = nối cùng dòng
✅ SOUR ở record-level và event-level
✅ OBJE FILE path missing không làm import fail
✅ _GIAPHA_LUNAR và legacy _LUNAR
```

## Commit RPC nguyên tắc

```text
- Nếu validation có error severity='error' thì không commit.
- Commit theo thứ tự: persons → person_names → families → family_parents → family_children → events → person_events → sources/media.
- Mỗi xref được ghi vào import_xrefs.
- Nếu lỗi transaction thì rollback toàn bộ commit.
- Staging vẫn giữ để xem lỗi.
```

## Test bắt buộc

```text
✅ Import FAM có HUSB/WIFE/CHIL không mất parent
✅ INDI.FAMC/FAMS cross-check warning đúng
✅ Child có FAMC nhưng FAM thiếu CHIL → warning
✅ FAM có CHIL nhưng child thiếu FAMC → warning
✅ Một người nhiều FAMS → nhiều family đúng
✅ Single parent family đúng
✅ Adopted child giữ adopted
✅ NOTE CONT/CONC giữ đúng newline/nối dòng
✅ SOUR event-level không bị mất
✅ OBJE missing path không fail
✅ _GIAPHA_LUNAR roundtrip
✅ FamilyGem sample import pass
✅ Export từ Giapha → import lại → không mất family/date/name/lunar
```

## Rollback

Nếu parse/staging lỗi:

```text
- Xóa import_session hoặc set status='cancelled'.
- Staging cascade delete.
- Không ảnh hưởng bảng chính.
```

Nếu commit lỗi:

```text
- Transaction rollback tự động.
- import_session status='failed'.
- Bảng chính không đổi.
```

Nếu commit đã thành công nhưng phát hiện sai:

```text
1. Dùng import_xrefs theo session để liệt kê local_id đã tạo.
2. Soft delete các records đã tạo theo session.
3. Không hard delete persons/families/events.
4. Nếu lỗi nghiêm trọng: restore database.sql từ backup Phase 0.
```

---

# PHASE 8 — Media, Source, Place sau core

## Mục tiêu

Sau khi Family/Event/GEDCOM core ổn mới nâng media/source/place.

## Files

```text
docs/migrations/2026xxxx_050_media_system.sql
docs/migrations/2026xxxx_051_source_citation.sql
docs/migrations/2026xxxx_052_place_system.sql
scripts/migrate-avatars-to-media.ts
scripts/seed-vietnam-places.ts
components/MediaGallery.tsx
components/SourceBadge.tsx
components/PlaceAutocomplete.tsx
```

## Thứ tự

```text
1. Media schema
2. Migrate avatar_url → media/person_media, không xóa avatar_url
3. Source/citation schema
4. Place schema
5. PlaceAutocomplete dùng place_text fallback
```

## Test bắt buộc

```text
✅ Avatar cũ vẫn hiển thị nếu media chưa có
✅ Media mới hiển thị nếu có
✅ privacy public/family/private đúng
✅ Citation gắn được person/event/family/person_name
✅ place_text không mất khi chưa normalize
```

## Rollback

```text
- Tắt UI media/source/place mới.
- Fallback avatar_url/place_text.
- Không drop schema mới.
```

---

# PHASE 9 — Search, Data Quality, Production

## Search setup sửa lỗi v2.2.1

```sql
CREATE EXTENSION IF NOT EXISTS unaccent;

CREATE OR REPLACE FUNCTION public.remove_diacritics(input TEXT)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE
AS $$
  SELECT lower(public.unaccent(input));
$$;
```

Materialized view phải có unique index nếu dùng refresh concurrently:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS person_search_index_id_uidx
ON public.person_search_index(id);

CREATE INDEX IF NOT EXISTS person_search_index_all_names_gin
ON public.person_search_index USING gin(to_tsvector('simple', all_names_search));
```

## Data Quality Dashboard

Kiểm tra:

```text
- Person không có tên chính
- Person không có birth/death nhưng legacy có
- Family không có parent
- Family child bị trùng
- Child trong nhiều family nhưng không có lý do
- Event không có owner
- Event date start > end
- Person birth after death
- Parent age bất thường
- GEDCOM import warnings chưa xử lý
- Media orphan
- Source/citation orphan
```

## Production checklist

```text
✅ Backup mới nhất có thể restore
✅ `npm run test` pass
✅ `npm run typecheck` pass
✅ GEDCOM roundtrip pass
✅ Family migration verify pass
✅ Event migration verify pass
✅ Data quality không có error nghiêm trọng
✅ Feature flags bật đúng thứ tự
✅ Có rollback script
✅ Admin đã kiểm tra migration_family_review
```

---

# 10. Sprint plan v2.3

| Sprint | Mục tiêu | Files chính | Pass criteria | Rollback |
|---|---|---|---|---|
| 0 | Backup + baseline verify | `scripts/backup-*`, `verify-current-data.ts` | Có backup + report | Revert code |
| 1 | GEDCOM hotfix hiện tại | `utils/gedcom/*`, `ExportButton` | Export đúng Unicode/tên/âm lịch | Tắt flag exporter |
| 2 | Safety DB/service | audit, soft delete, version, rules | Không còn hard delete core | Drop trigger/tắt service mới |
| 3 | Person Names song song | `person_names`, compat | UI fallback ổn | Tắt flag person names |
| 4 | Family schema + dry-run | `families`, dry-run script | Report không auto-gán nhầm | Truncate schema mới |
| 5 | Family adapter + review UI | graph compat, review UI | Tree/Kinship không gãy | Tắt flag readFamilies |
| 6 | Event schema + migrate | events, date helpers | Birth/death migrate đúng | Tắt flag readEvents |
| 7 | Date UI + age | GenealogyDatePicker, Timeline | Age partial đúng | Tắt event UI |
| 8 | GEDCOM import staging | import_sessions/staging | Preview không ghi DB chính | Delete staging session |
| 9 | GEDCOM commit + roundtrip | commit RPC, import_xrefs | Import FamilyGem không mất parent | Soft delete by import session |
| 10 | Media/source/place | media/source/place schema | Không mất avatar/place_text | Fallback legacy |
| 11 | Search/data quality/prod | search index, dashboard | Quality pass | Tắt dashboard/search mới |

---

# 11. File inventory v2.3

## Tạo mới

```text
lib/featureFlags.ts

compat/
  personName.compat.ts
  family.compat.ts
  relationships.compat.ts
  date.compat.ts

services/
  person.service.ts
  family.service.ts
  event.service.ts
  gedcom-import.service.ts
  migration.service.ts

rules/
  person.rules.ts
  family.rules.ts
  event.rules.ts

utils/gedcom/
  index.ts
  tokenizer.ts
  parser.ts
  normalizer.ts
  mapper.ts
  exporter.ts
  writer.ts
  validator.ts
  date.ts
  name.ts
  notes.ts
  sources.ts
  media.ts
  compat/familygem.ts
  compat/gramps.ts
  compat/legacy551.ts

utils/graph/
  buildFromLegacy.ts
  buildFromFamilies.ts
  buildUnifiedGraph.ts
  kinship.ts
  traverse.ts

utils/date-parser/
  normalizeDate.ts
  parseVietnameseDate.ts
  parseGedcomDate.ts
  formatGenealogyDate.ts

utils/calendar/
  canChi.ts
  lunarDate.ts
  ageCalculation.ts
  deathAnniversary.ts

scripts/
  backup-db.sh
  backup-json.ts
  backup-gedcom.ts
  restore-db.sh
  verify-current-data.ts
  family-migration-dry-run.ts
  migrate-to-family-model-safe.ts
  verify-family-migration.ts
  migrate-dates-to-events-safe.ts
  verify-event-migration.ts
  migrate-person-names.ts
  migrate-avatars-to-media.ts
  seed-vietnam-places.ts

components/
  FamilyMigrationReview.tsx
  GenealogyDatePicker.tsx
  PersonTimeline.tsx
  FamilyCard.tsx
  GedcomImportPreview.tsx
  DataQualityDashboard.tsx
  MediaGallery.tsx
  SourceBadge.tsx
  PlaceAutocomplete.tsx
```

## Sửa file hiện có

```text
utils/gedcom.ts
utils/treeHelpers.ts
components/FamilyTree.tsx
components/KinshipFinder.tsx
components/ExportButton.tsx
components/MemberForm.tsx
components/PersonDetail.tsx
app/actions/data.ts
app/dashboard/members/page.tsx
app/dashboard/kinship/page.tsx
app/api/export/gedcom/route.ts
package.json
.env.local.example
```

## Migrations

```text
docs/migrations/
  2026xxxx_001_audit_log.sql
  2026xxxx_002_soft_delete.sql
  2026xxxx_003_prevent_hard_delete.sql
  2026xxxx_004_optimistic_lock.sql
  2026xxxx_005_rpc_security_helpers.sql
  2026xxxx_010_person_names.sql
  2026xxxx_020_family_model.sql
  2026xxxx_021_rpc_create_family_unit.sql
  2026xxxx_022_migration_family_review.sql
  2026xxxx_030_event_system.sql
  2026xxxx_040_gedcom_import_staging.sql
  2026xxxx_041_rpc_commit_gedcom_import.sql
  2026xxxx_050_media_system.sql
  2026xxxx_051_source_citation.sql
  2026xxxx_052_place_system.sql
  2026xxxx_060_search_setup.sql
```

## Tests

```text
tests/
  smoke/currentData.test.ts
  gedcom/exporter.test.ts
  gedcom/parser.test.ts
  gedcom/roundtrip.test.ts
  gedcom/importParser.test.ts
  gedcom/importStaging.test.ts
  personNames/personNames.test.ts
  family/familyMigration.test.ts
  graph/legacyGraph.test.ts
  graph/familyGraph.test.ts
  graph/kinshipRegression.test.ts
  date/normalizeDate.test.ts
  date/ageCalculation.test.ts
  events/eventMigration.test.ts
  rules/personRules.test.ts
  rules/familyRules.test.ts
  rules/eventRules.test.ts
```

---

# 12. Cleanup cuối cùng — chỉ làm sau khi production ổn

Không cleanup sớm. Chỉ sau khi:

```text
- Đã chạy production ít nhất 2 vòng backup.
- Data Quality không còn lỗi nghiêm trọng.
- FamilyTree/Kinship/Event UI dùng schema mới ổn.
- Export/import GEDCOM roundtrip pass.
- Không còn code đọc trực tiếp cột legacy trừ compat fallback.
```

Mới được xem xét:

```text
- Drop birth_year/birth_month/birth_day/death_year/death_month/death_day/death_lunar_* khỏi persons.
- Không drop relationships ngay; nên giữ read-only thêm một thời gian.
- Nếu drop relationships, phải export backup riêng trước.
```

Cleanup migration phải có file riêng:

```text
docs/migrations/2026xxxx_999_legacy_cleanup.sql
```

Và phải có rollback bằng restore DB, vì drop column không rollback nhẹ được.

---

# 13. Quy tắc triển khai thực tế

## Thứ tự bật feature flags

```text
1. FF_GEDCOM_EXPORTER_V23=true
2. FF_READ_PERSON_NAMES=true
3. FF_WRITE_FAMILIES=true trong admin/internal only
4. Chạy family dry-run và review
5. FF_READ_FAMILIES=true
6. FF_WRITE_EVENTS=true trong admin/internal only
7. Chạy event migration
8. FF_READ_EVENTS=true
9. FF_GEDCOM_IMPORT_STAGING=true
```

Không bật `READ_FAMILIES` trước khi `verify-family-migration.ts` pass.

Không bật `READ_EVENTS` trước khi `verify-event-migration.ts` pass.

## Khi có lỗi production

Ưu tiên rollback mềm:

```text
1. Tắt feature flag.
2. Revert code deploy.
3. Không restore DB nếu dữ liệu legacy còn nguyên.
4. Chỉ restore DB khi legacy data bị ảnh hưởng hoặc migration ghi sai diện rộng.
```

---

# 14. Kết quả kỳ vọng sau v2.3

| Hạng mục | Trước | Sau v2.3 |
|---|---|---|
| GEDCOM tên Việt Nam | `Nguyễn Văn /An/` sai | `Văn An /Nguyễn/`, có SURN/GIVN |
| GEDCOM Unicode | dễ lỗi font | UTF-8 BOM + CHAR UTF-8 + CRLF |
| Âm lịch GEDCOM | mất hoặc tag chung | `_GIAPHA_LUNAR`, parser đọc legacy `_LUNAR` |
| Note dài tiếng Việt | có thể vượt 255 bytes | wrap đúng bằng CONT/CONC theo byte |
| Family migration | có thể gán nhầm con | dry-run + review queue + chỉ auto khi chắc |
| Tái hôn/con riêng | nguy cơ sai | không auto-gán nếu mơ hồ |
| Date chỉ năm/tháng | dễ hiển thị sai | start/end/sort/precision rõ ràng |
| Death month/year | v2.2.1 còn lỗi | normalize chung birth/death |
| GEDCOM import | ghi thẳng/rủi ro | staging + preview + commit RPC |
| Rollback | chưa rõ | flag + truncate schema mới + restore backup |
| Tree/Kinship | dễ gãy khi đổi schema | unified graph adapter |
| Soft delete | chỉ thêm cột | service + grep + optional trigger chặn hard delete |
| Search | thiếu unique index | refresh concurrently an toàn |
| Production | backup cuối phase | backup ngay Phase 0 |

---

# 15. Kết luận

Roadmap v2.3 là bản nên dùng để triển khai thật. Bản này không phủ nhận v2.2.1; ngược lại, nó giữ lại phần GEDCOM export rất tốt của v2.2.1, nhưng đặt vào một quy trình an toàn hơn:

```text
GEDCOM hotfix trước
Safety layer trước migration
Family/Event chạy song song
Import GEDCOM qua staging
UI đọc qua compatibility adapter
Test bắt buộc trước khi bật flag
Rollback rõ từng phase
```

Nguyên tắc quan trọng nhất: **không drop dữ liệu cũ cho đến khi schema mới đã chạy ổn, có backup, có verify report, và có đường quay lại.**
