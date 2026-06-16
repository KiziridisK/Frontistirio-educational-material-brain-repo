# Frontistirio — Educational Material Brain

> **Full-stack reference** for file upload, S3 storage, permission system, and pre-signed URL delivery. Covers multer file handling, the exact S3 folder structure, permission filtering, and the Angular frontend service/components.

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
    classes: [ObjectId → ClassModel],
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

### For Store-Users
`getStoreEducationalMaterials(storeId)` — returns all non-deleted materials for the store, no permission filtering. Store-users can see everything.

### For Students
`getStudentEducationalMaterial(storeId, student, periodId)`:
1. Find the student's grade and class for the period from `student.period_grade` and `student.period_class`
2. Filter materials where `period_permissions` for the active period includes either:
   - The student's `grade_id` in `grades[]`, OR
   - The student's `class_id` in `classes[]`

### `visibleToTeachers` / `visibleToParents`
Permission flags that future teacher/parent portal implementations can use to filter materials.

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
  period_permissions: [{
    period: string;
    grades: string[];
    classes: string[];
    visibleToTeachers: boolean;
    visibleToParents: boolean;
  }];
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
- File picker (`<input type="file">`) + metadata form
- Permission UI: select period, then grades/classes, toggle teacher/parent visibility
- Calls `addEducationalMaterial(material, file)`
- File is read as `File` object from the input

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
