<template>
  <div class="card p-4">
    <div class="mb-4 flex items-center justify-between gap-3">
      <h3 class="text-sm font-semibold text-gray-900 dark:text-white">
        {{ t('admin.dashboard.userTokenDistribution') }}
      </h3>
      <!-- Account search filter -->
      <div class="relative min-w-0 flex-1 max-w-xs" ref="dropdownRef">
        <!-- Selected tags + input -->
        <div
          class="flex flex-wrap items-center gap-1 rounded-lg border border-gray-200 bg-gray-50 px-2 py-1 text-xs dark:border-gray-700 dark:bg-dark-800 cursor-text"
          @click="focusInput"
        >
          <span
            v-for="id in selectedAccountIDs"
            :key="id"
            class="inline-flex items-center gap-0.5 rounded bg-primary-100 px-1.5 py-0.5 text-xs font-medium text-primary-700 dark:bg-primary-900/40 dark:text-primary-300"
          >
            <span class="max-w-[80px] truncate">{{ accountNameMap.get(id) || `#${id}` }}</span>
            <button type="button" class="ml-0.5 hover:text-primary-900 dark:hover:text-primary-100" @click.stop="removeAccount(id)">
              <svg class="h-2.5 w-2.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </span>
          <input
            ref="searchInputRef"
            v-model="searchText"
            type="text"
            class="min-w-[80px] flex-1 bg-transparent text-xs text-gray-700 outline-none placeholder-gray-400 dark:text-gray-300 dark:placeholder-gray-600"
            :placeholder="selectedAccountIDs.size === 0 ? t('admin.dashboard.allUsers') : ''"
            @focus="showDropdown = true"
            @input="showDropdown = true"
            @keydown.escape="showDropdown = false"
            @keydown.backspace="onBackspace"
          />
        </div>
        <!-- Suggestion dropdown -->
        <div
          v-if="showDropdown && filteredSuggestions.length > 0"
          class="absolute right-0 top-full z-50 mt-1 max-h-52 w-full overflow-y-auto rounded-lg border border-gray-200 bg-white py-1 shadow-lg dark:border-dark-600 dark:bg-dark-800"
        >
          <button
            v-for="account in filteredSuggestions"
            :key="account.user_id"
            class="flex w-full items-center justify-between px-3 py-1.5 text-left text-xs text-gray-700 hover:bg-gray-50 dark:text-gray-300 dark:hover:bg-dark-700"
            @mousedown.prevent="toggleAccount(account.user_id)"
          >
            <span class="truncate" :title="`${account.username || ''} ${account.email || String(account.user_id)}`">
              <span v-if="account.username" class="font-medium">{{ account.username }}</span>
              <span class="text-gray-400 dark:text-gray-500" :class="account.username ? 'ml-1 text-xs' : ''">{{ account.email || `#${account.user_id}` }}</span>
            </span>
            <svg v-if="selectedAccountIDs.has(account.user_id)" class="ml-1 h-3 w-3 shrink-0 text-primary-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M5 13l4 4L19 7" />
            </svg>
          </button>
        </div>
      </div>
    </div>

    <div v-if="loading" class="flex h-48 items-center justify-center">
      <LoadingSpinner />
    </div>
    <div v-else-if="displayStats.length > 0" class="max-h-64 overflow-y-auto">
      <table class="w-full text-xs">
        <thead>
          <tr class="text-gray-500 dark:text-gray-400">
            <th class="pb-2 text-left">{{ t('admin.dashboard.username') }}</th>
            <th class="pb-2 text-left text-gray-400 dark:text-gray-500">{{ t('admin.dashboard.userEmail') }}</th>
            <th class="pb-2 text-right">{{ t('admin.dashboard.requests') }}</th>
            <th class="pb-2 text-right">{{ t('admin.dashboard.tokens') }}</th>
          </tr>
        </thead>
        <tbody>
          <!-- Selected total row -->
          <tr
            v-if="selectedAccountIDs.size > 1"
            class="border-t border-gray-100 bg-blue-50/50 dark:border-gray-700 dark:bg-blue-900/10"
          >
            <td class="py-1.5 font-semibold text-blue-700 dark:text-blue-400" colspan="2">
              {{ t('admin.dashboard.selectedTotal') }} ({{ selectedAccountIDs.size }})
            </td>
            <td class="py-1.5 text-right font-semibold text-gray-700 dark:text-gray-300">
              {{ formatNumber(selectedTotals.requests) }}
            </td>
            <td class="py-1.5 text-right font-semibold text-gray-700 dark:text-gray-300">
              {{ formatTokens(selectedTotals.tokens) }}
            </td>
          </tr>
          <!-- Per-user rows -->
          <template v-for="account in displayStats" :key="account.user_id">
            <tr
              class="cursor-pointer border-t border-gray-100 transition-colors hover:bg-gray-50 dark:border-gray-700 dark:hover:bg-dark-700/40"
              @click="toggleBreakdown(account.user_id)"
            >
              <td class="max-w-[120px] truncate py-1.5 font-medium text-blue-600 hover:text-blue-800 dark:text-blue-400 dark:hover:text-blue-300" :title="account.username || account.email || String(account.user_id)">
                <span class="inline-flex items-center gap-1">
                  <svg v-if="expandedID === account.user_id" class="h-3 w-3 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                  </svg>
                  <svg v-else class="h-3 w-3 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                  </svg>
                  {{ account.username || `#${account.user_id}` }}
                </span>
              </td>
              <td class="max-w-[140px] truncate py-1.5 text-left text-gray-400 dark:text-gray-500" :title="account.email">{{ account.email || '—' }}</td>
              <td class="py-1.5 text-right text-gray-600 dark:text-gray-400">{{ formatNumber(account.requests) }}</td>
              <td class="py-1.5 text-right text-gray-600 dark:text-gray-400">{{ formatTokens(account.total_tokens) }}</td>
            </tr>
            <!-- Model breakdown sub-rows -->
            <tr v-if="expandedID === account.user_id">
              <td colspan="4" class="p-0">
                <div class="bg-gray-50/50 dark:bg-dark-700/30">
                  <div v-if="breakdownLoading" class="flex items-center justify-center py-3">
                    <LoadingSpinner />
                  </div>
                  <div v-else-if="breakdownItems.length === 0" class="py-2 text-center text-xs text-gray-400">
                    {{ t('admin.dashboard.noDataAvailable') }}
                  </div>
                  <table v-else class="w-full text-xs">
                    <tbody>
                      <tr
                        v-for="item in breakdownItems"
                        :key="item.model"
                        class="border-t border-gray-100/50 dark:border-gray-700/50"
                      >
                        <td class="max-w-[120px] truncate py-1 pl-6 text-gray-600 dark:text-gray-300" :title="item.model">
                          {{ item.model || '—' }}
                        </td>
                        <td class="py-1 text-left text-gray-300 dark:text-gray-600" />
                        <td class="py-1 text-right text-gray-500 dark:text-gray-400">
                          {{ formatNumber(item.requests) }}
                        </td>
                        <td class="py-1 text-right text-gray-500 dark:text-gray-400">
                          {{ formatTokens(item.total_tokens) }}
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </td>
            </tr>
          </template>
        </tbody>
      </table>
    </div>
    <div
      v-else
      class="flex h-48 items-center justify-center text-sm text-gray-500 dark:text-gray-400"
    >
      {{ t('admin.dashboard.noDataAvailable') }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, onMounted, onUnmounted } from 'vue'
