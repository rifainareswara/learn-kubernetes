# Struktur Project

> Dokumen ini menjelaskan struktur folder yang direkomendasikan dan kode aplikasi yang akan kita deploy.

> [!IMPORTANT]
> **Dua folder, dua tujuan berbeda:**
>
> | Folder | Letak | Tujuan |
> |---|---|---|
> | `my-todo-app/` | Di **komputer kamu** (luar repo) | Project yang kamu buat sendiri saat praktik |
> | `app/` | Di **repo ini** (`12-real-deploy-python-svelte/app/`) | Kode referensi lengkap — bisa langsung dijalankan |
>
> Saat belajar, kamu akan **membangun `my-todo-app/` dari awal** sambil merujuk ke `app/` jika butuh contoh lengkap. Anggap `app/` sebagai *answer key* — lihat setelah mencoba sendiri.

---

## Struktur Monorepo yang Direkomendasikan

Kita menggunakan pendekatan **monorepo** — semua kode (backend, frontend, kubernetes) berada dalam satu repository Git. Ini memudahkan pengelolaan versi dan deployment.

Buat struktur ini di komputer kamu (di luar repo belajar ini):

```bash
# Buat folder project kamu
mkdir -p my-todo-app/{backend,frontend/src/lib,frontend/public,k8s/{backend,frontend,database,ingress}}
cd my-todo-app

# Buat file placeholder agar struktur langsung terlihat jelas
touch backend/{main.py,requirements.txt,.env.example,Dockerfile}
touch frontend/{nginx.conf,package.json,vite.config.js,Dockerfile,.env.example}
touch frontend/src/{App.svelte,main.js}
touch frontend/src/lib/api.js
touch docker-compose.yml
```

Struktur lengkapnya akan terlihat seperti ini:

```
my-todo-app/                        ← Ini ada di komputer kamu (luar repo)
├── docker-compose.yml              # Jalankan semua service sekaligus (dev lokal)
│
├── backend/                        # Kode Python FastAPI
│   ├── main.py                     # Entry point aplikasi
│   ├── requirements.txt            # Dependensi Python
│   ├── .env.example                # Template environment variable
│   └── Dockerfile                  # Instruksi build Docker image (multi-stage)
│
├── frontend/                       # Kode Svelte
│   ├── src/
│   │   ├── App.svelte              # Komponen utama
│   │   ├── lib/
│   │   │   └── api.js             # Helper untuk memanggil API
│   │   └── main.js                # Entry point Svelte
│   ├── public/
│   │   └── favicon.png
│   ├── nginx.conf                  # Konfigurasi nginx untuk serve SPA
│   ├── package.json                # Dependensi Node.js
│   ├── vite.config.js              # Konfigurasi bundler Vite
│   ├── .env.example                # Template environment variable (VITE_API_URL)
│   └── Dockerfile                  # Instruksi build Docker image (multi-stage)
│
└── k8s/                            # Semua manifest Kubernetes
    ├── backend/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap-secret.yaml
    ├── frontend/
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── database/
    │   ├── statefulset.yaml
    │   ├── service.yaml
    │   └── pvc.yaml
    └── ingress/
        └── ingress.yaml
```

> [!TIP]
> Kode lengkap untuk semua file di atas sudah tersedia di [`../app/`](../app/) — gunakan sebagai referensi saat kamu mentok.

---

## Kode Backend: `backend/main.py`

Ini adalah kode lengkap FastAPI untuk aplikasi Todo:

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from datetime import datetime
from typing import Optional
import os

DATABASE_URL = (
    f"postgresql://{os.getenv('DB_USER', 'postgres')}"
    f":{os.getenv('DB_PASS', 'postgres')}"
    f"@{os.getenv('DB_HOST', 'localhost')}"
    f":{os.getenv('DB_PORT', '5432')}"
    f"/{os.getenv('DB_NAME', 'tododb')}"
)

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


class TodoModel(Base):
    __tablename__ = "todos"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False)
    description = Column(String, default="")
    done = Column(Boolean, default=False)          # ← field: "done" (bukan "completed")
    created_at = Column(DateTime, default=datetime.utcnow)


Base.metadata.create_all(bind=engine)

