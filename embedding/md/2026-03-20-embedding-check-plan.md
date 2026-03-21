# Embedding Check Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a full-stack platform for testing and evaluating embedding model retrieval accuracy, with document parsing, chunking, vector search, LLM scoring, and report export.

**Architecture:** FastAPI backend with SQLAlchemy ORM (SQLite/PostgreSQL), Chroma vector DB (PersistentClient), async task execution via ThreadPoolExecutor. Next.js frontend with Ant Design. OpenAI-compatible API protocol for all model calls.

**Tech Stack:** Python 3.11+, FastAPI, SQLAlchemy, Alembic, Chroma, OpenAI SDK, pypdfium2, python-docx, openpyxl, reportlab, cryptography | Next.js 14, TypeScript, Ant Design 5, Axios

**Spec:** `docs/superpowers/specs/2026-03-20-embedding-check-design.md`

---

## File Structure

### Backend

```
backend/
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/                    # auto-generated migration files
├── requirements.txt
├── uploads/                         # uploaded document files (UUID-named)
├── exports/                         # generated report files (temp)
└── app/
    ├── __init__.py
    ├── main.py                      # FastAPI app, lifespan, CORS, router includes
    ├── config.py                    # Settings via pydantic-settings (DB URL, CORS, upload limits)
    ├── database.py                  # SQLAlchemy engine, SessionLocal, get_db dependency
    ├── models/
    │   ├── __init__.py              # re-export all models
    │   ├── api_config.py            # ApiConfig ORM model
    │   ├── document.py              # Document ORM model
    │   ├── chunk.py                 # Chunk ORM model (with parent_id self-ref)
    │   ├── test_case.py             # TestCase ORM model
    │   ├── test_task.py             # TestTask ORM model (with parent_task_id self-ref)
    │   ├── test_result.py           # TestResult ORM model
    │   └── scoring_dimension.py     # ScoringDimension ORM model
    ├── schemas/
    │   ├── __init__.py
    │   ├── api_config.py            # Pydantic create/update/response schemas
    │   ├── document.py
    │   ├── chunk.py
    │   ├── test_case.py
    │   ├── test_task.py
    │   ├── test_result.py
    │   ├── scoring_dimension.py
    │   └── common.py                # PaginatedResponse, generic schemas
    ├── api/
    │   ├── __init__.py
    │   ├── configs.py               # CRUD + connection test for API configs
    │   ├── documents.py             # Upload, list, delete, re-chunk
    │   ├── chunks.py                # List chunks for a document
    │   ├── test_cases.py            # CRUD + file import (CSV/Excel)
    │   ├── tests.py                 # Create single/batch test, list tasks
    │   ├── tasks.py                 # Task progress, cancel, retry
    │   ├── scoring.py               # Scoring dimensions CRUD
    │   └── reports.py               # Export Excel/PDF
    ├── services/
    │   ├── __init__.py
    │   ├── document_parser.py       # Dispatcher: detect type → call extractor
    │   ├── extractors/
    │   │   ├── __init__.py
    │   │   ├── txt.py               # TXT extractor (encoding detection)
    │   │   ├── markdown.py          # Markdown extractor (header-aware)
    │   │   ├── pdf.py               # PDF extractor (pypdfium2)
    │   │   ├── word.py              # Word extractor (python-docx → markdown)
    │   │   └── excel.py             # Excel extractor (openpyxl → key-value rows)
    │   ├── text_cleaner.py          # Pre-processing: invalid chars, whitespace, URLs
    │   ├── chunking/
    │   │   ├── __init__.py          # ChunkStrategy enum, get_chunker()
    │   │   ├── base.py              # RecursiveCharacterTextSplitter base class
    │   │   ├── auto.py              # AutoChunker (EnhanceRecursiveCharacterTextSplitter)
    │   │   ├── custom.py            # CustomChunker (FixedRecursiveCharacterTextSplitter)
    │   │   └── parent_child.py      # ParentChildChunker (two-level splitting)
    │   ├── embedding.py             # OpenAI-compatible embedding client
    │   ├── retrieval.py             # Chroma query, similarity search
    │   ├── scoring.py               # LLM scoring (3 modes), prompt templates, JSON parsing
    │   ├── task_manager.py          # ThreadPoolExecutor, task scanner, progress tracking
    │   ├── task_executor.py         # Actual test execution logic (retrieval + scoring loop)
    │   └── report_export.py         # Excel (openpyxl) + PDF (reportlab) generation
    └── utils/
        ├── __init__.py
        ├── crypto.py                # Fernet encrypt/decrypt for API keys
        └── chroma_client.py         # Chroma PersistentClient singleton, collection helpers
```

### Frontend

```
frontend/
├── package.json
├── next.config.js
├── tsconfig.json
├── .env.local                       # NEXT_PUBLIC_API_URL=http://localhost:8000
└── src/
    ├── app/
    │   ├── layout.tsx               # Root layout with Ant Design ConfigProvider, sidebar nav
    │   ├── page.tsx                 # Dashboard: recent tasks, quick actions
    │   ├── settings/
    │   │   └── page.tsx             # API configs CRUD table
    │   ├── documents/
    │   │   ├── page.tsx             # Document list with upload
    │   │   └── [id]/
    │   │       └── page.tsx         # Document detail: chunk list, re-chunk
    │   ├── test-cases/
    │   │   └── page.tsx             # Test cases CRUD + file import
    │   ├── tests/
    │   │   ├── page.tsx             # Test hub: single test + batch test tabs
    │   │   └── [id]/
    │   │       └── page.tsx         # Task detail: progress, results
    │   ├── results/
    │   │   └── [taskId]/
    │   │       └── page.tsx         # Results detail + model comparison + export
    │   └── scoring/
    │       └── page.tsx             # Scoring dimensions CRUD
    ├── components/
    │   ├── AppLayout.tsx            # Sidebar + content layout
    │   ├── DocumentUpload.tsx       # Upload modal with chunk config
    │   ├── ChunkConfigForm.tsx      # Chunk strategy selector + params
    │   ├── TestCaseImport.tsx       # CSV/Excel import modal
    │   ├── BatchTestForm.tsx        # Multi-model selection form
    │   ├── TaskProgress.tsx         # Progress bar with polling
    │   ├── ResultsTable.tsx         # Scored results table
    │   └── ModelComparisonChart.tsx  # Comparison visualization
    ├── lib/
    │   ├── api.ts                   # Axios instance, typed API functions
    │   └── types.ts                 # Shared TypeScript types mirroring backend schemas
    └── types/
        └── index.ts                 # Re-exports
```

---

## Task 1: Backend Project Scaffolding

**Files:**
- Create: `backend/requirements.txt`
- Create: `backend/app/__init__.py`
- Create: `backend/app/config.py`
- Create: `backend/app/database.py`
- Create: `backend/app/main.py`
- Create: `backend/alembic.ini`
- Create: `backend/alembic/env.py`

- [ ] **Step 1: Create requirements.txt**

```txt
fastapi==0.115.6
uvicorn[standard]==0.34.0
sqlalchemy==2.0.36
alembic==1.14.1
chromadb==0.6.3
openai==1.59.0
pypdfium2==4.30.1
python-docx==1.1.2
openpyxl==3.1.5
reportlab==4.2.5
cryptography==44.0.0
python-multipart==0.0.19
pydantic-settings==2.7.1
chardet==5.2.0
psycopg2-binary==2.9.10
```

- [ ] **Step 2: Create config.py**

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str = "sqlite:///./embedding_check.db"
    CHROMA_PERSIST_DIR: str = "./chroma_data"
    CORS_ORIGINS: str = "http://localhost:3000"
    UPLOAD_DIR: str = "./uploads"
    EXPORT_DIR: str = "./exports"
    MAX_UPLOAD_SIZE: int = 50 * 1024 * 1024  # 50MB
    ENCRYPTION_KEY: str = ""  # Fernet key, auto-generated if empty
    TASK_POOL_SIZE: int = 4
    model_config = {"env_file": ".env"}

settings = Settings()
```

- [ ] **Step 3: Create database.py**

```python
from sqlalchemy import create_engine, event
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from sqlalchemy.pool import StaticPool
from app.config import settings

is_sqlite = settings.DATABASE_URL.startswith("sqlite")

