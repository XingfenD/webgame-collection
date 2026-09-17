<script setup lang="ts">
import { computed, onBeforeUnmount, ref, useId, watch } from 'vue'
import type { GameSummary, GameType } from '@/data/types'
import { GAME_TYPES } from '@/data/types'
import { GAME_TYPE_LABELS } from '@/lib/labels'
import { DEFAULT_FILTER, countByDuration, countByTag, countByType, type DurationBucket } from '@/lib/filter'
import { useFilterState } from '@/composables/useFilterState'

const props = defineProps<{ games: GameSummary[] }>()
const { state, update } = useFilterState()

const typeCounts = computed(() => countByType(props.games))
const durationCounts = computed(() => countByDuration(props.games))
const tagCounts = computed(() => countByTag(props.games))

const INITIAL_TAG_COUNT = 6
const visibleTags = computed(() =>
  tagCounts.value.filter(([tag], index) => index < INITIAL_TAG_COUNT || state.value.tags.includes(tag))
)
const hiddenTags = computed(() => {
  const shown = new Set(visibleTags.value.map(([tag]) => tag))
  return tagCounts.value.filter(([tag]) => !shown.has(tag))
})

const morePanelId = useId()
const moreButton = ref<HTMLButtonElement | null>(null)
const morePanel = ref<HTMLDivElement | null>(null)
const moreOpen = ref(false)

function positionMorePanel(): void {
  const button = moreButton.value
  const panel = morePanel.value
  if (!button || !panel) return
  const gap = 4
  const rect = button.getBoundingClientRect()
  const panelRect = panel.getBoundingClientRect()
  const openUp = window.innerHeight - rect.bottom - gap < panelRect.height && rect.top > window.innerHeight - rect.bottom
  panel.style.left = `${Math.max(gap, Math.min(rect.left, window.innerWidth - panelRect.width - gap))}px`
  panel.style.top = `${openUp ? rect.top - panelRect.height - gap : rect.bottom + gap}px`
}

function closeMore(): void {
  const panel = morePanel.value
  if (panel?.matches(':popover-open')) panel.hidePopover()
}

function toggleMore(): void {
  const panel = morePanel.value
  if (!panel) return
  if (panel.matches(':popover-open')) panel.hidePopover()
  else panel.showPopover()
}

function onMoreToggle(event: Event): void {
  const open = (event as Event & { newState?: string }).newState === 'open'
  moreOpen.value = open
  if (open) {
    positionMorePanel()
    window.addEventListener('scroll', closeMore, true)
    window.addEventListener('resize', closeMore)
  } else {
    window.removeEventListener('scroll', closeMore, true)
    window.removeEventListener('resize', closeMore)
  }
}

watch(hiddenTags, (tags) => {
  if (!tags.length) closeMore()
})

onBeforeUnmount(() => {
  window.removeEventListener('scroll', closeMore, true)
  window.removeEventListener('resize', closeMore)
})

interface Row<K extends string> {
  key: K
  label: string
  count: number
}

const typeRows = computed<Array<Row<GameType | 'all'>>>(() => [
  { key: 'all', label: '全部', count: props.games.length },
  ...GAME_TYPES.filter((type) => typeCounts.value[type] > 0).map((type) => ({
    key: type as GameType | 'all',
    label: GAME_TYPE_LABELS[type],
    count: typeCounts.value[type]
  }))
])

const durationRows = computed<Array<Row<DurationBucket | 'all'>>>(() => {
  const options: Array<{ value: DurationBucket | 'all'; label: string }> = [
    { value: 'all', label: '全部时长' },
    { value: 'short', label: '≤5 分钟' },
    { value: 'mid', label: '5–30 分钟' },
    { value: 'long', label: '>30 分钟' }
  ]
  return options
    .filter((option) => option.value === 'all' || durationCounts.value[option.value as DurationBucket] > 0)
    .map((option) => ({
      key: option.value,
      label: option.label,
      count: option.value === 'all' ? props.games.length : durationCounts.value[option.value as DurationBucket]
    }))
})

function toggleTag(tag: string): void {
  const tags = state.value.tags.includes(tag)
    ? state.value.tags.filter((t) => t !== tag)
    : [...state.value.tags, tag]
  update({ tags })
}
</script>

