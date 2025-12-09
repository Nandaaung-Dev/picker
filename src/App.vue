<template>
  <div class="mx-auto p-6">
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
            <!-- Disabled because fixed 5 males + 5 females -->
            <input type="number" v-model.number="teamSizes[team.key]" min="0" class="w-24 mt-1 p-1 rounded border"
              disabled />
            <div class="text-xs text-gray-500 mt-1">Fixed 10 (5 Male + 5 Female)</div>
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
          Total members: <strong>{{ members.length }}</strong> (Male: {{ maleMembers.length }}, Female: {{
            femaleMembers.length }})
        </div>

        <div v-if="warning" class="text-sm text-yellow-700">{{ warning }}</div>
      </div>
    </div>

    <!-- Teams Output -->
    <div v-if="hasAssignments" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-5 gap-4">

      <div v-for="team in teams" :key="team.key" class="p-4 rounded-xl border shadow-lg" :class="team.bg">

        <h3 class="font-semibold mb-2 text-lg">
          {{ team.key }}
        </h3>
        <ul class="space-y-2">
          <li v-for="(member, i) in combinedAssignments[team.key]" :key="`member-${i}`"
            class="flex items-center justify-between">
            <div>
              {{ i + 1 }}. {{ member.name }}
              <!-- <span v-if="member.gender === 'male'" class="text-blue-600 ml-2 text-xs font-semibold">
                (M)
              </span>
              <span v-else-if="member.gender === 'female'" class="text-pink-600 ml-2 text-xs font-semibold">
                (F)
              </span> -->
            </div>
            <button @click="toggleLock(team.key, member.name, member.gender)"
              class="text-xs px-2 py-1 rounded border cursor-pointer"
              :class="(member.gender === 'male' ? lockedMale[team.key] : lockedFemale[team.key]).has(member.name) ? 'bg-yellow-200' : ''">
              {{ (member.gender === 'male' ? lockedMale[team.key] : lockedFemale[team.key]).has(member.name) ? 'Locked'
                : 'Lock' }}
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

// Number of teams
const teamCount = 5

// Generate teams with colors
const teams = Array.from({ length: teamCount }, (_, i) => {
  const key = `Team ${String.fromCharCode(65 + i)}`
  return {
    key,
    bg: ['bg-blue-50', 'bg-green-50', 'bg-purple-50', 'bg-pink-50', 'bg-yellow-50'][i] || 'bg-gray-100'
  }
})

const membersText = ref('')
const warning = ref('')

// Fixed team sizes (disabled input, fixed 10 per team)
const teamSizes = ref({})
teams.forEach(t => teamSizes.value[t.key] = 10)

// Helper functions for gender detection
function isMale(name) {
  return /\b(U|Mg|Ko)\b/.test(name)
}
function isFemale(name) {
  return /\b(Ma|Daw)\b/.test(name)
}

// Parse all members
const members = computed(() =>
  membersText.value.split(/\r?\n/)
    .map(s => s.trim())
    .filter(Boolean)
)

// Separate by gender
const maleMembers = computed(() =>
  members.value.filter(m => isMale(m))
)

const femaleMembers = computed(() =>
  members.value.filter(m => isFemale(m))
)

// Locked male and female members per team
const lockedMale = ref({})
const lockedFemale = ref({})
teams.forEach(t => {
  lockedMale.value[t.key] = new Set()
  lockedFemale.value[t.key] = new Set()
})

// Assignments for male and female separately
const assignmentsMale = ref({})
const assignmentsFemale = ref({})
const assignments = ref({})
teams.forEach(t => {
  assignmentsMale.value[t.key] = []
  assignmentsFemale.value[t.key] = []
  assignments.value[t.key] = []
})

// Whether any assignment exists
const hasAssignments = computed(() =>
  teams.some(t => assignments.value[t.key].length > 0)
)

