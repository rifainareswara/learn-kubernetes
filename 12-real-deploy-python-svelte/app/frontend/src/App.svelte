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
    <input
      type="text"
      placeholder="Judul todo..."
      bind:value={newTitle}
      on:keydown={handleKeydown}
      disabled={submitting}
    />
    <input
      type="text"
      placeholder="Deskripsi (opsional)..."
      bind:value={newDescription}
      on:keydown={handleKeydown}
      disabled={submitting}
    />
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
          <button class="check" on:click={() => toggleDone(todo)} title={todo.done ? 'Tandai belum selesai' : 'Tandai selesai'}>
            {todo.done ? '✓' : '○'}
          </button>
          <div class="content">
            <span class="title">{todo.title}</span>
            {#if todo.description}
              <span class="desc">{todo.description}</span>
            {/if}
          </div>
          <button class="delete" on:click={() => handleDelete(todo.id)} title="Hapus">✕</button>
        </li>
      {/each}
    </ul>
    <p class="count">
      {todos.filter(t => t.done).length}/{todos.length} selesai
    </p>
  {/if}
</main>

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) { font-family: system-ui, sans-serif; background: #f5f5f5; color: #222; }

  main {
    max-width: 560px;
    margin: 48px auto;
    padding: 0 16px;
  }

  h1 { font-size: 2rem; font-weight: 700; }
  .subtitle { color: #666; margin-bottom: 32px; }

  .form {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-bottom: 24px;
  }

  input {
    padding: 10px 14px;
    border: 1px solid #ddd;
    border-radius: 8px;
    font-size: 1rem;
    outline: none;
    transition: border-color 0.15s;
  }
  input:focus { border-color: #4f46e5; }

  button {
    cursor: pointer;
    border: none;
    border-radius: 8px;
    font-size: 1rem;
    transition: opacity 0.15s;
  }
  button:disabled { opacity: 0.5; cursor: not-allowed; }

  .form button {
    padding: 10px;
    background: #4f46e5;
    color: #fff;
    font-weight: 600;
  }
  .form button:hover:not(:disabled) { background: #4338ca; }

  .error { color: #dc2626; margin-bottom: 12px; }
  .loading, .empty { color: #888; text-align: center; padding: 32px 0; }

  ul { list-style: none; display: flex; flex-direction: column; gap: 8px; }

  li {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px 16px;
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    transition: opacity 0.2s;
  }
  li.done { opacity: 0.55; }
  li.done .title { text-decoration: line-through; }

  .check {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: #f0f0f0;
    font-size: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .check:hover { background: #e0e7ff; }

  .content { flex: 1; min-width: 0; }
  .title { display: block; font-weight: 500; }
  .desc { display: block; font-size: 0.85rem; color: #888; margin-top: 2px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

  .delete {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    border-radius: 6px;
    background: transparent;
    color: #aaa;
    font-size: 0.85rem;
  }
  .delete:hover { background: #fee2e2; color: #dc2626; }

  .count { margin-top: 12px; text-align: right; font-size: 0.85rem; color: #888; }
</style>
