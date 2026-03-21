# Frontend Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate frontend from Next.js to Vite SPA, add i18n (zh/en), add JWT user auth with multi-user data isolation.

**Architecture:** Backend adds User model + JWT auth middleware + user_id columns on all tenant tables. Frontend is rebuilt as Vite + React Router v6 SPA with react-i18next, auth pages, and route guards. All existing page logic is preserved with framework-specific adaptations.

**Tech Stack:** Backend: passlib[bcrypt], PyJWT | Frontend: Vite, React 19, React Router v6, Ant Design 6, react-i18next, Axios

**Spec:** `docs/superpowers/specs/2026-03-20-frontend-refactor-design.md`

---

## File Structure

### Backend Changes

```
backend/
├── app/
│   ├── config.py                    # MODIFY: add JWT_SECRET, update CORS default
│   ├── main.py                      # MODIFY: add auth router, admin seeding
│   ├── models/
│   │   ├── __init__.py              # MODIFY: add User import
│   │   ├── user.py                  # CREATE: User ORM model
│   │   ├── api_config.py            # MODIFY: add user_id FK
│   │   ├── document.py              # MODIFY: add user_id FK
│   │   ├── test_case.py             # MODIFY: add user_id FK
│   │   ├── test_task.py             # MODIFY: add user_id FK
│   │   └── scoring_dimension.py     # MODIFY: add user_id FK (nullable)
│   ├── schemas/
│   │   └── auth.py                  # CREATE: auth request/response schemas
│   ├── api/
│   │   ├── auth.py                  # CREATE: register/login/change-password/profile
│   │   ├── configs.py               # MODIFY: add current_user dependency, filter by user_id
│   │   ├── documents.py             # MODIFY: add current_user dependency, filter by user_id
│   │   ├── test_cases.py            # MODIFY: add current_user dependency, filter by user_id
│   │   ├── tests.py                 # MODIFY: add current_user dependency, filter by user_id
│   │   ├── tasks.py                 # MODIFY: add current_user dependency, filter by user_id
│   │   ├── scoring.py               # MODIFY: add current_user dependency, filter by user_id
│   │   └── reports.py               # MODIFY: add current_user dependency, filter by user_id
│   └── utils/
│       └── auth.py                  # CREATE: JWT encode/decode, get_current_user, password hashing
├── requirements.txt                 # MODIFY: add passlib[bcrypt], PyJWT
└── alembic/
    └── versions/                    # NEW migration files
```

### Frontend (complete rebuild)

```
frontend/
├── index.html                       # Vite entry HTML
├── vite.config.ts                   # Vite config with dev proxy
├── tsconfig.json
├── tsconfig.app.json
├── package.json
├── .env                             # VITE_API_URL=http://localhost:8000
└── src/
    ├── main.tsx                     # React entry: Router + i18n + Ant ConfigProvider
    ├── App.tsx                      # Route definitions
    ├── router/
    │   └── AuthGuard.tsx            # Auth guard: no token→login, must_change→change-pw
    ├── pages/
    │   ├── Login.tsx
    │   ├── Register.tsx
    │   ├── ChangePassword.tsx
    │   ├── NotFound.tsx
    │   ├── Dashboard.tsx            # migrated from app/page.tsx
    │   ├── Settings.tsx             # migrated from app/settings/page.tsx
    │   ├── Documents.tsx            # migrated from app/documents/page.tsx
    │   ├── DocumentDetail.tsx       # migrated from app/documents/[id]/page.tsx
    │   ├── TestCases.tsx            # migrated from app/test-cases/page.tsx
    │   ├── Tests.tsx                # migrated from app/tests/page.tsx
    │   ├── TaskDetail.tsx           # migrated from app/tests/[id]/page.tsx
    │   ├── Results.tsx              # migrated from app/results/[taskId]/page.tsx
    │   └── Scoring.tsx              # migrated from app/scoring/page.tsx
    ├── components/
    │   └── AppLayout.tsx            # Sidebar + topbar (language switch, user menu, logout)
    ├── lib/
    │   ├── api.ts                   # Axios + Bearer token interceptor + 401 redirect
    │   ├── types.ts                 # All TS interfaces (add User type)
    │   └── auth.ts                  # Token get/set/clear, login state, must_change_password
    └── i18n/
        ├── index.ts                 # i18next init with language detection
        ├── zh.json                  # Chinese translations
        └── en.json                  # English translations
```

