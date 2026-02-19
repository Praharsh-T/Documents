# Agent Prompt: Migrate File Upload from Base64 to Streaming Architecture

I need you to migrate our current base64 file upload system to a production-grade streaming-based file upload architecture. Currently, we receive files as base64 encoded strings which is memory inefficient and has size limitations. 

## CURRENT PROBLEM:
- Files are sent as base64 strings from frontend
- This increases file size by ~33%
- High memory usage on backend (entire file loaded into memory)
- Limited to small files only
- No cloud storage integration

## TARGET ARCHITECTURE:
Implement a cloud-agnostic file upload system using the Strategy Pattern with these features:

### 1. **Multi-Storage Support** (at minimum, implement 2 of these):
- Local filesystem storage (for development)
- Google Cloud Storage (GCP)
- Azure Blob Storage
- AWS S3

### 2. **Streaming Upload Flow**:
- Use `formidable` library to parse multipart/form-data
- Stream files directly to storage (no temp files)
- Use PassThrough streams for memory efficiency
- Support multiple file uploads in single request

### 3. **Access Control**:
- Public mode: Publicly accessible files (avatars, product images)
- Private mode: Authenticated access with presigned URLs

### 4. **File Organization**:
- Context-based folders (e.g., "user-avatars/", "documents/", "invoices/")
- UUID-based unique filenames to prevent collisions
- Preserve original file extensions

### 5. **Security**:
- MIME type validation with whitelist
- File size limits
- Reject files before processing if invalid
- Signed URLs for private file access (15-min expiry)

## REFERENCE IMPLEMENTATION:
Use this structure as your guide:

```
objectStorage/
  ├── types.ts              # Shared types and interfaces
  ├── index.ts              # Dynamic storage adapter loader
  ├── localStorage.ts       # Local filesystem implementation
  ├── cloudStorage.ts       # GCP implementation
  └── azureBlobStorage.ts   # Azure implementation

controllers/
  ├── uploadFiles.ts        # Upload endpoint handler
  └── downloadFile.ts       # Download endpoint handler
```

## KEY IMPLEMENTATION DETAILS:

### 1. **Create Storage Abstraction Layer**:
Each storage adapter must implement these methods:
- `uploadFile(stream, options)` - Upload from a readable stream
- `getUploadFileStream(filename, options)` - Returns writable stream for direct upload
- `generateUrl(key, mode, filename)` - Generate download URL
- `generatePresignedUrl(key, expiryMins, filename)` - Temporary signed URL
- `deleteFiles(keys, mode)` - Batch delete
- `downloadFileAsStream(key, mode)` - Stream file for download
- `downloadFileAsBuffer(key, mode)` - Load file as buffer

### 2. **Upload Controller Pattern**:
```typescript
- Parse route params: mode (public/private), context (folder name)
- Use formidable with custom fileWriteStreamHandler
- In fileWriteStreamHandler: call getUploadFileStream()
- Formidable streams directly to storage (no temp files!)
- After parse complete: save metadata to database
- Return file IDs and download URLs to frontend
```

### 3. **File Metadata Database Schema**:
Store these fields:
- id (UUID)
- fileName (original name)
- mimeType
- storageKey (the path in storage)
- mode (public/private)
- uploadedBy (user ID)
- location (optional JSON - geo coordinates, etc.)
- createdAt, updatedAt

### 4. **Environment Configuration**:
```env
STORAGE_MODE=local|gcp|azure|s3
LOCAL_STORAGE_PATH=./storage
PRIVATE_BUCKET_NAME=my-private-bucket/optional-folder
PUBLIC_BUCKET_NAME=my-public-bucket/optional-folder
AZURE_STORAGE_ACCOUNT_NAME=...
GOOGLE_APPLICATION_CREDENTIALS=...
```

### 5. **MIME Type Whitelist**:
Support at minimum:
- Images: image/png, image/jpeg, image/gif, image/webp
- Documents: application/pdf, text/plain, text/csv
- MS Office: docx, xlsx, pptx
- Videos: mp4, webm (optional)
- Allow extending via EXTRA_MIMETYPES env variable

