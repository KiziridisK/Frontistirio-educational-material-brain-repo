# Frontistirio — Educational Material Brain

> **Full-stack reference** for file upload, S3 storage, permission system, and pre-signed URL delivery. Covers multer file handling, the exact S3 folder structure, permission filtering, and the Angular frontend service/components.

---

## Change log

### 2026-06-25 — Course-level permissions + modal redesign
Added a third permission level (**Course / μάθημα**) so a material can be restricted to the
students of a specific course within a class — students who don't take that course no longer
get access. Also redesigned the assignment modal into a 3-level cascade and fixed a latent
access bug.

**Backend (`Frontistirio-API`)**
- `models/educational-material.js` — `period_permissions[].courses: [ObjectId → Course]`.
- `controllers/handlers/educational-material.js`
  - `createStoreEducationalMaterial` + `upsertPeriodPermission` now persist `courses`.
  - `getStudentEducationalMaterial` rewritten to enforce **Grade AND Class AND Course**
    (reads `student.period_courses`). Fixed BUG-011 (grade was never actually enforced).

**Frontend (`Frontistirio`)**
- `models/educationalMaterial.model.ts` — `courses: string[]` in the permission object.
- `add-educational-material.component.{ts,html,scss}` — cascading Grade→Class→Course selects
  with auto-pruning, stepper UI, file dropzone, Visibility card; removed TEST button + dead
  code; added `ngOnDestroy`.
- `assets/i18n/{el,en}.json` — added the missing + new permission keys.

**Compatibility:** no migration — existing entries lack `courses`, treated as "no course
restriction". Requires a Node server restart for the schema change. `node --check` clean;
frontend `ng build` (dev) exit 0.

**Not done yet (deferred):** parent/teacher material delivery (the `visibleTo*` flags + the
`parent`/`teacher` bootstrap branches) — see BUG-010. Course-level access for those roles
applies once that's built.

---

## Backend: MongoDB Model (`models/educational-material.js`)

```javascript
{
  name: String,
  description: String,
  s3Key: String,            // S3 object key (used for download/delete)
  s3Url: String,            // Base S3 URL (not used for access)
  bucket: String,           // S3 bucket name
  filetype: String,         // MIME type (e.g. 'application/pdf', 'image/jpeg')
  filename: String,         // original filename (UTF-8 decoded)

  period_permissions: [{
    period: ObjectId → TeachingPeriod,
    grades: [ObjectId → Grade],
    classes: [ObjectId → Class],
    courses: [ObjectId → Course],     // ← added 2026-06-25 (course-level access)
    visibleToTeachers: Boolean (default: true),
    visibleToParents: Boolean (default: false)
  }],

  store_id: ObjectId → Store,
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  timestamps: true
}
```

---

## Backend: Routes (`routes/educational-material.js`)

Multer configuration uses **disk storage** with UTF-8 filename fix:
```javascript
const storage = multer.diskStorage({
  destination: 'uploads/',
  filename: (req, file, cb) => {
    const utf8Name = Buffer.from(file.originalname, 'latin1').toString('utf8');
    const safeName = Date.now() + '-' + utf8Name.replace(/\s+/g, '_');
    cb(null, safeName);
  }
});
const upload = multer({ storage });
```

| Method | Path | Middleware | Notes |
|---|---|---|---|
| GET | `/educational-material/get-educational-material/:materialId` | auth, isStoreOrStudentUser | Students can access |
| POST | `/educational-material/create-store-educational-material` | auth, isStoreUser, upload.single('file') | Multer parses file |
| POST | `/educational-material/edit-store-educational-material` | auth, isStoreUser, upload.single('file') | Optional file replace |
| DELETE | `/educational-material/delete-educational-material` | auth, isStoreUser | Soft delete + S3 delete |

---

## Backend: Upload Flow (`createStoreEducationalMaterial`)

```javascript
// 1. Validate: user, store, group must all be present
if (!user_id || !store || !group) throw new Error('unauthorized_or_missing_store_info');
if (!req.file) throw new Error('missing_file');

// 2. Parse period_permissions from FormData string
let period_permissions = JSON.parse(req.body.period_permissions);
const permission = period_permissions?.[0] ?? { grades:[], classes:[], ... };

// 3. Get default period
const activePeriodId = (await getStoreDefaultPeriod(req.user.store))._id;

// 4. Read local file written by multer
const fileContent = fs.readFileSync(req.file.path);
// Fix encoding: multer writes latin1, original name needs UTF-8
const filename = Buffer.from(req.file.originalname, 'latin1').toString('utf8');

// 5. Define S3 key with folder structure:
// "educational-materials/{group_id}/{store_id}/{timestamp}_{filename}"
const s3FolderPath = `educational-materials/${group_id}/${store_id}/`;
const s3Key = `${s3FolderPath}${Date.now()}_${filename}`;

// 6. Upload to S3 (bucket: "logeion-educational-material")
await s3Client.send(new PutObjectCommand({
  Bucket: 'logeion-educational-material',
  Key: s3Key, Body: fileContent, ContentType: req.file.mimetype
}));

// 7. Delete local temp file
fs.unlinkSync(req.file.path);

// 8. Save EducationalMaterial document with s3Key, s3Url, bucket, filename, permission
const educational_material = await educationalMaterialHandler
  .createStoreEducationalMaterial(fileData, s3Key, s3Url, bucket, filename,
    store_id, user_id, activePeriodId, permission, session);

// 9. Respond
res.json({ success: true, data: { name, description, filetype, s3Key, educational_material, s3Url } });
```

