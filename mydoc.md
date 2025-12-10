# 🎓 RADIAL CODE — SEMINAR & CERTIFICATION BACKEND DOCUMENTATION
## Strapi CMS — Final Architecture

---

## 📋 TABLE OF CONTENTS

1. [System Overview](#1-system-overview)
2. [Strapi Content Types](#2-strapi-content-types)
3. [Role Permissions](#3-role-permissions)
4. [Public Flows](#4-public-flows)
5. [Admin Flows](#5-admin-flows)
6. [Admin Filters](#6-important-admin-filters)
7. [Certificate Numbering](#7-certificate-numbering)
8. [Download Tracking](#8-download-tracking)
9. [Complete Endpoint Index](#9-complete-endpoint-index)
10. [Summary](#10-summary)

---

## 1. SYSTEM OVERVIEW

### What This Backend Provides

✅ Program (seminar) management  
✅ Public student registration  
✅ Admin attendance verification  
✅ Certificate issuing (manual/auto)  
✅ Certificate activation  
✅ Secure certificate download  
✅ QR-based certificate verification  
✅ Admin dashboard workflows  

### Technology Stack

- **CMS**: Strapi v4
- **Database**: PostgreSQL
- **APIs**: Strapi default CRUD + minimal custom endpoints
- **Deployment**: Render.com

---

## 2. STRAPI CONTENT TYPES

### 2.1 Program (Collection Type)

**API ID**: `api::program.program`  
**Endpoint**: `/api/programs`

#### Fields

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| `program_name` | Text | Yes | Name of the seminar |
| `program_slug` | UID | Yes | Auto-generated from name, used in public URL |
| `date` | Date | Yes | Seminar date |
| `venue` | Text | Yes | Location |
| `description` | Rich Text | No | Program details |
| `certificates_active` | Boolean | No | Default: false, controls certificate downloads |
| `next_serial_number` | Integer | No | Default: 1, tracks certificate numbering |
| `created_by` | Relation | No | Links to Admin User |
| `students` | Relation | No | One-to-Many with Student |

#### Purpose

- Represents a seminar event
- `program_slug` used for public URL: `radialcode.com/{program_slug}`
- `next_serial_number` used for certificate numbering

#### Example Data

```json
{
  "program_name": "Machine Learning Workshop",
  "program_slug": "machine-learning",
  "date": "2025-01-15",
  "venue": "Tech Hub, Bangalore",
  "certificates_active": false,
  "next_serial_number": 1
}
```

---

### 2.2 Student (Collection Type)

**API ID**: `api::student.student`  
**Endpoint**: `/api/students`  
**Public Role**: ONLY "create" allowed

#### Fields

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| `full_name` | Text | Yes | Student's full name |
| `email` | Email | Yes | Student's email (unique per program) |
| `phone` | Text | Yes | Student's phone number |
| `college` | Text | Yes | College/University name |
| `course_year` | Text | No | Course and year |
| `program` | Relation | Yes | Links to Program (Many-to-One) |
| `is_attendance_verified` | Boolean | No | Default: false, admin sets to true |
| `certificate_issued` | Boolean | No | Default: false, set when certificate uploaded |
| `certificate_revoked` | Boolean | No | Default: false, admin can revoke |
| `certificate_file` | Media | No | PDF file of certificate |
| `certificate_number` | Text | No | Unique certificate ID (e.g., ML-2025-0001) |
| `certificate_qr_url` | Text | No | Verification URL for QR code |
| `download_attempts` | Integer | No | Default: 0, tracks download count |
| `last_download_at` | DateTime | No | Last download timestamp |
| `submitted_at` | DateTime | Auto | Registration timestamp |

#### Important Rules

- `email + program` must be unique → prevents duplicate registration
- Certificate download requires exact match of: `email + phone + program`

#### Example Data

```json
{
  "full_name": "John Doe",
  "email": "john@example.com",
  "phone": "9876543210",
  "college": "ABC University",
  "course_year": "3rd Year CSE",
  "program": 1,
  "is_attendance_verified": true,
  "certificate_issued": true,
  "certificate_number": "ML-2025-0001",
  "certificate_qr_url": "https://radialcode.com/verify/ML-2025-0001"
}
```

---

### 2.3 Certificate Template (Optional — Single Type)

**API ID**: `certificate-template`  
**Endpoint**: `/api/certificate-template`

#### Fields
student_name
college_name
program_name
start_date
end_date
certificate_number
registration_number
qr_url

---

## 3. ROLE PERMISSIONS

### 3.1 Public Role (No Login Required)

#### ✅ Allowed Actions

| Action | Endpoint | Purpose |
|--------|----------|---------|
| Create | `POST /api/students` | Student registration |
| Custom | `POST /api/certificates/download` | Download certificate |
| Custom | `GET /api/verify/:certificate_number` | Verify certificate |
| Read | `GET /api/programs?filters[...]` | View public program info |

#### ❌ Not Allowed

- Read/Update/Delete students
- Read/Update/Delete programs
- Upload files

---

### 3.2 Admin Role (JWT Protected)

#### ✅ Full Access To

- Programs (CRUD)
- Students (CRUD)
- Attendance verification
- Certificate issuing
- Certificate revocation
- File uploads
- Data export

---

## 4. PUBLIC FLOWS

### 4.1 Student Registration

**Endpoint**: `POST /api/students`

**Request Body**:
```json
{
  "data": {
    "full_name": "John Doe",
    "email": "john@example.com",
    "phone": "9876543210",
    "college": "ABC University",
    "course_year": "3rd Year",
    "program": 1
  }
}
```

**Auto-Set Defaults**:
- `is_attendance_verified` = false
- `certificate_issued` = false
- `certificate_revoked` = false

**Response**:
```json
{
  "data": {
    "id": 101,
    "attributes": {
      "full_name": "John Doe",
      "email": "john@example.com",
      "is_attendance_verified": false
    }
  }
}
```

---

### 4.2 Public Program Details

**Endpoint**: `GET /api/programs?filters[program_slug][$eq]={slug}`

**Example**: `GET /api/programs?filters[program_slug][$eq]=machine-learning`

**Returns**: Name, date, venue, description, certificates_active status

---

### 4.3 Certificate Download (Public)

**Endpoint**: `POST /api/certificates/download` (Custom)

**Request Body**:
```json
{
  "email": "john@example.com",
  "phone": "9876543210",
  "program_slug": "machine-learning"
}
```

**Validation Checks** (All must pass):

| Check | Description |
|-------|-------------|
| ✔ Program exists | Program with slug found |
| ✔ Certificates active | `certificates_active` = true |
| ✔ Student exists | Student registered for program |
| ✔ Email + Phone match | Exact match required |
| ✔ Attendance verified | `is_attendance_verified` = true |
| ✔ Certificate issued | `certificate_issued` = true |
| ✔ Not revoked | `certificate_revoked` = false |
| ✔ Download limit | `download_attempts` < limit |

**Success Response**:
```json
{
  "success": true,
  "certificate_url": "https://cdn.com/cert-101.pdf",
  "certificate_number": "ML-2025-0001",
  "student_name": "John Doe"
}
```

**Error Response**:
```json
{
  "success": false,
  "message": "We couldn't find your record. Please check your details."
}
```

---

### 4.4 Certificate Verification (Public / QR)

**Endpoint**: `GET /api/verify/:certificate_number`

**Example**: `GET /api/verify/ML-2025-0001`

**Valid Certificate Response**:
```json
{
  "valid": true,
  "certificate_number": "ML-2025-0001",
  "student_name": "John Doe",
  "program_name": "Machine Learning Workshop",
  "issued_on": "2025-01-15",
  "revoked": false
}
```

**Invalid Certificate Response**:
```json
{
  "valid": false,
  "message": "Certificate not found or revoked"
}
```

**Purpose**: QR code on certificate points to this URL

---

## 5. ADMIN FLOWS

### 5.1 Create Program

**Endpoint**: `POST /api/programs`

**Request**:
```json
{
  "data": {
    "program_name": "AI Workshop",
    "date": "2025-02-20",
    "venue": "Innovation Center",
    "description": "Learn AI basics"
  }
}
```

---

### 5.2 Update Program / Activate Certificates

**Endpoint**: `PUT /api/programs/:id`

**Request** (Activate certificates):
```json
{
  "data": {
    "certificates_active": true
  }
}
```

---

### 5.3 Get All Students for a Program

**Endpoint**: `GET /api/students?filters[program][id][$eq]=1`

**Returns**: All students registered for program ID 1

---

### 5.4 Get Single Student

**Endpoint**: `GET /api/students/:id`

**Returns**: Detailed student information

---

### 5.5 Verify Attendance

**Endpoint**: `PUT /api/students/:id`

**Request**:
```json
{
  "data": {
    "is_attendance_verified": true
  }
}
```

**Purpose**: Mark student as attended

---

### 5.6 Revoke Certificate

**Endpoint**: `PUT /api/students/:id`

**Request**:
```json
{
  "data": {
    "certificate_revoked": true
  }
}
```

**Effect**: Certificate becomes invalid, download blocked

---

### 5.7 Issue Certificate (Manual Upload)

**Step 1**: Upload PDF

**Endpoint**: `POST /api/upload`  
**Body**: FormData with PDF file  
**Returns**: File ID (e.g., 7)

**Step 2**: Attach to student

**Endpoint**: `PUT /api/students/:id`

**Request**:
```json
{
  "data": {
    "certificate_file": 7,
    "certificate_issued": true,
    "certificate_number": "ML-2025-0001",
    "certificate_qr_url": "https://radialcode.com/verify/ML-2025-0001"
  }
}
```

---

## 6. IMPORTANT ADMIN FILTERS

### Common Filter Queries

| Filter Purpose | Endpoint |
|----------------|----------|
| All verified students | `GET /api/students?filters[is_attendance_verified][$eq]=true` |
| All unverified students | `GET /api/students?filters[is_attendance_verified][$eq]=false` |
| Certificates issued | `GET /api/students?filters[certificate_issued][$eq]=true` |
| Certificates pending | `GET /api/students?filters[certificate_issued][$eq]=false` |
| Program + verified | `GET /api/students?filters[program][id][$eq]=1&filters[is_attendance_verified][$eq]=true` |

---

## 7. CERTIFICATE NUMBERING

### How It Works

**Storage**: `program.next_serial_number`

**Process**:

1. Read current serial number (e.g., 1)
2. Format certificate number: `{PROGRAM-CODE}-{YEAR}-{SERIAL}`
   - Example: `ML-2025-0001`
3. Increment `program.next_serial_number` to 2
4. Generate QR URL: `https://radialcode.com/verify/{certificate_number}`

**Format Examples**:
- `ML-2025-0001`
- `AI-2025-0023`
- `WEB-2026-0153`

---

## 8. DOWNLOAD TRACKING

### Fields Used

| Field | Purpose |
|-------|---------|
| `download_attempts` | Count of downloads |
| `last_download_at` | Timestamp of last download |

### Use Cases

- Enforce download limits (e.g., max 5 per day)
- Prevent abuse
- Audit logging
- Analytics

---

## 9. COMPLETE ENDPOINT INDEX

### 📊 PUBLIC ENDPOINTS (No Authentication)

| Method | Endpoint | Purpose | Request Body | Notes |
|--------|----------|---------|--------------|-------|
| POST | `/api/students` | Student registration | `{ data: { full_name, email, phone, college, course_year, program } }` | Public role: ONLY create |
| POST | `/api/certificates/download` | Validate & return certificate URL | `{ email, phone, program_slug }` | Custom endpoint |
| GET | `/api/programs?filters[program_slug][$eq]={slug}` | Public program info | — | Slug filter allowed |
| GET | `/api/verify/:certificate_number` | Certificate authenticity check | — | For QR scan |

---

### 🔐 ADMIN ENDPOINTS (JWT Protected)

#### Authentication

| Method | Endpoint | Purpose | Request Body | Response |
|--------|----------|---------|--------------|----------|
| POST | `/api/auth/local` | Admin login | `{ identifier, password }` | Returns JWT token |

#### Program Management

| Method | Endpoint | Purpose | Request Body | Notes |
|--------|----------|---------|--------------|-------|
| GET | `/api/programs` | Get all programs | — | Protected |
| POST | `/api/programs` | Create program | `{ data: { program_name, date, venue, description } }` | Auto-generates slug |
| GET | `/api/programs/:id` | Get single program | — | Detailed view |
| PUT | `/api/programs/:id` | Update / activate certificates | `{ data: { certificates_active: true } }` | Set `certificates_active` |
| DELETE | `/api/programs/:id` | Delete program | — | Deletes linked students |

#### Student Management

| Method | Endpoint | Purpose | Request Body | Notes |
|--------|----------|---------|--------------|-------|
| GET | `/api/students` | Get all students (filterable) | — | By program, verified, etc. |
| GET | `/api/students/:id` | Get single student | — | Detailed view |
| PUT | `/api/students/:id` | Verify / revoke / issue certificate | `{ data: { is_attendance_verified, certificate_issued, certificate_revoked } }` | One endpoint handles all |
| POST | `/api/students/bulk-verify` | Bulk verify students | `{ student_ids: [1, 2, 3] }` | Custom endpoint |

#### File Management

| Method | Endpoint | Purpose | Request Body | Notes |
|--------|----------|---------|--------------|-------|
| POST | `/api/upload` | Upload certificate PDF | FormData (PDF file) | Returns file ID |
| PUT | `/api/students/:id` | Attach certificate PDF to student | `{ data: { certificate_file: <file_id> } }` | After upload |

---

## 10. SUMMARY

### ✅ This Backend Includes

- ✔ Correct Strapi content structures
- ✔ Accurate role permissions
- ✔ Public registration flow
- ✔ Admin verification workflow
- ✔ Manual/auto certificate issuing
- ✔ Certificate numbering system
- ✔ Download & QR verification
- ✔ Revocation support
- ✔ Complete endpoint documentation
- ✔ Production-ready architecture

### 🚀 Technology Stack

- **Backend**: Strapi v4
- **Database**: PostgreSQL
- **Hosting**: Render.com
- **Authentication**: JWT
- **File Storage**: Strapi Upload Plugin

### 📋 Ready For

1. Development team handoff
2. Strapi project setup
3. Content type creation
4. Custom controller implementation
5. Deployment to Render

---

**Everything follows Strapi v4 best practices and is ready for production deployment.**

---

🎉 **FINAL DOCUMENT DELIVERED**