### 6. **API Endpoints**:
```
POST /upload/:mode/:context
- mode: "public" or "private"
- context: folder name like "user-avatars" or "invoices"
- Body: multipart/form-data with files
- Optional: locations array (JSON) for metadata

GET /download/:fileId?token=xxx
- Returns file stream with proper Content-Type headers
- Validates JWT token for private files
```

### 7. **Frontend Migration**:
Change from base64 to FormData:

```typescript
// OLD (base64):
const base64 = await fileToBase64(file);
fetch('/upload', { 
  body: JSON.stringify({ file: base64 }) 
});

// NEW (multipart):
const formData = new FormData();
formData.append('file', file);
formData.append('locations', JSON.stringify({ lat, lng }));
fetch('/upload/public/user-avatars', { 
  method: 'POST',
  body: formData 
});
```

## STEP-BY-STEP IMPLEMENTATION PLAN:

1. Analyze current codebase to locate base64 upload logic
2. Install dependencies: formidable, uuid, cloud SDK
3. Create objectStorage/ directory with types.ts
4. Implement localStorage.ts first (easiest to test)
5. Create upload controller with formidable streaming
6. Create database migration for file metadata table
7. Implement download controller with stream response
8. Test with local storage thoroughly
9. Implement cloud storage adapters (GCP/Azure/S3)
10. Add dynamic loader in index.ts
11. Update frontend to use FormData instead of base64
12. Add error handling and validation
13. Write tests for each storage adapter
14. Update documentation and environment variables

## VALIDATION CRITERIA:
✅ Files stream directly to storage (verify with console.time logs)  
✅ Memory usage stays low even with large files (test with 100MB+ files)  
✅ Can switch storage backends via env variable  
✅ MIME type validation rejects unauthorized files  
✅ Private files require authentication to download  
✅ Public files are directly accessible  
✅ Multiple files can be uploaded in one request  
✅ Original filenames are preserved in database  
✅ Files organized by context in storage  

