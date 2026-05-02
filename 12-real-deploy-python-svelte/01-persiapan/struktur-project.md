# Struktur Project

> Dokumen ini menjelaskan struktur folder yang direkomendasikan dan kode aplikasi yang akan kita deploy.

---

## Struktur Monorepo yang Direkomendasikan

Kita menggunakan pendekatan **monorepo** — semua kode (backend, frontend, kubernetes) berada dalam satu repository Git. Ini memudahkan pengelolaan versi dan deployment.

```
my-todo-app/
├── backend/                    # Kode Python FastAPI
│   ├── main.py                 # Entry point aplikasi
│   ├── requirements.txt        # Dependensi Python
│   └── Dockerfile              # Instruksi build Docker image
│
├── frontend/                   # Kode Svelte
│   ├── src/
│   │   ├── App.svelte          # Komponen utama
│   │   ├── lib/
│   │   │   └── api.js          # Helper untuk memanggil API
│   │   └── main.js             # Entry point Svelte
│   ├── public/
│   │   └── favicon.png
│   ├── nginx.conf              # Konfigurasi nginx untuk production
│   ├── package.json
│   ├── vite.config.js
│   └── Dockerfile
│
└── k8s/                        # Semua manifest Kubernetes
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

---

## Kode Backend: `backend/main.py`

Ini adalah kode lengkap FastAPI untuk aplikasi Todo:

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from datetime import datetime
import os

# ─── Konfigurasi Database ──────────────────────────────────────────────────────
# Baca dari environment variables (diset oleh ConfigMap/Secret di Kubernetes)
DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = os.getenv("DB_PORT", "5432")
DB_NAME = os.getenv("DB_NAME", "tododb")
DB_USER = os.getenv("DB_USER", "postgres")
DB_PASS = os.getenv("DB_PASS", "postgres")

DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASS}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


# ─── Model Database ────────────────────────────────────────────────────────────
class TodoDB(Base):
    __tablename__ = "todos"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    description = Column(String(500), nullable=True)
    completed = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)


# Buat tabel jika belum ada
Base.metadata.create_all(bind=engine)


# ─── Pydantic Schemas ──────────────────────────────────────────────────────────
class TodoCreate(BaseModel):
    title: str
    description: Optional[str] = None


class TodoUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    completed: Optional[bool] = None


class TodoResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    completed: bool
    created_at: datetime

    class Config:
        from_attributes = True


# ─── Inisialisasi FastAPI ──────────────────────────────────────────────────────
app = FastAPI(
    title="Todo API",
    description="REST API untuk Todo List — Contoh deploy di Kubernetes",
    version="1.0.0",
)

# CORS: izinkan frontend Svelte mengakses API
# Di Kubernetes, frontend dan backend berada di domain yang sama (via Ingress)
# tapi kita tetap set CORS untuk keamanan
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Di production, ganti dengan domain spesifik
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# ─── Database Dependency ───────────────────────────────────────────────────────
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# ─── Health Check ──────────────────────────────────────────────────────────────
# PENTING: Endpoint ini digunakan oleh Kubernetes readinessProbe dan livenessProbe
@app.get("/health", tags=["System"])
async def health_check():
    return {"status": "healthy", "service": "todo-backend"}


@app.get("/", tags=["System"])
async def root():
    return {"message": "Todo API berjalan. Buka /docs untuk dokumentasi."}


# ─── CRUD Endpoints ────────────────────────────────────────────────────────────
@app.get("/todos", response_model=List[TodoResponse], tags=["Todos"])
async def get_todos(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    """Ambil semua todos"""
    todos = db.query(TodoDB).offset(skip).limit(limit).all()
    return todos


@app.post("/todos", response_model=TodoResponse, status_code=201, tags=["Todos"])
async def create_todo(todo: TodoCreate, db: Session = Depends(get_db)):
    """Buat todo baru"""
    db_todo = TodoDB(**todo.model_dump())
    db.add(db_todo)
    db.commit()
    db.refresh(db_todo)
    return db_todo


@app.get("/todos/{todo_id}", response_model=TodoResponse, tags=["Todos"])
async def get_todo(todo_id: int, db: Session = Depends(get_db)):
    """Ambil satu todo berdasarkan ID"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")
    return todo


@app.put("/todos/{todo_id}", response_model=TodoResponse, tags=["Todos"])
async def update_todo(
    todo_id: int,
    todo_update: TodoUpdate,
    db: Session = Depends(get_db)
):
    """Update todo (judul, deskripsi, atau status selesai)"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")

    update_data = todo_update.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(todo, field, value)

    db.commit()
    db.refresh(todo)
    return todo


@app.delete("/todos/{todo_id}", status_code=204, tags=["Todos"])
async def delete_todo(todo_id: int, db: Session = Depends(get_db)):
    """Hapus todo berdasarkan ID"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    if not todo:
        raise HTTPException(status_code=404, detail="Todo tidak ditemukan")

    db.delete(todo)
    db.commit()
    return None
```