if is_sqlite:
    engine = create_engine(
        settings.DATABASE_URL,
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    @event.listens_for(engine, "connect")
    def set_sqlite_pragma(dbapi_connection, connection_record):
        cursor = dbapi_connection.cursor()
        cursor.execute("PRAGMA journal_mode=WAL")
        cursor.execute("PRAGMA foreign_keys=ON")
        cursor.close()
else:
    engine = create_engine(settings.DATABASE_URL)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

- [ ] **Step 4: Create main.py with CORS and lifespan**

```python
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: create dirs, clean old exports
    os.makedirs(settings.UPLOAD_DIR, exist_ok=True)
    os.makedirs(settings.EXPORT_DIR, exist_ok=True)
    # TODO: start task manager, clean old exports
    yield
    # Shutdown: stop task manager

app = FastAPI(title="Embedding Check", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# TODO: include routers
```

- [ ] **Step 5: Initialize Alembic**

Run:
```bash
cd backend && pip install -r requirements.txt
alembic init alembic
```

Then edit `alembic/env.py` to import `Base` and all models, set `target_metadata = Base.metadata`. Edit `alembic.ini` to set `sqlalchemy.url` from config.

- [ ] **Step 6: Create __init__.py files**

Create empty `__init__.py` in `app/`, `app/models/`, `app/schemas/`, `app/api/`, `app/services/`, `app/services/extractors/`, `app/services/chunking/`, `app/utils/`.

- [ ] **Step 7: Verify server starts**

Run: `cd backend && uvicorn app.main:app --reload --port 8000`
Expected: Server starts, responds to `GET /docs` with Swagger UI.

- [ ] **Step 8: Commit**

```bash
git add backend/
git commit -m "feat: scaffold backend with FastAPI, SQLAlchemy, Alembic"
```

---

## Task 2: ORM Models & Initial Migration

**Files:**
- Create: `backend/app/models/api_config.py`
- Create: `backend/app/models/document.py`
- Create: `backend/app/models/chunk.py`
- Create: `backend/app/models/test_case.py`
- Create: `backend/app/models/test_task.py`
- Create: `backend/app/models/test_result.py`
- Create: `backend/app/models/scoring_dimension.py`
- Create: `backend/app/models/__init__.py`

- [ ] **Step 1: Create ApiConfig model**

```python
# backend/app/models/api_config.py
from sqlalchemy import Column, Integer, String, DateTime, func
from app.database import Base

class ApiConfig(Base):
    __tablename__ = "api_configs"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    model_type = Column(String(50), nullable=False)  # "embedding" or "llm"
    base_url = Column(String(500), nullable=False)
    api_key = Column(String(500), nullable=False)  # encrypted
    model_name = Column(String(255), nullable=False)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

- [ ] **Step 2: Create Document model**

```python
# backend/app/models/document.py
from sqlalchemy import Column, Integer, String, DateTime, JSON, func
from app.database import Base

class Document(Base):
    __tablename__ = "documents"
    id = Column(Integer, primary_key=True, index=True)
    filename = Column(String(255), nullable=False)
    file_path = Column(String(500), nullable=False)
    file_type = Column(String(20), nullable=False)
    file_size = Column(Integer, nullable=False)
    status = Column(String(50), default="uploaded")
    chunk_strategy = Column(String(50), nullable=True)
    chunk_params = Column(JSON, nullable=True)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

- [ ] **Step 3: Create Chunk model with self-referencing parent_id**

```python
# backend/app/models/chunk.py
from sqlalchemy import Column, Integer, String, Text, DateTime, JSON, ForeignKey, func
from sqlalchemy.orm import relationship, backref
from app.database import Base

class Chunk(Base):
    __tablename__ = "chunks"
    id = Column(Integer, primary_key=True, index=True)
    document_id = Column(Integer, ForeignKey("documents.id", ondelete="CASCADE"), nullable=False)
    parent_id = Column(Integer, ForeignKey("chunks.id", ondelete="SET NULL"), nullable=True)
    chunk_type = Column(String(20), default="normal")  # parent / child / normal
    content = Column(Text, nullable=False)
    chunk_index = Column(Integer, nullable=False)
    metadata_ = Column("metadata", JSON, nullable=True)
    chroma_id = Column(String(255), nullable=True)
    embedding_model = Column(String(255), nullable=True)
    created_at = Column(DateTime, default=func.now())
    children = relationship("Chunk", backref=backref("parent", remote_side=[id]), foreign_keys=[parent_id])
```

- [ ] **Step 4: Create TestCase model**

```python
# backend/app/models/test_case.py
from sqlalchemy import Column, Integer, String, Text, DateTime, JSON, func
from app.database import Base

class TestCase(Base):
    __tablename__ = "test_cases"
    id = Column(Integer, primary_key=True, index=True)
    group_name = Column(String(255), nullable=True)
    query = Column(Text, nullable=False)
    expected_chunks = Column(JSON, nullable=True)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

- [ ] **Step 5: Create TestTask model with self-referencing parent_task_id**

```python
# backend/app/models/test_task.py
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, JSON, ForeignKey, func
from app.database import Base

class TestTask(Base):
    __tablename__ = "test_tasks"
    id = Column(Integer, primary_key=True, index=True)
    parent_task_id = Column(Integer, ForeignKey("test_tasks.id", ondelete="CASCADE"), nullable=True)
    name = Column(String(255), nullable=False)
    type = Column(String(20), nullable=False)  # single / batch
    status = Column(String(50), default="pending")
    progress = Column(Integer, default=0)
    cancel_requested = Column(Boolean, default=False)
    document_ids = Column(JSON, nullable=False)
    test_case_ids = Column(JSON, nullable=True)
    embedding_model_id = Column(Integer, ForeignKey("api_configs.id"), nullable=False)
    scoring_model_id = Column(Integer, ForeignKey("api_configs.id"), nullable=True)
    scoring_mode = Column(String(50), nullable=True)
    scoring_dimensions_snapshot = Column(JSON, nullable=True)
    config_params = Column(JSON, nullable=True)
    error_message = Column(Text, nullable=True)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
    completed_at = Column(DateTime, nullable=True)
```

- [ ] **Step 6: Create TestResult model**

```python
# backend/app/models/test_result.py
from sqlalchemy import Column, Integer, String, Text, Float, DateTime, JSON, ForeignKey, func
from app.database import Base

class TestResult(Base):
    __tablename__ = "test_results"
    id = Column(Integer, primary_key=True, index=True)
    task_id = Column(Integer, ForeignKey("test_tasks.id", ondelete="CASCADE"), nullable=False)
    test_case_id = Column(Integer, ForeignKey("test_cases.id"), nullable=True)
    query = Column(Text, nullable=False)
    retrieved_chunks = Column(JSON, nullable=True)
    similarity_scores = Column(JSON, nullable=True)
    llm_scores = Column(JSON, nullable=True)
    llm_reasoning = Column(Text, nullable=True)
    total_score = Column(Float, nullable=True)
    created_at = Column(DateTime, default=func.now())
```

- [ ] **Step 7: Create ScoringDimension model**

```python
# backend/app/models/scoring_dimension.py
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, func
from app.database import Base

class ScoringDimension(Base):
    __tablename__ = "scoring_dimensions"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    description = Column(Text, nullable=True)
    max_score = Column(Integer, default=5)
    is_default = Column(Boolean, default=False)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

- [ ] **Step 8: Update models/__init__.py to import all models**

```python
from app.models.api_config import ApiConfig
from app.models.document import Document
from app.models.chunk import Chunk
from app.models.test_case import TestCase
from app.models.test_task import TestTask
from app.models.test_result import TestResult
from app.models.scoring_dimension import ScoringDimension
```

- [ ] **Step 9: Update alembic/env.py to use models metadata**

Ensure `alembic/env.py` imports `Base` from `app.database` and sets `target_metadata = Base.metadata`. Also import `app.models` to register all models.

- [ ] **Step 10: Generate and run initial migration**

Run:
```bash
cd backend
alembic revision --autogenerate -m "initial schema"
alembic upgrade head
```
Expected: Migration file created in `alembic/versions/`, database file `embedding_check.db` created with all tables.

- [ ] **Step 11: Commit**

```bash
git add backend/
git commit -m "feat: add all ORM models and initial migration"
```

---

## Task 3: Pydantic Schemas & Common Utilities

**Files:**
- Create: `backend/app/schemas/common.py`
- Create: `backend/app/schemas/api_config.py`
- Create: `backend/app/schemas/document.py`
- Create: `backend/app/schemas/chunk.py`
- Create: `backend/app/schemas/test_case.py`
- Create: `backend/app/schemas/test_task.py`
- Create: `backend/app/schemas/test_result.py`
- Create: `backend/app/schemas/scoring_dimension.py`
- Create: `backend/app/utils/crypto.py`
- Create: `backend/app/utils/chroma_client.py`

- [ ] **Step 1: Create common schemas**

```python
# backend/app/schemas/common.py
from typing import TypeVar, Generic, List
from pydantic import BaseModel

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    page_size: int
```

- [ ] **Step 2: Create api_config schemas**

```python
# backend/app/schemas/api_config.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class ApiConfigCreate(BaseModel):
    name: str
    model_type: str  # "embedding" or "llm"
    base_url: str
    api_key: str
    model_name: str

class ApiConfigUpdate(BaseModel):
    name: Optional[str] = None
    model_type: Optional[str] = None
    base_url: Optional[str] = None
    api_key: Optional[str] = None
    model_name: Optional[str] = None

class ApiConfigResponse(BaseModel):
    id: int
    name: str
    model_type: str
    base_url: str
    model_name: str
    created_at: datetime
    updated_at: datetime
    model_config = {"from_attributes": True}
```

- [ ] **Step 3: Create remaining schemas**

Each model gets Create, Update (optional fields), Response schemas. Key schemas:

```python
# backend/app/schemas/test_task.py
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime

class SingleTestCreate(BaseModel):
    query: str
    document_ids: List[int]
    embedding_model_id: int
    scoring_model_id: Optional[int] = None
    scoring_mode: Optional[str] = None
    config_params: Optional[dict] = None

class BatchTestCreate(BaseModel):
    name: str
    document_ids: List[int]
    test_case_ids: List[int]
    embedding_model_ids: List[int]  # plural: multiple for comparison
    scoring_model_id: Optional[int] = None
    scoring_mode: Optional[str] = None
    config_params: Optional[dict] = None

class TestTaskResponse(BaseModel):
    id: int
    parent_task_id: Optional[int] = None
    name: str
    type: str
    status: str
    progress: int
    document_ids: list
    embedding_model_id: int
    scoring_model_id: Optional[int] = None
    scoring_mode: Optional[str] = None
    config_params: Optional[dict] = None
    error_message: Optional[str] = None
    created_at: datetime
    completed_at: Optional[datetime] = None
    model_config = {"from_attributes": True}

# backend/app/schemas/test_result.py
class TestResultResponse(BaseModel):
    id: int
    task_id: int
    test_case_id: Optional[int] = None
    query: str
    retrieved_chunks: Optional[list] = None
    similarity_scores: Optional[list] = None
    llm_scores: Optional[dict] = None
    llm_reasoning: Optional[str] = None
    total_score: Optional[float] = None
    created_at: datetime
    model_config = {"from_attributes": True}
```

Other schemas (document, chunk, test_case, scoring_dimension) follow the same Create/Update/Response pattern mirroring their ORM models. `api_key` is excluded from all response schemas.

- [ ] **Step 4: Create crypto utility**

```python
# backend/app/utils/crypto.py
import os
from cryptography.fernet import Fernet
from app.config import settings

_KEY_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.dirname(__file__))), ".encryption_key")

def _get_fernet() -> Fernet:
    key = settings.ENCRYPTION_KEY
    if not key:
        # Try to read from key file
        if os.path.exists(_KEY_FILE):
            with open(_KEY_FILE, "r") as f:
                key = f.read().strip()
        else:
            # Generate and persist to file
            key = Fernet.generate_key().decode()
            with open(_KEY_FILE, "w") as f:
                f.write(key)
    return Fernet(key.encode() if isinstance(key, str) else key)

_fernet = None

def get_fernet() -> Fernet:
    global _fernet
    if _fernet is None:
        _fernet = _get_fernet()
    return _fernet

def encrypt_api_key(plain: str) -> str:
    return get_fernet().encrypt(plain.encode()).decode()

def decrypt_api_key(encrypted: str) -> str:
    return get_fernet().decrypt(encrypted.encode()).decode()
```

- [ ] **Step 5: Create Chroma client utility**

```python
# backend/app/utils/chroma_client.py
import chromadb
from app.config import settings

_client = None

def get_chroma_client() -> chromadb.ClientAPI:
    global _client
    if _client is None:
        _client = chromadb.PersistentClient(path=settings.CHROMA_PERSIST_DIR)
    return _client

def get_collection_name(document_id: int, model_config_id: int) -> str:
    return f"doc_{document_id}_model_{model_config_id}"

def get_or_create_collection(document_id: int, model_config_id: int):
    client = get_chroma_client()
    name = get_collection_name(document_id, model_config_id)
    return client.get_or_create_collection(name=name, metadata={"hnsw:space": "cosine"})

def delete_collection(document_id: int, model_config_id: int):
    client = get_chroma_client()
    name = get_collection_name(document_id, model_config_id)
    try:
        client.delete_collection(name=name)
    except ValueError:
        pass
```

- [ ] **Step 6: Commit**

```bash
git add backend/app/schemas/ backend/app/utils/
git commit -m "feat: add Pydantic schemas and utility modules"
```

---

## Task 4: API Configs CRUD API

**Files:**
- Create: `backend/app/api/configs.py`
- Modify: `backend/app/main.py` (include router)

- [ ] **Step 1: Create configs router with full CRUD**

```python
# backend/app/api/configs.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.database import get_db
from app.models.api_config import ApiConfig
from app.schemas.api_config import ApiConfigCreate, ApiConfigUpdate, ApiConfigResponse
from app.schemas.common import PaginatedResponse
from app.utils.crypto import encrypt_api_key, decrypt_api_key
from openai import OpenAI

router = APIRouter(prefix="/api/configs", tags=["configs"])

@router.get("", response_model=PaginatedResponse[ApiConfigResponse])
def list_configs(page: int = 1, page_size: int = 20, model_type: str = None, db: Session = Depends(get_db)):
    query = db.query(ApiConfig)
    if model_type:
        query = query.filter(ApiConfig.model_type == model_type)
    total = query.count()
    items = query.offset((page - 1) * page_size).limit(page_size).all()
    return PaginatedResponse(items=items, total=total, page=page, page_size=page_size)

@router.post("", response_model=ApiConfigResponse)
def create_config(data: ApiConfigCreate, db: Session = Depends(get_db)):
    config = ApiConfig(
        name=data.name, model_type=data.model_type, base_url=data.base_url,
        api_key=encrypt_api_key(data.api_key), model_name=data.model_name
    )
    db.add(config)
    db.commit()
    db.refresh(config)
    return config

@router.put("/{config_id}", response_model=ApiConfigResponse)
def update_config(config_id: int, data: ApiConfigUpdate, db: Session = Depends(get_db)):
    config = db.get(ApiConfig, config_id)
    if not config:
        raise HTTPException(404, "Config not found")
    for field, value in data.model_dump(exclude_unset=True).items():
        if field == "api_key" and value:
            value = encrypt_api_key(value)
        setattr(config, field, value)
    db.commit()
    db.refresh(config)
    return config

@router.delete("/{config_id}")
def delete_config(config_id: int, db: Session = Depends(get_db)):
    config = db.get(ApiConfig, config_id)
    if not config:
        raise HTTPException(404, "Config not found")
    db.delete(config)
    db.commit()
    return {"ok": True}

@router.post("/{config_id}/test")
def test_connection(config_id: int, db: Session = Depends(get_db)):
    config = db.get(ApiConfig, config_id)
    if not config:
        raise HTTPException(404, "Config not found")
    api_key = decrypt_api_key(config.api_key)
    client = OpenAI(api_key=api_key, base_url=config.base_url)
    try:
        if config.model_type == "embedding":
            client.embeddings.create(input="test", model=config.model_name)
        else:
            client.chat.completions.create(
                model=config.model_name,
                messages=[{"role": "user", "content": "hi"}],
                max_tokens=5
            )
        return {"ok": True, "message": "Connection successful"}
    except Exception as e:
        return {"ok": False, "message": str(e)}
```

- [ ] **Step 2: Include router in main.py**

Add to `main.py`:
```python
from app.api.configs import router as configs_router
app.include_router(configs_router)
```

- [ ] **Step 3: Test with curl**

Run: `uvicorn app.main:app --reload --port 8000`
Test: `curl -X POST http://localhost:8000/api/configs -H 'Content-Type: application/json' -d '{"name":"test","model_type":"embedding","base_url":"https://api.openai.com/v1","api_key":"sk-test","model_name":"text-embedding-3-small"}'`
Expected: 200 with config response (api_key excluded).

- [ ] **Step 4: Commit**

```bash
git add backend/app/api/configs.py backend/app/main.py
git commit -m "feat: add API configs CRUD with connection test"
```

---

## Task 5: Document Extractors

**Files:**
- Create: `backend/app/services/extractors/txt.py`
- Create: `backend/app/services/extractors/markdown.py`
- Create: `backend/app/services/extractors/pdf.py`
- Create: `backend/app/services/extractors/word.py`
- Create: `backend/app/services/extractors/excel.py`
- Create: `backend/app/services/document_parser.py`
- Create: `backend/app/services/text_cleaner.py`

- [ ] **Step 1: Create TXT extractor**

```python
# backend/app/services/extractors/txt.py
import chardet

def extract_txt(file_path: str) -> str:
    with open(file_path, "rb") as f:
        raw = f.read()
    encoding = chardet.detect(raw)["encoding"] or "utf-8"
    return raw.decode(encoding)
```

- [ ] **Step 2: Create Markdown extractor**

Implement header-aware parsing. Track code blocks (```) to avoid false header detection. Split by `# ` headers, return list of `(header, content)` tuples joined as plain text sections.

- [ ] **Step 3: Create PDF extractor**

```python
# backend/app/services/extractors/pdf.py
import pypdfium2 as pdfium

def extract_pdf(file_path: str) -> str:
    pdf = pdfium.PdfDocument(file_path)
    pages = []
    for i in range(len(pdf)):
        page = pdf[i]
        text = page.get_textpage().get_text_range()
        pages.append(text)
    pdf.close()
    return "\n\n".join(pages)
```

- [ ] **Step 4: Create Word extractor**

Use python-docx to extract paragraphs and tables. Convert tables to markdown table format. Join all content.

- [ ] **Step 5: Create Excel extractor**

Use openpyxl to read rows. Detect header row (first non-empty row). Convert each data row to `"column_name":"value"` key-value pair string, semicolon-separated.

- [ ] **Step 6: Create text_cleaner.py**

```python
# backend/app/services/text_cleaner.py
import re

def clean_text(text: str, remove_extra_spaces: bool = True, remove_urls: bool = False) -> str:
    # Remove control characters
    text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]', '', text)
    # Replace invalid symbols
    text = text.replace("<|", "<").replace("|>", ">")
    if remove_extra_spaces:
        text = re.sub(r'\n{3,}', '\n\n', text)
        text = re.sub(r'[ \t]{2,}', ' ', text)
    if remove_urls:
        text = re.sub(r'https?://\S+', '', text)
        text = re.sub(r'\S+@\S+\.\S+', '', text)
    return text.strip()
```

- [ ] **Step 7: Create document_parser.py dispatcher**

```python
# backend/app/services/document_parser.py
from app.services.extractors.txt import extract_txt
from app.services.extractors.markdown import extract_markdown
from app.services.extractors.pdf import extract_pdf
from app.services.extractors.word import extract_word
from app.services.extractors.excel import extract_excel
from app.services.text_cleaner import clean_text

EXTRACTORS = {
    "txt": extract_txt,
    "md": extract_markdown,
    "pdf": extract_pdf,
    "docx": extract_word,
    "xlsx": extract_excel,
}

def parse_document(file_path: str, file_type: str, clean_options: dict = None) -> str:
    extractor = EXTRACTORS.get(file_type)
    if not extractor:
        raise ValueError(f"Unsupported file type: {file_type}")
    text = extractor(file_path)
    options = clean_options or {}
    return clean_text(text, **options)
```

- [ ] **Step 8: Commit**

```bash
git add backend/app/services/
git commit -m "feat: add document extractors and text cleaner"
```

---

## Task 6: Chunking Engine (3 Strategies)

**Files:**
- Create: `backend/app/services/chunking/base.py`
- Create: `backend/app/services/chunking/auto.py`
- Create: `backend/app/services/chunking/custom.py`
- Create: `backend/app/services/chunking/parent_child.py`
- Create: `backend/app/services/chunking/__init__.py`

- [ ] **Step 1: Create base RecursiveCharacterTextSplitter**

```python
# backend/app/services/chunking/base.py
from typing import List

class RecursiveCharacterTextSplitter:
    def __init__(self, max_tokens: int = 500, chunk_overlap: int = 50,
                 separators: List[str] = None):
        self.max_tokens = max_tokens
        self.chunk_overlap = chunk_overlap
        self.separators = separators or ["\n\n", "。", ". ", " ", ""]

    def _token_count(self, text: str) -> int:
        # Approximate: 1 token ≈ 2 CJK chars / 4 latin chars
        cjk = sum(1 for c in text if '\u4e00' <= c <= '\u9fff')
        non_cjk = len(text) - cjk
        return cjk // 2 + non_cjk // 4

    def _split_text(self, text: str, separators: List[str]) -> List[str]:
        if not text:
            return []
        separator = separators[0] if separators else ""
        remaining_separators = separators[1:] if len(separators) > 1 else []

        if separator == "":
            chunks = list(text)
        else:
            chunks = text.split(separator)

        result = []
        current = ""
        for chunk in chunks:
            test = current + (separator if current else "") + chunk
            if self._token_count(test) <= self.max_tokens:
                current = test
            else:
                if current:
                    result.append(current)
                if self._token_count(chunk) > self.max_tokens and remaining_separators:
                    result.extend(self._split_text(chunk, remaining_separators))
                else:
                    current = chunk
                    continue
                current = ""
        if current:
            result.append(current)
        return result

    def _merge_with_overlap(self, chunks: List[str], separator: str = "\n\n") -> List[str]:
        """Merge chunks with overlap: append tail of previous chunk to start of next."""
        if not chunks or self.chunk_overlap <= 0:
            return chunks
        merged = [chunks[0]]
        for i in range(1, len(chunks)):
            prev = chunks[i - 1]
            # Take last chunk_overlap tokens worth of text from previous chunk
            overlap_chars = self.chunk_overlap * 4  # approximate chars per token
            overlap_text = prev[-overlap_chars:] if len(prev) > overlap_chars else prev
            merged.append(overlap_text + separator + chunks[i])
        return merged

    def split(self, text: str) -> List[str]:
        raw = self._split_text(text, self.separators)
        return self._merge_with_overlap(raw)
```

- [ ] **Step 2: Create AutoChunker**

```python
# backend/app/services/chunking/auto.py
from app.services.chunking.base import RecursiveCharacterTextSplitter

class AutoChunker(RecursiveCharacterTextSplitter):
    """Automatic mode: recursive character splitting with defaults."""
    def __init__(self, max_tokens: int = 500, chunk_overlap: int = 50):
        super().__init__(max_tokens=max_tokens, chunk_overlap=chunk_overlap)
```

- [ ] **Step 3: Create CustomChunker**

```python
# backend/app/services/chunking/custom.py
from typing import List
from app.services.chunking.base import RecursiveCharacterTextSplitter

class CustomChunker(RecursiveCharacterTextSplitter):
    """Custom mode: fixed separator first pass, then recursive fallback."""
    def __init__(self, max_tokens: int = 500, chunk_overlap: int = 50,
                 fixed_separator: str = "\n\n"):
        super().__init__(max_tokens=max_tokens, chunk_overlap=chunk_overlap)
        self.fixed_separator = fixed_separator

    def split(self, text: str) -> List[str]:
        # First pass: split by fixed separator
        parts = text.split(self.fixed_separator)
        result = []
        for part in parts:
            if self._token_count(part) <= self.max_tokens:
                result.append(part)
            else:
                # Second pass: recursive split for oversized parts
                result.extend(self._split_text(part, self.separators))
        return [p for p in result if p.strip()]
```

- [ ] **Step 4: Create ParentChildChunker**

```python
# backend/app/services/chunking/parent_child.py
from typing import List, Tuple
from app.services.chunking.custom import CustomChunker

class ParentChildChunker:
    """Parent-child mode: two-level splitting."""
    def __init__(self, parent_max_tokens: int = 1000, parent_overlap: int = 100,
                 parent_separator: str = "\n\n", child_max_tokens: int = 200,
                 child_overlap: int = 50, child_separator: str = "\n\n",
                 parent_mode: str = "paragraph"):
        self.parent_mode = parent_mode  # "paragraph" or "full_doc"
        self.parent_chunker = CustomChunker(parent_max_tokens, parent_overlap, parent_separator)
        self.child_chunker = CustomChunker(child_max_tokens, child_overlap, child_separator)

    def split(self, text: str) -> List[Tuple[str, List[str]]]:
        """Returns list of (parent_text, [child_texts])."""
        if self.parent_mode == "full_doc":
            parents = [text]
        else:
            parents = self.parent_chunker.split(text)
        result = []
        for parent in parents:
            children = self.child_chunker.split(parent)
            result.append((parent, children))
        return result
```

- [ ] **Step 5: Create __init__.py with factory function**

```python
# backend/app/services/chunking/__init__.py
from enum import Enum
from app.services.chunking.auto import AutoChunker
from app.services.chunking.custom import CustomChunker
from app.services.chunking.parent_child import ParentChildChunker

class ChunkStrategy(str, Enum):
    AUTO = "auto"
    CUSTOM = "custom"
    PARENT_CHILD = "parent_child"

def get_chunker(strategy: str, params: dict = None):
    params = params or {}
    if strategy == ChunkStrategy.AUTO:
        return AutoChunker(**{k: v for k, v in params.items() if k in ("max_tokens", "chunk_overlap")})
    elif strategy == ChunkStrategy.CUSTOM:
        return CustomChunker(**{k: v for k, v in params.items() if k in ("max_tokens", "chunk_overlap", "fixed_separator")})
    elif strategy == ChunkStrategy.PARENT_CHILD:
        return ParentChildChunker(**{k: v for k, v in params.items()
            if k in ("parent_max_tokens", "parent_overlap", "parent_separator",
                      "child_max_tokens", "child_overlap", "child_separator", "parent_mode")})
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
```

- [ ] **Step 6: Commit**

```bash
git add backend/app/services/chunking/
git commit -m "feat: add chunking engine with auto, custom, parent-child strategies"
```

---

## Task 7: Embedding Service & Document Upload API

**Files:**
- Create: `backend/app/services/embedding.py`
- Create: `backend/app/api/documents.py`
- Create: `backend/app/api/chunks.py`
- Modify: `backend/app/main.py` (include routers)

- [ ] **Step 1: Create embedding service**

```python
# backend/app/services/embedding.py
from typing import List
from openai import OpenAI
from app.utils.crypto import decrypt_api_key

def get_embeddings(texts: List[str], base_url: str, api_key_encrypted: str, model_name: str) -> List[List[float]]:
    api_key = decrypt_api_key(api_key_encrypted)
    client = OpenAI(api_key=api_key, base_url=base_url)
    # Batch in groups of 100 to avoid token limits
    all_embeddings = []
    batch_size = 100
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        response = client.embeddings.create(input=batch, model=model_name)
        all_embeddings.extend([d.embedding for d in response.data])
    return all_embeddings
```

- [ ] **Step 2: Create documents API**

Endpoints:
- `POST /api/documents/upload` — multipart file upload, saves to uploads/ with UUID name, creates Document record
- `GET /api/documents` — list with pagination
- `GET /api/documents/{id}` — detail
- `DELETE /api/documents/{id}` — delete file + chunks + Chroma collections
- `POST /api/documents/{id}/parse` — trigger parsing + chunking + embedding (accepts chunk strategy, params, embedding_model_id). Creates chunks in DB, embeds and stores in Chroma. For parent-child mode: store parent chunks (chunk_type="parent") and child chunks (chunk_type="child", with parent_id set, chroma_id set).
- `POST /api/documents/{id}/rechunk` — delete existing chunks/collections, re-parse with new strategy

- [ ] **Step 3: Create chunks API**

Endpoints:
- `GET /api/documents/{doc_id}/chunks` — list chunks with pagination, filter by chunk_type

- [ ] **Step 4: Include routers in main.py**

```python
from app.api.documents import router as documents_router
from app.api.chunks import router as chunks_router
app.include_router(documents_router)
app.include_router(chunks_router)
```

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/embedding.py backend/app/api/documents.py backend/app/api/chunks.py backend/app/main.py
git commit -m "feat: add document upload, parsing, chunking, embedding pipeline"
```

---

## Task 8: Test Cases CRUD & File Import

**Files:**
- Create: `backend/app/api/test_cases.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create test_cases router**

Endpoints:
- `GET /api/test-cases` — list with pagination, optional `group_name` filter
- `POST /api/test-cases` — create single test case
- `PUT /api/test-cases/{id}` — update
- `DELETE /api/test-cases/{id}` — delete
- `POST /api/test-cases/import` — upload CSV or Excel file. CSV columns: `query`, `expected_chunks` (optional), `group_name` (optional). Excel: same columns. Parse file, create TestCase records for each row.
- `GET /api/test-cases/groups` — list distinct group names

- [ ] **Step 2: Implement file import logic**

For CSV: use built-in `csv` module. For Excel: use `openpyxl`. Auto-detect header row, map columns by name. Return count of imported cases.

- [ ] **Step 3: Include router in main.py**

- [ ] **Step 4: Commit**

```bash
git add backend/app/api/test_cases.py backend/app/main.py
git commit -m "feat: add test cases CRUD with CSV/Excel import"
```

---

## Task 9: Scoring Dimensions CRUD

**Files:**
- Create: `backend/app/api/scoring.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create scoring router**

Endpoints:
- `GET /api/scoring-dimensions` — list all
- `POST /api/scoring-dimensions` — create
- `PUT /api/scoring-dimensions/{id}` — update
- `DELETE /api/scoring-dimensions/{id}` — delete (prevent delete if is_default)
- `POST /api/scoring-dimensions/seed` — seed default dimensions (Relevance, Completeness, Accuracy) if none exist

- [ ] **Step 2: Seed defaults on first startup in lifespan**

In `main.py` lifespan, check if scoring_dimensions table is empty, if so create defaults.

- [ ] **Step 3: Include router, commit**

```bash
git add backend/app/api/scoring.py backend/app/main.py
git commit -m "feat: add scoring dimensions CRUD with default seeding"
```

---

## Task 10: Retrieval Service

**Files:**
- Create: `backend/app/services/retrieval.py`

- [ ] **Step 1: Create retrieval service**

```python
# backend/app/services/retrieval.py
from typing import List, Dict, Any
from app.utils.chroma_client import get_or_create_collection
from app.services.embedding import get_embeddings

def retrieve_similar_chunks(
    query: str, document_ids: List[int], embedding_config: dict,
    top_k: int = 5, threshold: float = 0.0
) -> List[Dict[str, Any]]:
    """
    Query Chroma for similar chunks across multiple document collections.
    embedding_config: {id, base_url, api_key, model_name}
    Returns: [{chunk_id, content, score, document_id, metadata}, ...]
    """
    query_embedding = get_embeddings(
        [query], embedding_config["base_url"],
        embedding_config["api_key"], embedding_config["model_name"]
    )[0]

    all_results = []
    for doc_id in document_ids:
        collection = get_or_create_collection(doc_id, embedding_config["id"])
        results = collection.query(
            query_embeddings=[query_embedding],
            n_results=top_k,
            include=["documents", "metadatas", "distances"]
        )
        if results["ids"][0]:
            for i, chunk_id in enumerate(results["ids"][0]):
                score = 1 - results["distances"][0][i]  # cosine distance to similarity
                if score >= threshold:
                    all_results.append({
                        "chroma_id": chunk_id,
                        "content": results["documents"][0][i],
                        "score": score,
                        "document_id": doc_id,
                        "metadata": results["metadatas"][0][i] if results["metadatas"] else {},
                    })

    # Sort by score descending, take top_k overall
    all_results.sort(key=lambda x: x["score"], reverse=True)
    return all_results[:top_k]
```

- [ ] **Step 2: Commit**

```bash
git add backend/app/services/retrieval.py
git commit -m "feat: add retrieval service with multi-document Chroma search"
```

---

## Task 11: LLM Scoring Service

**Files:**
- Create: `backend/app/services/scoring.py`

- [ ] **Step 1: Create scoring service with 3 modes**

```python
# backend/app/services/scoring.py
import json
import re
from typing import List, Dict, Any
from openai import OpenAI
from app.utils.crypto import decrypt_api_key

SINGLE_PROMPT = """You are evaluating the relevance of a retrieved text chunk to a query.

Query: {query}
Retrieved chunk: {chunk}

Rate the relevance on a scale of 0-5. Respond in JSON:
{{"score": <0-5>, "reasoning": "<one sentence explanation>"}}"""

MULTI_PROMPT = """You are evaluating a retrieved text chunk against a query on multiple dimensions.

Query: {query}
Retrieved chunk: {chunk}

Rate each dimension on 0-5. Respond in JSON:
{{
  "relevance": {{"score": <0-5>, "reasoning": "<explanation>"}},
  "completeness": {{"score": <0-5>, "reasoning": "<explanation>"}},
  "accuracy": {{"score": <0-5>, "reasoning": "<explanation>"}}
}}"""

CUSTOM_PROMPT = """You are evaluating a retrieved text chunk against a query.

Query: {query}
Retrieved chunk: {chunk}

Evaluate on these dimensions:
{dimensions_text}

Respond in JSON with each dimension as a key:
{expected_json}"""

def _parse_json_response(text: str) -> dict:
    """Extract JSON from LLM response, handling markdown code fences."""
    # Try direct parse
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # Try extracting from code fence
    match = re.search(r'```(?:json)?\s*(.*?)\s*```', text, re.DOTALL)
    if match:
        try:
            return json.loads(match.group(1))
        except json.JSONDecodeError:
            pass
    # Try finding first { to last }
    start = text.find('{')
    end = text.rfind('}')
    if start != -1 and end != -1:
        try:
            return json.loads(text[start:end + 1])
        except json.JSONDecodeError:
            pass
    raise ValueError(f"Could not parse JSON from: {text[:200]}")

def score_chunk(
    query: str, chunk_content: str, scoring_mode: str,
    dimensions: List[Dict], llm_config: dict,
    weights: Dict[str, float] = None
) -> Dict[str, Any]:
    """
    Score a single chunk. Returns {scores: dict, reasoning: str, total_score: float}.
    llm_config: {base_url, api_key (encrypted), model_name}
    """
    api_key = decrypt_api_key(llm_config["api_key"])
    client = OpenAI(api_key=api_key, base_url=llm_config["base_url"])

    if scoring_mode == "single":
        prompt = SINGLE_PROMPT.format(query=query, chunk=chunk_content)
    elif scoring_mode == "multi":
        prompt = MULTI_PROMPT.format(query=query, chunk=chunk_content)
    else:  # custom
        dims_text = "\n".join(f"- {d['name']} (0-{d['max_score']}): {d['description']}" for d in dimensions)
        expected = {d["name"]: {"score": f"<0-{d['max_score']}>", "reasoning": "<explanation>"} for d in dimensions}
        prompt = CUSTOM_PROMPT.format(query=query, chunk=chunk_content,
                                       dimensions_text=dims_text, expected_json=json.dumps(expected, indent=2))

    try:
        response = client.chat.completions.create(
            model=llm_config["model_name"],
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0,
        )
        parsed = _parse_json_response(response.choices[0].message.content)
    except Exception as e:
        return {"scores": {}, "reasoning": f"Error: {str(e)}", "total_score": 0}

    # Calculate total score
    if scoring_mode == "single":
        return {
            "scores": {"relevance": parsed.get("score", 0)},
            "reasoning": parsed.get("reasoning", ""),
            "total_score": parsed.get("score", 0)
        }
    else:
        scores = {}
        reasonings = []
        for key, val in parsed.items():
            if isinstance(val, dict):
                scores[key] = val.get("score", 0)
                reasonings.append(f"{key}: {val.get('reasoning', '')}")
        # Apply weights or average
        if weights:
            total = sum(scores.get(k, 0) * w for k, w in weights.items())
            total /= sum(weights.values()) if weights.values() else 1
        else:
            total = sum(scores.values()) / len(scores) if scores else 0
        return {
            "scores": scores,
            "reasoning": " | ".join(reasonings),
            "total_score": round(total, 2)
        }
```

- [ ] **Step 2: Commit**

```bash
git add backend/app/services/scoring.py
git commit -m "feat: add LLM scoring service with 3 modes"
```

---

## Task 12: Async Task Manager & Task Executor

**Files:**
- Create: `backend/app/services/task_manager.py`
- Create: `backend/app/services/task_executor.py`
- Create: `backend/app/api/tasks.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create task_manager.py**

```python
# backend/app/services/task_manager.py
import time
import threading
from concurrent.futures import ThreadPoolExecutor
from sqlalchemy.orm import Session
from app.database import SessionLocal
from app.models.test_task import TestTask
from app.config import settings

class TaskManager:
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=settings.TASK_POOL_SIZE)
        self._running = False
        self._scanner_thread = None

    def start(self):
        self._running = True
        self._scanner_thread = threading.Thread(target=self._scan_loop, daemon=True)
        self._scanner_thread.start()

    def stop(self):
        self._running = False
        self.executor.shutdown(wait=False)

    def _scan_loop(self):
        while self._running:
            self._process_pending_tasks()
            time.sleep(2)  # scan every 2 seconds

    def _process_pending_tasks(self):
        db = SessionLocal()
        try:
            pending = db.query(TestTask).filter(
                TestTask.status == "pending",
                TestTask.parent_task_id == None  # noqa: E711
            ).all()
            for task in pending:
                task.status = "running"
                db.commit()
                self.executor.submit(self._run_task, task.id)
        except Exception:
            db.rollback()
        finally:
            db.close()

    def _run_task(self, task_id: int):
        from app.services.task_executor import execute_task
        db = SessionLocal()
        try:
            execute_task(task_id, db)
        except Exception as e:
            task = db.query(TestTask).get(task_id)
            if task:
                task.status = "failed"
                task.error_message = str(e)
                db.commit()
        finally:
            db.close()

task_manager = TaskManager()
```

- [ ] **Step 2: Create task_executor.py**

```python
# backend/app/services/task_executor.py
import time
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor, as_completed
from sqlalchemy.orm import Session
from app.models.test_task import TestTask
from app.models.test_result import TestResult
from app.models.test_case import TestCase
from app.models.api_config import ApiConfig
from app.services.retrieval import retrieve_similar_chunks
from app.services.scoring import score_chunk
from app.utils.crypto import decrypt_api_key

def execute_task(task_id: int, db: Session):
    task = db.get(TestTask, task_id)
    if not task:
        return

    # Check if this is a parent task (multi-model comparison)
    if task.parent_task_id is None and task.type == "batch":
        # Check for child tasks
        children = db.query(TestTask).filter(TestTask.parent_task_id == task_id).all()
        if children:
            _monitor_parent_task(task, children, db)
            return

    embedding_config = db.get(ApiConfig, task.embedding_model_id)
    scoring_config = db.get(ApiConfig, task.scoring_model_id) if task.scoring_model_id else None
    config_params = task.config_params or {}
    top_k = config_params.get("top_k", 5)
    threshold = config_params.get("threshold", 0.0)
    concurrency = config_params.get("concurrency", 5)

    emb_dict = {
        "id": embedding_config.id, "base_url": embedding_config.base_url,
        "api_key": embedding_config.api_key, "model_name": embedding_config.model_name,
    }

    if task.type == "single":
        # Single test: one query from test_case or direct
        test_cases = db.query(TestCase).filter(TestCase.id.in_(task.test_case_ids or [])).all()
        if not test_cases:
            task.status = "failed"
            task.error_message = "No test cases found"
            db.commit()
            return
        _execute_single(task, test_cases[0], emb_dict, scoring_config, top_k, threshold, db)
    else:
        # Batch test: iterate all test cases with concurrency
        test_cases = db.query(TestCase).filter(TestCase.id.in_(task.test_case_ids or [])).all()
        _execute_batch(task, test_cases, emb_dict, scoring_config, top_k, threshold, concurrency, db)

    task.status = "completed"
    task.progress = 100
    task.completed_at = datetime.utcnow()
    db.commit()

def _execute_single(task, test_case, emb_dict, scoring_config, top_k, threshold, db):
    results = retrieve_similar_chunks(
        test_case.query, task.document_ids, emb_dict, top_k, threshold
    )
    score_data = None
    if scoring_config and task.scoring_mode:
        llm_dict = {
            "base_url": scoring_config.base_url,
            "api_key": scoring_config.api_key,
            "model_name": scoring_config.model_name,
        }
        dimensions = task.scoring_dimensions_snapshot or []
        score_data = score_chunk(
            test_case.query, results[0]["content"] if results else "",
            task.scoring_mode, dimensions, llm_dict
        )
    result = TestResult(
        task_id=task.id, test_case_id=test_case.id, query=test_case.query,
        retrieved_chunks=[r["content"] for r in results],
        similarity_scores=[r["score"] for r in results],
        llm_scores=score_data["scores"] if score_data else None,
        llm_reasoning=score_data["reasoning"] if score_data else None,
        total_score=score_data["total_score"] if score_data else None,
    )
    db.add(result)
    task.progress = 100
    db.commit()

def _execute_batch(task, test_cases, emb_dict, scoring_config, top_k, threshold, concurrency, db):
    total = len(test_cases)
    completed = 0

    def process_one(tc):
        """Process a single test case (retrieval + scoring)."""
        results = retrieve_similar_chunks(tc.query, task.document_ids, emb_dict, top_k, threshold)
        score_data = None
        if scoring_config and task.scoring_mode:
            llm_dict = {
                "base_url": scoring_config.base_url,
                "api_key": scoring_config.api_key,
                "model_name": scoring_config.model_name,
            }
            dims = task.scoring_dimensions_snapshot or []
            # Score each retrieved chunk, aggregate
            chunk_scores = []
            for r in results:
                for attempt in range(3):  # retry up to 3 times
                    try:
                        s = score_chunk(tc.query, r["content"], task.scoring_mode, dims, llm_dict)
                        chunk_scores.append(s)
                        break
                    except Exception:
                        if attempt == 2:
                            chunk_scores.append({"scores": {}, "reasoning": "Error after 3 retries", "total_score": 0})
                        time.sleep(1)
            avg_score = sum(s["total_score"] for s in chunk_scores) / len(chunk_scores) if chunk_scores else 0
            score_data = {
                "scores": {f"chunk_{i}": s["scores"] for i, s in enumerate(chunk_scores)},
                "reasoning": " | ".join(s["reasoning"] for s in chunk_scores),
                "total_score": round(avg_score, 2),
            }
        return TestResult(
            task_id=task.id, test_case_id=tc.id, query=tc.query,
            retrieved_chunks=[r["content"] for r in results],
            similarity_scores=[r["score"] for r in results],
            llm_scores=score_data["scores"] if score_data else None,
            llm_reasoning=score_data["reasoning"] if score_data else None,
            total_score=score_data["total_score"] if score_data else None,
        )

    # Use thread pool for concurrent LLM calls
    with ThreadPoolExecutor(max_workers=concurrency) as pool:
        futures = {pool.submit(process_one, tc): tc for tc in test_cases}
        for future in as_completed(futures):
            # Check cancellation between each result
            db.refresh(task)
            if task.cancel_requested:
                task.status = "cancelled"
                db.commit()
                pool.shutdown(wait=False, cancel_futures=True)
                return
            result = future.result()
            db.add(result)
            completed += 1
            task.progress = int(completed / total * 100)
            db.commit()

def _monitor_parent_task(parent_task, children, db):
    """Monitor child tasks until all complete."""
    while True:
        db.expire_all()
        statuses = [db.get(TestTask, c.id).status for c in children]
        if all(s in ("completed", "failed", "cancelled") for s in statuses):
            parent_task.status = "completed" if all(s == "completed" for s in statuses) else "failed"
            parent_task.progress = 100
            parent_task.completed_at = datetime.utcnow()
            db.commit()
            return
        # Update parent progress as average of children
        progresses = [db.get(TestTask, c.id).progress for c in children]
        parent_task.progress = int(sum(progresses) / len(progresses))
        db.commit()
        time.sleep(2)
```

- [ ] **Step 3: Create tasks API**

Endpoints:
- `GET /api/tasks` — list tasks with pagination, filter by status/type
- `GET /api/tasks/{id}` — task detail with sub-tasks if parent
- `GET /api/tasks/{id}/progress` — lightweight progress endpoint (just status + progress)
- `POST /api/tasks/{id}/cancel` — set cancel_requested=true
- `POST /api/tasks/{id}/retry` — reset status to pending, clear error
- `GET /api/tasks/{id}/results` — list test results for a task with pagination. Returns `PaginatedResponse[TestResultResponse]`. Supports sorting by `total_score`, filtering by `min_score`.

- [ ] **Step 4: Start/stop task manager in lifespan**

```python
# In main.py lifespan
from app.services.task_manager import task_manager
# startup:
task_manager.start()
# shutdown:
task_manager.stop()
```

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/task_manager.py backend/app/services/task_executor.py backend/app/api/tasks.py backend/app/main.py
git commit -m "feat: add async task manager with ThreadPoolExecutor"
```

---

## Task 13: Tests API (Create Single & Batch Tests)

**Files:**
- Create: `backend/app/api/tests.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create tests router**

Endpoints:

`POST /api/tests/single` — Create a single retrieval test:
```json
{
  "query": "...",
  "document_ids": [1, 2],
  "embedding_model_id": 1,
  "scoring_model_id": 2,     // optional
  "scoring_mode": "single",  // optional
  "config_params": {"top_k": 5, "threshold": 0.0}
}
```
Creates a TestTask (type="single"), sets to pending. Task manager picks it up.

`POST /api/tests/batch` — Create a batch test:
```json
{
  "name": "Batch Test v1",
  "document_ids": [1, 2],
  "test_case_ids": [1, 2, 3, ...],
  "embedding_model_ids": [1, 2],    // multiple for comparison
  "scoring_model_id": 3,
  "scoring_mode": "multi",
  "config_params": {"top_k": 5, "threshold": 0.0, "concurrency": 5}
}
```
If multiple embedding models: create parent task + child tasks per model. If single model: create one task directly.

Snapshot scoring dimensions into `scoring_dimensions_snapshot` at creation time.

- [ ] **Step 2: Include router, commit**

```bash
git add backend/app/api/tests.py backend/app/main.py
git commit -m "feat: add single and batch test creation APIs"
```

---

## Task 14: Report Export Service & API

**Files:**
- Create: `backend/app/services/report_export.py`
- Create: `backend/app/api/reports.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create Excel export**

Use openpyxl to create workbook with 3 sheets:
- Sheet 1 "Overview": task name, time, models, scoring mode, average score, Top-K
- Sheet 2 "Detailed Results": columns = query, chunk content, similarity score, per-dimension scores, reasoning
- Sheet 3 "Model Comparison": (only if parent task with children) per-model averages

Return file path.

- [ ] **Step 2: Create PDF export**

Use reportlab:
- Cover page with title and summary
- Stats section with charts (using reportlab.graphics): score distribution bar chart, dimension radar, model comparison bars
- Detailed results table

Return file path.

- [ ] **Step 3: Create reports API**

Endpoints:
- `POST /api/reports/export` — body: `{task_id, format: "excel"|"pdf"}`. Generates file, returns download URL.
- `GET /api/reports/download/{filename}` — serve file from exports/ directory.

- [ ] **Step 4: Add export cleanup in lifespan**

On startup, delete files in `exports/` older than 24 hours.

- [ ] **Step 5: Include router, commit**

```bash
git add backend/app/services/report_export.py backend/app/api/reports.py backend/app/main.py
git commit -m "feat: add report export service (Excel + PDF)"
```

---

## Task 15: Frontend Project Scaffolding

**Files:**
- Create: `frontend/` project via create-next-app
- Create: `frontend/src/lib/api.ts`
- Create: `frontend/src/lib/types.ts`
- Create: `frontend/src/components/AppLayout.tsx`
- Create: `frontend/src/app/layout.tsx`

- [ ] **Step 1: Create Next.js project**

Run:
```bash
npx create-next-app@latest frontend --typescript --tailwind --app --src-dir --no-import-alias
cd frontend
npm install antd @ant-design/icons axios dayjs
```

- [ ] **Step 2: Create types.ts mirroring backend schemas**

```typescript
// frontend/src/lib/types.ts
export interface ApiConfig {
  id: number; name: string; model_type: 'embedding' | 'llm';
  base_url: string; model_name: string;
  created_at: string; updated_at: string;
}

export interface Document {
  id: number; filename: string; file_type: string; file_size: number;
  status: string; chunk_strategy: string | null; chunk_params: any;
  created_at: string; updated_at: string;
}

// ... Chunk, TestCase, TestTask, TestResult, ScoringDimension

export interface PaginatedResponse<T> {
  items: T[]; total: number; page: number; page_size: number;
}
```

- [ ] **Step 3: Create api.ts with Axios instance and typed functions**

```typescript
// frontend/src/lib/api.ts
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000',
});