// Shuffle helper
function shuffle(arr) {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
      ;[a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

function resetAll() {
  teams.forEach(t => {
    assignments.value[t.key] = []
    assignmentsMale.value[t.key] = []
    assignmentsFemale.value[t.key] = []
    lockedMale.value[t.key] = new Set()
    lockedFemale.value[t.key] = new Set()
  })
  warning.value = ''
}

function useSample() {
  const malePrefixes = ['U', 'Mg', 'Ko']
  const femalePrefixes = ['Ma', 'Daw']

  const males = Array.from({ length: 25 }, (_, i) => {
    // Cycle through prefixes
    const prefix = malePrefixes[i % malePrefixes.length]
    return `${prefix} Member ${i + 1}`
  })

  const females = Array.from({ length: 25 }, (_, i) => {
    const prefix = femalePrefixes[i % femalePrefixes.length]
    return `${prefix} Member ${i + 1}`
  })

  // Combine and shuffle so males and females are mixed
  const combined = [...males, ...females]

  // Optional shuffle for mixed order
  function shuffle(arr) {
    const a = [...arr]
    for (let i = a.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1))
        ;[a[i], a[j]] = [a[j], a[i]]
    }
    return a
  }

  membersText.value = shuffle(combined).join('\n')
}

function clearMembers() {
  membersText.value = ''
  resetAll()
}

const enoughMembers = computed(() => {
  // Need at least 5 males * teams and 5 females * teams
  return maleMembers.value.length >= teamCount * 5 &&
    femaleMembers.value.length >= teamCount * 5
})

function generate() {
  warning.value = ''

  const malesNeeded = teamCount * 5
  const femalesNeeded = teamCount * 5

  if (maleMembers.value.length < malesNeeded) {
    warning.value = `Not enough male members. Need ${malesNeeded}, have ${maleMembers.value.length}.`
    resetAll()
    return
  }
  if (femaleMembers.value.length < femalesNeeded) {
    warning.value = `Not enough female members. Need ${femalesNeeded}, have ${femaleMembers.value.length}.`
    resetAll()
    return
  }

  // Collect locked males and females globally
  const lockedMalesGlobal = new Set()
  const lockedFemalesGlobal = new Set()
  teams.forEach(t => {
    lockedMale.value[t.key].forEach(m => lockedMalesGlobal.add(m))
    lockedFemale.value[t.key].forEach(f => lockedFemalesGlobal.add(f))
  })

  // Unlocked males and females shuffled
  const unlockedMales = shuffle(maleMembers.value.filter(m => !lockedMalesGlobal.has(m)))
  const unlockedFemales = shuffle(femaleMembers.value.filter(f => !lockedFemalesGlobal.has(f)))

  let maleIndex = 0
  let femaleIndex = 0

  // Assign per team males and females
  teams.forEach(t => {
    const maleTeamArr = []
    const femaleTeamArr = []

    // Add locked males/females first
    lockedMale.value[t.key].forEach(m => maleTeamArr.push(m))
    lockedFemale.value[t.key].forEach(f => femaleTeamArr.push(f))

    while (maleTeamArr.length < 5) {
      maleTeamArr.push(unlockedMales[maleIndex++])
    }
    while (femaleTeamArr.length < 5) {
      femaleTeamArr.push(unlockedFemales[femaleIndex++])
    }

    assignmentsMale.value[t.key] = maleTeamArr
    assignmentsFemale.value[t.key] = femaleTeamArr
    // Combine for generic assignment (optional, for export or else)
    assignments.value[t.key] = [...maleTeamArr, ...femaleTeamArr]
  })

  // Remove assigned members from textarea
  const assignedMembers = new Set()
  teams.forEach(t => {
    assignments.value[t.key].forEach(m => assignedMembers.add(m))
  })

  membersText.value = members.value.filter(m => !assignedMembers.has(m)).join('\n')
}

function toggleLock(teamKey, member, gender) {
  if (gender === 'male') {
    const set = lockedMale.value[teamKey]
    set.has(member) ? set.delete(member) : set.add(member)
  } else {
    const set = lockedFemale.value[teamKey]
    set.has(member) ? set.delete(member) : set.add(member)
  }
}

const combinedAssignments = computed(() => {
  const combined = {}
  teams.forEach(t => {
    // Map male members to {name, gender}
    const males = assignmentsMale.value[t.key].map(name => ({ name, gender: 'male' }))
    // Map female members similarly
    const females = assignmentsFemale.value[t.key].map(name => ({ name, gender: 'female' }))
    // Combine and keep order: males then females (or shuffle if you want)
    combined[t.key] = [...males, ...females]
  })
  return combined
})

function exportCsv() {
  // Find max length of combined lists (should be 10 fixed, but dynamic if needed)
  const maxLen = Math.max(...teams.map(t => assignments.value[t.key].length))

  // Header with team names only (no male/female separate columns)
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
