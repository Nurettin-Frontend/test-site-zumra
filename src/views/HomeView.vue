<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue'

const defaultRecords = [
  {
    id: 'rec-1',
    title: 'Cloud Infrastructure Migration to AWS',
    category: 'Engineering',
    status: 'Active',
    assignee: 'Alex Morgan',
    date: '2026-06-15',
    desc: 'Migrate legacy microservices and PostgreSQL databases to containerized AWS ECS environment with zero downtime.',
  },
  {
    id: 'rec-2',
    title: 'Design System 2.0 Token Refactor',
    category: 'Design',
    status: 'Pending',
    assignee: 'Jessica Taylor',
    date: '2026-06-28',
    desc: 'Consolidate component design tokens, typography scales, and color palettes into a unified Figma library and Tailwind setup.',
  },
  {
    id: 'rec-3',
    title: 'Q3 Global Growth Marketing Campaign',
    category: 'Marketing',
    status: 'Urgent',
    assignee: 'Marcus Vance',
    date: '2026-05-30',
    desc: 'Plan multi-channel performance ad campaigns across LinkedIn, Google Ads, and tech newsletters targeting enterprise clients.',
  },
  {
    id: 'rec-4',
    title: 'Automated CI/CD Pipeline Optimization',
    category: 'Operations',
    status: 'Active',
    assignee: 'David Kim',
    date: '2026-06-10',
    desc: 'Reduce GitHub Actions build times by caching node modules and splitting parallel testing test suites.',
  },
]

const STORAGE_KEY = 'cardflow_records'

function loadRecords() {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) return JSON.parse(saved)
  } catch {
    // Fall back to the sample records when storage is unavailable.
  }
  return defaultRecords.map((record) => ({ ...record }))
}

const records = ref(loadRecords())
const search = ref('')
const filterCategory = ref('all')
const filterStatus = ref('all')
const sortOrder = ref('newest')

const modalOpen = ref(false)
const modalMode = ref('create')
const currentEditingId = ref(null)
const titleError = ref(false)

const emptyForm = () => ({
  title: '',
  category: '',
  status: 'Active',
  assignee: '',
  date: '',
  desc: '',
})

const form = ref(emptyForm())

const toast = ref({
  visible: false,
  message: '',
  type: 'success',
})

let toastTimer = null

watch(
  records,
  (value) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(value))
  },
  { deep: true },
)

const stats = computed(() => ({
  total: records.value.length,
  active: records.value.filter((record) => record.status === 'Active').length,
  pending: records.value.filter((record) => record.status === 'Pending').length,
  urgent: records.value.filter((record) => record.status === 'Urgent').length,
}))

const filteredRecords = computed(() => {
  const searchVal = search.value.toLowerCase().trim()

  const filtered = records.value.filter((record) => {
    const matchesSearch =
      record.title.toLowerCase().includes(searchVal) ||
      record.desc.toLowerCase().includes(searchVal) ||
      (record.assignee && record.assignee.toLowerCase().includes(searchVal))
    const matchesCategory = filterCategory.value === 'all' || record.category === filterCategory.value
    const matchesStatus = filterStatus.value === 'all' || record.status === filterStatus.value
    return matchesSearch && matchesCategory && matchesStatus
  })

  return [...filtered].sort((a, b) => {
    if (sortOrder.value === 'newest') return b.id.localeCompare(a.id)
    if (sortOrder.value === 'oldest') return a.id.localeCompare(b.id)
    return a.title.localeCompare(b.title)
  })
})

const activeRecord = computed(
  () => records.value.find((record) => record.id === currentEditingId.value) || null,
)