app = FastAPI(title="Todo API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "*").split(","),  # ← baca dari env
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


class TodoCreate(BaseModel):
    title: str
    description: Optional[str] = ""


class TodoUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    done: Optional[bool] = None                    # ← field: "done"


class TodoResponse(BaseModel):
    id: int
    title: str
    description: str
    done: bool
    created_at: datetime

    class Config:
        from_attributes = True


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.get("/health")
def health_check():
    return {"status": "ok"}


@app.get("/todos", response_model=list[TodoResponse])
def list_todos(db: Session = Depends(get_db)):
    return db.query(TodoModel).order_by(TodoModel.created_at.desc()).all()


@app.post("/todos", response_model=TodoResponse, status_code=201)
def create_todo(payload: TodoCreate, db: Session = Depends(get_db)):
    todo = TodoModel(**payload.model_dump())
    db.add(todo)
    db.commit()
    db.refresh(todo)
    return todo


@app.get("/todos/{todo_id}", response_model=TodoResponse)
def get_todo(todo_id: int, db: Session = Depends(get_db)):
    todo = db.query(TodoModel).filter(TodoModel.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")
    return todo


@app.patch("/todos/{todo_id}", response_model=TodoResponse)  # ← PATCH, bukan PUT
def update_todo(todo_id: int, payload: TodoUpdate, db: Session = Depends(get_db)):
    todo = db.query(TodoModel).filter(TodoModel.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")
    for field, value in payload.model_dump(exclude_unset=True).items():
        setattr(todo, field, value)
    db.commit()
    db.refresh(todo)
    return todo


@app.delete("/todos/{todo_id}", status_code=204)
def delete_todo(todo_id: int, db: Session = Depends(get_db)):
    todo = db.query(TodoModel).filter(TodoModel.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")
    db.delete(todo)
    db.commit()
```

> [!IMPORTANT]
> Perhatikan beberapa perbedaan dari contoh umum di internet:
> - Field status todo menggunakan `done` (bukan `completed`)
> - Update menggunakan method **`PATCH`** (bukan `PUT`) — hanya update field yang dikirim
> - `CORS_ORIGINS` dibaca dari environment variable, bukan hardcode `"*"`
> - `health_check` mengembalikan `{"status": "ok"}` (bukan `"healthy"`)

---

## Backend: `backend/requirements.txt`

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
sqlalchemy==2.0.35
psycopg2-binary==2.9.9
python-dotenv==1.0.1
```

| Package | Kegunaan |
|---|---|
| `fastapi` | Framework web utama |
| `uvicorn[standard]` | ASGI server dengan extras (watchfiles, websockets) |
| `pydantic` | Validasi data dan serialisasi |
| `sqlalchemy` | ORM untuk interaksi database |
| `psycopg2-binary` | Driver PostgreSQL untuk Python |
| `python-dotenv` | Load `.env` file saat development lokal |

---

## Backend: `backend/.env.example`

Salin file ini menjadi `.env` untuk development lokal:

```bash
cp backend/.env.example backend/.env
```

Isi `.env.example`:

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=tododb
DB_USER=postgres
DB_PASS=postgres
CORS_ORIGINS=http://localhost:5173
```

> [!NOTE]
> `CORS_ORIGINS` menggunakan `http://localhost:5173` (port Vite dev server) saat development lokal.
> Di production Kubernetes, nilai ini diset via ConfigMap.

---

## Backend: `backend/Dockerfile`

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Penjelasan multi-stage build:**
- **Stage `builder`** — install semua package Python (image sementara, tidak di-push)
- **Stage final** — salin hanya hasil install dari builder, tanpa tool build → image lebih kecil

---

## Frontend: `frontend/src/lib/api.js`

Helper terpusat untuk semua HTTP request ke backend:

```javascript
const BASE_URL = import.meta.env.VITE_API_URL || '/api'

async function request(path, options = {}) {
  const res = await fetch(`${BASE_URL}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  })
  if (!res.ok) {
    const err = await res.json().catch(() => ({ detail: 'Terjadi kesalahan' }))
    throw new Error(err.detail || 'Request gagal')
  }
  if (res.status === 204) return null
  return res.json()
}

export const api = {
  getTodos: () => request('/todos'),
  createTodo: (data) => request('/todos', { method: 'POST', body: JSON.stringify(data) }),
  updateTodo: (id, data) => request(`/todos/${id}`, { method: 'PATCH', body: JSON.stringify(data) }),
  deleteTodo: (id) => request(`/todos/${id}`, { method: 'DELETE' }),
}
```

> [!NOTE]
> `App.svelte` mengimport dari file ini dengan `import { api } from './lib/api.js'` — bukan memanggil `fetch` langsung. Ini memisahkan logika HTTP dari logika UI.

---

## Frontend: `frontend/src/main.js`

```javascript
import App from './App.svelte'

const app = new App({ target: document.getElementById('app') })

export default app
```

---

## Frontend: `frontend/src/App.svelte`

```svelte
<script>
  import { onMount } from 'svelte'
  import { api } from './lib/api.js'

  let todos = []
  let loading = true
  let error = ''
  let newTitle = ''
  let newDescription = ''
  let submitting = false

  onMount(fetchTodos)

  async function fetchTodos() {
    loading = true
    error = ''
    try {
      todos = await api.getTodos()
    } catch (e) {
      error = e.message
    } finally {
      loading = false
    }
  }

  async function handleCreate() {
    if (!newTitle.trim()) return
    submitting = true
    error = ''
    try {
      const todo = await api.createTodo({ title: newTitle.trim(), description: newDescription.trim() })
      todos = [todo, ...todos]
      newTitle = ''
      newDescription = ''
    } catch (e) {
      error = e.message
    } finally {
      submitting = false
    }
  }

  async function toggleDone(todo) {
    try {
      const updated = await api.updateTodo(todo.id, { done: !todo.done })
      todos = todos.map(t => t.id === updated.id ? updated : t)
    } catch (e) {
      error = e.message
    }
  }

  async function handleDelete(id) {
    try {
      await api.deleteTodo(id)
      todos = todos.filter(t => t.id !== id)
    } catch (e) {
      error = e.message
    }
  }

  function handleKeydown(e) {
    if (e.key === 'Enter') handleCreate()
  }
</script>

<main>
  <h1>Todo App</h1>
  <p class="subtitle">Belajar deploy ke Kubernetes</p>

  <div class="form">
    <input type="text" placeholder="Judul todo..." bind:value={newTitle}
      on:keydown={handleKeydown} disabled={submitting} />
    <input type="text" placeholder="Deskripsi (opsional)..." bind:value={newDescription}
      on:keydown={handleKeydown} disabled={submitting} />
    <button on:click={handleCreate} disabled={submitting || !newTitle.trim()}>
      {submitting ? 'Menyimpan...' : '+ Tambah'}
    </button>
  </div>

  {#if error}
    <p class="error">{error}</p>
  {/if}

  {#if loading}
    <p class="loading">Memuat...</p>
  {:else if todos.length === 0}
    <p class="empty">Belum ada todo. Tambahkan yang pertama!</p>
  {:else}
    <ul>
      {#each todos as todo (todo.id)}
        <li class:done={todo.done}>
          <button class="check" on:click={() => toggleDone(todo)}>
            {todo.done ? '✓' : '○'}
          </button>
          <div class="content">
            <span class="title">{todo.title}</span>
            {#if todo.description}
              <span class="desc">{todo.description}</span>
            {/if}
          </div>
          <button class="delete" on:click={() => handleDelete(todo.id)}>✕</button>
        </li>
      {/each}
    </ul>
    <p class="count">
      {todos.filter(t => t.done).length}/{todos.length} selesai
    </p>
  {/if}
</main>

<style>
  /* ... lihat file lengkap di app/frontend/src/App.svelte ... */
</style>
```

---

## Frontend: `frontend/package.json`

```json
{
  "name": "todo-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "@sveltejs/vite-plugin-svelte": "^3.1.2",
    "svelte": "^4.2.19",
    "vite": "^5.4.8"
  }
}
```

---

## Frontend: `frontend/vite.config.js`

```javascript
import { defineConfig } from 'vite'
import { svelte } from '@sveltejs/vite-plugin-svelte'

export default defineConfig({
  plugins: [svelte()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
})
```

> [!NOTE]
> Proxy `/api` → `http://localhost:8000` hanya aktif saat **development** (`npm run dev`).
> Di production (Docker/Kubernetes), routing `/api` ditangani oleh Nginx Ingress.

---

## Frontend: `frontend/nginx.conf`

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> [!NOTE]
> File ini sengaja dibuat minimal — cukup untuk SPA fallback.
> Fitur seperti gzip dan cache header bisa ditambahkan nanti sesuai kebutuhan.

---

## Frontend: `frontend/Dockerfile`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Perbedaan penting dengan backend Dockerfile:**
- Base image: `node:20-alpine` (builder) → `nginx:alpine` (runtime)
- `VITE_API_URL` di-set sebagai `ARG` karena dibutuhkan saat **build time** (`npm run build`)
- Hasil build (`dist/`) di-copy ke Nginx, bukan kode sumber

---

## Root: `docker-compose.yml`

Untuk menjalankan semua service sekaligus saat development lokal:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: tododb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: tododb
      DB_USER: postgres
      DB_PASS: postgres
      CORS_ORIGINS: http://localhost:5173,http://localhost:80
    depends_on:
      postgres:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
      args:
        VITE_API_URL: http://localhost:8000
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  postgres_data:
```

> [!NOTE]
> Di `docker-compose.yml`, `VITE_API_URL` diset ke `http://localhost:8000` karena frontend (di port 80) perlu memanggil backend secara langsung dari browser. Berbeda dengan di Kubernetes, di mana Ingress yang menangani routing sehingga `VITE_API_URL=/api`.

---

## Selanjutnya

- [02-backend-python/README.md](../02-backend-python/README.md) — Dockerize dan deploy FastAPI

