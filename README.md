# DevChronicle - Production MERN Stack Blog Management System

[![MERN Stack](https://img.shields.io/badge/Stack-MERN-61DAFB.svg?style=flat-square&logo=react)](https://reactjs.org/)
[![Node.js Version](https://img.shields.io/badge/Node.js-v18%2B-339933.svg?style=flat-square&logo=node.js)](https://nodejs.org/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Tests Passing](https://img.shields.io/badge/Tests-43%20Passing-brightgreen.svg?style=flat-square)](backend/src/tests)

A robust, enterprise-grade Blog Management System built on the **MERN** stack (MongoDB, Express.js, React.js, Node.js). Engineered with **Clean Architecture**, separation of concerns, Role-Based Access Control (RBAC), URL slug generation, soft deletion, real-time Socket.IO notifications, and a comprehensive Jest & Supertest integration suite.

---

## Table of Contents

1. [Architectural Overview](#architectural-overview)
2. [Folder & File Structure](#folder--file-structure)
3. [Core Feature Breakdown](#core-feature-breakdown)
4. [Prerequisites & Environment Setup](#prerequisites--environment-setup)
5. [Installation & Local Setup](#installation--local-setup)
6. [Docker Deployment](#docker-deployment)
7. [Demo Accounts & Credentials](#demo-accounts--credentials)
8. [Automated Testing Suite](#automated-testing-suite)
9. [REST API Documentation](#rest-api-documentation)
10. [5–10 Minute Demo Video Walkthrough Guide](#510-minute-demo-video-walkthrough-guide)

---

## Architectural Overview

This application adheres to **Clean Architecture** principles to guarantee modularity, testability, and maintainability:

```text
[ Client (React + Vite) ]
        │ HTTP / Bearer JWT + HttpOnly Cookie / WebSocket (Socket.IO)
        ▼
[ Routing & Rate Limiting (Express Router, express-rate-limit) ]
        ▼
[ Middleware Pipeline (JWT Auth, RBAC Authorization, Zod Validation, Activity Logger) ]
        ▼
[ Controller Layer (HTTP Request parsing, Status Code mapping, Response Formatting) ]
        ▼
[ Service Layer (Core Business Rules, Slugs, Soft-Delete Logic, Permissions) ]
        ▼
[ Data Access Layer (Mongoose ODM, Indexed MongoDB Schemas) ]
        ▼
[ MongoDB Database ]
```

### Key Architectural Tenets
- **Controller-Service Separation**: Controllers never interact directly with database logic; business rules reside strictly in stateless services.
- **Stateless Authentication with Dual Tokens**: Short-lived access tokens (15m) paired with database-backed HTTP-only refresh tokens (7d) for secure rotation and replay attack mitigation.
- **Defensive RBAC at API Layer**: Authorization checks (`authorize('admin')`, `isOwner || isAdmin`) are enforced deterministically on the server, ensuring security even if frontend checks are bypassed.
- **Centralized Error Handling**: Unified `ApiError` class processed by a global error handler for predictable, structured JSON responses.

---

## Folder & File Structure

```text
blog-management-system/
├── backend/
│   ├── src/
│   │   ├── config/              # MongoDB connection, Passport OAuth strategies, Socket.IO
│   │   ├── controllers/         # HTTP controllers (auth, post, comment, user, admin, tag)
│   │   ├── middleware/          # auth, rbac, activityLogger, rateLimiter, validate, errorHandler
│   │   ├── models/              # User, Post, Comment, ActivityLog, Tag schemas & indexes
│   │   ├── routes/              # Express API versioned routes (/api/v1/...)
│   │   ├── services/            # Business logic layer (AuthService, PostService, etc.)
│   │   ├── utils/               # ApiError, ApiResponse, slug generator, token helpers, seeder
│   │   ├── validators/          # Zod validation schemas (payload validation)
│   │   ├── tests/               # Jest integration tests (auth, posts, comments, admin-rbac)
│   │   ├── app.js               # Express application initialization & middleware stack
│   │   └── server.js            # HTTP server startup & Socket.IO listener
│   ├── .env.example
│   ├── package.json
│   └── jest.config.js
│
├── frontend/
│   ├── src/
│   │   ├── admin/               # Dedicated Admin Panel (Dashboard, Users, Posts, Comments, Tags)
│   │   ├── components/          # Reusable UI (Navbar, Footer, BlogCard, CommentSection, Modal, Pagination)
│   │   ├── context/             # Global AuthContext, SocketContext, ToastContext
│   │   ├── pages/               # Home, PostDetails, CreatePost, EditPost, Profile, Login, Register
│   │   ├── routes/              # ProtectedRoute, AdminRoute
│   │   ├── services/            # Axios instance with automatic 401 refresh interceptor
│   │   ├── App.jsx              # Application router & layout structure
│   │   ├── main.jsx             # React DOM root entry
│   │   └── index.css            # Responsive vanilla design system
│   ├── .env.example
│   ├── package.json
│   └── vite.config.js
│
├── docker-compose.yml
└── README.md
```

---

## Core Feature Breakdown

### 1. User Authentication & Authorization
- **Registration & Login**: Secure credential exchange with `bcryptjs` password hashing (salt rounds: 10).
- **JWT Dual-Token Workflow**: Short-lived (15 min) access tokens in `Authorization: Bearer` headers; long-lived (7 days) refresh tokens stored in `httpOnly`, `sameSite: lax` cookies.
- **Automatic Token Refresh**: Axios response interceptors transparently refresh expired tokens and retry queued requests.
- **Rate Limiting**: `express-rate-limit` guards authentication endpoints against brute-force attacks (20 req / 15 min per IP).
- **Social Media Login (OAuth 2.0)**: Passport.js configured for **Google** and **Facebook** OAuth authentication.
- **Role-Based Access Control (RBAC)**: Distinguishes between `admin` and `user` roles enforced via server-side middleware.

### 2. Post Management
- **Full CRUD**: Create, read, update, delete, and restore articles.
- **Dynamic Slug Generation**: Automatic URL-friendly slugs using `slugify` with collision resolution (e.g., `modern-web-development-1`).
- **Soft Deletion**: Non-destructive deletion model via `isDeleted: true` and `deletedAt: Date`. Normal listings automatically filter out deleted items; admins can view and restore them.
- **Validation**: Strict schema validation using **Zod** (validates title length, content length, array tags, and URL strings).

### 3. Comment Management
- **Discussion Threads**: Any registered user can participate in discussions on published posts.
- **Fine-Grained Permissions**:
  - Regular users can update or delete only their own comments.
  - The author of a post can moderate/delete any comments on their own article.
  - Administrators can moderate/delete all comments across the entire system.
- **Mongoose Associations**: Relational references linking `Post`, `User`, and `Comment` documents.

### 4. Admin Panel & Moderation Dashboard
- **Dedicated Administrative UI**: Isolated under `/admin` and guarded by `AdminRoute`.
- **KPI Metrics Dashboard**:
  - **Total Users**
  - **Total Posts** (with active and soft-deleted breakdown)
  - **Total Comments**
- **User Management**: View paginated users, change user roles (`user` <-> `admin`), toggle account activation, delete accounts.
- **Post Moderation**: Filter by `all`, `active`, or `deleted` posts, preview articles, soft-delete, or restore posts.
- **Comment Moderation**: Search comments globally by text and remove inappropriate comments.
- **Activity Audit Trail**: Live log of system events (`USER_LOGIN`, `USER_REGISTER`, `POST_CREATE`, `POST_DELETE`, etc.) capturing user IDs, timestamps, and IP addresses.

### 5. Performance Optimizations
- **MongoDB Indexing**: Compound index `{ isDeleted: 1, createdAt: -1 }`, author index `{ author: 1, isDeleted: 1 }`, text index on `{ title: 'text', content: 'text', tags: 'text' }`, and comment index `{ post: 1, isDeleted: 1, createdAt: -1 }`.
- **Smart Population**: Restricts populated user fields to only `.populate('author', 'name email avatar role')` avoiding heavy memory footprints.
- **Lean Queries**: Uses `.lean()` on read endpoints for high-throughput serialization.
- **Cursor/Offset Pagination**: Every listing endpoint enforces pagination with metadata (`page`, `limit`, `total`, `totalPages`, `hasNextPage`, `hasPrevPage`).

### 6. Real-Time Notifications (Socket.IO)
- Socket server connects with clients on WebSocket / polling transport.
- Authenticated users join their private room (`user:<userId>`).
- When a comment is posted, the article author receives an instantaneous real-time notification toast.

---

## Prerequisites & Environment Setup

### Prerequisites
- **Node.js**: `v18.x` or higher (`v20+` recommended)
- **MongoDB**: Local MongoDB instance running on port `27017` OR MongoDB Atlas connection URI

### Backend Environment Configuration (`backend/.env`)

```env
NODE_ENV=development
PORT=5000
CLIENT_URL=http://localhost:5173
MONGODB_URI=mongodb://localhost:27017/blog_management_db

JWT_ACCESS_SECRET=dev_jwt_access_secret_super_secure_key_1234567890
JWT_REFRESH_SECRET=dev_jwt_refresh_secret_super_secure_key_0987654321
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Google OAuth 2.0 (Optional for local testing; demo credentials available)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:5000/api/v1/auth/google/callback

# Facebook OAuth 2.0
FACEBOOK_CLIENT_ID=your_facebook_client_id
FACEBOOK_CLIENT_SECRET=your_facebook_client_secret
FACEBOOK_CALLBACK_URL=http://localhost:5000/api/v1/auth/facebook/callback
```

### Frontend Environment Configuration (`frontend/.env`)

```env
VITE_API_URL=http://localhost:5000/api/v1
```

---

## Installation & Local Setup

### 1. Clone & Setup Backend
```bash
cd backend
npm install
cp .env.example .env
npm run seed       # Seeds Admin, 2 Regular Users, 6 Articles, Comments, and Logs
npm run dev        # Starts backend API on http://localhost:5000
```

### 2. Setup Frontend
```bash
cd ../frontend
npm install
cp .env.example .env
npm run dev        # Starts Vite dev server on http://localhost:5173
```

Open your browser at **`http://localhost:5173`**.

---

## Docker Deployment

To spin up the entire application stack (MongoDB, Express backend, React frontend) with a single command:

```bash
docker-compose up --build
```

- Frontend: `http://localhost:5173`
- Backend API: `http://localhost:5000`
- MongoDB: `localhost:27017`

---

## Demo Accounts & Credentials

The automated seeder (`npm run seed`) provisions pre-configured accounts:

| Role | Email | Password | Permissions & Testing Capabilities |
| :--- | :--- | :--- | :--- |
| **System Admin** | `admin@example.com` | `Admin@123456` | Full platform access: `/admin`, user management, post soft-delete/restore, comment moderation |
| **Regular User 1** | `john@example.com` | `User@123456` | Author articles, edit/delete own articles, comment on posts |
| **Regular User 2** | `jane@example.com` | `User@123456` | Author articles, edit/delete own articles, comment on posts |

> **Quick-Fill**: The sign-in page (`/login`) includes "Admin Demo" and "User Demo" buttons that populate credentials instantly for rapid evaluation.

---

## Automated Testing Suite

The backend contains an automated testing suite built with **Jest** and **Supertest** running in native ES Module mode.

To execute all test suites:

```bash
cd backend
npm test
```

### Test Coverage Summary:
- **`auth.test.js`**:
  - User registration, password hashing verification, duplicate email checks, password confirmation validation.
  - User login, invalid password checks, missing email checks.
  - JWT refresh token rotation via cookies and body.
  - Authenticated `/api/v1/auth/me` and logout session invalidation.
- **`posts.test.js`**:
  - Post creation with automated slug generation.
  - Slug collision handling (`post-title`, `post-title-1`).
  - Request validation with Zod (rejects short/empty payloads).
  - Public pagination, tag filtering, slug-based post retrieval.
  - RBAC verification: author can edit/delete; unauthorized users receive **403 Forbidden**; admin can manage all.
  - Soft-deletion validation and admin restoration.
- **`comments.test.js`**:
  - Comment creation linked to posts and authors.
  - Post comment listing.
  - Comment author updates and deletions.
  - Bystander user edit/delete rejection (**403 Forbidden**).
  - Post author moderation rights & platform admin moderation rights.
- **`admin-rbac.test.js`**:
  - API-level RBAC enforcement on `/admin/*` and `/users/*` (401 when unauthenticated, 403 for regular users).
  - Admin dashboard statistics verification (Total Users, Total Posts, Total Comments).
  - User role promotion/demotion and account deactivation.
  - Self-deletion prevention for admins (400 Bad Request).

---

## REST API Documentation

All routes are prefixed with `/api/v1`.

### 1. Authentication Endpoints (`/api/v1/auth`)

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/auth/register` | Public | Register new user account. Returns `{ user, accessToken }` + sets refresh cookie |
| `POST` | `/auth/login` | Public | Sign in with email & password |
| `POST` | `/auth/refresh` | Cookie/Body | Rotate refresh token and issue new access token |
| `POST` | `/auth/logout` | Bearer Token | Invalidate refresh token and clear cookie |
| `GET` | `/auth/me` | Bearer Token | Retrieve authenticated user profile |
| `PATCH` | `/auth/profile` | Bearer Token | Update profile name, bio, avatar, or password |
| `GET` | `/auth/google` | Public | Initiates Google OAuth 2.0 authentication |
| `GET` | `/auth/google/callback` | Public | Google OAuth redirect callback |
| `GET` | `/auth/facebook` | Public | Initiates Facebook OAuth 2.0 authentication |
| `GET` | `/auth/facebook/callback` | Public | Facebook OAuth redirect callback |

### 2. Posts Endpoints (`/api/v1/posts`)

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/posts` | Public | Get active posts with pagination, search, tag, author, and sorting |
| `GET` | `/posts/:id` | Public | Get post details by MongoDB ID |
| `GET` | `/posts/slug/:slug` | Public | Get post details by URL-friendly slug |
| `POST` | `/posts` | Bearer Token | Create new article. Payload: `{ title, content, tags, coverImage, readTime }` |
| `PATCH` | `/posts/:id` | Author/Admin | Update post. Payload: `{ title, content, tags, coverImage }` |
| `DELETE` | `/posts/:id` | Author/Admin | Soft-delete post (`isDeleted: true`) |
| `PATCH` | `/posts/:id/restore`| Admin Only | Restore a soft-deleted post |

### 3. Comments Endpoints (`/api/v1/comments` & `/api/v1/posts/:postId/comments`)

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/posts/:postId/comments` | Public | Get comments for an article (paginated) |
| `POST` | `/posts/:postId/comments` | Bearer Token | Add comment to article. Payload: `{ content }` |
| `PATCH` | `/comments/:id` | Author/Admin | Update comment content |
| `DELETE` | `/comments/:id` | Author/Post Owner/Admin | Delete comment |

### 4. Admin & User Moderation Endpoints (`/api/v1/admin` & `/api/v1/users`)

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/admin/stats` | Admin Only | Get total counts (users, posts, comments), active/deleted posts, and audit log |
| `GET` | `/admin/posts` | Admin Only | List all posts including deleted with status filter (`all`, `active`, `deleted`) |
| `GET` | `/admin/comments` | Admin Only | Global comments moderation table with search |
| `DELETE` | `/admin/comments/:id` | Admin Only | Administrative comment removal |
| `GET` | `/admin/activity-logs` | Admin Only | System activity audit logs |
| `GET` | `/users` | Admin Only | Paginated list of registered users with search and role filters |
| `GET` | `/users/:id` | Admin Only | Get detailed user information |
| `PATCH` | `/users/:id` | Admin Only | Update user role (`user`/`admin`) or active status (`isActive`) |
| `DELETE` | `/users/:id` | Admin Only | Delete user account and cascade soft-delete their posts |

---

## 5–10 Minute Demo Video Walkthrough Guide

When presenting or recording the walkthrough video for this project, follow this structured agenda:

### **Part 1: Introduction & Architecture (0:00 - 1:30)**
1. **Introduction**: Introduce the system as a production-grade MERN Blog Management System.
2. **Architecture Highlights**:
   - Clean Architecture: Service layer pattern separating business rules from Express controllers.
   - Dual-token JWT authentication (15-minute access token + 7-day HttpOnly refresh token).
   - Server-side Role-Based Access Control (RBAC) preventing unauthorized operations.

### **Part 2: Public Exploration & Article Discovery (1:30 - 3:00)**
1. **Landing Page**: Showcase the responsive card layout, modern typography, and clean dark aesthetic.
2. **Search & Multi-Filtering**:
   - Type in the search bar or press `/` to focus the search field.
   - Filter by categories/tags (`#react`, `#nodejs`, `#architecture`).
   - Demonstrate sorting (Newest, Oldest, Title A-Z) and date presets.
   - Show responsive pagination controls.
3. **Article Detail Page**:
   - Click an article to navigate via its **URL slug** (e.g. `/posts/mastering-the-mern-stack-from-clean-architecture-to-production`).
   - Point out reading time, author metadata, and formatted content.

### **Part 3: User Authentication & Post Management (3:00 - 5:30)**
1. **User Sign-In**:
   - Navigate to `/login`. Click "User Demo" to sign in as **John Doe** (`john@example.com`).
   - Mention the availability of Google and Facebook OAuth buttons.
2. **Authoring an Article**:
   - Click "Write" in the top navigation (`/create-post`).
   - Input a title, markdown content, tags, and cover image URL.
   - Point out live permalink preview based on the generated slug.
   - Submit and show immediate redirect to the new slug URL.
3. **Editing & Permissions**:
   - Click "Edit" to modify the post title or body.
   - Demonstrate that John cannot edit or delete articles authored by other members (buttons are hidden on the UI and blocked on the API).
4. **Commenting**:
   - Scroll down to the comment section. Add a comment.
   - Show edit and delete options available to the comment author.

### **Part 4: Dedicated Admin Panel & Moderation (5:30 - 8:30)**
1. **Admin Authentication**:
   - Log out and log in as **Admin** (`admin@example.com`) using the "Admin Demo" quick-fill button.
2. **Admin Dashboard (`/admin`)**:
   - Highlight the **KPI Metrics**: Total Users, Total Posts (with active and soft-deleted breakdown), and Total Comments.
   - Review the live **Recent Activity Logs** tracking logins, registrations, and post creations.
3. **User Management (`/admin/users`)**:
   - Search users by name or email.
   - Change a user's role from `user` to `admin` dynamically.
   - Toggle account status between `Active` and `Deactivated`.
4. **Post Moderation & Soft-Delete Recovery (`/admin/posts`)**:
   - Filter by "Soft-Deleted Only" to reveal the archived demo post.
   - Click the "Restore" button to restore the article.
   - Navigate to the public articles page to prove the post is now live again.
   - Soft-delete an article and verify it disappears from public view without database data loss.
5. **Comment Moderation (`/admin/comments`)**:
   - Search comments across all articles and demonstrate admin deletion.

### **Part 5: Automated Testing & Conclusion (8:30 - 10:00)**
1. **Terminal Walkthrough**:
   - Run `npm test` inside the backend directory.
   - Highlight the passing test suites: Auth, Posts, Comments, and Admin RBAC (43/43 tests passing).
2. **Wrap-up**: Summarize adherence to MERN best practices, security measures, and clean modular code.

---

## License

This project is licensed under the [MIT License](LICENSE).