**Important:** The temp file at `uploads/` is deleted after S3 upload (`fs.unlinkSync`). The local file is only a staging area.

---

## Backend: Download Flow (`getStoreEducationalMaterial`)

```javascript
// 1. Fetch material document
const material = await educationalMaterialHandler.getEducationalMaterial(params.materialId);

// 2. Build filename with extension
const materialExt = material.filetype?.split('/')[1] || 'bin';
const filename = `${material.name}.${materialExt}`;

// 3. Generate pre-signed URL (60s expiry)
const command = new GetObjectCommand({
  Bucket: material.bucket,
  Key: material.s3Key,
  ResponseContentDisposition: `attachment; filename="${filename}"`
});
const signedUrl = await getSignedUrl(s3Client, command, { expiresIn: 60 });

// 4. Return URL (S3 serves the file directly to the browser)
res.json({ success: true, downloadUrl: signedUrl, filename, filetype: material.filetype });
```

---

## Backend: Permission System

### Three-level cascade: Grade → Class → Course
Access is stored per period as `{ grades[], classes[], courses[] }` and narrows
**progressively** — each level is an additional AND-constraint, not an alternative:

- **Grades** — which grades (e.g. Γ ΓΕΛ) the material targets. Required: a material with
  an empty `grades[]` is visible to **no** student.
- **Classes** — optionally restrict to specific classes/τμήματα of those grades. Empty ⇒
  grade-level access is enough.
- **Courses** — optionally restrict to students taking specific courses/μαθήματα. Empty ⇒
  class-level access is enough. This is what lets a file reach only the students of one
  course within a class (e.g. only those taking Πληροφορική in *Γ ΓΕΛ Οικονομικών*).

### For Store-Users
`getStoreEducationalMaterials(storeId)` — returns all materials for the store, no permission
filtering. Store-users can see everything. (NB still missing `isDeleted:false` — BUG-005.)

### For Students
`getStudentEducationalMaterial(storeId, student, periodId)` resolves the student's
`period_grade`, `period_class` **and `period_courses`** for the active period, then keeps a
material only if **all** of these hold:
1. `perm.grades` is non-empty **and** contains the student's grade.
2. If `perm.classes` is non-empty, it contains the student's class.
3. If `perm.courses` is non-empty, the student takes at least one of those courses.

> ⚠️ Before 2026-06-25 the grade match was computed (`exists`) but never used in the return
> decision, so access was driven by class only and the grade level was effectively ignored —
> see **BUG-011** (now fixed alongside the course-level rewrite).

### `visibleToTeachers` / `visibleToParents`
Permission flags persisted per entry but **not consumed anywhere yet** — `bootstrap.js` has
`parent` / `teacher` role branches that fetch **no** materials. Course-level visibility for
those roles applies once that delivery is built (see BUG-010).

### Saving permissions
Both write paths persist `courses` (mapped to ObjectIds):
- `createStoreEducationalMaterial` handler — initial `period_permissions` entry.
- `upsertPeriodPermission` handler — `permDoc.courses` + `"period_permissions.$.courses"` in
  the positional `$set` (and pushed in the new-entry branch).

---

## Backend: Handler Exports

```javascript
exports.createStoreEducationalMaterial(fileData, s3Key, s3Url, bucket, filename, store_id, user_id, periodId, permission, session)
exports.editStoreEducationalMaterial(materialId, fileData, s3Key, s3Url, bucket, filename, periodId, permission, session)
exports.getEducationalMaterial(materialId)
exports.getStoreEducationalMaterials(storeId)          // all for store-user
exports.getStudentEducationalMaterial(storeId, student, periodId) // filtered for student
exports.deleteStoreEducationalMaterial(materialId, permanently, session)
exports.watchEducationalMaterial(io, userSockets)       // Change Stream
```

---

## Backend: Real-Time Watcher

`watchEducationalMaterial` emits `materialAdded`, `materialUpdated`, `materialDeleted` to store-users of the affected store. Same pattern as other watchers.

---

## Frontend: TypeScript Model (`models/educationalMaterial.model.ts`)