// Configs
export const listConfigs = (params?: any) => api.get('/api/configs', { params });
export const createConfig = (data: any) => api.post('/api/configs', data);
export const updateConfig = (id: number, data: any) => api.put(`/api/configs/${id}`, data);
export const deleteConfig = (id: number) => api.delete(`/api/configs/${id}`);
export const testConfig = (id: number) => api.post(`/api/configs/${id}/test`);

// Documents
export const listDocuments = (params?: any) => api.get('/api/documents', { params });
export const uploadDocument = (file: File) => { /* FormData upload */ };
export const parseDocument = (id: number, data: any) => api.post(`/api/documents/${id}/parse`, data);
export const deleteDocument = (id: number) => api.delete(`/api/documents/${id}`);

// Chunks
export const listChunks = (docId: number, params?: any) => api.get(`/api/documents/${docId}/chunks`, { params });

// Test Cases
export const listTestCases = (params?: any) => api.get('/api/test-cases', { params });
export const createTestCase = (data: any) => api.post('/api/test-cases', data);
export const importTestCases = (file: File) => { /* FormData upload */ };

// Tests
export const createSingleTest = (data: any) => api.post('/api/tests/single', data);
export const createBatchTest = (data: any) => api.post('/api/tests/batch', data);

// Tasks
export const listTasks = (params?: any) => api.get('/api/tasks', { params });
export const getTaskProgress = (id: number) => api.get(`/api/tasks/${id}/progress`);
export const cancelTask = (id: number) => api.post(`/api/tasks/${id}/cancel`);

