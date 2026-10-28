# Akatsuki's Draft - Backend API

> An AI-enabled content sharing and publishing backend built with Node.js, Express, MongoDB, and Google Gemini API. Users create, edit, and submit posts, and admins review and approve them before they go public. Authors can enhance their articles using AI-driven editorial improvements.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Architecture](#project-architecture)
- [Environment Variables](#environment-variables)
- [Installation & Setup](#installation--setup)
- [Database Seeding](#database-seeding)
- [API Endpoints Overview](#api-endpoints-overview)
- [Detailed API Specifications & Examples](#detailed-api-specifications--examples)
  - [Health Check](#health-check)
  - [Authentication](#authentication)
  - [Post Management](#post-management)
  - [Admin Approval Workflow](#admin-approval-workflow)
  - [AI Editorial Improvement](#ai-editorial-improvement)
- [Business Logic & Rules](#business-logic--rules)
- [Security & Quality Measures](#security--quality-measures)
- [Automated Verification & Testing](#automated-verification--testing)

---

## Tech Stack

- **Runtime & Framework:** Node.js (v18+) & Express (CommonJS)
- **Database & ODM:** MongoDB with Mongoose (with automated fallback to `mongodb-memory-server` in development)
- **Authentication & Security:** JWT (`jsonwebtoken`), password hashing with `bcryptjs`, `helmet`, `cors`
- **Validation & Sanitization:** `express-validator`, custom NoSQL injection sanitizer (stripping `$` and `.` operators), MongoDB ObjectId validator
- **Traffic Control:** `express-rate-limit` (custom policies for `/auth/*` and `/ai/*`)
- **AI Integration:** Google Gemini API via `@google/generative-ai`
- **Logging & Utilities:** `morgan`, centralized `ApiError`, `asyncHandler` wrapper

---

## Project Architecture

```
.
├── src/
│   ├── config/
│   │   ├── db.js             # Mongoose connection with embedded dev fallback
│   │   └── env.js            # Centralized environment variable validation
│   ├── controllers/
│   │   ├── admin.controller.js # Admin approval / rejection handlers
│   │   ├── ai.controller.js    # Gemini AI post improvement handler
│   │   ├── auth.controller.js  # Registration, login, profile handlers
│   │   └── post.controller.js  # Post CRUD and query handlers
│   ├── middleware/
│   │   ├── auth.js           # JWT Bearer verification & user extraction
│   │   ├── error.js          # Centralized error handler & 404 middleware
│   │   ├── role.js           # Role-based authorization middleware
│   │   └── validate.js       # express-validator, ObjectId check, NoSQL sanitizer
│   ├── models/
│   │   ├── Post.js           # Post schema with indexes & timestamps
│   │   └── User.js           # User schema with bcrypt hook & select:false
│   ├── routes/
│   │   ├── admin.routes.js   # Admin routes (/admin/posts/...)
│   │   ├── ai.routes.js      # AI routes (/ai/improve/...)
│   │   ├── auth.routes.js    # Auth routes (/auth/...)
│   │   └── post.routes.js    # Post routes (/posts/...)
│   ├── services/
│   │   └── ai.service.js     # Google Gemini API client & prompt engineer
│   ├── utils/
│   │   ├── ApiError.js       # Custom operational error subclass
│   │   └── asyncHandler.js   # Promise resolution wrapper
│   ├── app.js                # Express app configuration & middleware pipeline
│   └── server.js             # HTTP listener with graceful shutdown
├── scripts/
│   ├── seed.js               # Admin account seeder
│   └── test-e2e.js           # Automated end-to-end verification suite
├── api-tests.http            # VS Code / REST Client test queries
├── postman_collection.json   # Exported Postman v2.1 collection
├── .env.example              # Template environment configuration
├── package.json
└── README.md
```

---

## Environment Variables

Copy `.env.example` to `.env` and configure your credentials:

```bash
cp .env.example .env
```

| Variable | Description | Default Value | Required |
| :--- | :--- | :--- | :--- |
| `PORT` | Port for the HTTP server to listen on | `5000` | No |
| `NODE_ENV` | Application environment (`development`, `production`, `test`) | `development` | No |
| `MONGO_URI` | MongoDB connection URI | `mongodb://127.0.0.1:27017/akatsuki_draft` | Yes |
| `JWT_SECRET` | Secret key used to sign and verify JSON Web Tokens | `akatsuki_draft_super_secret_jwt_key_2026` | Yes |
| `JWT_EXPIRES_IN`| Lifespan for issued authentication tokens | `7d` | No |
| `GEMINI_API_KEY`| API key for Google Gemini Generative AI | `""` | For AI features |
| `GEMINI_MODEL` | Gemini model name | `gemini-1.5-flash` | No |
| `ADMIN_EMAIL` | Default email used when seeding initial admin | `admin@akatsuki.draft` | No |
| `ADMIN_PASSWORD`| Default password used when seeding initial admin | `Admin@123456` | No |

---

## Installation & Setup

### 1. Clone & Install Dependencies

```bash
cd "Post event task"
npm install
```

### 2. Configure Environment

Create your `.env` file (see [Environment Variables](#environment-variables)).

### 3. Run the Database Seeder

Creates the initial administrator account specified in `.env`:

```bash
npm run seed
```

### 4. Start the Application

```bash
# Production mode
npm start

# Development mode with nodemon auto-reload
npm run dev
```

---

## Database Seeding

The seed script creates the default admin user with an encrypted password:

```bash
npm run seed
```

Output:
```
[Seed] Connecting to MongoDB...
[MongoDB] Connected successfully to host: ...
[Seed] Successfully created admin user: admin@akatsuki.draft (ID: ...)
[Seed] Seeding completed successfully.
```

If the admin account already exists, the seeder ensures its role is set to `admin` without overwriting other data.

---

## API Endpoints Overview

| Method | Endpoint | Auth | Role | Description |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/health` | Public | Any | System health check |
| `POST` | `/auth/register` | Public | Any | Register user account (`role` forced to `"user"`) |
| `POST` | `/auth/login` | Public | Any | Authenticate user or admin and return JWT |
| `GET` | `/auth/me` | JWT | Any | Get current authenticated user profile |
| `GET` | `/posts` | Public | Any | List approved posts (paginated, search, category filter) |
| `GET` | `/posts/my-posts` | JWT | Any | List current user's posts across all statuses |
| `POST` | `/posts` | JWT | Any | Create post (`status` limited to `"draft"` or `"pending"`) |
| `GET` | `/posts/:id` | Public | Any | Retrieve single post (404 if not approved) |
| `PATCH` | `/posts/:id` | JWT | Owner | Update post (resets to `"pending"` if was approved/rejected) |
| `DELETE`| `/posts/:id` | JWT | Owner/Admin | Delete post |
| `GET` | `/admin/posts/pending` | JWT | Admin | List all pending posts with author populated |
| `PATCH` | `/admin/posts/approve/:id` | JWT | Admin | Approve pending post |
| `PATCH` | `/admin/posts/reject/:id` | JWT | Admin | Reject pending post with optional reason |
| `POST` | `/ai/improve/:postId` | JWT | Owner | Improve post using Gemini AI, saving as a new draft |

---

## Detailed API Specifications & Examples

### Health Check

#### `GET /health`
- **Description:** Verifies service uptime.
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Akatsuki's Draft API is healthy",
  "timestamp": "2026-10-28T14:30:00.000Z"
}
```

---

### Authentication

#### `POST /auth/register`
- **Description:** Register a new user. The `role` is strictly assigned to `"user"` regardless of request body payload.
- **Request Body:**
```json
{
  "name": "Itachi Uchiha",
  "email": "itachi@akatsuki.draft",
  "password": "Password123!"
}
```
- **Response `201 Created`:**
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "user": {
      "_id": "6ae2070eb261083fd1735179",
      "name": "Itachi Uchiha",
      "email": "itachi@akatsuki.draft",
      "role": "user",
      "createdAt": "2026-10-28T14:30:05.120Z",
      "updatedAt": "2026-10-28T14:30:05.120Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

#### `POST /auth/login`
- **Description:** Verify credentials and retrieve a 7-day JWT.
- **Request Body:**
```json
{
  "email": "itachi@akatsuki.draft",
  "password": "Password123!"
}
```
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "_id": "6ae2070eb261083fd1735179",
      "name": "Itachi Uchiha",
      "email": "itachi@akatsuki.draft",
      "role": "user"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

#### `GET /auth/me`
- **Description:** Fetch profile of currently logged-in user.
- **Headers:** `Authorization: Bearer <token>`
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "User profile retrieved successfully",
  "data": {
    "user": {
      "_id": "6ae2070eb261083fd1735179",
      "name": "Itachi Uchiha",
      "email": "itachi@akatsuki.draft",
      "role": "user",
      "createdAt": "2026-10-28T14:30:05.120Z",
      "updatedAt": "2026-10-28T14:30:05.120Z"
    }
  }
}
```

---

### Post Management

#### `POST /posts`
- **Description:** Create a post. Users may only specify status `"draft"` or `"pending"`. Defaults to `"draft"`.
- **Headers:** `Authorization: Bearer <token>`
- **Request Body:**
```json
{
  "title": "The True Meaning of Peace",
  "description": "Reflections on empathy and mutual understanding.",
  "content": "True peace is not merely the absence of conflict; it is the presence of deep mutual comprehension.",
  "category": "Philosophy",
  "image": "https://images.unsplash.com/photo-1506744038136-46273834b3fb",
  "status": "draft"
}
```
- **Response `201 Created`:**
```json
{
  "success": true,
  "message": "Post created successfully",
  "data": {
    "post": {
      "_id": "6ae2071ab261083fd1735182",
      "title": "The True Meaning of Peace",
      "description": "Reflections on empathy and mutual understanding.",
      "content": "True peace is not merely the absence of conflict; it is the presence of deep mutual comprehension.",
      "category": "Philosophy",
      "image": "https://images.unsplash.com/photo-1506744038136-46273834b3fb",
      "status": "draft",
      "rejectionReason": null,
      "createdBy": "6ae2070eb261083fd1735179",
      "createdAt": "2026-10-28T14:32:10.000Z",
      "updatedAt": "2026-10-28T14:32:10.000Z"
    }
  }
}
```

#### `GET /posts`
- **Description:** Public catalog of approved posts with pagination and filtering.
- **Query Parameters:**
  - `page`: Page number (default `1`)
  - `limit`: Items per page (default `10`, max `100`)
  - `category`: Optional category filter
  - `search`: Full text search on title, description, and content
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Approved posts retrieved successfully",
  "data": {
    "posts": [
      {
        "_id": "6ae2071ab261083fd1735182",
        "title": "The True Meaning of Peace",
        "description": "Reflections on empathy and mutual understanding.",
        "content": "True peace is not merely the absence of conflict...",
        "category": "Philosophy",
        "image": "https://images.unsplash.com/photo-1506744038136-46273834b3fb",
        "status": "approved",
        "createdBy": {
          "_id": "6ae2070eb261083fd1735179",
          "name": "Itachi Uchiha",
          "email": "itachi@akatsuki.draft"
        },
        "createdAt": "2026-10-28T14:32:10.000Z",
        "updatedAt": "2026-10-28T14:35:00.000Z"
      }
    ],
    "pagination": {
      "total": 1,
      "page": 1,
      "limit": 10,
      "totalPages": 1,
      "hasNextPage": false,
      "hasPrevPage": false
    }
  }
}
```

#### `GET /posts/my-posts`
- **Description:** Retrieve all posts belonging to the authenticated user across all statuses (`draft`, `pending`, `approved`, `rejected`).
- **Headers:** `Authorization: Bearer <token>`
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "User posts retrieved successfully",
  "data": {
    "posts": [
      {
        "_id": "6ae2071ab261083fd1735182",
        "title": "The True Meaning of Peace",
        "status": "draft",
        "category": "Philosophy",
        "createdAt": "2026-10-28T14:32:10.000Z"
      }
    ],
    "pagination": {
      "total": 1,
      "page": 1,
      "limit": 10,
      "totalPages": 1
    }
  }
}
```

#### `GET /posts/:id`
- **Description:** Retrieve a single post. **Returns `404 Not Found` if status is not `"approved"`.**
- **Response `200 OK` (when approved):**
```json
{
  "success": true,
  "message": "Post retrieved successfully",
  "data": {
    "post": {
      "_id": "6ae2071ab261083fd1735182",
      "title": "The True Meaning of Peace",
      "content": "True peace is not merely the absence of conflict...",
      "status": "approved",
      "createdBy": {
        "_id": "6ae2070eb261083fd1735179",
        "name": "Itachi Uchiha",
        "email": "itachi@akatsuki.draft"
      }
    }
  }
}
```

#### `PATCH /posts/:id`
- **Description:** Update a post. Only the owner can update.
  - If an `approved` or `rejected` post is edited, its status is automatically reset to `"pending"` (or `"draft"` if explicitly requested).
  - Users are strictly forbidden from setting status to `"approved"` or `"rejected"`.
- **Headers:** `Authorization: Bearer <token>`
- **Request Body:**
```json
{
  "title": "The True Meaning of Peace - Expanded",
  "content": "Expanded reflections on how empathy resolves long-standing discord.",
  "status": "pending"
}
```
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Post updated successfully",
  "data": {
    "post": {
      "_id": "6ae2071ab261083fd1735182",
      "title": "The True Meaning of Peace - Expanded",
      "status": "pending"
    }
  }
}
```

#### `DELETE /posts/:id`
- **Description:** Delete a post. Only the post author or an administrator can delete.
- **Headers:** `Authorization: Bearer <token>`
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Post deleted successfully",
  "data": null
}
```

---

### Admin Approval Workflow

#### `GET /admin/posts/pending`
- **Description:** View all posts submitted with status `"pending"`, with author `name` and `email` populated.
- **Headers:** `Authorization: Bearer <admin_token>`
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Pending posts retrieved successfully",
  "data": {
    "posts": [
      {
        "_id": "6ae2071ab261083fd1735182",
        "title": "The True Meaning of Peace",
        "status": "pending",
        "createdBy": {
          "_id": "6ae2070eb261083fd1735179",
          "name": "Itachi Uchiha",
          "email": "itachi@akatsuki.draft"
        }
      }
    ],
    "pagination": {
      "total": 1,
      "page": 1,
      "limit": 10,
      "totalPages": 1
    }
  }
}
```

#### `PATCH /admin/posts/approve/:id`
- **Description:** Sets status to `"approved"` **only if currently `"pending"`**.
- **Headers:** `Authorization: Bearer <admin_token>`
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Post approved successfully",
  "data": {
    "post": {
      "_id": "6ae2071ab261083fd1735182",
      "title": "The True Meaning of Peace",
      "status": "approved",
      "rejectionReason": null
    }
  }
}
```

#### `PATCH /admin/posts/reject/:id`
- **Description:** Sets status to `"rejected"` **only if currently `"pending"`**. Accepts an optional `reason` in the request body and stores it as `rejectionReason`.
- **Headers:** `Authorization: Bearer <admin_token>`
- **Request Body:**
```json
{
  "reason": "Please cite additional practical examples in section 2 before publication."
}
```
- **Response `200 OK`:**
```json
{
  "success": true,
  "message": "Post rejected successfully",
  "data": {
    "post": {
      "_id": "6ae2071ab261083fd1735182",
      "title": "The True Meaning of Peace",
      "status": "rejected",
      "rejectionReason": "Please cite additional practical examples in section 2 before publication."
    }
  }
}
```

---

### AI Editorial Improvement

#### `POST /ai/improve/:postId`
- **Description:** Only the post author can request AI improvement.
  1. Sends title, description, and content to Google Gemini API.
  2. The prompt enhances clarity, grammar, structure, and readability while preserving facts, tone, and author voice.
  3. Saves the output as a **brand-new Post document** (with a unique `_id`), retaining `createdBy`, `category`, and `image`.
  4. Status is set to `"draft"`.
  5. Title is suffixed with `" (AI Improved)"`.
  6. **The original post remains completely untouched.**
  7. If AI times out or fails, returns a clean `502 Bad Gateway` or `503 Service Unavailable`, leaving zero partial documents in MongoDB.
- **Headers:** `Authorization: Bearer <user_token>`
- **Response `201 Created`:**
```json
{
  "success": true,
  "message": "Post improved by AI successfully and saved as a new draft",
  "data": {
    "post": {
      "_id": "6ae20799b261083fd1735199",
      "title": "The True Meaning of Peace (AI Improved)",
      "description": "A comprehensive examination of how empathy resolves discord.",
      "content": "True peace extends beyond a temporary cessation of hostilities. It requires profound empathy and structural understanding between parties...",
      "category": "Philosophy",
      "image": "https://images.unsplash.com/photo-1506744038136-46273834b3fb",
      "status": "draft",
      "createdBy": "6ae2070eb261083fd1735179",
      "createdAt": "2026-10-28T14:40:00.000Z",
      "updatedAt": "2026-10-28T14:40:00.000Z"
    }
  }
}
```

---

## Business Logic & Rules

1. **Role Security & Privilege Escalation Prevention:**
   - Registration strictly enforces `role: "user"`. Any `role` field sent in `POST /auth/register` is discarded.
   - Admin accounts can only be provisioned via the secure seeding script (`npm run seed`) or direct database administration.

2. **Strict Post Lifecycle State Machine:**
   ```
   [Author: draft] ──> [Author: pending] ──> [Admin approves] ──> [approved (Public)]
                              │
                              └──> [Admin rejects] ──> [rejected (Private)]
                                                            │
   [Author edits approved/rejected post] ────────────────────┘
          │
          └──> Auto-resets to [pending] (or [draft])
   ```

3. **Public Exposure Rules:**
   - Public queries (`GET /posts` and `GET /posts/:id`) **only** expose posts whose status is `"approved"`.
   - Accessing a draft, pending, or rejected post through `GET /posts/:id` returns `404 Not Found`.

4. **Non-Destructive AI Post Generation:**
   - AI improvements never mutate or overwrite the source post.
   - Saves a distinct document with status `"draft"` and title suffixed with `" (AI Improved)"`.

---

## Security & Quality Measures

- **NoSQL Injection Defense:** All incoming request bodies, query strings, and URL parameters are sanitized to strip any keys beginning with `$` or containing `.`.
- **ObjectId Validation:** Dedicated middleware validates all route ObjectIds before MongoDB queries are dispatched, returning clear `400 Bad Request` errors instead of internal database cast exceptions.
- **Strict Rate Limiting:**
  - `/auth/*` limited to prevent credential stuffing.
  - `/ai/*` limited to protect API quotas and prevent resource exhaustion.
- **Data Protection:** `password` is defined with `select: false` on the User schema and omitted from all JSON outputs.
- **Stack Trace Suppression:** Stack traces are suppressed in non-development environments.
- **Centralized Error Responses:** Consistent `{ success: false, message, errors?: [...] }` structure across all HTTP status codes (`400`, `401`, `403`, `404`, `409`, `422`, `500`, `502`, `503`).

---

## Automated Verification & Testing

An end-to-end integration test is included covering the entire operational workflow:

```bash
npm test
```

### Verified Flow:
1. Register standard user (verifying body `role: 'admin'` is ignored and default `user` is assigned).
2. Login and retrieve JWT.
3. Fetch user profile with `GET /auth/me`.
4. Create draft post and verify visibility is limited to author (`GET /posts/my-posts` includes it, `GET /posts` excludes it, `GET /posts/:id` yields 404).
5. Submit draft as pending. Verify user cannot self-approve.
6. Register admin and verify admin access to `GET /admin/posts/pending` with populated author details.
7. Admin approves post. Confirm post appears in public `GET /posts` and `GET /posts/:id`.
8. User calls `POST /ai/improve/:postId`. Confirm a new draft document with a new ID is created with title suffixed `" (AI Improved)"`, and verify original post remains completely untouched.
9. Verify non-owner gets `403 Forbidden` on update, delete, and AI improve endpoints.
10. Verify non-admin gets `403 Forbidden` on admin endpoints.
11. Verify editing an approved post automatically resets status to `pending`.
12. Verify admin rejection accepts optional `reason` and records `rejectionReason`.
13. Verify owner/admin deletion of posts.

