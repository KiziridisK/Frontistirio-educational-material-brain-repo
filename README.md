# Frontistirio — Educational Material Brain

> **Purpose:** Documents how educational files (PDFs, documents, etc.) are uploaded, stored in AWS S3, permissioned by grade/class/role, and served to users via pre-signed URLs.

---

## Overview

Educational materials are files (any type) uploaded by store admins and made available to students, teachers, and/or parents. Files are stored in **AWS S3** and never served directly — all access goes through **pre-signed URLs** with short expiry.

Access is controlled by **period-scoped permissions** (which grades/classes can see the file, and whether teachers and parents can see it).

---

## Model (`models/educational-material.js`)

```
EducationalMaterial {
  name          String
  description   String
  s3Key         String          // S3 object key
  s3Url         String          // Base S3 URL (not used for access)
  bucket        String          // S3 bucket name
  filetype      String          // MIME type (e.g., "application/pdf")
  filename      String

  period_permissions [{
    period              → TeachingPeriod
    grades              → [Grade]     // which grade levels can access
    classes             → [Class]     // which specific classes can access
    visibleToTeachers   Boolean (default: true)
    visibleToParents    Boolean (default: false)
  }]

  store_id      → Store
  isDeleted     Boolean
  deletedAt     Date
  createdBy / updatedBy → User
}
```

---

## API Endpoints (`routes/educational-material.js`)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/educational-material/upload` | Upload a file to S3 | store-user |
| GET | `/educational-material/:materialId` | Get download URL | store-user, student, teacher, parent |
| PUT | `/educational-material/:materialId` | Edit metadata/permissions | store-user |
| DELETE | `/educational-material/:materialId` | Soft-delete material | store-user |
| GET | `/educational-material/` | List all store materials | store-user |
| GET | `/educational-material/student/:studentId` | Get materials accessible to student | student, store-user |

---

## Upload Flow

```
Client → POST /educational-material/upload (multipart/form-data)
         ↓
  multer middleware parses the file
         ↓
  PutObjectCommand to S3
  Key: <generated unique key>
  Bucket: process.env.AWS_BUCKET_NAME
         ↓
  Save EducationalMaterial document with s3Key, bucket, filetype
         ↓
  Return: { success: true, material: { _id, name, ... } }
```

Uses:
- `multer` for multipart file parsing
- `@aws-sdk/client-s3` for S3 operations
- `services/s3Client.js` for the configured S3 client

---

## Download Flow (Pre-signed URL)

```
Client → GET /educational-material/:materialId
         ↓
  Fetch EducationalMaterial document (s3Key, bucket, filetype)
         ↓
  GetObjectCommand with ResponseContentDisposition: attachment; filename="<name>.<ext>"
         ↓
  getSignedUrl(s3Client, command, { expiresIn: 60 })
         ↓
  Return: { downloadUrl, filename, filetype }
         ↓
  Client opens/downloads the pre-signed URL directly from S3
```

**The backend never streams file bytes** — S3 serves the file directly to the client via the pre-signed URL. The URL expires in **60 seconds**.

---

## Permission System

Permissions are stored per teaching period and checked when listing materials for a user:

```javascript
// Example period_permissions entry:
{
  period: "...",
  grades: ["grade_id_1", "grade_id_2"],   // accessible to these grades
  classes: ["class_id_1"],                 // or these specific classes
  visibleToTeachers: true,
  visibleToParents: false
}
```

When a **student** requests their materials:
1. Fetch student's current grade and class for active period
2. Filter materials where `period_permissions` includes their grade or class
3. Return filtered list

When a **parent** requests materials:
1. Check `visibleToParents: true` in relevant permissions
2. Return materials their child's grade/class can see

---

## AWS S3 Configuration

Environment variables (`.env`):
```
AWS_REGION=eu-central-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_BUCKET_NAME=...
```

S3 client initialized in `services/s3Client.js` using `@aws-sdk/client-s3`.

---

## Real-Time Sync

`watchEducationalMaterial(io, userSockets)`:
- MongoDB Change Stream on `EducationalMaterial` collection
- When a new material is added or permissions change, connected store users are notified in real time

---

## File Deletion

Soft delete only on the DB side:
```javascript
material.isDeleted = true;
material.deletedAt = new Date();
```

The S3 object is removed via `DeleteObjectCommand`:
```javascript
await s3Client.send(new DeleteObjectCommand({ Bucket: bucket, Key: s3Key }));
```