// Scoring Dimensions
export const listDimensions = () => api.get('/api/scoring-dimensions');
export const createDimension = (data: any) => api.post('/api/scoring-dimensions', data);

// Reports
export const exportReport = (data: any) => api.post('/api/reports/export', data);

export default api;
```

- [ ] **Step 4: Create AppLayout component with sidebar navigation**

Use Ant Design `Layout`, `Menu`, `Sider`. Menu items: Dashboard, Settings, Documents, Test Cases, Tests, Results, Scoring Dimensions. Use Next.js `usePathname` for active menu highlighting.

- [ ] **Step 5: Create root layout.tsx**

Wrap with Ant Design `ConfigProvider` (zh_CN locale), import AppLayout. Set up `AntdRegistry` for SSR compatibility.

- [ ] **Step 6: Create .env.local**

```
NEXT_PUBLIC_API_URL=http://localhost:8000
```

- [ ] **Step 7: Verify frontend starts**

Run: `cd frontend && npm run dev`
Expected: Next.js running on localhost:3000 with sidebar layout.

- [ ] **Step 8: Commit**

```bash
git add frontend/
git commit -m "feat: scaffold Next.js frontend with Ant Design layout"
```

---

## Task 16: Settings Page (API Configs UI)

**Files:**
- Create: `frontend/src/app/settings/page.tsx`

- [ ] **Step 1: Build settings page**

Use `@ui-ux-pro-max` skill for design. Features:
- Ant Design `Table` listing all configs (name, model_type, base_url, model_name, actions)
- "Add Config" button → `Modal` with `Form` (name, model_type select, base_url, api_key password input, model_name)
- Edit action → same modal pre-filled
- Delete action → `Popconfirm`
- "Test Connection" button per row → calls test endpoint, shows success/error `message`

- [ ] **Step 2: Commit**

```bash
git add frontend/src/app/settings/
git commit -m "feat: add API configs management page"
```

---

## Task 17: Documents Page

**Files:**
- Create: `frontend/src/app/documents/page.tsx`
- Create: `frontend/src/app/documents/[id]/page.tsx`
- Create: `frontend/src/components/DocumentUpload.tsx`
- Create: `frontend/src/components/ChunkConfigForm.tsx`

- [ ] **Step 1: Build document list page**

Use `@ui-ux-pro-max` skill. Features:
- Table: filename, type, size, status (with `Tag` colors), chunk strategy, actions
- Upload button → `DocumentUpload` modal
- Delete action

- [ ] **Step 2: Build DocumentUpload component**

`Upload.Dragger` for file selection (accept .txt,.md,.pdf,.docx,.xlsx). After upload, show `ChunkConfigForm`.

- [ ] **Step 3: Build ChunkConfigForm component**

- Strategy selector: Radio.Group (Auto / Custom / Parent-Child)
- Auto: max_tokens slider (50-2000, default 500), overlap slider
- Custom: + fixed_separator input
- Parent-Child: parent params group + child params group + parent_mode select
- Embedding model selector: dropdown from API configs (filter model_type="embedding")
- Submit: calls parse endpoint

- [ ] **Step 4: Build document detail page**

- Document info card
- Chunks table with pagination: index, content (truncated), type, chroma_id
- "Re-chunk" button → ChunkConfigForm modal

- [ ] **Step 5: Commit**

```bash
git add frontend/src/app/documents/ frontend/src/components/DocumentUpload.tsx frontend/src/components/ChunkConfigForm.tsx
git commit -m "feat: add documents management pages"
```

---

## Task 18: Test Cases Page

**Files:**
- Create: `frontend/src/app/test-cases/page.tsx`
- Create: `frontend/src/components/TestCaseImport.tsx`

- [ ] **Step 1: Build test cases page**

- Table: group_name, query (truncated), expected_chunks, actions
- Filter by group_name (Select with distinct groups)
- Add button → Modal with Form (group_name, query, expected_chunks textarea)
- Edit/Delete actions
- "Import" button → TestCaseImport modal

- [ ] **Step 2: Build TestCaseImport component**

- Upload.Dragger for CSV/Excel
- Preview table showing parsed rows before confirm
- group_name input for batch assignment
- Confirm button → calls import endpoint

- [ ] **Step 3: Commit**

```bash
git add frontend/src/app/test-cases/ frontend/src/components/TestCaseImport.tsx
git commit -m "feat: add test cases page with file import"
```

---

## Task 19: Tests Page (Single & Batch)

**Files:**
- Create: `frontend/src/app/tests/page.tsx`
- Create: `frontend/src/app/tests/[id]/page.tsx`
- Create: `frontend/src/components/BatchTestForm.tsx`
- Create: `frontend/src/components/TaskProgress.tsx`

- [ ] **Step 1: Build tests page with tabs**

Tabs:
- **Single Test**: query input, document multi-select, embedding model select, Top-K slider, "Test" button. Results shown inline below: table of retrieved chunks with scores.
- **Batch Test**: BatchTestForm component. Below: task list table (name, status, progress, created_at, actions).

- [ ] **Step 2: Build BatchTestForm**

- Task name input
- Document multi-select
- Test case select (multi-select or group filter)
- Embedding models multi-select (checkbox group from API configs)
- Scoring model select
- Scoring mode radio (single/multi/custom)
- Top-K, threshold, concurrency number inputs
- "Start Batch Test" button

- [ ] **Step 3: Build TaskProgress component**

- Poll `GET /api/tasks/{id}/progress` every 2 seconds
- Show `Progress` bar with percentage
- Show status badge
- Cancel button (if running)
- Auto-stop polling when completed/failed/cancelled

- [ ] **Step 4: Build task detail page**

- Task info card (name, type, status, models, scoring mode)
- Progress component
- Sub-tasks list (if parent task)
- Results summary when completed
- "View Results" button → navigate to /results/[taskId]
- "Export" button → export modal

- [ ] **Step 5: Commit**

```bash
git add frontend/src/app/tests/ frontend/src/components/BatchTestForm.tsx frontend/src/components/TaskProgress.tsx
git commit -m "feat: add tests page with single and batch test UI"
```

---

## Task 20: Results Page & Model Comparison

**Files:**
- Create: `frontend/src/app/results/[taskId]/page.tsx`
- Create: `frontend/src/components/ResultsTable.tsx`
- Create: `frontend/src/components/ModelComparisonChart.tsx`

- [ ] **Step 1: Build results page**

- Task summary card at top
- Tabs: "Detailed Results" | "Model Comparison" (only if parent task)
- Export buttons: "Export Excel" / "Export PDF"

- [ ] **Step 2: Build ResultsTable**

Ant Design Table with columns:
- Query
- Retrieved Chunk (expandable row to show full content)
- Similarity Score (with color coding)
- Per-dimension LLM Scores
- Reasoning (tooltip or expandable)
- Total Score (sortable)

Supports pagination, sorting, filtering.

- [ ] **Step 3: Build ModelComparisonChart**

For parent tasks with multiple child tasks (different embedding models):
- Bar chart comparing average scores per model
- Per-dimension comparison table
- Use Ant Design Charts or simple HTML/CSS bars (keep it simple, no heavy chart lib)

- [ ] **Step 4: Commit**

```bash
git add frontend/src/app/results/ frontend/src/components/ResultsTable.tsx frontend/src/components/ModelComparisonChart.tsx
git commit -m "feat: add results page with model comparison"
```

---

## Task 21: Scoring Dimensions Page

**Files:**
- Create: `frontend/src/app/scoring/page.tsx`

- [ ] **Step 1: Build scoring dimensions page**

- Table: name, description, max_score, is_default (tag), actions
- Add button → Modal with Form
- Edit/Delete (prevent delete of defaults)

- [ ] **Step 2: Commit**

```bash
git add frontend/src/app/scoring/
git commit -m "feat: add scoring dimensions management page"
```

---

## Task 22: Dashboard Page

**Files:**
- Create: `frontend/src/app/page.tsx`

- [ ] **Step 1: Build dashboard**

Use `@ui-ux-pro-max` skill. Features:
- Statistics cards: total documents, total test cases, recent tasks count, average score
- Recent tasks table (last 10): name, status, progress, created_at
- Quick action buttons: Upload Document, Create Test, View Results

- [ ] **Step 2: Commit**

```bash
git add frontend/src/app/page.tsx
git commit -m "feat: add dashboard page"
```

---

## Task 23: README & Deployment Config

**Files:**
- Create: `README.md`
- Create: `.env.example`
- Create: `docker-compose.yml`

- [ ] **Step 1: Write README.md**

Contents:
- Project overview
- Prerequisites (Python 3.11+, Node.js 18+)
- Backend setup: venv, pip install, alembic upgrade, uvicorn
- Frontend setup: npm install, npm run dev
- Chroma deployment: explain PersistentClient (auto, no separate setup needed), mention `CHROMA_PERSIST_DIR` config
- PostgreSQL setup (optional): install PostgreSQL, set `DATABASE_URL=postgresql://user:pass@localhost/embedding_check`
- Environment variables reference
- Docker Compose deployment option

