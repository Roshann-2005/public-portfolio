# Portfolio & Family Platform

A full-stack web and mobile application comprising a **private family social network** and a **personal portfolio/CV engine**, built with Django, React, and SwiftUI.

This is a personal project I built and actively maintain for my family to share posts, photos, and updates privately. It also serves as my professional portfolio and CV platform.

> _This repository documents the architecture and features of the application. Source code is available upon request._

## Architecture

```
┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
│  React SPA   │   │  iOS App     │   │  Django Templates │
│  (React 19)  │   │  (SwiftUI)   │   │  (Server-rendered)│
└──────┬───────┘   └──────┬───────┘   └────────┬─────────┘
       │                  │                     │
       └──────────────────┼─────────────────────┘
                          │
                   ┌──────▼───────┐
                   │  Django REST  │
                   │  API (DRF)   │
                   │  + OpenAPI   │
                   └──────┬───────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
       ┌──────▼──┐  ┌─────▼────┐  ┌──▼───────────┐
       │PostgreSQL│  │DigitalOcean│ │  Django Auth  │
       │         │  │  Spaces   │  │  + Token Auth │
       │         │  │  (S3)     │  │  + RBAC       │
       └─────────┘  └──────────┘  └───────────────┘
```

Three clients consume a single RESTful API — a React single-page application, a native iOS app, and server-rendered Django templates — demonstrating API-first design that supports multiple consumers without duplication.

## Tech Stack

| Layer                         | Technology                                                                   |
| ----------------------------- | ---------------------------------------------------------------------------- |
| Backend                       | Python, Django 5, Django REST Framework, drf-spectacular (OpenAPI/Swagger)   |
| Frontend (Web)                | React 19, React Router, date-fns                                             |
| Frontend (iOS) - Not deployed | SwiftUI, Kingfisher (image caching), async/await concurrency                 |
| Database                      | PostgreSQL                                                                   |
| Object Storage                | DigitalOcean Spaces (S3-compatible) with presigned URLs                      |
| Auth                          | Token-based authentication (DRF), role-based access control (custom RBAC)    |
| Rich Text                     | CKEditor 5                                                                   |
| Deployment                    | Gunicorn, environment-based configuration via python-dotenv, dj-database-url |

## Core Features

### Multi-Family Social Feed

The central feature is a social feed where family members share posts with photos, videos, and rich-text content. The feed query uses **raw SQL with Common Table Expressions (CTEs)** to efficiently determine post visibility across family boundaries — a member sees posts from all members of all families they belong to, with no duplicates even when multiple family memberships overlap.

The CTE chain resolves the authenticated user → their family memberships → all members of those families → their posts, and also attaches the latest valid share code for posts owned by the current user. This approach was chosen over ORM-based queries for the performance and clarity it provides when navigating a many-to-many graph with aggregation.

### Role-Based Access Control (RBAC)

A custom permissions system built from scratch, independent of Django's built-in permissions framework:

- **Permission** — granular capabilities (e.g., `can_invite_members`, `can_edit_posts`, `can_manage_roles`)
- **FamilyRole** — named role grouping multiple permissions (e.g., Admin, Viewer)
- **FamilyMembership** — the through-model linking a Member to a Family, carrying both a role _and_ per-membership custom permission overrides

This design supports scenarios where a member has different roles in different families, and where individual permission exceptions can be granted without creating a new role. Permission checks cascade: custom overrides are checked first, then role-level permissions.

A management command (`create_permissions`) seeds the default permission set for new deployments.

### Media Upload Pipeline

Three upload strategies are supported to accommodate different client capabilities:

1. **Presigned S3 PUT URLs** — The API generates time-limited presigned URLs so clients upload directly to DigitalOcean Spaces, bypassing the application server entirely. After upload, the client registers the Spaces object key with the API, which validates ownership (the key must match the authenticated member's GUID prefix) before creating the Photo record.

2. **Multipart file upload** — Traditional form-based upload for simpler clients.

3. **Base64-encoded upload** — JSON payload with embedded image data, decoded server-side and saved to storage.

The presigned URL flow reduces server load and bandwidth costs for media-heavy usage, and the ownership validation prevents one member from claiming another's uploads.

### Invite & Onboarding System

New users join through a **single-use invite code** system:

- An existing member with the `can_invite_members` permission generates a UUID invite code, selecting which families the new member will be added to
- The invite link leads to a registration form; on submission, the system uses `select_for_update()` within an atomic transaction to prevent race conditions from concurrent invite redemption
- The new user is created, linked to a Member, enrolled in the selected families with a default role, and added to the appropriate Django auth group — all atomically
- Once redeemed, the invite code is permanently bound to the new member and cannot be reused

The React frontend adds a sharing UX on top: the Web Share API (with clipboard fallback) lets members share invite links natively on mobile.

### Shareable Post Links

Post authors can generate **time-bound share links** that allow anyone (including unauthenticated users) to view a specific post:

- Each share link is a `SharePostCode` with a UUID and configurable expiry date
- Validity is checked on access; expired links return 403
- Every access is logged in `PostAccessLog` with timestamp and request metadata
- The feed query surfaces the latest valid share code for the current user's own posts, enabling one-tap sharing from the feed UI
- A dedicated public API endpoint (`SharedPostAPIView`) serves shared posts with no authentication required

### React Frontend

The React single-page application provides the primary web experience:

- **Masonry Feed** — posts rendered in a responsive grid layout with thumbnail images, author info, and interaction controls
- **Image Gallery Modal** — full-screen image viewer with prev/next navigation, image counter, and loading spinners for large media
- **Comment System** — inline comment forms with optimistic UI; comments are posted via the API and the full comment list is re-fetched to ensure consistency
- **Post Sharing** — share-post flow with date picker for link expiry, using the Web Share API on supported devices with an automatic clipboard fallback on desktop
- **Invite Members** — in-app invite flow where members select families via checkboxes, generate an invite code, and share it via native share or clipboard
- **Token Authentication** — login form that stores the auth token in React state, gating all routes behind authentication

### iOS App (SwiftUI) - Not deployed, in development

A native iOS client built with SwiftUI, featuring:

- **Authentication** — `AuthenticationManager` as an `ObservableObject` using async/await for login, with published state driving UI reactivity
- **Feed View** — scrollable feed with thumbnail cards, navigation to full post detail views with paginated image galleries (TabView)
- **Post Creation** — sheet-based new post flow with edit/discard actions
- **Image Loading** — Kingfisher library for async image loading with placeholder states and fade transitions
- **Theming** — asset catalog-based colour system with named colour sets for consistent theming

### Portfolio & CV Engine

Separate from the family platform, the application also serves as a personal portfolio:

- **People** — custom `User` model extending `AbstractUser` with about text (rich text via CKEditor), skills with proficiency levels (0–5 with database-enforced constraints), and referee management
- **Experience** — work history with positions, institutions, date ranges, and detailed descriptions; project showcase with categories, thumbnails, and external links
- **Qualifications** — academic qualifications linked to an international qualification framework, with institution details and evidence file uploads
- **CV Generation** — server-rendered curriculum vitae pages pulling from all of the above models

## Data Model (Family App)

```
  Permission ◄──────── FamilyRole
       │                    │
       │ (custom overrides) │ (role assignment)
       ▼                    ▼
  FamilyMembership ◄─── Member ───► User
       │                    │
       ▼                    │
    Family                  ▼
                          Post ──────► Photo
                           │           (S3 storage)
                           ▼
                        Comment

  InviteCode ──► Family (M2M)
       │
       ▼
    Member (one-time bind)

  SharePostCode ──► Post
       │
       ▼
  PostAccessLog
```

Key relationships:

- **Member ↔ Family** is many-to-many through `FamilyMembership`, which carries role and custom permissions
- **Post → Photo** supports multiple photos/videos per post, with S3-backed storage
- **InviteCode** bridges the onboarding flow: created by a member, bound to families, consumed once
- **SharePostCode** enables public access to individual posts with expiry and audit trail

## API Design

The REST API is built with Django REST Framework and documented with **drf-spectacular** (OpenAPI 3.0), served via Swagger UI at `/api/swagger/`.

Key endpoint groups:

| Endpoint Group                       | Description                                              |
| ------------------------------------ | -------------------------------------------------------- |
| `POST /family/api/login`             | Token-based authentication, returns token + member data  |
| `GET /family/api/feed/`              | Authenticated member's feed with posts, comments, photos |
| `POST /family/post/create/`          | Create a new post                                        |
| `POST /family/api/presigned-upload/` | Generate presigned S3 PUT URLs for direct upload         |
| `POST/DELETE /posts/<guid>/photos/`  | Attach or soft/hard-delete photos on a post              |
| `GET/PATCH /posts/<guid>/`           | Retrieve or update a post                                |
| `GET/PATCH /comments/<guid>/`        | Retrieve or update a comment                             |
| `POST /family/api/new_comment/`      | Create a comment on a post                               |
| `POST /family/api/invite_code/`      | Generate an invite code for onboarding                   |
| `GET /family/api/families/invite/`   | List families the member can invite into                 |
| `POST /family/api/post/share`        | Create a time-bound share link for a post                |
| `GET /family/api/post/share/<guid>/` | Public: retrieve a shared post (no auth required)        |

All authenticated endpoints use **token authentication** (`Authorization: Token <key>`). Serializers handle read/write asymmetry — for example, `CommentSerialiser` accepts GUIDs on write but returns nested member and post data on read.

## Testing

The project includes a comprehensive test suite covering:

- **Permissions cascade** — verifying that role permissions, custom overrides, and permission removal all propagate correctly through the RBAC chain
- **Concurrent invite redemption** — ensuring `select_for_update()` prevents race conditions when two users attempt to redeem the same invite simultaneously
- **Feed visibility matrix** — testing that members in overlapping families see exactly the correct set of posts with no duplicates
- **Share code lifecycle** — creation, valid access, expired access, invalid GUIDs, and access log generation
- **Invite flow end-to-end** — from invite creation through user registration, family enrollment, and group assignment
- **Model integrity** — unique constraints, cascade deletes, and validation enforcement
- **Edge cases** — empty database scenarios, form validation failures, authentication redirects

## Infrastructure & Deployment

- **Environment-based configuration** — all secrets, database URLs, CORS origins, and debug flags driven by environment variables via `python-dotenv`
- **Database** — PostgreSQL, configured via `dj-database-url` for portability across environments
- **Object storage** — DigitalOcean Spaces (S3-compatible) with a custom `PublicMediaStorage` backend; files are stored as private with signed query-string URLs for access
- **CORS** — configurable allowed origins for cross-origin API access from the React SPA and iOS app
- **Production server** — Digital ocean app platform webpack (Heroku for django, static react site)
