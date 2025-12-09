<template>
  <div class="max-w-6xl mx-auto p-6">
    <h2 class="text-3xl font-bold mb-6 text-center">Random Team Picker ({{ teamCount }} Teams)</h2>

    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
      <!-- Members Input -->
      <div>
        <textarea v-model="membersText" rows="12"
          class="w-full p-3 rounded-xl border-2 border-blue-300 focus:ring focus:outline-none bg-blue-50"
          placeholder="Paste or type member names, one per line"></textarea>

        <div class="flex items-center gap-2 mt-3">
          <button @click="useSample" class="px-3 py-1 rounded bg-purple-200 hover:bg-purple-300 shadow cursor-pointer">
            Use sample
          </button>

          <button @click="clearMembers" class="px-3 py-1 rounded bg-red-200 hover:bg-red-300 shadow cursor-pointer">
            Clear
          </button>
        </div>
      </div>

      <!-- Options -->
      <div class="space-y-4 bg-white p-4 rounded-xl border shadow">
        <label class="block text-sm font-medium mb-1">Team Sizes</label>

        <div class="grid grid-cols-2 md:grid-cols-3 gap-4">
          <div v-for="team in teams" :key="team.key" class="p-3 bg-gray-50 rounded-lg border">
            <div class="text-sm font-semibold">
              {{ team.key }} size
            </div>
            <input type="number" v-model.number="teamSizes[team.key]" min="0" class="w-24 mt-1 p-1 rounded border" />
          </div>
        </div>

        <div class="flex items-center gap-2">
          <button @click="generate" :disabled="!enoughMembers"
            class="px-4 py-2 rounded bg-blue-600 text-white hover:bg-blue-700 disabled:opacity-50 cursor-pointer">
            Generate
          </button>

          <button @click="exportCsv" :disabled="!hasAssignments"
            class="px-4 py-2 rounded bg-yellow-300 hover:bg-yellow-400 disabled:opacity-50 cursor-pointer">
            Export CSV
          </button>
        </div>

        <div class="text-sm text-gray-600">
          Total members: <strong>{{ members.length }}</strong>
        </div>

        <div v-if="warning" class="text-sm text-yellow-700">{{ warning }}</div>
      </div>
    </div>

    <!-- Teams Output -->
    <div v-if="hasAssignments" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-5 gap-4">

      <div v-for="team in teams" :key="team.key" class="p-4 rounded-xl border shadow-lg" :class="team.bg">

        <h3 class="font-semibold mb-2 text-lg">
          {{ team.key }} ({{ teamSizes[team.key] }})
        </h3>

        <ul class="space-y-2">
          <li v-for="(m, i) in assignments[team.key]" :key="i" class="flex items-center justify-between">

            <div>{{ i + 1 }}. {{ m }}</div>

            <button @click="toggleLock(team.key, m)" class="text-xs px-2 py-1 rounded border cursor-pointer"
              :class="locked[team.key].has(m) ? 'bg-yellow-200' : ''">
              {{ locked[team.key].has(m) ? 'Locked' : 'Lock' }}
            </button>
          </li>
        </ul>
      </div>

    </div>

    <div v-else class="text-center text-gray-500 py-10">
      No assignment yet — generate teams to see results.
    </div>
  </div>
</template>
<script setup>
import { ref, computed } from 'vue'

// ❗ SET YOUR TEAM COUNT HERE (YOU WANT 5 TEAMS)
const teamCount = 5

// Auto-generate team labels: A, B, C, D, E...
const teams = Array.from({ length: teamCount }, (_, i) => {
  const key = `Team ${String.fromCharCode(65 + i)}`
  return {
    key,
    bg: ['bg-blue-50', 'bg-green-50', 'bg-purple-50', 'bg-pink-50', 'bg-yellow-50', 'bg-orange-50'][i] || 'bg-gray-100'
  }
})

const membersText = ref('')
const warning = ref('')

// Dynamic sizes
const teamSizes = ref({})
teams.forEach(t => teamSizes.value[t.key] = 10)

// Storage for locks and results
const locked = ref({})
const assignments = ref({})
teams.forEach(t => {
  locked.value[t.key] = new Set()
  assignments.value[t.key] = []
})

const members = computed(() =>
  membersText.value.split(/\r?\n/)
    .map(s => s.trim())
    .filter(Boolean)
)

const hasAssignments = computed(() =>
  teams.some(t => assignments.value[t.key].length > 0)
)

function useSample() {
  membersText.value = Array.from({ length: 50 })
    .map((_, i) => `Member ${i + 1}`)
    .join('\n')
}

function clearMembers() {
  membersText.value = ''
  resetAll()
}

function resetAll() {
  teams.forEach(t => {
    assignments.value[t.key] = []
    locked.value[t.key] = new Set()
  })
}

function shuffle(arr) {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
      ;[a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

function generate() {
  warning.value = ''

  const totalNeeded = teams.reduce(
    (sum, t) => sum + teamSizes.value[t.key], 0
  )

  if (members.value.length < totalNeeded) {
    warning.value = `Not enough members. Need ${totalNeeded}, have ${members.value.length}.`
    resetAll()
    return
  }

  // all locked members
  const lockedMembers = new Set()
  teams.forEach(t => locked.value[t.key].forEach(m => lockedMembers.add(m)))

  const unlocked = shuffle(
    members.value.filter(m => !lockedMembers.has(m))
  )

  let idx = 0

  // assign per team
  teams.forEach(t => {
    const size = teamSizes.value[t.key]
    const teamArr = []

    locked.value[t.key].forEach(m => teamArr.push(m))

    while (teamArr.length < size) {
      teamArr.push(unlocked[idx++])
    }

    assignments.value[t.key] = teamArr
  })

  // === REMOVE assigned members from textarea ===
  const assignedMembers = new Set()
  teams.forEach(t => {
    assignments.value[t.key].forEach(m => assignedMembers.add(m))
  })

  // Keep only members not assigned
  membersText.value = members.value
    .filter(m => !assignedMembers.has(m))
    .join('\n')
}

const enoughMembers = computed(() => {
  const totalNeeded = teams.reduce(
    (sum, t) => sum + teamSizes.value[t.key], 0
  )
  return members.value.length >= totalNeeded
})

function toggleLock(teamKey, member) {
  const set = locked.value[teamKey]
  set.has(member) ? set.delete(member) : set.add(member)
}

function exportCsv() {
  const maxLen = Math.max(
    ...teams.map(t => assignments.value[t.key].length)
  )

  const header = teams.map(t => t.key).join(',')
  const rows = [header]

  for (let i = 0; i < maxLen; i++) {
    const row = teams.map(t => assignments.value[t.key][i] ?? '')
    rows.push(row.map(c => `"${c}"`).join(','))
  }

  const csv = rows.join('\n')
  const blob = new Blob([csv], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'teams.csv'
  a.click()
  URL.revokeObjectURL(url)
}
</script>
