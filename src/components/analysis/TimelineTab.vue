<script setup lang="ts">
import { computed } from 'vue'
import type { DailyActivity, DivingAnalysis } from '@/types/chat'
import dayjs from 'dayjs'
import { LineChart } from '@/components/charts'
import type { LineChartData } from '@/components/charts'
import { SectionCard, StatCard, EmptyState, LoadingState } from '@/components/UI'
import { useAsyncData } from '@/composables'
import { formatFullDateTime, formatDaysSince } from '@/utils'

interface TimeFilter {
  startTs?: number
  endTs?: number
}

const props = defineProps<{
  sessionId: string
  dailyActivity: DailyActivity[]
  timeRange: { start: number; end: number } | null
  timeFilter?: TimeFilter
}>()

// 检测是否跨年
const isMultiYear = computed(() => {
  if (props.dailyActivity.length < 2) return false
  const years = new Set(props.dailyActivity.map((d) => dayjs(d.date).year()))
  return years.size > 1
})

// 每日趋势图数据
const dailyChartData = computed<LineChartData>(() => {
  const dateFormat = isMultiYear.value ? 'YYYY/MM/DD' : 'MM/DD'
  return {
    labels: props.dailyActivity.map((d) => dayjs(d.date).format(dateFormat)),
    values: props.dailyActivity.map((d) => d.messageCount),
  }
})

// 最活跃的一天
const peakDay = computed(() => {
  if (!props.dailyActivity.length) return null
  return props.dailyActivity.reduce((max, d) => (d.messageCount > max.messageCount ? d : max), props.dailyActivity[0])
})

// 平均每日消息数
const avgDailyMessages = computed(() => {
  if (!props.dailyActivity.length) return 0
  const total = props.dailyActivity.reduce((sum, d) => sum + d.messageCount, 0)
  return Math.round(total / props.dailyActivity.length)
})

// 活跃天数
const activeDays = computed(() => {
  return props.dailyActivity.filter((d) => d.messageCount > 0).length
})

// 总天数
const totalDays = computed(() => {
  if (!props.timeRange) return 0
  const start = dayjs.unix(props.timeRange.start)
  const end = dayjs.unix(props.timeRange.end)
  return end.diff(start, 'day') + 1
})

// 活跃率
const activeRate = computed(() => {
  return totalDays.value > 0 ? Math.round((activeDays.value / totalDays.value) * 100) : 0
})

// ==================== 潜水分析 ====================
const sessionIdRef = computed(() => props.sessionId)
const timeFilterRef = computed(() => props.timeFilter)

const { data: divingAnalysis, isLoading: isLoadingDiving } = useAsyncData<DivingAnalysis>(
  (sessionId, filter) => window.chatApi.getDivingAnalysis(sessionId, filter),
  sessionIdRef,
  timeFilterRef
)
</script>

<template>
  <div class="space-y-6">
    <!-- 标题 -->
    <div>
      <h2 class="text-xl font-bold text-gray-900 dark:text-white">时间轴分析</h2>
      <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">追踪群聊的活跃趋势变化</p>
    </div>

    <!-- 统计卡片 -->
    <div class="grid grid-cols-2 gap-4 lg:grid-cols-4">
      <StatCard
        label="最活跃日期"
        :value="peakDay ? dayjs(peakDay.date).format('MM/DD') : '-'"
        :subtext="`${peakDay?.messageCount ?? 0} 条消息`"
      />
      <StatCard label="日均消息" :value="avgDailyMessages" subtext="条/天" />
      <StatCard label="活跃天数" :value="activeDays" :subtext="`/ ${totalDays} 天`" />
      <StatCard label="活跃率" :value="`${activeRate}%`" subtext="有消息的天数占比" />
    </div>

    <!-- 每日趋势 -->
    <SectionCard title="每日消息趋势" :show-divider="false">
      <div class="p-5">
        <LineChart :data="dailyChartData" :height="288" />
      </div>
    </SectionCard>

    <!-- 潜水排名 -->
    <SectionCard title="🤿 潜水排名" description="按最后发言时间排序，最久没发言的在前面">
      <LoadingState v-if="isLoadingDiving" text="正在统计潜水数据..." />

      <div
        v-else-if="divingAnalysis && divingAnalysis.rank.length > 0"
        class="divide-y divide-gray-100 dark:divide-gray-800"
      >
        <div
          v-for="(member, index) in divingAnalysis.rank"
          :key="member.memberId"
          class="flex items-center gap-3 px-5 py-3 transition-colors hover:bg-gray-50 dark:hover:bg-gray-800/50"
        >
          <!-- 排名 -->
          <div
            class="flex h-8 w-8 shrink-0 items-center justify-center rounded-full text-sm font-bold"
            :class="
              index === 0
                ? 'bg-gradient-to-r from-blue-400 to-cyan-500 text-white'
                : index === 1
                  ? 'bg-gradient-to-r from-blue-300 to-cyan-400 text-white'
                  : index === 2
                    ? 'bg-gradient-to-r from-blue-200 to-cyan-300 text-gray-700'
                    : 'bg-gray-100 text-gray-600 dark:bg-gray-800 dark:text-gray-400'
            "
          >
            {{ index + 1 }}
          </div>

          <!-- 名字 -->
          <div class="w-32 shrink-0">
            <p class="truncate font-medium text-gray-900 dark:text-white">
              {{ member.name }}
            </p>
          </div>

          <!-- 最后发言时间 -->
          <div class="flex flex-1 items-center gap-2">
            <span class="text-sm text-gray-600 dark:text-gray-400">
              {{ formatFullDateTime(member.lastMessageTs) }}
            </span>
          </div>

          <!-- 距今天数 -->
          <div class="shrink-0 text-right">
            <span
              class="text-sm font-medium"
              :class="
                member.daysSinceLastMessage > 30
                  ? 'text-red-600 dark:text-red-400'
                  : member.daysSinceLastMessage > 7
                    ? 'text-orange-600 dark:text-orange-400'
                    : 'text-gray-600 dark:text-gray-400'
              "
            >
              {{ formatDaysSince(member.daysSinceLastMessage) }}
            </span>
          </div>
        </div>
      </div>

      <EmptyState v-else />
    </SectionCard>
  </div>
</template>
