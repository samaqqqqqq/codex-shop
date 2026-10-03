<script setup>
import { computed, ref, watch } from 'vue'

const storageKey = 'shiri-todo-items-v1'
const defaults = [
  { id: 'welcome-1', text: '整理一下今天的優先順序', done: false },
  { id: 'welcome-2', text: '留 20 分鐘給自己，好好休息', done: false },
  { id: 'welcome-3', text: '回覆還沒處理的訊息', done: true },
]
const filters = [
  { id: 'all', label: '全部事項', icon: '◷' },
  { id: 'active', label: '尚未完成', icon: '○' },
  { id: 'completed', label: '已經完成', icon: '✓' },
]
const labels = { all: '所有事項', active: '尚未完成', completed: '已經完成' }

function loadTasks() {
  try {
    const saved = JSON.parse(localStorage.getItem(storageKey))
    return Array.isArray(saved) ? saved : defaults
  } catch {
    return defaults
  }
}

const tasks = ref(loadTasks())
const activeFilter = ref('all')
const newTask = ref('')
const taskInput = ref(null)
const today = new Date()
const weekday = new Intl.DateTimeFormat('zh-TW', { weekday: 'long' }).format(today).toUpperCase()
const date = new Intl.DateTimeFormat('zh-TW', { month: 'long', day: 'numeric' }).format(today)

const completedCount = computed(() => tasks.value.filter((task) => task.done).length)
const activeCount = computed(() => tasks.value.length - completedCount.value)
const progress = computed(() => tasks.value.length ? completedCount.value / tasks.value.length * 100 : 0)
const visibleTasks = computed(() => tasks.value.filter((task) => {
  if (activeFilter.value === 'active') return !task.done
  if (activeFilter.value === 'completed') return task.done
  return true
}))
const listTitle = computed(() => activeFilter.value === 'all' ? '今天的待辦' : labels[activeFilter.value])
const listHint = computed(() => activeFilter.value === 'completed' ? '每一步都值得記得' : '照自己的步調就好')
const emptyTitle = computed(() => activeFilter.value === 'completed' ? '完成的事情會在這裡' : '清單還是空的')
const emptyNote = computed(() => activeFilter.value === 'completed'
  ? '先完成一件小事，讓今天留下記號。'
  : '寫下一件想做的事，開始今天的步調。')

watch(tasks, (value) => {
  try {
    localStorage.setItem(storageKey, JSON.stringify(value))
  } catch {
    // Keep the in-memory list usable when browser storage is unavailable.
  }
}, { deep: true })

function addTask() {
  const text = newTask.value.trim()
  if (!text) {
    taskInput.value?.focus()
    return
  }

  tasks.value.unshift({
    id: globalThis.crypto?.randomUUID?.() ?? `${Date.now()}-${Math.random()}`,
    text,
    done: false,
  })
  newTask.value = ''
  activeFilter.value = 'all'
  taskInput.value?.focus()
}

function removeTask(id) {
  tasks.value = tasks.value.filter((task) => task.id !== id)
}

function clearCompleted() {
  tasks.value = tasks.value.filter((task) => !task.done)
}
</script>

<template>
  <header class="topbar shell">
    <a class="brand" href="#top" aria-label="拾日首頁">
      <span class="brand-mark">拾</span>
      <span>拾日<small>MAKE ROOM FOR TODAY</small></span>
    </a>
    <div class="top-note">留一點空間，給重要的事。</div>
  </header>

  <main id="top" class="shell layout">
    <aside class="sidebar">
      <div class="greeting">
        <div class="greeting-copy">
          <div class="greeting-label">YOUR LITTLE DAY PLANNER</div>
          <h1>早安，今天也慢慢來。</h1>
          <p>把想做的事寫下來，一件一件完成。</p>
        </div>
        <div class="date-card">
          <span>{{ weekday }}</span>
          <strong>{{ date }}</strong>
        </div>
      </div>

      <div class="nav-label">檢視清單</div>
      <nav class="filters" aria-label="待辦事項篩選">
        <button
          v-for="filter in filters"
          :key="filter.id"
          class="filter"
          :class="{ active: activeFilter === filter.id }"
          type="button"
          :aria-pressed="activeFilter === filter.id"
          @click="activeFilter = filter.id"
        >
          <span class="filter-name"><span class="filter-icon">{{ filter.icon }}</span>{{ filter.label }}</span>
          <span class="filter-count">{{ filter.id === 'all' ? tasks.length : filter.id === 'active' ? activeCount : completedCount }}</span>
        </button>
      </nav>
      <div class="side-foot">小提醒：不必一次完成所有事，<br />每一步都算數。</div>
    </aside>

    <section class="content" aria-labelledby="listTitle">
      <div class="content-head">
        <div>
          <div class="eyebrow">A FRESH START</div>
          <h2 id="listTitle">{{ listTitle }}</h2>
        </div>
        <div class="progress-copy">
          <strong>{{ completedCount }} / {{ tasks.length }}</strong>
          今日完成進度
          <div class="progress-track" role="progressbar" aria-label="今日完成進度" :aria-valuenow="progress" aria-valuemin="0" aria-valuemax="100">
            <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
          </div>
        </div>
      </div>

      <form class="composer" @submit.prevent="addTask">
        <span class="plus" aria-hidden="true">＋</span>
        <input
          ref="taskInput"
          v-model="newTask"
          type="text"
          maxlength="120"
          placeholder="新增一件想完成的事…"
          autocomplete="off"
          aria-label="新增待辦事項"
        />
        <button class="add-button" type="submit">加入清單</button>
      </form>

      <div class="task-card">
        <div class="list-head">
          <div class="list-heading">{{ labels[activeFilter] }} <span>{{ listHint }}</span></div>
          <button v-if="completedCount" class="clear-button" type="button" @click="clearCompleted">清除已完成</button>
        </div>
        <ul class="task-list" aria-live="polite">
          <li v-if="!visibleTasks.length" class="empty">
            <div class="empty-icon">{{ activeFilter === 'completed' ? '✦' : '＋' }}</div>
            <strong>{{ emptyTitle }}</strong>
            <span>{{ emptyNote }}</span>
          </li>
          <li v-for="task in visibleTasks" :key="task.id" class="task" :class="{ done: task.done }">
            <button
              class="task-check"
              type="button"
              :aria-label="task.done ? '標記為未完成' : '標記為完成'"
              :aria-pressed="task.done"
              @click="task.done = !task.done"
            >{{ task.done ? '✓' : '' }}</button>
            <span class="task-text">{{ task.text }}</span>
            <span class="task-tag">{{ task.done ? '已完成' : '今天' }}</span>
            <button class="delete-button" type="button" :aria-label="`刪除：${task.text}`" @click="removeTask(task.id)">×</button>
          </li>
        </ul>
      </div>
      <div class="bottom-note"><span>一件一件來，今天就很棒。</span><span>YOUR OWN PACE IS JUST RIGHT</span></div>
    </section>
  </main>
</template>
