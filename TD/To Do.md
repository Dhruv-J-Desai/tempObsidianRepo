Here’s a **senior-level FastAPI CRUD scaffold** you can use in interviews. It’s **small enough to finish in 45–60 mins**, but structured enough to evaluate **architecture, DI, validation, error handling, and tests**.

It uses **SQLite** by default (zero setup), but is designed so you can swap to Postgres easily.

---

## Project layout (what you hand the candidate)

```text
employee_api/
├─ app/
│  ├─ main.py
│  ├─ deps.py
│  ├─ core/
│  │  ├─ config.py
│  │  └─ errors.py
│  ├─ db/
│  │  ├─ base.py
│  │  ├─ session.py
│  │  └─ models.py
│  ├─ schemas/
│  │  └─ employee.py
│  ├─ repositories/
│  │  └─ employee_repo.py
│  ├─ services/
│  │  └─ employee_service.py
│  └─ api/
│     └─ employees.py
├─ tests/
│  └─ test_employees.py
├─ pyproject.toml
└─ README.md
```

---

## `pyproject.toml` (minimal, interview-friendly)

```toml
[project]
name = "employee-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "fastapi>=0.110",
  "uvicorn[standard]>=0.27",
  "pydantic>=2.6",
  "pydantic-settings>=2.2",
  "sqlalchemy>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "httpx>=0.27", "ruff>=0.3"]

[tool.ruff]
line-length = 100
```

---

## `app/core/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    app_name: str = "Employee API"
    database_url: str = "sqlite:///./employees.db"  # swap for Postgres in real use

settings = Settings()
```

---

## `app/db/base.py`

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

---

## `app/db/session.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

connect_args = {"check_same_thread": False} if settings.database_url.startswith("sqlite") else {}
engine = create_engine(settings.database_url, pool_pre_ping=True, connect_args=connect_args)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## `app/db/models.py`

```python
from datetime import datetime
from sqlalchemy import String, Boolean, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base

class Employee(Base):
    __tablename__ = "employees"

    id: Mapped[int] = mapped_column(primary_key=True)
    employee_code: Mapped[str] = mapped_column(String(32), unique=True, index=True)
    first_name: Mapped[str] = mapped_column(String(100))
    last_name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    department: Mapped[str | None] = mapped_column(String(100), nullable=True)
    title: Mapped[str | None] = mapped_column(String(100), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )
```

---

## `app/schemas/employee.py`

```python
from pydantic import BaseModel, EmailStr, Field
from typing import Optional

class EmployeeCreate(BaseModel):
    employee_code: str = Field(min_length=2, max_length=32)
    first_name: str = Field(min_length=1, max_length=100)
    last_name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    department: Optional[str] = Field(default=None, max_length=100)
    title: Optional[str] = Field(default=None, max_length=100)

class EmployeeUpdate(BaseModel):
    first_name: Optional[str] = Field(default=None, min_length=1, max_length=100)
    last_name: Optional[str] = Field(default=None, min_length=1, max_length=100)
    email: Optional[EmailStr] = None
    department: Optional[str] = Field(default=None, max_length=100)
    title: Optional[str] = Field(default=None, max_length=100)
    is_active: Optional[bool] = None

class EmployeeOut(BaseModel):
    id: int
    employee_code: str
    first_name: str
    last_name: str
    email: EmailStr
    department: Optional[str] = None
    title: Optional[str] = None
    is_active: bool

    class Config:
        from_attributes = True
```

---

## `app/core/errors.py` (domain errors)

```python
class NotFoundError(Exception):
    pass

class ConflictError(Exception):
    pass
```

---

## `app/repositories/employee_repo.py` (DB access)

```python
from sqlalchemy.orm import Session
from sqlalchemy import select
from app.db.models import Employee

class EmployeeRepo:
    def __init__(self, db: Session):
        self.db = db

    def get(self, employee_id: int) -> Employee | None:
        return self.db.get(Employee, employee_id)

    def get_by_email_or_code(self, email: str, code: str) -> Employee | None:
        stmt = select(Employee).where((Employee.email == email) | (Employee.employee_code == code))
        return self.db.execute(stmt).scalar_one_or_none()

    def list(self, limit: int, offset: int, is_active: bool | None) -> list[Employee]:
        stmt = select(Employee).offset(offset).limit(limit)
        if is_active is not None:
            stmt = stmt.where(Employee.is_active == is_active)
        return list(self.db.execute(stmt).scalars().all())

    def create(self, emp: Employee) -> Employee:
        self.db.add(emp)
        self.db.commit()
        self.db.refresh(emp)
        return emp

    def save(self, emp: Employee) -> Employee:
        self.db.commit()
        self.db.refresh(emp)
        return emp

    def delete(self, emp: Employee) -> None:
        self.db.delete(emp)
        self.db.commit()
```

---

## `app/services/employee_service.py` (business rules)