## IMPORTANT NOTES:
- Use PassThrough streams to bridge formidable and storage APIs
- Never save files to temp directory first
- Generate UUIDs for storage keys to prevent filename collisions
- For cloud storage, use resumable=false for files <10MB (faster)
- Implement proper error handling for stream failures
- Set appropriate Content-Type and Content-Disposition headers
- Use streaming for downloads too (don't load entire file into memory)

## DETAILED IMPLEMENTATION EXAMPLES:

### Example: localStorage.ts Implementation

```typescript
import fs from "fs";
import path from "path";
import { v4 as uuidv4 } from "uuid";
import Stream, { PassThrough } from "stream";
import { pipeline } from "stream/promises";
import { BlobMode, BlobOptions, defaultMode } from "./types";

const getStoragePath = (mode: BlobMode) => {
  const basePath = process.env.LOCAL_STORAGE_PATH || "./storage";
  return path.join(basePath, mode);
};

const ensureDirectoryExists = (dirPath: string) => {
  if (!fs.existsSync(dirPath)) {
    fs.mkdirSync(dirPath, { recursive: true });
  }
};

export const getUploadFileStream = (filename: string, options: BlobOptions = {}) => {
  const { context, mode = "private" } = options;
  
  const id = uuidv4();
  const extension = filename ? `.${filename.split(".").pop()}` : "";
  const key = `${context ? context + "/" : ""}${id}${extension}`;
  
  const storagePath = getStoragePath(mode);
  const fullPath = path.join(storagePath, key);
  const dirPath = path.dirname(fullPath);
  
  ensureDirectoryExists(dirPath);
  
  const passThrough = new PassThrough();
  const writeStream = fs.createWriteStream(fullPath);
  
  passThrough.pipe(writeStream);
  
  return {
    name: key,
    stream: passThrough,
    id,
  };
};

export const downloadFileAsStream = async (
  key: string,
  mode = "private" as BlobMode | null
) => {
  const storagePath = getStoragePath(mode || "private");
  const fullPath = path.join(storagePath, key);
  
  if (!fs.existsSync(fullPath)) {
    throw new Error(`File not found: ${key}`);
  }
  
  return fs.createReadStream(fullPath);
};
```

### Example: Upload Controller with Formidable

```typescript
import * as formidable from 'formidable';
import { generateUrl, getUploadFileStream } from '../objectStorage';
import { Response } from 'express';
import db from '@/db';
import { v4 as uuid } from 'uuid';

const allowedMimetypes: Record<string, string> = [
  'image/png',
  'image/jpeg',
  'image/gif',
  'image/webp',
  'application/pdf',
  'text/plain',
].reduce((acc, type) => ({ ...acc, [type]: type }), {});

const uploadFiles = (req: ExpressRequest, res: Response): void => {
  const { mode = 'private', context } = req.params;

  const form = new formidable.IncomingForm({
    multiples: true,
    filter: ({ mimetype }) => {
      return Boolean(allowedMimetypes[mimetype || '']);
    },
    fileWriteStreamHandler: (file) => {
      const { stream, name, id } = getUploadFileStream(
        file?.originalFilename || '',
        {
          context: context || '',
          mimetype: file?.mimetype || '',
          mode: mode as 'private' | 'public',
        }
      );
      if (file) {
        file.newFilename = name;
        file.id = id;
      }
      return stream;
    },
  });

  form.parse(req, async (err, fields, files) => {
    if (err) {
      return res.status(500).json({ 
        message: 'Upload failed', 
        error: err.message 
      });
    }

    const uploadedFiles = Object.keys(files)
      .sort()
      .flatMap(key => files[key]);

    try {
      const dbDocs = [];
      const responseFiles = await Promise.all(
        uploadedFiles.map(async (file) => {
          const newFile = {
            id: file.id || uuid(),
            fileName: file.originalFilename || '',
            mimeType: file.mimetype || '',
            storageKey: file.newFilename || '',
            uploadedBy: req.user?.id || '',
            mode: mode as 'private' | 'public',
          };

          dbDocs.push(newFile);

          return {
            ...newFile,
            url: await generateUrl(
              newFile.storageKey,
              mode,
              newFile.fileName
            ),
          };
        })
      );

      await db.file.createMany({ data: dbDocs });

      return res.json({
        message: 'Files uploaded successfully',
        files: responseFiles,
      });
    } catch (err) {
      return res.status(500).json({
        message: 'DB insert failed',
        error: err.message,
      });
    }
  });
};

export default uploadFiles;
```

### Example: Download Controller

```typescript
import { Request, Response } from 'express';
import { downloadFileAsStream } from '../objectStorage';
import db from '@/db';

const downloadFile = async (req: Request, res: Response) => {
  try {
    const { fileId } = req.params;
    const { token } = req.query;

    const file = await db.file.findFirst({
      where: { id: fileId },
    });

    if (!file) {
      return res.status(404).json({ message: 'File not found' });
    }

    if (!token && file.mode === 'private') {
      return res.status(401).json({ message: 'Unauthorized' });
    }

    res.setHeader('Content-Type', file.mimeType);
    res.setHeader('Content-Disposition', `inline; filename="${file.fileName}"`);

    const fileStream = await downloadFileAsStream(
      file.storageKey,
      file.mode as 'private' | 'public'
    );
    
    fileStream.pipe(res);
  } catch (error) {
    res.status(500).json({
      message: 'Download failed',
      error: error.message,
    });
  }
};

export default downloadFile;
```

## GETTING STARTED:

Start by analyzing my current upload implementation and create a migration plan. Then implement the new architecture step by step, testing each storage adapter individually before moving to the next.

Focus on:
1. Finding all places where base64 file uploads are currently handled
2. Creating the storage abstraction layer
3. Implementing local storage first for easy testing
4. Migrating controllers to use streaming
5. Updating frontend to send FormData
6. Testing thoroughly before deploying

Good luck! This will significantly improve your application's performance and scalability.