<template>
  <div class="space-y-6">
    <h2 class="sr-only">筛选</h2>
    <section>
      <h3 class="text-[0.625rem] font-extrabold tracking-[0.22em] text-accent-ink">类型</h3>
      <ul class="mt-2">
        <li
          v-for="row in typeRows"
          :key="row.key"
          class="border-b-[1.5px] border-dashed"
          :class="state.type === row.key ? 'border-transparent' : 'border-ink'"
        >
          <button
            type="button"
            class="flex min-h-11 w-full items-center justify-between gap-2 border-2 px-2 text-left text-[0.8125rem] lg:min-h-9"
            :class="state.type === row.key ? 'border-ink bg-highlight font-bold' : 'border-transparent hover:bg-paper'"
            :aria-pressed="state.type === row.key"
            @click="update({ type: row.key })"
          >
            <span class="truncate">{{ row.label }}</span>
            <span class="font-mono text-[0.6875rem]">{{ row.count }}</span>
          </button>
        </li>
      </ul>
    </section>

    <section>
      <h3 class="text-[0.625rem] font-extrabold tracking-[0.22em] text-accent-ink">时长</h3>
      <ul class="mt-2">
        <li
          v-for="row in durationRows"
          :key="row.key"
          class="border-b-[1.5px] border-dashed"
          :class="state.dur === row.key ? 'border-transparent' : 'border-ink'"
        >
          <button
            type="button"
            class="flex min-h-11 w-full items-center justify-between gap-2 border-2 px-2 text-left text-[0.8125rem] lg:min-h-9"
            :class="state.dur === row.key ? 'border-ink bg-highlight font-bold' : 'border-transparent hover:bg-paper'"
            :aria-pressed="state.dur === row.key"
            @click="update({ dur: row.key })"
          >
            <span class="truncate">{{ row.label }}</span>
            <span class="font-mono text-[0.6875rem]">{{ row.count }}</span>
          </button>
        </li>
      </ul>
    </section>

    <section v-if="tagCounts.length">
      <h3 class="text-[0.625rem] font-extrabold tracking-[0.22em] text-accent-ink">标签</h3>
      <ul class="mt-2">
        <li
          class="border-b-[1.5px] border-dashed"
          :class="state.tags.length === 0 ? 'border-transparent' : 'border-ink'"
        >
          <button
            type="button"
            class="flex min-h-11 w-full items-center justify-between gap-2 border-2 px-2 text-left text-[0.8125rem] lg:min-h-9"
            :class="state.tags.length === 0 ? 'border-ink bg-highlight font-bold' : 'border-transparent hover:bg-paper'"
            :aria-pressed="state.tags.length === 0"
            @click="update({ tags: [] })"
          >
            <span class="truncate">全部</span>
            <span class="font-mono text-[0.6875rem]">{{ games.length }}</span>
          </button>
        </li>
        <li
          v-for="[tag, tagCount] in visibleTags"
          :key="tag"
          class="border-b-[1.5px] border-dashed"
          :class="state.tags.includes(tag) ? 'border-transparent' : 'border-ink'"
        >
          <button
            type="button"
            class="flex min-h-11 w-full items-center justify-between gap-2 border-2 px-2 text-left text-[0.8125rem] lg:min-h-9"
            :class="state.tags.includes(tag) ? 'border-ink bg-highlight font-bold' : 'border-transparent hover:bg-paper'"
            :aria-pressed="state.tags.includes(tag)"
            @click="toggleTag(tag)"
          >
            <span class="truncate">{{ tag }}</span>
            <span class="font-mono text-[0.6875rem]">{{ tagCount }}</span>
          </button>
        </li>
      </ul>
      <button
        v-if="hiddenTags.length"
        ref="moreButton"
        type="button"
        class="mt-2 flex min-h-11 w-full items-center justify-center border-2 border-transparent px-2 text-[0.8125rem] font-bold text-accent-ink hover:bg-paper lg:min-h-9"
        :aria-expanded="moreOpen"
        :aria-controls="morePanelId"
        @click="toggleMore"
      >展开更多（{{ hiddenTags.length }}）</button>
      <div
        :id="morePanelId"
        ref="morePanel"
        popover="auto"
        class="m-0 max-h-64 w-48 max-w-[calc(100vw-2rem)] overflow-y-auto border-2 border-ink bg-surface p-0 shadow-hard"
        :class="moreOpen ? '' : 'invisible'"
        @toggle="onMoreToggle"
      >
        <ul>
          <li
            v-for="[tag, tagCount] in hiddenTags"
            :key="tag"
            class="border-b-[1.5px] border-dashed last:border-b-0"
            :class="state.tags.includes(tag) ? 'border-transparent' : 'border-ink'"
          >
            <button
              type="button"
              class="flex min-h-11 w-full items-center justify-between gap-2 border-2 px-2 text-left text-[0.8125rem] lg:min-h-9"
              :class="state.tags.includes(tag) ? 'border-ink bg-highlight font-bold' : 'border-transparent hover:bg-paper'"
              :aria-pressed="state.tags.includes(tag)"
              @click="toggleTag(tag)"
            >
              <span class="truncate">{{ tag }}</span>
              <span class="font-mono text-[0.6875rem]">{{ tagCount }}</span>
            </button>
          </li>
        </ul>
      </div>
    </section>

    <button
      type="button"
      class="lift w-full border-2 border-ink bg-surface px-3 py-2 text-sm font-extrabold shadow-hard-sm hover:shadow-hard active:shadow-none"
      @click="update(DEFAULT_FILTER)"
    >重置筛选</button>
  </div>
</template>