```python
from app.core.errors import NotFoundError, ConflictError
from app.db.models import Employee
from app.schemas.employee import EmployeeCreate, EmployeeUpdate
from app.repositories.employee_repo import EmployeeRepo

class EmployeeService:
    def __init__(self, repo: EmployeeRepo):
        self.repo = repo

    def create_employee(self, payload: EmployeeCreate) -> Employee:
        existing = self.repo.get_by_email_or_code(payload.email, payload.employee_code)
        if existing:
            raise ConflictError("Employee with same email or employee_code already exists")
        emp = Employee(**payload.model_dump(), is_active=True)
        return self.repo.create(emp)

    def get_employee(self, employee_id: int) -> Employee:
        emp = self.repo.get(employee_id)
        if not emp:
            raise NotFoundError(f"Employee {employee_id} not found")
        return emp

    def list_employees(self, limit: int, offset: int, is_active: bool | None) -> list[Employee]:
        return self.repo.list(limit=limit, offset=offset, is_active=is_active)

    def update_employee(self, employee_id: int, payload: EmployeeUpdate) -> Employee:
        emp = self.get_employee(employee_id)
        data = payload.model_dump(exclude_unset=True)

        # TODO (senior): if email changes, enforce uniqueness check
        for k, v in data.items():
            setattr(emp, k, v)
        return self.repo.save(emp)

    def delete_employee(self, employee_id: int) -> None:
        emp = self.get_employee(employee_id)
        self.repo.delete(emp)
```

---

## `app/deps.py` (dependency injection)

```python
from sqlalchemy.orm import Session
from fastapi import Depends
from app.db.session import get_db
from app.repositories.employee_repo import EmployeeRepo
from app.services.employee_service import EmployeeService

def get_employee_service(db: Session = Depends(get_db)) -> EmployeeService:
    return EmployeeService(EmployeeRepo(db))
```

---

## `app/api/employees.py` (router)

```python
from fastapi import APIRouter, Depends, HTTPException, Query, status
from app.schemas.employee import EmployeeCreate, EmployeeUpdate, EmployeeOut
from app.core.errors import NotFoundError, ConflictError
from app.deps import get_employee_service
from app.services.employee_service import EmployeeService

router = APIRouter(prefix="/employees", tags=["employees"])

@router.post("", response_model=EmployeeOut, status_code=status.HTTP_201_CREATED)
def create_employee(payload: EmployeeCreate, svc: EmployeeService = Depends(get_employee_service)):
    try:
        return svc.create_employee(payload)
    except ConflictError as e:
        raise HTTPException(status_code=409, detail=str(e))

@router.get("/{employee_id}", response_model=EmployeeOut)
def get_employee(employee_id: int, svc: EmployeeService = Depends(get_employee_service)):
    try:
        return svc.get_employee(employee_id)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))

@router.get("", response_model=list[EmployeeOut])
def list_employees(
    svc: EmployeeService = Depends(get_employee_service),
    limit: int = Query(50, ge=1, le=200),
    offset: int = Query(0, ge=0),
    is_active: bool | None = Query(None),
):
    return svc.list_employees(limit=limit, offset=offset, is_active=is_active)

@router.put("/{employee_id}", response_model=EmployeeOut)
def update_employee(employee_id: int, payload: EmployeeUpdate, svc: EmployeeService = Depends(get_employee_service)):
    try:
        return svc.update_employee(employee_id, payload)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except ConflictError as e:
        raise HTTPException(status_code=409, detail=str(e))

@router.delete("/{employee_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_employee(employee_id: int, svc: EmployeeService = Depends(get_employee_service)):
    try:
        svc.delete_employee(employee_id)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

---

## `app/main.py` (startup + health + router)

```python
from fastapi import FastAPI
from app.core.config import settings
from app.db.base import Base
from app.db.session import engine
from app.api.employees import router as employees_router

app = FastAPI(title=settings.app_name)

@app.on_event("startup")
def on_startup():
    # Interview-friendly: auto-create tables (in real prod, use Alembic)
    Base.metadata.create_all(bind=engine)

@app.get("/health")
def health():
    return {"status": "ok"}

app.include_router(employees_router)
```

---

## `tests/test_employees.py` (minimal test, shows senior habits)

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_employee_crud_happy_path():
    payload = {
        "employee_code": "E-1001",
        "first_name": "Ada",
        "last_name": "Lovelace",
        "email": "ada@example.com",
        "department": "Engineering",
        "title": "Staff",
    }

    r = client.post("/employees", json=payload)
    assert r.status_code == 201
    emp = r.json()
    emp_id = emp["id"]

    r = client.get(f"/employees/{emp_id}")
    assert r.status_code == 200

    r = client.put(f"/employees/{emp_id}", json={"title": "Principal"})
    assert r.status_code == 200
    assert r.json()["title"] == "Principal"

    r = client.delete(f"/employees/{emp_id}")
    assert r.status_code == 204
```

---

## `README.md` (candidate instructions)

```md
### Run
pip install -e ".[dev]"
uvicorn app.main:app --reload --port 8000

### Test
pytest -q

### Endpoints
GET  /health
POST /employees
GET  /employees
GET  /employees/{id}
PUT  /employees/{id}
DELETE /employees/{id}
```

---

# Interview TODO list (what you ask them to implement)

Pick 2–3 items depending on time:

### Core (must-have)

-  Add **unique email check on update** (return 409)
    
-  Add **search**: `?q=` matches first_name/last_name/email
    
-  Add **soft delete** instead of hard delete (set `is_active=false`)
    

### Senior signals

-  Add **pagination metadata** `{items, limit, offset, total}`
    
-  Add **structured error responses** (consistent error schema)
    
-  Add **transaction handling** / rollback for failures
    
-  Add **pytest** tests for conflict + not found paths
    

### Optional extensions

-  Add auth stub (e.g., `X-API-Key` header)
    
-  Add OpenAPI examples / response docs
    

---

If you want, I can also give you:

- a **grading rubric (1–5)** for this exact exercise
    
- a **“broken version”** of the scaffold for debugging interviews
    
- a Postgres version using Alembic migrations