const modalMeta = computed(() => {
  if (modalMode.value === 'edit') {
    return {
      title: 'Edit Record',
      subtitle: `Updating record #${currentEditingId.value}`,
      icon: 'fa-solid fa-pen-to-square',
      wrap: 'w-10 h-10 rounded-xl bg-amber-100 text-amber-600 flex items-center justify-center',
      submit: 'Update Record',
    }
  }

  if (modalMode.value === 'read') {
    return {
      title: 'Record Details',
      subtitle: `Viewing record ID: ${currentEditingId.value}`,
      icon: 'fa-solid fa-eye',
      wrap: 'w-10 h-10 rounded-xl bg-emerald-100 text-emerald-600 flex items-center justify-center',
      submit: '',
    }
  }

  return {
    title: 'Create New Record',
    subtitle: 'Fill out the details below to add a card to your dashboard.',
    icon: 'fa-solid fa-plus',
    wrap: 'w-10 h-10 rounded-xl bg-indigo-100 text-indigo-600 flex items-center justify-center',
    submit: 'Create Record',
  }
})

function statusBadgeClass(status) {
  if (status === 'Pending') return 'bg-amber-50 text-amber-600 border-amber-200'
  if (status === 'Urgent') return 'bg-rose-50 text-rose-600 border-rose-200'
  return 'bg-emerald-50 text-emerald-600 border-emerald-200'
}

function readStatusClass(status) {
  if (status === 'Active') return 'bg-emerald-100 text-emerald-700'
  if (status === 'Pending') return 'bg-amber-100 text-amber-700'
  return 'bg-rose-100 text-rose-700'
}

function resetFilters() {
  search.value = ''
  filterCategory.value = 'all'
  filterStatus.value = 'all'
  sortOrder.value = 'newest'
  showToast('Filters reset successfully', 'info')
}

function fillForm(record) {
  form.value = {
    title: record.title,
    category: record.category,
    status: record.status,
    assignee: record.assignee || '',
    date: record.date || '',
    desc: record.desc,
  }
}

function openModal(mode, id = null) {
  if (mode !== 'create') {
    const record = records.value.find((item) => item.id === id)
    if (!record) return
    currentEditingId.value = id
    fillForm(record)
  } else {
    currentEditingId.value = null
    form.value = emptyForm()
  }

  titleError.value = false
  modalMode.value = mode
  modalOpen.value = true
  document.body.style.overflow = 'hidden'
}

function closeModal() {
  modalOpen.value = false
  document.body.style.overflow = ''
}

function handleFormSubmit() {
  const title = form.value.title.trim()
  const category = form.value.category
  const status = form.value.status
  const assignee = form.value.assignee.trim()
  const date = form.value.date
  const desc = form.value.desc.trim()

  if (title.length < 3) {
    titleError.value = true
    return
  }

  titleError.value = false

  if (modalMode.value === 'edit' && currentEditingId.value) {
    const index = records.value.findIndex((record) => record.id === currentEditingId.value)
    if (index !== -1) {
      records.value[index] = {
        id: currentEditingId.value,
        title,
        category,
        status,
        assignee,
        date,
        desc,
      }
      showToast('Record updated successfully!')
    }
  } else {
    records.value.unshift({
      id: `rec-${Date.now()}`,
      title,
      category,
      status,
      assignee,
      date,
      desc,
    })
    showToast('New record created successfully!')
  }

  closeModal()
}

function deleteRecord(id) {
  if (!confirm('Are you sure you want to delete this record? This action cannot be undone.')) return
  records.value = records.value.filter((record) => record.id !== id)
  showToast('Record deleted successfully', 'info')
}

function showToast(message, type = 'success') {
  toast.value = { visible: true, message, type }
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => {
    toast.value.visible = false
  }, 3000)
}

onBeforeUnmount(() => {
  clearTimeout(toastTimer)
  document.body.style.overflow = ''
})
</script>

