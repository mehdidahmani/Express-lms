# ClassroomIO Backend API Documentation

## Overview

The ClassroomIO Backend API is a comprehensive backend service designed to support the ClassroomIO education platform. This API handles resource-intensive operations including certificate generation, course content management, file uploads, email notifications, and course cloning functionality.

**Base URL (Production)**: `https://api.classroomio.com`
**Base URL (Development)**: `http://localhost:3002`

## Table of Contents

- [Features](#features)
- [Technology Stack](#technology-stack)
- [Getting Started](#getting-started)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [API Endpoints](#api-endpoints)
- [Error Handling](#error-handling)
- [Environment Variables](#environment-variables)
- [Development](#development)
- [Deployment](#deployment)

## Features

### Core Capabilities

1. **Certificate Generation**
   - Generate professional PDF certificates for course completion
   - Multiple customizable themes (blue, purple, lined patterns, professional badges)
   - Organization branding support with custom logos and colors

2. **Course Content Management**
   - Generate comprehensive PDF documents of entire courses
   - Create individual lesson PDF downloads
   - Support for markdown content rendering
   - Include supplementary resources (videos, slides)

3. **File Storage & Management**
   - Secure video uploads to Cloudflare R2
   - Document uploads (PDF, DOC, DOCX)
   - Pre-signed URL generation for secure uploads and downloads
   - File size validation and content type checking

4. **Course Cloning**
   - Complete course duplication including all content
   - Clone lessons, exercises, questions, and options
   - Support for multi-language content
   - Preserve course structure and metadata

5. **Email Notifications**
   - Send transactional emails via Nodemailer or Zoho
   - Custom HTML email templates
   - Bulk email support

6. **Math Expression Rendering**
   - LaTeX/KaTeX rendering for mathematical expressions
   - MathML output for accessibility

## Technology Stack

- **Runtime**: Node.js
- **Framework**: Hono.js (Fast, lightweight web framework)
- **Language**: TypeScript
- **Database**: Supabase (PostgreSQL)
- **File Storage**: Cloudflare R2 (S3-compatible)
- **PDF Generation**: Cloudflare Browser Rendering API
- **Email**: Nodemailer / Zoho ZeptoMail
- **Rate Limiting**: Redis with ioredis
- **Validation**: Zod
- **Authentication**: Supabase Auth (JWT-based)

## Getting Started

### Prerequisites

- Node.js 18+
- pnpm (package manager)
- Supabase account
- Cloudflare account (for R2 storage and PDF rendering)
- Redis instance (for rate limiting)

### Installation

```bash
# Clone the repository
git clone https://github.com/classroomio/classroomio.git
cd classroomio/apps/backend

# Install dependencies
pnpm install

# Copy environment variables
cp .env.example .env

# Configure your environment variables (see Environment Variables section)

# Start development server
pnpm dev

# Build for production
pnpm build

# Start production server
pnpm start
```

The server will start on port 3002 by default.

## Authentication

Most endpoints require JWT-based authentication using Supabase Auth.

### How to Authenticate

Include the authorization token in the request header:

```
Authorization: Bearer <your_supabase_access_token>
```

### Protected Endpoints

The following endpoints require authentication:
- All `/course/presign/*` endpoints (video/document upload and download)
- `/course/clone` endpoint

### Getting an Access Token

Users authenticate through the ClassroomIO web application. The frontend application handles authentication with Supabase and passes the access token to API requests.

## Rate Limiting

The API implements Redis-based rate limiting to prevent abuse.

### Default Limits

- **Window**: 60 seconds (1 minute)
- **Max Requests**: 100 requests per window
- **Key Generation**: Based on user ID (authenticated) or IP address (anonymous)

### Rate Limit Headers

Responses include the following headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 2024-12-05T10:30:00.000Z
```

When rate limit is exceeded:

```
Status: 429 Too Many Requests
Retry-After: 45

{
  "error": "Too Many Requests",
  "message": "Rate limit exceeded. Please try again later.",
  "retryAfter": 45
}
```

## API Endpoints

### Root

#### Welcome Message
```http
GET /
```

Returns a welcome message with a link to API documentation.

**Response**
```json
{
  "message": "Welcome to Classroomio.com API - docs are at https://api.classroomio.com/docs"
}
```

---

### Course Endpoints

#### Generate Certificate

Generate a PDF certificate for course completion.

```http
POST /course/download/certificate
```

**Request Body**
```json
{
  "theme": "blueProfessionalBadge",
  "studentName": "John Doe",
  "courseName": "Introduction to Web Development",
  "courseDescription": "A comprehensive course covering HTML, CSS, and JavaScript fundamentals.",
  "orgName": "ClassroomIO",
  "orgLogoUrl": "https://example.com/logo.png",
  "facilitator": "Jane Smith"
}
```

**Request Fields**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| theme | string | Yes | Certificate theme identifier |
| studentName | string | Yes | Name of the student |
| courseName | string | Yes | Name of the course |
| courseDescription | string | Yes | Course description |
| orgName | string | Yes | Organization name |
| orgLogoUrl | string | No | URL to organization logo |
| facilitator | string | No | Name of course facilitator |

**Response**

Binary PDF file with appropriate headers:
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="certificate-{courseName}.pdf"
```

**Available Themes**
- `blueProfessionalBadge` - Blue themed certificate with professional badge
- `purpleProfessionalBadge` - Purple themed certificate with professional badge
- `blueLinedBackground` - Blue certificate with lined background
- `purpleLinedBackground` - Purple certificate with lined background

---

#### Download Course Content

Generate a comprehensive PDF of all course content including all lessons.

```http
POST /course/download/content
```

**Request Body**
```json
{
  "courseTitle": "Web Development Bootcamp",
  "orgName": "ClassroomIO",
  "orgTheme": "#0030FF",
  "lessons": [
    {
      "lessonTitle": "Introduction to HTML",
      "lessonNumber": "1",
      "lessonNote": "# HTML Basics\n\nHTML is the foundation...",
      "slideUrl": "https://example.com/slides/lesson1.pdf",
      "video": ["https://example.com/videos/lesson1.mp4"]
    }
  ]
}
```

**Request Fields**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| courseTitle | string | Yes | Title of the course |
| orgName | string | Yes | Organization name |
| orgTheme | string | Yes | Hex color code for theme |
| lessons | array | Yes | Array of lesson objects |
| lessons[].lessonTitle | string | Yes | Title of the lesson |
| lessons[].lessonNumber | string | Yes | Lesson number/order |
| lessons[].lessonNote | string | Yes | Markdown content of lesson |
| lessons[].slideUrl | string | No | URL to presentation slides |
| lessons[].video | array | No | Array of video URLs |

**Response**

Binary PDF file with appropriate headers:
```
Content-Type: application/pdf
```

---

#### Clone Course

Clone an existing course with all its content, structure, and materials.

```http
POST /course/clone
```

**Authentication Required**: Yes

**Request Body**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Web Development Bootcamp - Copy",
  "description": "A copy of the original course",
  "slug": "web-dev-bootcamp-copy",
  "organizationId": "org_123456"
}
```

**Request Fields**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | string | Yes | UUID of the course to clone |
| title | string | Yes | Title for the new course |
| description | string | No | Description for the new course |
| slug | string | Yes | URL slug for the new course |
| organizationId | string | Yes | Organization ID to assign the course to |

**What Gets Cloned**
- Course metadata and settings
- All lesson sections and their structure
- All lessons with content
- Multi-language lesson content
- All exercises and questions
- Multiple choice options
- Course group and member assignments

**Response**
```json
{
  "success": true,
  "course": {
    "id": "650e8400-e29b-41d4-a716-446655440001",
    "title": "Web Development Bootcamp - Copy",
    "description": "A copy of the original course",
    "slug": "web-dev-bootcamp-copy",
    "group_id": "group_123",
    "created_at": "2024-12-05T10:00:00.000Z",
    ...
  }
}
```

**Error Response**
```json
{
  "error": "Failed to clone course",
  "details": "Course not found"
}
```

---

### Lesson Endpoints

#### Download Lesson PDF

Generate a PDF for a single lesson.

```http
POST /course/lesson/download/pdf
```

**Request Body**
```json
{
  "title": "Introduction to JavaScript",
  "number": "5",
  "orgName": "ClassroomIO",
  "note": "# JavaScript Basics\n\nJavaScript is a programming language...",
  "slideUrl": "https://example.com/slides.pdf",
  "video": ["https://example.com/video1.mp4"],
  "courseTitle": "Web Development Bootcamp"
}
```

**Response**

Binary PDF file with `Content-Type: application/pdf`

---

### Presigned URL Endpoints

These endpoints generate secure, time-limited URLs for uploading and downloading files from Cloudflare R2.

#### Generate Video Upload URL

```http
POST /course/presign/video/upload
```

**Authentication Required**: Yes

**Request Body**
```json
{
  "fileName": "lesson-video.mp4",
  "fileType": "video/mp4"
}
```

**Allowed Video Types**
- `video/mp4`
- `video/quicktime`
- `video/x-msvideo`
- `video/x-matroska`

**Max File Size**: 500MB

**Response**
```json
{
  "success": true,
  "url": "https://{account_id}.r2.cloudflarestorage.com/videos/{unique_key}.mp4?X-Amz-Algorithm=...",
  "fileKey": "{unique_key}.mp4",
  "message": "Pre-signed URL generated successfully"
}
```

**URL Expiration**: 1 hour

---

#### Generate Document Upload URL

```http
POST /course/presign/document/upload
```

**Authentication Required**: Yes

**Request Body**
```json
{
  "fileName": "course-materials.pdf",
  "fileType": "application/pdf"
}
```

**Allowed Document Types**
- `application/pdf`
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (.docx)
- `application/msword` (.doc)

**Max File Size**: 5MB

**Response**
```json
{
  "success": true,
  "url": "https://{account_id}.r2.cloudflarestorage.com/documents/{unique_key}.pdf?X-Amz-Algorithm=...",
  "fileKey": "{unique_key}.pdf",
  "message": "Document pre-signed URL generated successfully"
}
```

---

#### Generate Video Download URLs

Generate pre-signed URLs for downloading videos.

```http
POST /course/presign/video/download
```

**Authentication Required**: Yes

**Request Body**
```json
{
  "keys": [
    "abc123.mp4",
    "def456.mp4"
  ]
}
```

**Response**
```json
{
  "success": true,
  "urls": {
    "abc123.mp4": "https://{account_id}.r2.cloudflarestorage.com/videos/abc123.mp4?X-Amz-Algorithm=...",
    "def456.mp4": "https://{account_id}.r2.cloudflarestorage.com/videos/def456.mp4?X-Amz-Algorithm=..."
  },
  "message": "Video URLs retrieved successfully"
}
```

---

#### Generate Document Download URLs

```http
POST /course/presign/document/download
```

**Authentication Required**: Yes

**Request Body**
```json
{
  "keys": [
    "document1.pdf",
    "document2.docx"
  ]
}
```

**Response**
```json
{
  "success": true,
  "urls": {
    "document1.pdf": "https://{account_id}.r2.cloudflarestorage.com/documents/document1.pdf?...",
    "document2.docx": "https://{account_id}.r2.cloudflarestorage.com/documents/document2.docx?..."
  },
  "message": "Document URLs retrieved successfully"
}
```

---

### Math Rendering

#### Render LaTeX Expression

Render a LaTeX mathematical expression to MathML.

```http
GET /course/katex?x^2+y^2=z^2
```

**Query Parameters**: Raw LaTeX string in the query (URL-encoded)

**Special Encoding**
- Use `&plus;` for `+` symbol
- Use `&space;` for spaces

**Example**
```
GET /course/katex?x^2&plus;y^2=z^2
```

**Response**

HTML with MathML content:
```html
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <!-- MathML content -->
</math>
```

---

### Email Endpoints

#### Send Email

Send transactional emails using configured email service.

```http
POST /mail/send
```

**Request Body**
```json
[
  {
    "from": "ClassroomIO <notify@mail.classroomio.com>",
    "to": "student@example.com",
    "subject": "Welcome to ClassroomIO",
    "content": "<h1>Welcome!</h1><p>Thank you for joining...</p>",
    "replyTo": "support@classroomio.com"
  }
]
```

**Request Fields**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| from | string | No | Sender email (defaults to configured sender) |
| to | string | Yes | Recipient email address |
| subject | string | Yes | Email subject line |
| content | string | Yes | HTML email content |
| replyTo | string | No | Reply-to email address |

**Important Notes**
- Emails MUST be sent from `@mail.classroomio.com` domain
- Cannot send to test domains (e.g., `@test.com`)
- Content is automatically wrapped in ClassroomIO email template
- Supports batch sending (array of email objects)

**Response (Success)**
```json
{
  "success": true,
  "details": [
    {
      "success": true
    }
  ]
}
```

**Response (Partial Failure)**
```json
{
  "success": false,
  "error": "Some emails failed to send",
  "details": [
    {
      "success": true
    },
    {
      "success": false,
      "error": "Invalid email address"
    }
  ]
}
```

---

## Error Handling

### Standard Error Response Format

```json
{
  "error": "Error type or message",
  "details": "Additional error information"
}
```

### Common HTTP Status Codes

| Code | Meaning | When It Occurs |
|------|---------|----------------|
| 200 | OK | Successful request |
| 400 | Bad Request | Invalid request body or parameters |
| 401 | Unauthorized | Missing or invalid authentication token |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error occurred |

### Validation Errors

When request validation fails (using Zod schemas):

```json
{
  "success": false,
  "error": "Validation error",
  "details": "Invalid content type. Allowed types: video/mp4, video/quicktime..."
}
```

## Environment Variables

### Required Variables

```bash
# Supabase Configuration
PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
PUBLIC_SUPABASE_URL=https://your-project.supabase.co
PRIVATE_SUPABASE_SERVICE_ROLE=your_service_role_key

# Cloudflare R2 Storage
CLOUDFLARE_ACCESS_KEY=your_r2_access_key
CLOUDFLARE_SECRET_ACCESS_KEY=your_r2_secret_key
CLOUDFLARE_ACCOUNT_ID=your_cloudflare_account_id
CLOUDFLARE_BUCKET_DOMAIN=https://cdn.example.com
CLOUDFLARE_RENDERING_API_KEY=your_rendering_api_key

# Redis (Rate Limiting)
REDIS_URL=redis://localhost:6379

# Email Configuration (Choose one)
# Option 1: Nodemailer (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_SENDER="ClassroomIO <notify@mail.classroomio.com>"

# Option 2: Zoho ZeptoMail
ZOHO_TOKEN=your_zoho_api_token

# Server Configuration
NODE_ENV=production
PORT=3002
```

### Optional Variables

```bash
# OpenAPI Documentation URL
OPENAPI_URL=https://your-bucket.r2.dev/openapi/openapi-latest.json

# Sentry Error Tracking
SENTRY_DNS=your_sentry_dsn
```

## Development

### Project Structure

```
src/
├── app.ts                  # Main application setup
├── index.ts                # Server entry point
├── config/
│   └── env.ts             # Environment variable validation
├── constants/             # Application constants
├── middlewares/
│   ├── auth.ts           # JWT authentication middleware
│   └── rate-limiter.ts   # Rate limiting middleware
├── routes/
│   ├── course/           # Course-related endpoints
│   │   ├── clone.ts      # Course cloning
│   │   ├── course.ts     # Main course routes
│   │   ├── katex.ts      # Math rendering
│   │   ├── lesson.ts     # Lesson endpoints
│   │   └── presign.ts    # Presigned URL generation
│   └── mail.ts           # Email endpoints
├── services/
│   ├── course/
│   │   └── clone.ts      # Course cloning logic
│   └── mail.ts           # Email service
├── types/                # TypeScript type definitions
├── utils/                # Utility functions
│   ├── certificate.ts    # Certificate generation
│   ├── course.ts         # Course PDF generation
│   ├── lesson.ts         # Lesson PDF generation
│   ├── email.ts          # Email client setup
│   ├── s3.ts            # S3/R2 operations
│   ├── supabase.ts      # Supabase client
│   ├── redis/           # Redis utilities
│   └── upload.ts        # File upload utilities
└── rpc-types.ts         # Hono RPC type exports
```

### Running Tests

```bash
# Run tests
pnpm test

# Run tests with coverage
pnpm test:coverage
```

### Code Quality

```bash
# Lint code
pnpm lint

# Format code
pnpm format
```

### Generate OpenAPI Specification

```bash
# Generate and upload OpenAPI spec
pnpm run upload:openapi
```

This generates the OpenAPI specification and uploads it to Cloudflare R2.

## Deployment

### Docker Deployment

The project includes a Dockerfile for containerized deployment.

```bash
# Build Docker image
docker build \
  --build-arg PUBLIC_SUPABASE_ANON_KEY=$PUBLIC_SUPABASE_ANON_KEY \
  --build-arg PUBLIC_SUPABASE_URL=$PUBLIC_SUPABASE_URL \
  # ... other build args
  -t classroomio-api .

# Run container
docker run -p 3002:3002 classroomio-api
```

### Fly.io Deployment

The project is configured for Fly.io deployment.

```bash
# Deploy to Fly.io
fly deploy
```

Configuration is in `fly.toml`:
- Auto-start and auto-stop machines
- Minimum 0 machines running (cost optimization)
- Health checks on port 3002
- HTTPS enforcement

### Health Checks

The API includes a health check that runs every 30 seconds:

```bash
curl -f http://localhost:3002 || exit 1
```

## API Documentation

### Interactive Documentation

When deployed, interactive API documentation is available at:

```
https://api.classroomio.com/docs
```

This uses Scalar API Reference for beautiful, interactive OpenAPI documentation.

## Support & Contributing

- **GitHub Repository**: [classroomio/classroomio](https://github.com/classroomio/classroomio)
- **Issues**: Report bugs and request features via GitHub Issues
- **Community**: Join the ClassroomIO community for support

## License

ISC License - See repository for full license details.

## Security

### Reporting Security Issues

If you discover a security vulnerability, please email security@classroomio.com instead of using the issue tracker.

### Best Practices Implemented

- JWT-based authentication
- Rate limiting on all endpoints
- Input validation using Zod schemas
- Secure file upload with content type validation
- Pre-signed URLs for S3/R2 access
- CORS configuration
- Secure headers middleware
- Environment variable validation

---

**API Version**: 1.0.0
**Last Updated**: December 2024
**Maintained by**: ClassroomIO Team