---

## Dependensi Backend: `backend/requirements.txt`

```
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.4.2
psycopg2-binary==2.9.9
sqlalchemy==2.0.23
python-dotenv==1.0.0
```

**Penjelasan setiap dependensi:**

| Package | Kegunaan |
|---|---|
| `fastapi` | Framework web utama |
| `uvicorn` | ASGI server untuk menjalankan FastAPI |
| `pydantic` | Validasi data dan serialisasi |
| `psycopg2-binary` | Driver PostgreSQL untuk Python |
| `sqlalchemy` | ORM untuk interaksi database |
| `python-dotenv` | Load `.env` file saat development lokal |

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
    "@sveltejs/vite-plugin-svelte": "^3.0.2",
    "svelte": "^4.2.8",
    "vite": "^5.0.8"
  }
}
```

## Frontend: `frontend/vite.config.js`

```javascript
import { defineConfig } from 'vite'
import { svelte } from '@sveltejs/vite-plugin-svelte'

export default defineConfig({
  plugins: [svelte()],
  server: {
    // Proxy untuk development lokal — teruskan /api ke backend
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        rewrite: (path) => path.replace(/^\/api/, ''),
      }
    }
  }
})
```

---

## Frontend: `frontend/src/App.svelte`

Komponen utama Svelte yang menampilkan dan mengelola todo list:

```svelte
<script>
  import { onMount } from 'svelte';

  // URL API diset saat build time via VITE_API_URL environment variable
  // Di Kubernetes: VITE_API_URL=/api (diteruskan oleh Ingress ke backend)
  const API_URL = import.meta.env.VITE_API_URL || '/api';

  let todos = [];
  let newTitle = '';
  let newDescription = '';
  let loading = false;
  let error = null;

  // Ambil semua todos dari API
  async function fetchTodos() {
    loading = true;
    try {
      const response = await fetch(`${API_URL}/todos`);
      if (!response.ok) throw new Error('Gagal mengambil data todos');
      todos = await response.json();
    } catch (err) {
      error = err.message;
    } finally {
      loading = false;
    }
  }

  // Buat todo baru
  async function createTodo() {
    if (!newTitle.trim()) return;

    try {
      const response = await fetch(`${API_URL}/todos`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title: newTitle,
          description: newDescription || null
        })
      });
      if (!response.ok) throw new Error('Gagal membuat todo');

      newTitle = '';
      newDescription = '';
      await fetchTodos(); // Refresh list
    } catch (err) {
      error = err.message;
    }
  }

  // Toggle status selesai/belum
  async function toggleTodo(todo) {
    try {
      const response = await fetch(`${API_URL}/todos/${todo.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed: !todo.completed })
      });
      if (!response.ok) throw new Error('Gagal mengupdate todo');
      await fetchTodos();
    } catch (err) {
      error = err.message;
    }
  }

  // Hapus todo
  async function deleteTodo(id) {
    try {
      const response = await fetch(`${API_URL}/todos/${id}`, {
        method: 'DELETE'
      });
      if (!response.ok) throw new Error('Gagal menghapus todo');
      await fetchTodos();
    } catch (err) {
      error = err.message;
    }
  }

  // Load todos saat komponen pertama kali dimount
  onMount(fetchTodos);
</script>

<main>
  <h1>Todo List - Kubernetes Edition</h1>

  {#if error}
    <div class="error">
      Error: {error}
      <button on:click={() => error = null}>Tutup</button>
    </div>
  {/if}

  <!-- Form tambah todo baru -->
  <form on:submit|preventDefault={createTodo} class="form-add">
    <input
      bind:value={newTitle}
      placeholder="Judul todo..."
      required
    />
    <input
      bind:value={newDescription}
      placeholder="Deskripsi (opsional)..."
    />
    <button type="submit">Tambah Todo</button>
  </form>

  <!-- Daftar todos -->
  {#if loading}
    <p>Memuat...</p>
  {:else if todos.length === 0}
    <p class="empty">Belum ada todo. Tambahkan yang pertama!</p>
  {:else}
    <ul class="todo-list">
      {#each todos as todo (todo.id)}
        <li class:completed={todo.completed}>
          <input
            type="checkbox"
            checked={todo.completed}
            on:change={() => toggleTodo(todo)}
          />
          <div class="todo-content">
            <span class="title">{todo.title}</span>
            {#if todo.description}
              <span class="description">{todo.description}</span>
            {/if}
          </div>
          <button
            class="btn-delete"
            on:click={() => deleteTodo(todo.id)}
          >
            Hapus
          </button>
        </li>
      {/each}
    </ul>
  {/if}
</main>

<style>
  main {
    max-width: 600px;
    margin: 0 auto;
    padding: 2rem;
    font-family: sans-serif;
  }

  h1 { color: #2c3e50; }

  .error {
    background: #fee;
    border: 1px solid #f99;
    padding: 1rem;
    border-radius: 4px;
    margin-bottom: 1rem;
  }

  .form-add {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 2rem;
  }

  .form-add input {
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 1rem;
  }

  .form-add button {
    padding: 0.5rem 1rem;
    background: #3498db;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
  }

  .todo-list {
    list-style: none;
    padding: 0;
  }

  .todo-list li {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 0.75rem;
    border: 1px solid #eee;
    border-radius: 4px;
    margin-bottom: 0.5rem;
  }

  .todo-list li.completed .title {
    text-decoration: line-through;
    color: #999;
  }

  .todo-content {
    flex: 1;
    display: flex;
    flex-direction: column;
  }

  .description {
    font-size: 0.85rem;
    color: #777;
  }

  .btn-delete {
    padding: 0.25rem 0.75rem;
    background: #e74c3c;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  .empty {
    text-align: center;
    color: #999;
    font-style: italic;
  }
</style>
```

---

## Frontend: `frontend/nginx.conf`

File konfigurasi Nginx untuk melayani Svelte SPA di production:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression untuk performa lebih baik
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # SPA fallback: semua route diteruskan ke index.html
    # Ini penting agar client-side routing Svelte berfungsi dengan benar
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache assets statis lebih lama
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

> **Tips:** Simpan file `nginx.conf` di folder `frontend/` bersama `Dockerfile`. Saat Docker build, file ini akan disalin ke dalam image nginx.

> **Perhatian:** Jangan lupa buat file `.dockerignore` di folder `frontend/` yang berisi `node_modules` dan `dist`. Ini akan mempercepat proses build Docker secara signifikan.

---

## Selanjutnya

- [02-backend-python/README.md](../02-backend-python/README.md) — Dockerize dan deploy FastAPI
