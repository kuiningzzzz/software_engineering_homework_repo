<script setup>
import { computed, nextTick, onMounted, onUnmounted, reactive, ref, watch } from 'vue'
import {
  ArrowRight,
  CalendarDays,
  Check,
  CheckCheck,
  CircleCheck,
  ClipboardList,
  GripVertical,
  Inbox,
  LayoutDashboard,
  LoaderCircle,
  Moon,
  Pencil,
  Plus,
  Search,
  Sun,
  Trash2,
  X,
} from '@lucide/vue'

const TASKS_KEY = 'xushi.tasks.v1'
const THEME_KEY = 'xushi.theme.v1'

const columns = [
  {
    id: 'todo',
    label: '待办',
    hint: '准备开始的任务',
    icon: ClipboardList,
    color: 'bg-[#8c7cf5]',
    iconStyle: 'bg-[#eeeaff] text-[#745fdf] dark:bg-[#342b62] dark:text-[#b6a9ff]',
    countStyle: 'bg-[#eeeaff] text-[#745fdf] dark:bg-[#342b62] dark:text-[#b6a9ff]',
  },
  {
    id: 'doing',
    label: '进行中',
    hint: '正在专注处理',
    icon: LoaderCircle,
    color: 'bg-[#f3b65c]',
    iconStyle: 'bg-[#fff2dc] text-[#d18c2d] dark:bg-[#493722] dark:text-[#f6c77f]',
    countStyle: 'bg-[#fff2dc] text-[#c7872c] dark:bg-[#493722] dark:text-[#f6c77f]',
  },
  {
    id: 'done',
    label: '已完成',
    hint: '每一步都值得记录',
    icon: CircleCheck,
    color: 'bg-[#64caa9]',
    iconStyle: 'bg-[#ddf8ed] text-[#369d79] dark:bg-[#21453d] dark:text-[#7bdfbc]',
    countStyle: 'bg-[#ddf8ed] text-[#369d79] dark:bg-[#21453d] dark:text-[#7bdfbc]',
  },
]

const priorities = [
  { id: 'high', label: '高优先级', shortLabel: '高', dot: 'bg-[#f26b75]', badge: 'bg-[#fff0f1] text-[#df5a67] dark:bg-[#4c2932] dark:text-[#ff9ba4]' },
  { id: 'medium', label: '中优先级', shortLabel: '中', dot: 'bg-[#f2bb62]', badge: 'bg-[#fff6e9] text-[#c68a34] dark:bg-[#493722] dark:text-[#f7ca82]' },
  { id: 'low', label: '低优先级', shortLabel: '低', dot: 'bg-[#65c9a3]', badge: 'bg-[#eaf9f0] text-[#33966f] dark:bg-[#21453d] dark:text-[#88ddb9]' },
]

function loadTasks() {
  try {
    const saved = JSON.parse(localStorage.getItem(TASKS_KEY) || '[]')
    if (!Array.isArray(saved)) return []
    return saved.filter((task) =>
      task &&
      typeof task.id === 'string' &&
      typeof task.title === 'string' &&
      task.title.trim() &&
      columns.some((column) => column.id === task.status) &&
      priorities.some((priority) => priority.id === task.priority),
    ).map((task) => ({
      id: task.id,
      title: task.title,
      description: typeof task.description === 'string' ? task.description : '',
      status: task.status,
      priority: task.priority,
      createdAt: task.createdAt || new Date().toISOString(),
      updatedAt: task.updatedAt || task.createdAt || new Date().toISOString(),
    }))
  } catch {
    return []
  }
}

function loadTheme() {
  try {
    return localStorage.getItem(THEME_KEY) === 'dark' ? 'dark' : 'light'
  } catch {
    return 'light'
  }
}

const tasks = ref(loadTasks())
const theme = ref(loadTheme())
const search = ref('')
const priorityFilter = ref('all')
const activeDropColumn = ref(null)
const draggedId = ref(null)
const modalMode = ref(null)
const editingId = ref(null)
const deletingTask = ref(null)
const titleInput = ref(null)
const titleError = ref('')
const storageError = ref(false)
const form = reactive({ title: '', description: '', status: 'todo', priority: 'medium' })