---

## Task 1: Backend Auth Utilities & User Model

**Files:**
- Create: `backend/app/utils/auth.py`
- Create: `backend/app/models/user.py`
- Create: `backend/app/schemas/auth.py`
- Modify: `backend/app/models/__init__.py`
- Modify: `backend/app/config.py`
- Modify: `backend/requirements.txt`

- [ ] **Step 1: Add dependencies to requirements.txt**

Append to `backend/requirements.txt`:
```
passlib[bcrypt]==1.7.4
PyJWT==2.10.1
```

Install: `cd backend && source venv/bin/activate && pip install passlib[bcrypt] PyJWT`

- [ ] **Step 2: Add JWT_SECRET to config.py**

Add to Settings class:
```python
JWT_SECRET: str = ""  # auto-generated if empty
JWT_EXPIRE_HOURS: int = 24
```

Update CORS default:
```python
CORS_ORIGINS: str = "http://localhost:5173"
```

- [ ] **Step 3: Create auth utilities**

```python
# backend/app/utils/auth.py
import os
import jwt
from datetime import datetime, timedelta, timezone
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.orm import Session
from app.config import settings
from app.database import get_db

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

_JWT_SECRET_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.dirname(__file__))), ".jwt_secret")

def _get_jwt_secret() -> str:
    secret = settings.JWT_SECRET
    if not secret:
        if os.path.exists(_JWT_SECRET_FILE):
            with open(_JWT_SECRET_FILE, "r") as f:
                secret = f.read().strip()
        else:
            import secrets
            secret = secrets.token_hex(32)
            with open(_JWT_SECRET_FILE, "w") as f:
                f.write(secret)
    return secret

_jwt_secret = None
def get_jwt_secret() -> str:
    global _jwt_secret
    if _jwt_secret is None:
        _jwt_secret = _get_jwt_secret()
    return _jwt_secret

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: int, username: str) -> str:
    payload = {
        "user_id": user_id,
        "username": username,
        "exp": datetime.now(timezone.utc) + timedelta(hours=settings.JWT_EXPIRE_HOURS),
    }
    return jwt.encode(payload, get_jwt_secret(), algorithm="HS256")

def decode_access_token(token: str) -> dict:
    try:
        return jwt.decode(token, get_jwt_secret(), algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db),
):
    from app.models.user import User
    payload = decode_access_token(credentials.credentials)
    user = db.get(User, payload["user_id"])
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

- [ ] **Step 4: Create User model**

```python
# backend/app/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, func
from app.database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(100), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    must_change_password = Column(Boolean, default=True)
    language = Column(String(10), default="zh")
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

- [ ] **Step 5: Create auth schemas**

```python
# backend/app/schemas/auth.py
from pydantic import BaseModel
from typing import Optional

class RegisterRequest(BaseModel):
    username: str
    password: str

class LoginRequest(BaseModel):
    username: str
    password: str

class LoginResponse(BaseModel):
    access_token: str
    must_change_password: bool
    language: str
    username: str

class ChangePasswordRequest(BaseModel):
    old_password: str
    new_password: str

class ProfileUpdateRequest(BaseModel):
    language: Optional[str] = None

class UserResponse(BaseModel):
    id: int
    username: str
    language: str
    must_change_password: bool
    model_config = {"from_attributes": True}
```

- [ ] **Step 6: Update models/__init__.py**

Add: `from app.models.user import User`

- [ ] **Step 7: Commit**

```bash
git add backend/
git commit -m "feat: add User model, JWT auth utilities, auth schemas"
```

---

## Task 2: Auth API Router

**Files:**
- Create: `backend/app/api/auth.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create auth router**

```python
# backend/app/api/auth.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.database import get_db
from app.models.user import User
from app.schemas.auth import (
    RegisterRequest, LoginRequest, LoginResponse,
    ChangePasswordRequest, ProfileUpdateRequest, UserResponse,
)
from app.utils.auth import hash_password, verify_password, create_access_token, get_current_user

router = APIRouter(prefix="/api/auth", tags=["auth"])