import { useI18n } from 'vue-i18n'
import LoadingSpinner from '@/components/common/LoadingSpinner.vue'
import { getUserModelBreakdown } from '@/api/admin/dashboard'
import type { UserTokenStat, UserModelBreakdown } from '@/types'

const { t } = useI18n()

const props = withDefaults(defineProps<{
  accountStats: UserTokenStat[]
  loading?: boolean
  startDate?: string
  endDate?: string
}>(), {
  loading: false,
})

const dropdownRef = ref<HTMLElement | null>(null)
const searchInputRef = ref<HTMLInputElement | null>(null)
const showDropdown = ref(false)
const searchText = ref('')
const selectedAccountIDs = ref<Set<number>>(new Set())
const expandedID = ref<number | null>(null)
const breakdownItems = ref<UserModelBreakdown[]>([])
const breakdownLoading = ref(false)

const accountNameMap = computed(() => {
  const m = new Map<number, string>()
  for (const a of props.accountStats) m.set(a.user_id, a.username || a.email || `#${a.user_id}`)
  return m
})

const filteredSuggestions = computed(() => {
  const q = searchText.value.trim().toLowerCase()
  if (!q) return props.accountStats
  return props.accountStats.filter(a =>
    (a.username || '').toLowerCase().includes(q) ||
    (a.email || '').toLowerCase().includes(q) ||
    String(a.user_id).includes(q)
  )
})

const focusInput = () => searchInputRef.value?.focus()

const onBackspace = () => {
  if (searchText.value === '' && selectedAccountIDs.value.size > 0) {
    const last = [...selectedAccountIDs.value].at(-1)!
    removeAccount(last)
  }
}

const removeAccount = (id: number) => {
  const next = new Set(selectedAccountIDs.value)
  next.delete(id)
  selectedAccountIDs.value = next
}

const displayStats = computed(() => {
  if (selectedAccountIDs.value.size === 0) return props.accountStats
  return props.accountStats.filter(a => selectedAccountIDs.value.has(a.user_id))
})

const selectedTotals = computed(() => {
  const rows = selectedAccountIDs.value.size === 0 ? props.accountStats : displayStats.value
  return rows.reduce(
    (acc, a) => ({ requests: acc.requests + a.requests, tokens: acc.tokens + a.total_tokens }),
    { requests: 0, tokens: 0 }
  )
})

const toggleAccount = (id: number) => {
  const next = new Set(selectedAccountIDs.value)
  if (next.has(id)) next.delete(id)
  else next.add(id)
  selectedAccountIDs.value = next
  searchText.value = ''
  searchInputRef.value?.focus()
}

const toggleBreakdown = async (userID: number) => {
  if (expandedID.value === userID) {
    expandedID.value = null
    return
  }
  expandedID.value = userID
  breakdownLoading.value = true
  breakdownItems.value = []
  try {
    const res = await getUserModelBreakdown({
      user_id: userID,
      start_date: props.startDate,
      end_date: props.endDate,
    })
    breakdownItems.value = res.breakdown || []
  } catch {
    breakdownItems.value = []
  } finally {
    breakdownLoading.value = false
  }
}

const handleClickOutside = (e: MouseEvent) => {
  if (dropdownRef.value && !dropdownRef.value.contains(e.target as Node)) {
    showDropdown.value = false
  }
}

onMounted(() => document.addEventListener('mousedown', handleClickOutside))
onUnmounted(() => document.removeEventListener('mousedown', handleClickOutside))

const formatTokens = (value: number): string => {
  if (value >= 1_000_000_000) return `${(value / 1_000_000_000).toFixed(2)}B`
  if (value >= 1_000_000) return `${(value / 1_000_000).toFixed(2)}M`
  if (value >= 1_000) return `${(value / 1_000).toFixed(2)}K`
  return value.toLocaleString()
}

const formatNumber = (value: number): string => value.toLocaleString()
</script>