- [ ] **Step 2: Create .env.example**

```
DATABASE_URL=sqlite:///./embedding_check.db
# DATABASE_URL=postgresql://user:password@localhost:5432/embedding_check
CHROMA_PERSIST_DIR=./chroma_data
CORS_ORIGINS=http://localhost:3000
UPLOAD_DIR=./uploads
EXPORT_DIR=./exports
MAX_UPLOAD_SIZE=52428800
ENCRYPTION_KEY=
TASK_POOL_SIZE=4
```

- [ ] **Step 3: Create docker-compose.yml (optional)**

```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    volumes:
      - ./data/uploads:/app/uploads
      - ./data/chroma:/app/chroma_data
      - ./data/db:/app/data
    env_file: .env
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    depends_on: [backend]
```

- [ ] **Step 4: Commit**

```bash
git add README.md .env.example docker-compose.yml
git commit -m "docs: add README with deployment instructions"
```

---

## Task Order & Dependencies

```
Task 1 (Scaffold) → Task 2 (Models) → Task 3 (Schemas/Utils)
  ├─→ Task 4 (Configs API)
  ├─→ Task 5 (Extractors) → Task 6 (Chunking) → Task 7 (Documents API + Embedding)
  ├─→ Task 8 (Test Cases API)
  ├─→ Task 9 (Scoring Dims API)
  ├─→ Task 7 → Task 10 (Retrieval) → Task 11 (Scoring) → Task 12 (Task Manager) → Task 13 (Tests API)
  ├─→ Task 12 → Task 14 (Reports)
  ├─→ Task 15 (Frontend Scaffold) → Tasks 16-22 (Frontend pages, can be parallelized)
  └─→ Task 23 (README)
```

**Parallelization notes:**
- Tasks 4, 5, 8, 9, 15 can run in parallel after Task 3
- Task 10 depends on Task 7 (needs embedding.py)
- Tasks 16-22 are independent of each other after Task 15
- Frontend tasks can run in parallel with backend tasks