@router.post("/register", response_model=UserResponse)
def register(data: RegisterRequest, db: Session = Depends(get_db)):
    if db.query(User).filter(User.username == data.username).first():
        raise HTTPException(400, "Username already exists")
    user = User(
        username=data.username,
        password_hash=hash_password(data.password),
        must_change_password=False,  # new registrations don't need to change
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

@router.post("/login", response_model=LoginResponse)
def login(data: LoginRequest, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.username == data.username).first()
    if not user or not verify_password(data.password, user.password_hash):
        raise HTTPException(401, "Invalid username or password")
    token = create_access_token(user.id, user.username)
    return LoginResponse(
        access_token=token,
        must_change_password=user.must_change_password,
        language=user.language,
        username=user.username,
    )

@router.post("/change-password")
def change_password(data: ChangePasswordRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    if not verify_password(data.old_password, user.password_hash):
        raise HTTPException(400, "Old password is incorrect")
    user.password_hash = hash_password(data.new_password)
    user.must_change_password = False
    db.commit()
    return {"ok": True}

@router.put("/profile", response_model=UserResponse)
def update_profile(data: ProfileUpdateRequest, user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    if data.language is not None:
        user.language = data.language
    db.commit()
    db.refresh(user)
    return user

@router.get("/me", response_model=UserResponse)
def get_me(user: User = Depends(get_current_user)):
    return user
```

- [ ] **Step 2: Include auth router in main.py (before other routers)**

Add to `backend/app/main.py`:
```python
from app.api.auth import router as auth_router
app.include_router(auth_router)
```

- [ ] **Step 3: Add admin user seeding in main.py lifespan**

In the lifespan startup, after scoring dimension seeding, add:
```python
from app.models.user import User
from app.utils.auth import hash_password

if not db.query(User).filter(User.username == "admin").first():
    admin = User(
        username="admin",
        password_hash=hash_password("admin"),
        must_change_password=True,
    )
    db.add(admin)
    db.commit()
```

- [ ] **Step 4: Commit**

```bash
git add backend/
git commit -m "feat: add auth API with register, login, change-password"
```

---

## Task 3: Add user_id to Models & Migration

**Files:**
- Modify: `backend/app/models/api_config.py`
- Modify: `backend/app/models/document.py`
- Modify: `backend/app/models/test_case.py`
- Modify: `backend/app/models/test_task.py`
- Modify: `backend/app/models/scoring_dimension.py`
- New: Alembic migration

- [ ] **Step 1: Add user_id FK to each model**

For `api_config.py`, `document.py`, `test_case.py`, `test_task.py`:
```python
user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
```

For `scoring_dimension.py`:
```python
user_id = Column(Integer, ForeignKey("users.id"), nullable=True)  # null = global default
```

Add `from sqlalchemy import ForeignKey` import where missing.

- [ ] **Step 2: Generate migration**

```bash
cd backend && source venv/bin/activate
alembic revision --autogenerate -m "add users table and user_id columns"
```

- [ ] **Step 3: Edit migration for data backfill**

The auto-generated migration will try to add non-nullable user_id columns to tables with existing data. Edit the migration to:

1. Create users table first
2. Add user_id columns as **nullable** initially
3. Insert admin user if not exists
4. Backfill existing rows with admin's user_id
5. Alter user_id columns to non-nullable (except scoring_dimensions)

```python
# In upgrade():
# Step 1: Create users table
op.create_table('users', ...)

# Step 2: Add user_id as nullable
op.add_column('api_configs', sa.Column('user_id', sa.Integer(), nullable=True))
# ... same for other tables

# Step 3: Insert admin user (use passlib directly, no app imports)
from passlib.context import CryptContext
pwd_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")
admin_hash = pwd_ctx.hash("admin")
op.execute(sa.text("INSERT INTO users (username, password_hash, must_change_password, language) VALUES (:u, :p, 1, 'zh')").bindparams(u="admin", p=admin_hash))

# Step 4: Backfill — set user_id to admin for all existing data
# NOTE: Do NOT backfill scoring_dimensions where is_default=True (keep user_id=null for global defaults)
op.execute("UPDATE api_configs SET user_id = (SELECT id FROM users WHERE username = 'admin')")
op.execute("UPDATE documents SET user_id = (SELECT id FROM users WHERE username = 'admin')")
op.execute("UPDATE test_cases SET user_id = (SELECT id FROM users WHERE username = 'admin')")
op.execute("UPDATE test_tasks SET user_id = (SELECT id FROM users WHERE username = 'admin')")
op.execute("UPDATE scoring_dimensions SET user_id = (SELECT id FROM users WHERE username = 'admin') WHERE is_default = 0")

# Step 5: Make non-nullable (for SQLite, need batch mode)
with op.batch_alter_table('api_configs') as batch_op:
    batch_op.alter_column('user_id', nullable=False)
# ... same for documents, test_cases, test_tasks
# scoring_dimensions stays nullable (global defaults have user_id=null)
```

- [ ] **Step 4: Run migration**

```bash
alembic upgrade head
```

- [ ] **Step 5: Commit**

```bash
git add backend/
git commit -m "feat: add user_id to all tenant models with migration"
```

---

## Task 4: Add Auth to All Existing API Routes

**Files:**
- Modify: `backend/app/api/configs.py`
- Modify: `backend/app/api/documents.py`
- Modify: `backend/app/api/test_cases.py`
- Modify: `backend/app/api/tests.py`
- Modify: `backend/app/api/tasks.py`
- Modify: `backend/app/api/scoring.py`
- Modify: `backend/app/api/reports.py`
- Modify: `backend/app/api/chunks.py`

- [ ] **Step 1: Update configs.py**

Add `get_current_user` dependency to all endpoints. Pattern:

```python
from app.utils.auth import get_current_user
from app.models.user import User

@router.get("")
def list_configs(..., user: User = Depends(get_current_user)):
    query = db.query(ApiConfig).filter(ApiConfig.user_id == user.id)
    ...

@router.post("")
def create_config(..., user: User = Depends(get_current_user)):
    config = ApiConfig(..., user_id=user.id)
    ...
```

Apply the same pattern to `update_config`, `delete_config`, `test_connection` — add ownership check (verify the record's user_id matches current user).

- [ ] **Step 2: Update documents.py**

Same pattern: add `user: User = Depends(get_current_user)` to all endpoints.
- `list_documents`: filter by user_id
- `upload_document`: set user_id on create
- `get_document`: check user_id ownership
- `delete_document`: check user_id ownership
- `parse_document`: check document ownership
- `rechunk_document`: check document ownership

- [ ] **Step 3: Update chunks.py**

Add auth dependency. Verify the parent document belongs to the current user.

- [ ] **Step 4: Update test_cases.py**

Same pattern for all CRUD + import endpoints.

- [ ] **Step 5: Update tests.py**

Add auth dependency. Set user_id on created tasks. Verify document_ids and test_case_ids belong to the user.

- [ ] **Step 6: Update tasks.py**

Add auth dependency. Filter task list by user_id. Verify task ownership for progress/cancel/retry/results.

- [ ] **Step 7: Update scoring.py**

Special logic: list returns user's own dimensions + global defaults (user_id is null). Create sets user_id. Edit/delete only allowed for user's own (not global defaults).

- [ ] **Step 8: Update reports.py**

Add auth dependency. Verify task ownership before export. For `download_report` endpoint: change `getReportDownloadUrl` pattern to use Axios blob download instead of direct URL (since the endpoint now requires auth). Update the frontend `api.ts` `exportReport` function to return the file as a blob download via Axios.

- [ ] **Step 9: Update _snapshot_dimensions in tests.py**

The `_snapshot_dimensions` helper must be updated to filter by current user: return user's own dimensions + global defaults (`user_id == user.id OR user_id IS NULL`).

- [ ] **Step 10: Commit**

```bash
git add backend/app/api/
git commit -m "feat: add JWT auth and user_id filtering to all API routes"
```

---

## Task 5: Delete Next.js Frontend

**Files:**
- Delete: `frontend/` directory entirely

- [ ] **Step 1: Delete the entire frontend directory**

```bash
rm -rf frontend/
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "chore: remove Next.js frontend for Vite migration"
```

---

## Task 6: Scaffold Vite Frontend

**Files:**
- Create: `frontend/` via Vite scaffold
- Create: `frontend/src/lib/types.ts`
- Create: `frontend/src/lib/auth.ts`
- Create: `frontend/src/lib/api.ts`
- Create: `frontend/.env`

- [ ] **Step 1: Create Vite project**

```bash
cd /Users/chenhao/codes/myself/embedding-check
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
npm install antd @ant-design/icons axios dayjs react-router-dom react-i18next i18next i18next-browser-languagedetector
```

- [ ] **Step 2: Create .env**

```
# Dev: leave empty to use Vite proxy. Production: set to backend URL.
VITE_API_URL=
```

- [ ] **Step 3: Configure vite.config.ts with dev proxy**

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
})
```

- [ ] **Step 4: Create auth.ts**

```typescript
// frontend/src/lib/auth.ts
const TOKEN_KEY = 'access_token';
const MUST_CHANGE_KEY = 'must_change_password';

export const getToken = () => localStorage.getItem(TOKEN_KEY);
export const setToken = (token: string) => localStorage.setItem(TOKEN_KEY, token);
export const clearToken = () => {
  localStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(MUST_CHANGE_KEY);
};
export const isLoggedIn = () => !!getToken();
export const getMustChangePassword = () => localStorage.getItem(MUST_CHANGE_KEY) === 'true';
export const setMustChangePassword = (val: boolean) => localStorage.setItem(MUST_CHANGE_KEY, String(val));
```

- [ ] **Step 5: Create types.ts**

Copy and extend from the existing `frontend/src/lib/types.ts` (already in git history). Add:

```typescript
export interface User {
  id: number;
  username: string;
  language: string;
  must_change_password: boolean;
}

export interface LoginResponse {
  access_token: string;
  must_change_password: boolean;
  language: string;
  username: string;
}
```

Keep all existing interfaces (ApiConfig, Document, Chunk, TestCase, TestTask, TestResult, ScoringDimension, PaginatedResponse).

- [ ] **Step 6: Create api.ts with auth interceptors**

```typescript
// frontend/src/lib/api.ts
import axios from 'axios';
import { getToken, clearToken } from './auth';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '',
});

api.interceptors.request.use((config) => {
  const token = getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      clearToken();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// Auth
export const login = (data: { username: string; password: string }) => api.post('/api/auth/login', data);
export const register = (data: { username: string; password: string }) => api.post('/api/auth/register', data);
export const changePassword = (data: { old_password: string; new_password: string }) => api.post('/api/auth/change-password', data);
export const updateProfile = (data: { language?: string }) => api.put('/api/auth/profile', data);
export const getMe = () => api.get('/api/auth/me');

// ... all existing API functions (configs, documents, chunks, test cases, tests, tasks, scoring, reports)
// Same as before but using import.meta.env.VITE_API_URL instead of process.env.NEXT_PUBLIC_API_URL

export default api;
```

- [ ] **Step 7: Commit**

```bash
git add frontend/
git commit -m "feat: scaffold Vite frontend with auth and API client"
```

---

## Task 7: i18n Setup & Translation Files

**Files:**
- Create: `frontend/src/i18n/index.ts`
- Create: `frontend/src/i18n/zh.json`
- Create: `frontend/src/i18n/en.json`

- [ ] **Step 1: Create i18n init**

```typescript
// frontend/src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import zh from './zh.json';
import en from './en.json';

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: { zh: { translation: zh }, en: { translation: en } },
    fallbackLng: 'zh',
    detection: {
      order: ['localStorage', 'navigator'],
      lookupLocalStorage: 'language',
    },
    interpolation: { escapeValue: false },
  });

export default i18n;
```

- [ ] **Step 2: Create zh.json**

Complete Chinese translation file covering all modules: common, nav, auth, settings, documents, testCases, tests, scoring, results, dashboard. Every user-facing string in all pages.

- [ ] **Step 3: Create en.json**

Complete English translation file mirroring zh.json structure.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/i18n/
git commit -m "feat: add i18n with Chinese and English translations"
```

---

## Task 8: Router, AuthGuard & Layout

**Files:**
- Create: `frontend/src/router/AuthGuard.tsx`
- Create: `frontend/src/components/AppLayout.tsx`
- Create: `frontend/src/App.tsx`
- Modify: `frontend/src/main.tsx`

- [ ] **Step 1: Create AuthGuard**

```typescript
// frontend/src/router/AuthGuard.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { isLoggedIn, getMustChangePassword } from '../lib/auth';

export default function AuthGuard() {
  const location = useLocation();
  if (!isLoggedIn()) return <Navigate to="/login" replace />;
  if (getMustChangePassword() && location.pathname !== '/change-password') {
    return <Navigate to="/change-password" replace />;
  }
  return <Outlet />;
}
```

- [ ] **Step 2: Create AppLayout with language switch and user menu**

Ant Design Layout + Sider + Menu (same nav items as before). Topbar adds:
- Language switcher (Dropdown: 中文 / English) → calls `updateProfile`, changes i18n language
- User menu (Dropdown: username + Logout) → clears token, redirects to /login
- Use `useTranslation()` for all labels

- [ ] **Step 3: Create App.tsx with route definitions**

```typescript
// frontend/src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import AuthGuard from './router/AuthGuard';
import AppLayout from './components/AppLayout';
import Login from './pages/Login';
import Register from './pages/Register';
import ChangePassword from './pages/ChangePassword';
import NotFound from './pages/NotFound';
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';
// ... all other page imports

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
        <Route element={<AuthGuard />}>
          <Route path="/change-password" element={<ChangePassword />} />
          <Route element={<AppLayout />}>
            <Route path="/" element={<Dashboard />} />
            <Route path="/settings" element={<Settings />} />
            <Route path="/documents" element={<Documents />} />
            <Route path="/documents/:id" element={<DocumentDetail />} />
            <Route path="/test-cases" element={<TestCases />} />
            <Route path="/tests" element={<Tests />} />
            <Route path="/tests/:id" element={<TaskDetail />} />
            <Route path="/results/:taskId" element={<Results />} />
            <Route path="/scoring" element={<Scoring />} />
          </Route>
        </Route>
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

- [ ] **Step 4: Update main.tsx**

```typescript
// frontend/src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { ConfigProvider } from 'antd';
import App from './App';
import './i18n';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <ConfigProvider>
      <App />
    </ConfigProvider>
  </React.StrictMode>
);
```

- [ ] **Step 5: Commit**

```bash
git add frontend/src/
git commit -m "feat: add router, auth guard, and app layout"
```

---

## Task 9: Auth Pages (Login, Register, ChangePassword)

**Files:**
- Create: `frontend/src/pages/Login.tsx`
- Create: `frontend/src/pages/Register.tsx`
- Create: `frontend/src/pages/ChangePassword.tsx`
- Create: `frontend/src/pages/NotFound.tsx`

- [ ] **Step 1: Create Login page**

Ant Design Card centered on page. Form with username + password inputs. Submit calls `login()` API. On success: setToken, setMustChangePassword, set language in i18n and localStorage, navigate to `/` or `/change-password`. Link to register. Language switcher in corner.

- [ ] **Step 2: Create Register page**

Similar to login. Form with username + password + confirm password. Submit calls `register()` API. On success: navigate to `/login` with success message.

- [ ] **Step 3: Create ChangePassword page**

Card with form: old password, new password, confirm new password. Submit calls `changePassword()` API. On success: setMustChangePassword(false), navigate to `/`.

- [ ] **Step 4: Create NotFound page**

Simple 404 page with Ant Design Result component and "Back to Home" button.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/pages/
git commit -m "feat: add login, register, change-password, 404 pages"
```

---

## Task 10: Migrate Dashboard & Settings Pages

**Files:**
- Create: `frontend/src/pages/Dashboard.tsx`
- Create: `frontend/src/pages/Settings.tsx`

- [ ] **Step 1: Migrate Dashboard**

Port from `app/page.tsx`. Changes:
- Remove `"use client"`
- Replace `useRouter().push()` with `useNavigate()`
- Replace all hardcoded Chinese text with `t('dashboard.xxx')` calls
- Keep all logic intact (stats cards, recent tasks, quick actions)

- [ ] **Step 2: Migrate Settings**

Port from `app/settings/page.tsx`. Changes:
- Same framework adaptations as Dashboard
- Replace all Chinese text with `t('settings.xxx')` calls

- [ ] **Step 3: Commit**

```bash
git add frontend/src/pages/
git commit -m "feat: migrate dashboard and settings pages to Vite"
```

---

## Task 11: Migrate Documents & Detail Pages

**Files:**
- Create: `frontend/src/pages/Documents.tsx`
- Create: `frontend/src/pages/DocumentDetail.tsx`

- [ ] **Step 1: Migrate Documents list page**

Port from `app/documents/page.tsx`:
- `useRouter().push(`/documents/${id}`)` → `useNavigate()(`/documents/${id}`)`
- All Chinese text → `t()` calls

- [ ] **Step 2: Migrate DocumentDetail page**

Port from `app/documents/[id]/page.tsx`:
- `useParams()` from react-router-dom (returns `{ id }` as string, parse to int)
- All Chinese text → `t()` calls

- [ ] **Step 3: Commit**

```bash
git add frontend/src/pages/
git commit -m "feat: migrate documents pages to Vite"
```

---

## Task 12: Migrate TestCases, Tests, TaskDetail Pages

**Files:**
- Create: `frontend/src/pages/TestCases.tsx`
- Create: `frontend/src/pages/Tests.tsx`
- Create: `frontend/src/pages/TaskDetail.tsx`

- [ ] **Step 1: Migrate TestCases**

Port from `app/test-cases/page.tsx`. Framework + i18n adaptations.

- [ ] **Step 2: Migrate Tests**

Port from `app/tests/page.tsx`. Navigate calls and i18n.

- [ ] **Step 3: Migrate TaskDetail**

Port from `app/tests/[id]/page.tsx`. useParams from react-router-dom.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/pages/
git commit -m "feat: migrate test cases, tests, task detail pages to Vite"
```

---

## Task 13: Migrate Results & Scoring Pages

**Files:**
- Create: `frontend/src/pages/Results.tsx`
- Create: `frontend/src/pages/Scoring.tsx`

- [ ] **Step 1: Migrate Results**

Port from `app/results/[taskId]/page.tsx`. useParams, navigate, i18n.

- [ ] **Step 2: Migrate Scoring**

Port from `app/scoring/page.tsx`. Framework + i18n adaptations.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/pages/
git commit -m "feat: migrate results and scoring pages to Vite"
```

---

## Task 14: Ant Design Locale Sync & Final Integration

**Files:**
- Modify: `frontend/src/main.tsx` or `frontend/src/App.tsx`
- Modify: `frontend/src/components/AppLayout.tsx`

- [ ] **Step 1: Add Ant Design locale switching**

In the app root, dynamically set Ant Design locale based on current i18n language:

```typescript
import zhCN from 'antd/locale/zh_CN';
import enUS from 'antd/locale/en_US';
import { useTranslation } from 'react-i18next';

function App() {
  const { i18n } = useTranslation();
  const antdLocale = i18n.language === 'en' ? enUS : zhCN;
  return (
    <ConfigProvider locale={antdLocale}>
      ...
    </ConfigProvider>
  );
}
```

- [ ] **Step 2: Verify build**

```bash
cd frontend && npm run build
```

- [ ] **Step 3: Commit**

```bash
git add frontend/
git commit -m "feat: add Ant Design locale sync and finalize integration"
```

---

## Task 15: Update README & Docker Config

**Files:**
- Modify: `README.md`
- Modify: `docker-compose.yml`
- Modify: `frontend/Dockerfile`
- Modify: `.env.example`

- [ ] **Step 1: Update README**

- Update frontend setup instructions (Vite instead of Next.js)
- Add default credentials section (admin/admin, must change on first login)
- Update environment variables table (add JWT_SECRET, JWT_EXPIRE_HOURS)
- Update CORS_ORIGINS default to 5173

- [ ] **Step 2: Update .env.example**

Add:
```
JWT_SECRET=
JWT_EXPIRE_HOURS=24
```

Update CORS:
```
CORS_ORIGINS=http://localhost:5173
```

- [ ] **Step 3: Update frontend Dockerfile for Vite**

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
RUN npm install -g serve
CMD ["serve", "-s", "dist", "-l", "3000"]
```

- [ ] **Step 4: Update docker-compose.yml**

Update frontend environment: `VITE_API_URL` instead of `NEXT_PUBLIC_API_URL`.

- [ ] **Step 5: Commit**

```bash
git add README.md .env.example docker-compose.yml frontend/Dockerfile
git commit -m "docs: update README and deploy config for Vite + auth"
```

---

## Task Order & Dependencies

```
Task 1 (Auth Utils + User Model)
  → Task 2 (Auth API)
  → Task 3 (user_id Migration)
  → Task 4 (Auth on all APIs)
Task 5 (Delete Next.js) → Task 6 (Scaffold Vite) → Task 7 (i18n)
  → Task 8 (Router + Layout) → Task 9 (Auth Pages)
  → Tasks 10-13 (Page migrations, can be parallelized)
  → Task 14 (Locale Sync)
  → Task 15 (README)
```

Backend tasks (1-4) and frontend deletion (5) can run before frontend rebuild (6-14). Tasks 10-13 are independent page migrations and can be parallelized after Task 9.