const totalCount = computed(() => tasks.value.length)
const doneCount = computed(() => tasks.value.filter((task) => task.status === 'done').length)
const doingCount = computed(() => tasks.value.filter((task) => task.status === 'doing').length)
const progress = computed(() => totalCount.value ? Math.round(doneCount.value / totalCount.value * 100) : 0)
const filteredTasks = computed(() => {
  const query = search.value.trim().toLocaleLowerCase()
  return tasks.value.filter((task) =>
    (priorityFilter.value === 'all' || task.priority === priorityFilter.value) &&
    (!query || `${task.title} ${task.description}`.toLocaleLowerCase().includes(query)),
  )
})

function tasksFor(status) {
  return filteredTasks.value.filter((task) => task.status === status)
}

function countFor(status) {
  return tasks.value.filter((task) => task.status === status).length
}

function priorityFor(id) {
  return priorities.find((item) => item.id === id) || priorities[1]
}

function formatDate(value) {
  const date = new Date(value)
  return Number.isNaN(date.getTime()) ? '' : new Intl.DateTimeFormat('zh-CN', { month: 'numeric', day: 'numeric' }).format(date)
}

function newId() {
  return globalThis.crypto?.randomUUID?.() || `${Date.now()}-${Math.random().toString(36).slice(2)}`
}

async function openCreate(status = 'todo') {
  editingId.value = null
  Object.assign(form, { title: '', description: '', status, priority: 'medium' })
  titleError.value = ''
  modalMode.value = 'form'
  await nextTick()
  titleInput.value?.focus()
}

async function openEdit(task) {
  editingId.value = task.id
  Object.assign(form, {
    title: task.title,
    description: task.description,
    status: task.status,
    priority: task.priority,
  })
  titleError.value = ''
  modalMode.value = 'form'
  await nextTick()
  titleInput.value?.focus()
}

function closeModal() {
  modalMode.value = null
  editingId.value = null
  deletingTask.value = null
  titleError.value = ''
}

function saveTask() {
  const title = form.title.trim()
  if (!title) {
    titleError.value = '请输入任务标题'
    titleInput.value?.focus()
    return
  }
  const now = new Date().toISOString()
  if (editingId.value) {
    tasks.value = tasks.value.map((task) => task.id === editingId.value
      ? { ...task, title, description: form.description.trim(), status: form.status, priority: form.priority, updatedAt: now }
      : task)
  } else {
    tasks.value = [{ id: newId(), title, description: form.description.trim(), status: form.status, priority: form.priority, createdAt: now, updatedAt: now }, ...tasks.value]
  }
  closeModal()
}

function requestDelete(task) {
  deletingTask.value = task
  modalMode.value = 'delete'
}

function deleteTask() {
  if (deletingTask.value) tasks.value = tasks.value.filter((task) => task.id !== deletingTask.value.id)
  closeModal()
}

function onDragStart(event, task) {
  draggedId.value = task.id
  event.dataTransfer.effectAllowed = 'move'
  event.dataTransfer.setData('text/plain', task.id)
}

function onDrop(event, status) {
  event.preventDefault()
  const id = event.dataTransfer.getData('text/plain') || draggedId.value
  if (id && tasks.value.some((task) => task.id === id)) {
    tasks.value = tasks.value.map((task) => task.id === id && task.status !== status
      ? { ...task, status, updatedAt: new Date().toISOString() }
      : task)
  }
  activeDropColumn.value = null
  draggedId.value = null
}

function onKeydown(event) {
  if (event.key === 'Escape' && modalMode.value) closeModal()
}

watch(tasks, (value) => {
  try {
    localStorage.setItem(TASKS_KEY, JSON.stringify(value))
    storageError.value = false
  } catch {
    storageError.value = true
  }
}, { deep: true })

watch(theme, (value) => {
  document.documentElement.classList.toggle('dark', value === 'dark')
  try {
    localStorage.setItem(THEME_KEY, value)
  } catch {
    storageError.value = true
  }
}, { immediate: true })

onMounted(() => window.addEventListener('keydown', onKeydown))
onUnmounted(() => window.removeEventListener('keydown', onKeydown))
</script>