```typescript
interface EducationalMaterial {
  _id?: string;
  name: string;
  description?: string;
  s3Key?: string; s3Url?: string; bucket?: string;
  filetype?: string; filename?: string;
  period_permissions?: {
    period?: string | TeachingPeriod;
    grades: string[];
    classes: string[];
    courses: string[];               // ← added 2026-06-25
    visibleToTeachers: boolean;
    visibleToParents: boolean;
  }[];
  store_id?: string;
  isDeleted?: boolean;
}
```

---

## Frontend: NgRx State

```typescript
// Slice: educational_materials
interface EducationalMaterialState { educational_materials: EducationalMaterial[] }

// Actions:
setEducationalMaterial({ materials })          // bootstrap
addEducationalMaterial({ material })           // socket: materialAdded
updateMaterialFields({ id, changes })          // socket: materialUpdated
deleteEducationalMaterial({ id })              // socket: materialDeleted

// Selector:
selectEducationalMaterials → Observable<EducationalMaterial[]>
```

---

## Frontend: Service (`services/educational-material.service.ts`)

### Upload (Create)
```typescript
addEducationalMaterial(material, file?):
  const formData = new FormData();
  formData.append('file', file, file.name);     // binary file
  formData.append('name', material.name);
  formData.append('description', material.description || '');
  formData.append('period_permissions', JSON.stringify(material.period_permissions));
  formData.append('filetype', material.filetype);
  // Angular auto-sets Content-Type: multipart/form-data with boundary
  → POST /educational-material/create-store-educational-material
```

### Edit
```typescript
editEducationalMaterial(material, newFile: boolean, file?):
  // newFile=true → includes file in FormData → backend replaces S3 object
  // newFile=false → no file → backend keeps existing S3 object
  formData.append('newFile', String(newFile));
  formData.append('materialId', material._id);
  → POST /educational-material/edit-store-educational-material
```

### Download
```typescript
getStoreEducationalMaterial(materialId)
  → GET /educational-material/get-educational-material/:materialId
  → returns { success, downloadUrl, filename, filetype }
  // Frontend:
  window.open(downloadUrl, '_blank');
  // or: <a [href]="downloadUrl" download>Download</a>
```

### Delete
```typescript
deleteEducationalMaterial(materialId, permanently)
  → DELETE /educational-material/delete-educational-material
  body: { materialId, permanently }
```

---

## Frontend: Components

### `EducationalMaterialComponent` (`/educational-material`)
- Route accessible to `store-user` AND `student` (RoleGuard: `['store-user', 'student']`)
- Lists materials from NgRx `selectEducationalMaterials`
- Store-user sees all; student sees only permitted materials (filtering done server-side in bootstrap)
- Shows `EducationalMaterialItemComponent` cards

### `AddEducationalMaterialComponent`
- Metadata form (name, description) + styled file dropzone.
- **Permission UI = 3-level cascade** (redesigned 2026-06-25): a numbered "stepper" of
  Grade → Class → Course, each with a selected-count badge and Select-all/Clear.
  - `filteredClasses` getter → classes whose `grade_id` ∈ selected grades (all if none).
  - `filteredCourses` getter → courses by `grade_id`, further narrowed to those taught in a
    selected class via `course.period_classes` (union across periods).
  - `onGradesChange()` / `onClassesChange()` **prune** downstream selections that become
    invalid, so no orphan class/course ids are saved.
- Separate **Visibility** card with teacher/parent toggles.
- On save: `material.period_permissions = [activePermission]` (still a single-entry array;
  the backend takes `[0]` and attaches it to the store's default period).
- Calls `addEducationalMaterial(material, file)` / `editEducationalMaterial(...)`.
- **Cleanup done in the same pass:** removed the debug `TEST` button + `test()`, the dead
  `addPeriodPermission` / `removePeriodPermission`, and added `ngOnDestroy` unsubscribes
  (grades/classes/courses subs).
- **i18n:** added the previously-missing keys (`educational_material.permissions`,
  `permissions_subtitle`, `permissions_hint`, `visible_to_*`) plus the new cascade keys
  (`level_grade/class/course` + hints, `selected_count`, `no_courses_for_selection`,
  `visibility`, `visibility_hint`) to **both** `el.json` and `en.json`. Cancel button uses
  the existing `general.cancel`; Save keeps the app-wide literal `"SAVE"`.

### `EducationalMaterialItemComponent`
- Shows material name, description, filetype icon
- Download button: calls `getStoreEducationalMaterial(materialId)` → opens `downloadUrl` in new tab

---

## S3 Folder Structure

```
logeion-educational-material/
  └── educational-materials/
        └── {group_id}/
              └── {store_id}/
                    └── {timestamp}_{original_filename}
```

Separate bucket from private lesson PDFs (`logeion-private-lesson-costs`).