<template>
  <div class="min-h-screen flex flex-col">
    <header class="bg-white border-b border-slate-200 sticky top-0 z-30 shadow-sm">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 h-16 flex items-center justify-between">
        <div class="flex items-center space-x-3">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-tr from-indigo-600 to-violet-600 flex items-center justify-center text-white shadow-md shadow-indigo-200">
            <i class="fa-solid fa-layer-group text-lg"></i>
          </div>
          <div>
            <h1 class="text-lg font-bold bg-gradient-to-r from-indigo-600 to-violet-600 bg-clip-text text-transparent">CardFlow Pro</h1>
            <p class="text-xs text-slate-400 hidden sm:block">Advanced Modern CRUD Dashboard</p>
          </div>
        </div>

        <div class="flex items-center space-x-3">
          <button
            type="button"
            class="inline-flex items-center space-x-2 bg-indigo-600 hover:bg-indigo-700 text-white font-medium px-4 py-2 rounded-xl shadow-lg shadow-indigo-100 transition-all duration-200 hover:scale-[1.02] active:scale-[0.98]"
            @click="openModal('create')"
          >
            <i class="fa-solid fa-plus"></i>
            <span>New Record</span>
          </button>
        </div>
      </div>
    </header>

    <main class="flex-1 max-w-7xl w-full mx-auto px-4 sm:px-6 lg:px-8 py-8 flex flex-col space-y-6">
      <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <div class="bg-white rounded-2xl p-5 border border-slate-200/80 shadow-sm flex items-center space-x-4">
          <div class="w-12 h-12 rounded-xl bg-indigo-50 text-indigo-600 flex items-center justify-center text-xl">
            <i class="fa-solid fa-folder-open"></i>
          </div>
          <div>
            <p class="text-xs font-medium text-slate-400 uppercase tracking-wider">Total Records</p>
            <h3 class="text-2xl font-bold text-slate-800">{{ stats.total }}</h3>
          </div>
        </div>
        <div class="bg-white rounded-2xl p-5 border border-slate-200/80 shadow-sm flex items-center space-x-4">
          <div class="w-12 h-12 rounded-xl bg-emerald-50 text-emerald-600 flex items-center justify-center text-xl">
            <i class="fa-solid fa-circle-check"></i>
          </div>
          <div>
            <p class="text-xs font-medium text-slate-400 uppercase tracking-wider">Active Status</p>
            <h3 class="text-2xl font-bold text-slate-800">{{ stats.active }}</h3>
          </div>
        </div>
        <div class="bg-white rounded-2xl p-5 border border-slate-200/80 shadow-sm flex items-center space-x-4">
          <div class="w-12 h-12 rounded-xl bg-amber-50 text-amber-600 flex items-center justify-center text-xl">
            <i class="fa-solid fa-clock"></i>
          </div>
          <div>
            <p class="text-xs font-medium text-slate-400 uppercase tracking-wider">Pending Status</p>
            <h3 class="text-2xl font-bold text-slate-800">{{ stats.pending }}</h3>
          </div>
        </div>
        <div class="bg-white rounded-2xl p-5 border border-slate-200/80 shadow-sm flex items-center space-x-4">
          <div class="w-12 h-12 rounded-xl bg-rose-50 text-rose-600 flex items-center justify-center text-xl">
            <i class="fa-solid fa-triangle-exclamation"></i>
          </div>
          <div>
            <p class="text-xs font-medium text-slate-400 uppercase tracking-wider">Archived / Urgent</p>
            <h3 class="text-2xl font-bold text-slate-800">{{ stats.urgent }}</h3>
          </div>
        </div>
      </div>

      <div class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200/80 shadow-sm flex flex-col lg:flex-row items-center justify-between gap-4">
        <div class="relative w-full lg:w-96">
          <span class="absolute inset-y-0 left-0 pl-3.5 flex items-center pointer-events-none text-slate-400">
            <i class="fa-solid fa-magnifying-glass"></i>
          </span>
          <input
            v-model="search"
            type="text"
            placeholder="Search by title, category, description..."
            class="w-full pl-10 pr-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
          />
        </div>

        <div class="flex flex-wrap items-center gap-3 w-full lg:w-auto justify-end">
          <div class="flex items-center space-x-2">
            <span class="text-xs font-medium text-slate-500 hidden sm:inline">Category:</span>
            <select
              v-model="filterCategory"
              class="bg-slate-50 border border-slate-200 rounded-xl px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500"
            >
              <option value="all">All Categories</option>
              <option value="Engineering">Engineering</option>
              <option value="Design">Design</option>
              <option value="Marketing">Marketing</option>
              <option value="Operations">Operations</option>
            </select>
          </div>

          <div class="flex items-center space-x-2">
            <span class="text-xs font-medium text-slate-500 hidden sm:inline">Status:</span>
            <select
              v-model="filterStatus"
              class="bg-slate-50 border border-slate-200 rounded-xl px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500"
            >
              <option value="all">All Statuses</option>
              <option value="Active">Active</option>
              <option value="Pending">Pending</option>
              <option value="Urgent">Urgent</option>
            </select>
          </div>

          <div class="flex items-center space-x-2">
            <span class="text-xs font-medium text-slate-500 hidden sm:inline">Sort:</span>
            <select
              v-model="sortOrder"
              class="bg-slate-50 border border-slate-200 rounded-xl px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500"
            >
              <option value="newest">Newest First</option>
              <option value="oldest">Oldest First</option>
              <option value="title">Title (A-Z)</option>
            </select>
          </div>

          <button
            type="button"
            title="Reset Filters"
            class="p-2.5 bg-slate-100 hover:bg-slate-200 text-slate-600 rounded-xl transition-all"
            @click="resetFilters"
          >
            <i class="fa-solid fa-rotate-right"></i>
          </button>
        </div>
      </div>

      <div v-if="filteredRecords.length" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        <div
          v-for="record in filteredRecords"
          :key="record.id"
          class="bg-white rounded-2xl border border-slate-200/80 shadow-sm hover:shadow-md transition-all duration-300 flex flex-col justify-between overflow-hidden group animate-fade-in"
        >
          <div class="p-6 flex-1 flex flex-col">
            <div class="flex items-center justify-between mb-3">
              <span class="px-3 py-1 rounded-full bg-slate-100 text-slate-600 text-xs font-semibold">
                {{ record.category }}
              </span>
              <span class="px-3 py-1 rounded-full border text-xs font-semibold" :class="statusBadgeClass(record.status)">
                {{ record.status }}
              </span>
            </div>

            <h3
              class="text-base font-bold text-slate-800 hover:text-indigo-600 cursor-pointer transition-colors mb-2 line-clamp-1"
              @click="openModal('read', record.id)"
            >
              {{ record.title }}
            </h3>

            <p class="text-slate-500 text-sm mb-4 line-clamp-2 leading-relaxed flex-1">
              {{ record.desc }}
            </p>

            <div class="pt-4 border-t border-slate-100 flex items-center justify-between text-xs text-slate-400">
              <div class="flex items-center space-x-1.5">
                <i class="fa-solid fa-user-tie text-slate-400"></i>
                <span class="font-medium text-slate-600">{{ record.assignee || 'Unassigned' }}</span>
              </div>
              <div class="flex items-center space-x-1.5">
                <i class="fa-solid fa-calendar-days text-slate-400"></i>
                <span>{{ record.date || 'No date' }}</span>
              </div>
            </div>
          </div>

          <div class="bg-slate-50 px-6 py-3 border-t border-slate-100 flex items-center justify-between">
            <button
              type="button"
              class="text-xs font-semibold text-indigo-600 hover:text-indigo-800 transition-colors flex items-center space-x-1"
              @click="openModal('read', record.id)"
            >
              <i class="fa-solid fa-eye"></i>
              <span>View Details</span>
            </button>
            <div class="flex items-center space-x-1">
              <button
                type="button"
                title="Edit Record"
                class="w-8 h-8 rounded-lg hover:bg-slate-200/70 text-slate-500 flex items-center justify-center transition-colors"
                @click="openModal('edit', record.id)"
              >
                <i class="fa-solid fa-pen-to-square text-xs"></i>
              </button>
              <button
                type="button"
                title="Delete Record"
                class="w-8 h-8 rounded-lg hover:bg-rose-100 text-rose-500 flex items-center justify-center transition-colors"
                @click="deleteRecord(record.id)"
              >
                <i class="fa-solid fa-trash text-xs"></i>
              </button>
            </div>
          </div>
        </div>
      </div>

      <div
        v-else
        class="flex flex-col items-center justify-center py-16 px-4 bg-white rounded-2xl border border-slate-200/80 shadow-sm text-center"
      >
        <div class="w-16 h-16 rounded-2xl bg-indigo-50 text-indigo-500 flex items-center justify-center text-2xl mb-4">
          <i class="fa-solid fa-folder-closed"></i>
        </div>
        <h3 class="text-lg font-bold text-slate-800 mb-1">No records found</h3>
        <p class="text-slate-400 text-sm max-w-sm mb-6">
          We couldn't find any records matching your search or filter criteria. Try adjusting your filters or create a new record.
        </p>
        <button
          type="button"
          class="bg-indigo-600 hover:bg-indigo-700 text-white font-medium px-5 py-2.5 rounded-xl shadow-lg shadow-indigo-100 transition-all"
          @click="openModal('create')"
        >
          Create New Record
        </button>
      </div>
    </main>

    <div
      v-if="modalOpen"
      class="fixed inset-0 z-50 overflow-y-auto bg-slate-900/40 backdrop-blur-sm flex items-center justify-center p-4 sm:p-6"
    >
      <div class="bg-white w-full max-w-2xl rounded-3xl shadow-2xl border border-slate-100 overflow-hidden transform transition-all animate-fade-in flex flex-col max-h-[90vh]">
        <div class="px-6 py-5 border-b border-slate-100 flex items-center justify-between bg-slate-50/50">
          <div class="flex items-center space-x-3">
            <div :class="modalMeta.wrap">
              <i :class="modalMeta.icon"></i>
            </div>
            <div>
              <h3 class="text-lg font-bold text-slate-800">{{ modalMeta.title }}</h3>
              <p class="text-xs text-slate-400">{{ modalMeta.subtitle }}</p>
            </div>
          </div>
          <button
            type="button"
            class="w-9 h-9 rounded-full bg-slate-100 hover:bg-slate-200 text-slate-500 flex items-center justify-center transition-all"
            @click="closeModal"
          >
            <i class="fa-solid fa-xmark"></i>
          </button>
        </div>

        <div class="p-6 overflow-y-auto flex-1">
          <form v-if="modalMode !== 'read'" class="space-y-5" @submit.prevent="handleFormSubmit">
            <div>
              <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Record Title *</label>
              <input
                v-model="form.title"
                type="text"
                required
                placeholder="e.g. Q3 Cloud Infrastructure Migration"
                class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
              />
              <p v-if="titleError" class="text-xs text-rose-500 mt-1">Title is required (min 3 characters).</p>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <div>
                <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Category *</label>
                <select
                  v-model="form.category"
                  required
                  class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
                >
                  <option value="" disabled>Select category</option>
                  <option value="Engineering">Engineering</option>
                  <option value="Design">Design</option>
                  <option value="Marketing">Marketing</option>
                  <option value="Operations">Operations</option>
                </select>
              </div>
              <div>
                <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Status *</label>
                <select
                  v-model="form.status"
                  required
                  class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
                >
                  <option value="Active">Active</option>
                  <option value="Pending">Pending</option>
                  <option value="Urgent">Urgent</option>
                </select>
              </div>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <div>
                <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Assignee Name</label>
                <input
                  v-model="form.assignee"
                  type="text"
                  placeholder="e.g. Sarah Jenkins"
                  class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
                />
              </div>
              <div>
                <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Target Due Date</label>
                <input
                  v-model="form.date"
                  type="date"
                  class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
                />
              </div>
            </div>

            <div>
              <label class="block text-xs font-semibold text-slate-600 uppercase tracking-wider mb-2">Detailed Description *</label>
              <textarea
                v-model="form.desc"
                rows="4"
                required
                placeholder="Provide comprehensive details about this record..."
                class="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-500 transition-all"
              ></textarea>
            </div>

            <div class="pt-4 border-t border-slate-100 flex items-center justify-end space-x-3">
              <button
                type="button"
                class="px-5 py-2.5 rounded-xl border border-slate-200 hover:bg-slate-100 text-slate-600 font-medium text-sm transition-all"
                @click="closeModal"
              >
                Cancel
              </button>
              <button
                type="submit"
                class="px-6 py-2.5 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-medium text-sm shadow-lg shadow-indigo-100 transition-all"
              >
                {{ modalMeta.submit }}
              </button>
            </div>
          </form>

          <div v-else-if="activeRecord" class="space-y-6">
            <div class="flex items-center space-x-3">
              <span class="px-3 py-1 rounded-full text-xs font-semibold" :class="readStatusClass(activeRecord.status)">
                {{ activeRecord.status }}
              </span>
              <span class="px-3 py-1 rounded-full bg-slate-100 text-slate-600 text-xs font-semibold">
                {{ activeRecord.category }}
              </span>
            </div>
            <div>
              <h4 class="text-xs font-semibold text-slate-400 uppercase tracking-wider mb-1">Record Title</h4>
              <p class="text-xl font-bold text-slate-800">{{ activeRecord.title }}</p>
            </div>
            <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 bg-slate-50 p-4 rounded-2xl border border-slate-100">
              <div>
                <span class="text-xs font-semibold text-slate-400 uppercase tracking-wider block mb-1">Assignee</span>
                <p class="text-sm font-medium text-slate-700 flex items-center space-x-2">
                  <i class="fa-solid fa-user-circle text-indigo-500"></i>
                  <span>{{ activeRecord.assignee || 'Unassigned' }}</span>
                </p>
              </div>
              <div>
                <span class="text-xs font-semibold text-slate-400 uppercase tracking-wider block mb-1">Due Date</span>
                <p class="text-sm font-medium text-slate-700 flex items-center space-x-2">
                  <i class="fa-solid fa-calendar text-indigo-500"></i>
                  <span>{{ activeRecord.date || 'Not specified' }}</span>
                </p>
              </div>
            </div>
            <div>
              <h4 class="text-xs font-semibold text-slate-400 uppercase tracking-wider mb-2">Description</h4>
              <p class="text-slate-600 text-sm leading-relaxed bg-slate-50 p-4 rounded-2xl border border-slate-100 whitespace-pre-wrap">
                {{ activeRecord.desc }}
              </p>
            </div>
            <div class="pt-4 border-t border-slate-100 flex items-center justify-end space-x-3">
              <button
                type="button"
                class="px-5 py-2.5 rounded-xl border border-slate-200 hover:bg-slate-100 text-slate-600 font-medium text-sm transition-all"
                @click="closeModal"
              >
                Close
              </button>
              <button
                type="button"
                class="px-6 py-2.5 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-medium text-sm shadow-lg shadow-indigo-100 transition-all flex items-center space-x-2"
                @click="openModal('edit', activeRecord.id)"
              >
                <i class="fa-solid fa-pen"></i>
                <span>Edit Record</span>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div
      class="fixed bottom-6 right-6 z-50 transform transition-all duration-300 pointer-events-none"
      :class="toast.visible ? 'translate-y-0 opacity-100' : 'translate-y-20 opacity-0'"
    >
      <div class="bg-slate-900 text-white px-5 py-3 rounded-2xl shadow-xl flex items-center space-x-3 border border-slate-800">
        <div :class="toast.type === 'success' ? 'text-emerald-400' : 'text-sky-400'">
          <i :class="toast.type === 'success' ? 'fa-solid fa-circle-check text-lg' : 'fa-solid fa-circle-info text-lg'"></i>
        </div>
        <p class="text-sm font-medium">{{ toast.message || 'Operation completed successfully.' }}</p>
      </div>
    </div>
  </div>
</template>