<template>
  <div class="min-h-screen bg-[#f7f8fc] text-[#25283c] transition-colors duration-200 dark:bg-[#10121d] dark:text-[#f4f2fb]">
    <header class="border-b border-[#e9eaf2] bg-white/90 backdrop-blur dark:border-[#292b3b] dark:bg-[#171925]/90">
      <div class="mx-auto flex h-[76px] max-w-[1440px] items-center justify-between px-5 sm:px-8 lg:px-12">
        <div class="flex items-center gap-3">
          <div class="flex h-10 w-10 items-center justify-center rounded-[13px] bg-gradient-to-br from-[#9988f8] to-[#6855d8] text-white shadow-[0_6px_16px_rgba(110,87,220,.22)]">
            <CheckCheck :size="23" :stroke-width="2.5" />
          </div>
          <div class="flex items-baseline gap-2">
            <span class="text-[20px] font-bold tracking-[-.05em]">序事</span>
            <span class="hidden text-[11px] font-semibold tracking-[.16em] text-[#a4a6b8] sm:inline">TASK SPACE</span>
          </div>
        </div>
        <div class="flex items-center gap-3">
          <span class="hidden rounded-full bg-[#f3f1fe] px-3.5 py-1.5 text-xs font-semibold text-[#7b68da] dark:bg-[#302a4b] dark:text-[#c5b8ff] sm:inline-flex">我的工作台</span>
          <button
            type="button"
            class="flex h-10 w-10 items-center justify-center rounded-xl border border-[#e9e9f1] text-[#73768a] transition hover:border-[#cac3f5] hover:bg-[#f7f5ff] hover:text-[#7562dd] focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-[#8875ea] dark:border-[#37394b] dark:text-[#babccf] dark:hover:bg-[#302a4b]"
            :aria-label="theme === 'dark' ? '切换浅色模式' : '切换深色模式'"
            :title="theme === 'dark' ? '切换浅色模式' : '切换深色模式'"
            @click="theme = theme === 'dark' ? 'light' : 'dark'"
          >
            <Sun v-if="theme === 'dark'" :size="19" />
            <Moon v-else :size="19" />
          </button>
        </div>
      </div>
    </header>

    <main class="mx-auto max-w-[1440px] px-5 pb-16 pt-9 sm:px-8 lg:px-12 lg:pt-12">
      <section class="mb-9 grid gap-6 lg:grid-cols-[minmax(0,1fr)_340px] lg:items-end">
        <div>
          <div class="mb-4 inline-flex items-center gap-2 rounded-full border border-[#e7e2ff] bg-[#f1efff] px-3 py-1 text-[11px] font-bold tracking-[.08em] text-[#7561d8] dark:border-[#403963] dark:bg-[#292542] dark:text-[#b9aaff]">
            <span class="h-1.5 w-1.5 rounded-full bg-[#8e79eb]"></span>
            轻松规划 · 专注完成
          </div>
          <h1 class="text-[31px] font-bold leading-tight tracking-[-.055em] sm:text-[40px]">让每件事，都<span class="text-[#806be4]">有条不紊。</span></h1>
          <p class="mt-3 text-sm leading-7 text-[#898b9e] dark:text-[#a1a4b9] sm:text-[15px]">把想法放进清单，让进度一目了然。今天，从下一件小事开始。</p>
        </div>
        <div class="relative overflow-hidden rounded-[22px] bg-[#7565df] px-6 py-5 text-white shadow-[0_14px_30px_rgba(103,83,207,.19)]">
          <div class="pointer-events-none absolute -right-8 -top-14 h-36 w-36 rounded-full border-[26px] border-white/10"></div>
          <div class="pointer-events-none absolute bottom-[-60px] right-14 h-32 w-32 rounded-full bg-white/5"></div>
          <div class="relative flex items-start justify-between">
            <div>
              <p class="text-xs font-medium text-white/75">整体完成进度</p>
              <div class="mt-2 flex items-baseline gap-2"><span class="text-[34px] font-bold leading-none tracking-[-.04em]">{{ progress }}%</span><span class="text-xs text-white/75">已完成 {{ doneCount }} / {{ totalCount }} 项</span></div>
            </div>
            <div class="flex h-9 w-9 items-center justify-center rounded-xl bg-white/15"><Check :size="18" /></div>
          </div>
          <div class="relative mt-5 h-2 overflow-hidden rounded-full bg-white/20" role="progressbar" :aria-valuenow="progress" aria-valuemin="0" aria-valuemax="100" aria-label="整体完成进度">
            <div class="h-full rounded-full bg-white transition-[width] duration-300" :style="{ width: `${progress}%` }"></div>
          </div>
        </div>
      </section>

      <section class="mb-8 grid grid-cols-3 gap-3 sm:gap-4" aria-label="任务概览">
        <div class="rounded-[18px] border border-[#ececf3] bg-white px-4 py-4 shadow-[0_5px_18px_rgba(37,40,60,.025)] dark:border-[#2c2e3f] dark:bg-[#1a1c2a] sm:px-5">
          <div class="flex items-center justify-between"><span class="text-xs font-medium text-[#8c8fa3] dark:text-[#a2a5b9] sm:text-sm">全部任务</span><LayoutDashboard class="hidden text-[#9c8eeb] sm:block" :size="18" /></div>
          <div class="mt-2 text-[26px] font-bold leading-none tracking-[-.04em] sm:text-[30px]">{{ totalCount }}</div>
        </div>
        <div class="rounded-[18px] border border-[#ececf3] bg-white px-4 py-4 shadow-[0_5px_18px_rgba(37,40,60,.025)] dark:border-[#2c2e3f] dark:bg-[#1a1c2a] sm:px-5">
          <div class="flex items-center justify-between"><span class="text-xs font-medium text-[#8c8fa3] dark:text-[#a2a5b9] sm:text-sm">进行中</span><LoaderCircle class="hidden text-[#e9b359] sm:block" :size="18" /></div>
          <div class="mt-2 text-[26px] font-bold leading-none tracking-[-.04em] sm:text-[30px]">{{ doingCount }}</div>
        </div>
        <div class="rounded-[18px] border border-[#ececf3] bg-white px-4 py-4 shadow-[0_5px_18px_rgba(37,40,60,.025)] dark:border-[#2c2e3f] dark:bg-[#1a1c2a] sm:px-5">
          <div class="flex items-center justify-between"><span class="text-xs font-medium text-[#8c8fa3] dark:text-[#a2a5b9] sm:text-sm">已完成</span><CircleCheck class="hidden text-[#64c9a4] sm:block" :size="18" /></div>
          <div class="mt-2 text-[26px] font-bold leading-none tracking-[-.04em] sm:text-[30px]">{{ doneCount }}</div>
        </div>
      </section>

      <section aria-labelledby="board-title">
        <div class="mb-5 flex flex-col gap-4 xl:flex-row xl:items-end xl:justify-between">
          <div>
            <div class="flex items-center gap-2.5"><h2 id="board-title" class="text-xl font-bold tracking-[-.04em] sm:text-[22px]">任务看板</h2><span class="rounded-full bg-[#eeebfc] px-2.5 py-0.5 text-[11px] font-bold text-[#806ce1] dark:bg-[#302a4b] dark:text-[#c5b8ff]">{{ totalCount }}</span></div>
            <p class="mt-1 text-xs text-[#a0a2b1] dark:text-[#868a9f]">拖动卡片到其他列，即可更新任务状态</p>
          </div>
          <div class="flex flex-wrap items-center gap-2.5">
            <label class="relative min-w-[180px] flex-1 sm:w-[240px] sm:flex-none">
              <Search class="pointer-events-none absolute left-3.5 top-1/2 -translate-y-1/2 text-[#a6a8b6]" :size="17" />
              <input v-model="search" type="search" placeholder="搜索任务..." aria-label="搜索任务" class="h-10 w-full rounded-xl border border-[#e7e8f0] bg-white pl-10 pr-3 text-sm outline-none transition placeholder:text-[#afb0bc] focus:border-[#9b8bf0] focus:ring-3 focus:ring-[#9b8bf0]/15 dark:border-[#36384a] dark:bg-[#1b1d2b] dark:placeholder:text-[#777b91]" />
            </label>
            <label class="sr-only" for="priority-filter">筛选优先级</label>
            <select id="priority-filter" v-model="priorityFilter" class="h-10 rounded-xl border border-[#e7e8f0] bg-white px-3 text-sm text-[#55596e] outline-none focus:border-[#9b8bf0] dark:border-[#36384a] dark:bg-[#1b1d2b] dark:text-[#d3d2e1]">
              <option value="all">全部优先级</option>
              <option v-for="priority in priorities" :key="priority.id" :value="priority.id">{{ priority.label }}</option>
            </select>
            <button type="button" class="inline-flex h-10 items-center justify-center gap-2 rounded-xl bg-[#7865df] px-4 text-sm font-semibold text-white shadow-[0_5px_14px_rgba(113,91,218,.22)] transition hover:bg-[#6854d2] focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-[#7865df]" @click="openCreate()"><Plus :size="18" :stroke-width="2.5" />新建任务</button>
          </div>
        </div>

        <div v-if="storageError" role="alert" class="mb-4 rounded-xl border border-[#f6cf9b] bg-[#fff8eb] px-4 py-3 text-sm text-[#9a642d] dark:border-[#745633] dark:bg-[#3c3022] dark:text-[#f6ce91]">浏览器无法保存数据。请检查浏览器的存储权限，当前更改可能在刷新后丢失。</div>

        <div class="grid gap-5 lg:grid-cols-3 lg:items-start">
          <section
            v-for="column in columns"
            :key="column.id"
            :aria-label="`${column.label}任务`"
            class="min-h-[330px] overflow-hidden rounded-[20px] border border-[#e9eaf1] bg-[#f1f2f8] transition-colors dark:border-[#2d2f40] dark:bg-[#191b28]"
            :class="activeDropColumn === column.id ? 'ring-2 ring-[#a293f4] ring-offset-2 ring-offset-[#f7f8fc] dark:ring-offset-[#10121d]' : ''"
            @dragenter.prevent="activeDropColumn = column.id"
            @dragover.prevent="activeDropColumn = column.id"
            @drop="onDrop($event, column.id)"
          >
            <div class="h-1 w-full" :class="column.color"></div>
            <div class="flex items-center justify-between px-4 pb-3 pt-4 sm:px-5">
              <div class="flex items-center gap-2.5">
                <span class="flex h-9 w-9 items-center justify-center rounded-[11px]" :class="column.iconStyle"><component :is="column.icon" :size="19" :stroke-width="2.2" /></span>
                <div><h3 class="text-sm font-bold">{{ column.label }}</h3><p class="mt-0.5 text-[11px] text-[#a0a2b2] dark:text-[#85899e]">{{ column.hint }}</p></div>
                <span class="ml-1 rounded-full px-2 py-0.5 text-[11px] font-bold" :class="column.countStyle">{{ countFor(column.id) }}</span>
              </div>
              <button type="button" class="flex h-8 w-8 items-center justify-center rounded-lg text-[#999cab] transition hover:bg-white hover:text-[#7965df] dark:hover:bg-[#282b3b]" :aria-label="`在${column.label}中新建任务`" @click="openCreate(column.id)"><Plus :size="18" /></button>
            </div>

            <div class="space-y-3 px-3 pb-4 sm:px-4">
              <article
                v-for="task in tasksFor(column.id)"
                :key="task.id"
                draggable="true"
                class="group rounded-[15px] border border-[#e8e8ef] bg-white p-4 shadow-[0_3px_12px_rgba(31,34,63,.035)] transition hover:-translate-y-0.5 hover:shadow-[0_8px_20px_rgba(31,34,63,.09)] dark:border-[#343647] dark:bg-[#222433] dark:hover:shadow-[0_8px_20px_rgba(0,0,0,.14)]"
                :class="draggedId === task.id ? 'opacity-50' : ''"
                @dragstart="onDragStart($event, task)"
                @dragend="draggedId = null; activeDropColumn = null"
              >
                <div class="flex items-start justify-between gap-2">
                  <span class="inline-flex items-center gap-1.5 rounded-md px-2 py-1 text-[11px] font-semibold" :class="priorityFor(task.priority).badge"><span class="h-1.5 w-1.5 rounded-full" :class="priorityFor(task.priority).dot"></span>{{ priorityFor(task.priority).label }}</span>
                  <GripVertical class="mt-0.5 shrink-0 text-[#c9cad4] dark:text-[#606478]" :size="16" aria-hidden="true" />
                </div>
                <h4 class="mt-3 break-words text-[15px] font-semibold leading-6">{{ task.title }}</h4>
                <p v-if="task.description" class="mt-1 line-clamp-3 whitespace-pre-line break-words text-[12px] leading-[1.7] text-[#9496a7] dark:text-[#a1a3b5]">{{ task.description }}</p>
                <div class="mt-4 flex items-center justify-between border-t border-[#f0f0f5] pt-3 dark:border-[#393b4b]">
                  <span class="inline-flex items-center gap-1.5 text-[11px] text-[#a4a6b5] dark:text-[#8e92a8]"><CalendarDays :size="13" />{{ formatDate(task.createdAt) }}</span>
                  <div class="flex items-center gap-1">
                    <button type="button" class="flex h-7 w-7 items-center justify-center rounded-lg text-[#9b9dab] transition hover:bg-[#f2efff] hover:text-[#7967df] focus-visible:outline-2 focus-visible:outline-[#8875ea] dark:hover:bg-[#393252]" :aria-label="`编辑任务：${task.title}`" @click="openEdit(task)"><Pencil :size="14" /></button>
                    <button type="button" class="flex h-7 w-7 items-center justify-center rounded-lg text-[#9b9dab] transition hover:bg-[#fff0f1] hover:text-[#e65c68] focus-visible:outline-2 focus-visible:outline-[#e65c68] dark:hover:bg-[#4b2c34]" :aria-label="`删除任务：${task.title}`" @click="requestDelete(task)"><Trash2 :size="14" /></button>
                  </div>
                </div>
              </article>

              <div v-if="tasksFor(column.id).length === 0" class="flex min-h-[180px] flex-col items-center justify-center rounded-[15px] border border-dashed border-[#dfe1ec] bg-white/45 px-4 py-6 text-center dark:border-[#373a4b] dark:bg-[#202230]/50">
                <span class="mb-3 flex h-10 w-10 items-center justify-center rounded-xl bg-white text-[#bab3ec] dark:bg-[#292b3d] dark:text-[#897bca]"><Inbox :size="21" /></span>
                <p class="text-[13px] font-medium text-[#9295a6] dark:text-[#a2a4b6]">{{ search || priorityFilter !== 'all' ? '没有匹配的任务' : '这里还没有任务' }}</p>
                <button v-if="!search && priorityFilter === 'all'" type="button" class="mt-2 inline-flex items-center gap-1 text-xs font-semibold text-[#8672e5] hover:underline dark:text-[#b5a5ff]" @click="openCreate(column.id)">添加一项 <ArrowRight :size="13" /></button>
              </div>
            </div>
          </section>
        </div>
      </section>
      <p class="mt-8 text-center text-xs text-[#b1b2bf] dark:text-[#696e85]">所有任务仅保存在当前浏览器中</p>
    </main>

    <div v-if="modalMode" class="fixed inset-0 z-50 flex items-center justify-center bg-[#151727]/50 px-4 py-6 backdrop-blur-[3px]" @click.self="closeModal">
      <div v-if="modalMode === 'form'" role="dialog" aria-modal="true" :aria-label="editingId ? '编辑任务' : '新建任务'" class="modal-in w-full max-w-[500px] rounded-[22px] bg-white p-6 shadow-[0_24px_70px_rgba(25,20,54,.24)] dark:bg-[#242635] sm:p-7">
        <div class="mb-6 flex items-start justify-between gap-4">
          <div><p class="mb-1 text-xs font-semibold tracking-[.08em] text-[#8c78e8]">TASK DETAILS</p><h2 class="text-xl font-bold tracking-[-.03em]">{{ editingId ? '编辑任务' : '新建任务' }}</h2></div>
          <button type="button" class="flex h-8 w-8 items-center justify-center rounded-lg text-[#9da0ae] hover:bg-[#f4f3f9] dark:hover:bg-[#343649]" aria-label="关闭" @click="closeModal"><X :size="19" /></button>
        </div>
        <form @submit.prevent="saveTask">
          <label for="task-title" class="mb-2 block text-sm font-semibold">任务标题 <span class="text-[#e76571]">*</span></label>
          <input id="task-title" ref="titleInput" v-model="form.title" type="text" maxlength="80" required placeholder="例如：完成课程项目报告" class="h-11 w-full rounded-xl border border-[#e4e5ed] bg-white px-3.5 text-sm outline-none transition placeholder:text-[#b0b2bf] focus:border-[#9583ee] focus:ring-3 focus:ring-[#9583ee]/15 dark:border-[#424455] dark:bg-[#1c1e2b] dark:placeholder:text-[#74788b]" :aria-invalid="Boolean(titleError)" :aria-describedby="titleError ? 'title-error' : undefined" @input="titleError = ''" />
          <p v-if="titleError" id="title-error" class="mt-1.5 text-xs text-[#df5967]">{{ titleError }}</p>
          <label for="task-description" class="mb-2 mt-5 block text-sm font-semibold">任务描述 <span class="font-normal text-[#a5a7b5]">（选填）</span></label>
          <textarea id="task-description" v-model="form.description" rows="4" maxlength="500" placeholder="补充一些细节，让任务更清晰..." class="w-full resize-y rounded-xl border border-[#e4e5ed] bg-white px-3.5 py-3 text-sm leading-6 outline-none transition placeholder:text-[#b0b2bf] focus:border-[#9583ee] focus:ring-3 focus:ring-[#9583ee]/15 dark:border-[#424455] dark:bg-[#1c1e2b] dark:placeholder:text-[#74788b]"></textarea>
          <div class="mt-4 grid grid-cols-2 gap-3">
            <div><label for="task-status" class="mb-2 block text-sm font-semibold">任务状态</label><select id="task-status" v-model="form.status" class="h-11 w-full rounded-xl border border-[#e4e5ed] bg-white px-3 text-sm outline-none focus:border-[#9583ee] dark:border-[#424455] dark:bg-[#1c1e2b]"><option v-for="column in columns" :key="column.id" :value="column.id">{{ column.label }}</option></select></div>
            <div><label for="task-priority" class="mb-2 block text-sm font-semibold">优先级</label><select id="task-priority" v-model="form.priority" class="h-11 w-full rounded-xl border border-[#e4e5ed] bg-white px-3 text-sm outline-none focus:border-[#9583ee] dark:border-[#424455] dark:bg-[#1c1e2b]"><option v-for="priority in priorities" :key="priority.id" :value="priority.id">{{ priority.label }}</option></select></div>
          </div>
          <div class="mt-7 flex justify-end gap-2.5"><button type="button" class="h-10 rounded-xl border border-[#e5e5ee] px-4 text-sm font-semibold text-[#73768a] hover:bg-[#f7f7fa] dark:border-[#454759] dark:text-[#c4c5d4] dark:hover:bg-[#303243]" @click="closeModal">取消</button><button type="submit" class="h-10 rounded-xl bg-[#7865df] px-5 text-sm font-semibold text-white hover:bg-[#6854d2]">{{ editingId ? '保存修改' : '创建任务' }}</button></div>
        </form>
      </div>

      <div v-else role="alertdialog" aria-modal="true" aria-label="确认删除任务" class="modal-in w-full max-w-[420px] rounded-[22px] bg-white p-6 shadow-[0_24px_70px_rgba(25,20,54,.24)] dark:bg-[#242635] sm:p-7">
        <div class="mb-4 flex h-12 w-12 items-center justify-center rounded-2xl bg-[#fff0f1] text-[#e8616d] dark:bg-[#4d2d36]"><Trash2 :size="22" /></div>
        <h2 class="text-xl font-bold">删除这项任务？</h2>
        <p class="mt-2 break-words text-sm leading-6 text-[#898c9e] dark:text-[#a8aabc]">“{{ deletingTask?.title }}”将从看板中移除，此操作无法撤销。</p>
        <div class="mt-7 flex justify-end gap-2.5"><button type="button" class="h-10 rounded-xl border border-[#e5e5ee] px-4 text-sm font-semibold text-[#73768a] hover:bg-[#f7f7fa] dark:border-[#454759] dark:text-[#c4c5d4] dark:hover:bg-[#303243]" @click="closeModal">取消</button><button type="button" class="h-10 rounded-xl bg-[#e86672] px-5 text-sm font-semibold text-white hover:bg-[#d85865]" @click="deleteTask">确认删除</button></div>
      </div>
    </div>
  </div>
</template>
