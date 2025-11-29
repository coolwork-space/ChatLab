<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import type { HourlyActivity, WeekdayActivity, NightOwlAnalysis } from '@/types/chat'
import { BarChart, RankListPro } from '@/components/charts'
import type { BarChartData } from '@/components/charts'
import { SectionCard, StatCard } from '@/components/UI'

const props = defineProps<{
  sessionId: string
  hourlyActivity: HourlyActivity[]
  timeFilter?: { startTs?: number; endTs?: number }
}>()

// 星期活跃度数据
const weekdayActivity = ref<WeekdayActivity[]>([])
const isLoadingWeekday = ref(false)

// 夜猫分析数据
const nightOwlData = ref<NightOwlAnalysis | null>(null)
const isLoadingNightOwl = ref(false)

// 星期名称映射（周一开始）
const weekdayNames = ['周一', '周二', '周三', '周四', '周五', '周六', '周日']

// 称号颜色映射
const titleColors: Record<string, string> = {
  养生达人: 'text-green-600 dark:text-green-400',
  偶尔失眠: 'text-blue-600 dark:text-blue-400',
  夜猫子: 'text-yellow-600 dark:text-yellow-400',
  秃头预备役: 'text-orange-600 dark:text-orange-400',
  修仙练习生: 'text-pink-600 dark:text-pink-400',
  守夜冠军: 'text-purple-600 dark:text-purple-400',
  不睡觉の神: 'text-red-600 dark:text-red-400',
}

// 加载星期活跃度数据
async function loadWeekdayActivity() {
  if (!props.sessionId) return
  isLoadingWeekday.value = true
  try {
    weekdayActivity.value = await window.chatApi.getWeekdayActivity(props.sessionId, props.timeFilter)
  } catch (error) {
    console.error('加载星期活跃度失败:', error)
  } finally {
    isLoadingWeekday.value = false
  }
}

// 加载夜猫分析数据
async function loadNightOwlAnalysis() {
  if (!props.sessionId) return
  isLoadingNightOwl.value = true
  try {
    nightOwlData.value = await window.chatApi.getNightOwlAnalysis(props.sessionId, props.timeFilter)
  } catch (error) {
    console.error('加载夜猫分析失败:', error)
  } finally {
    isLoadingNightOwl.value = false
  }
}

// 监听 sessionId 和 timeFilter 变化
watch(
  () => [props.sessionId, props.timeFilter],
  () => {
    loadWeekdayActivity()
    loadNightOwlAnalysis()
  },
  { immediate: true, deep: true }
)

// 24小时分布图数据
const hourlyChartData = computed<BarChartData>(() => {
  return {
    labels: props.hourlyActivity.map((h) => `${h.hour}:00`),
    values: props.hourlyActivity.map((h) => h.messageCount),
  }
})

// 星期分布图数据
const weekdayChartData = computed<BarChartData>(() => {
  return {
    labels: weekdayActivity.value.map((w) => weekdayNames[w.weekday - 1]),
    values: weekdayActivity.value.map((w) => w.messageCount),
  }
})

// 分析指标
const peakHour = computed(() => {
  if (!props.hourlyActivity.length) return null
  return props.hourlyActivity.reduce((max, h) => (h.messageCount > max.messageCount ? h : max), props.hourlyActivity[0])
})

const peakWeekday = computed(() => {
  if (!weekdayActivity.value.length) return null
  return weekdayActivity.value.reduce(
    (max, w) => (w.messageCount > max.messageCount ? w : max),
    weekdayActivity.value[0]
  )
})

// 时段占比计算
const totalMessages = computed(() => props.hourlyActivity.reduce((sum, h) => sum + h.messageCount, 0))

const lateNightRatio = computed(() => {
  const lateNight = props.hourlyActivity
    .filter((h) => h.hour >= 0 && h.hour < 6)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((lateNight / totalMessages.value) * 100) : 0
})

const morningRatio = computed(() => {
  const morning = props.hourlyActivity
    .filter((h) => h.hour >= 6 && h.hour < 12)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((morning / totalMessages.value) * 100) : 0
})

const afternoonRatio = computed(() => {
  const afternoon = props.hourlyActivity
    .filter((h) => h.hour >= 12 && h.hour < 18)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((afternoon / totalMessages.value) * 100) : 0
})

const eveningRatio = computed(() => {
  const evening = props.hourlyActivity
    .filter((h) => h.hour >= 18 && h.hour < 24)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((evening / totalMessages.value) * 100) : 0
})

const weekdayVsWeekend = computed(() => {
  if (!weekdayActivity.value.length) return { weekday: 0, weekend: 0 }
  const weekdaySum = weekdayActivity.value
    .filter((w) => w.weekday >= 1 && w.weekday <= 5)
    .reduce((sum, w) => sum + w.messageCount, 0)
  const weekendSum = weekdayActivity.value
    .filter((w) => w.weekday >= 6 && w.weekday <= 7)
    .reduce((sum, w) => sum + w.messageCount, 0)
  const total = weekdaySum + weekendSum
  return {
    weekday: total > 0 ? Math.round((weekdaySum / total) * 100) : 0,
    weekend: total > 0 ? Math.round((weekendSum / total) * 100) : 0,
  }
})

// 最晚下班排行
const lastSpeakerMembers = computed(() => {
  if (!nightOwlData.value) return []
  return nightOwlData.value.lastSpeakerRank.map((item) => ({
    id: String(item.memberId),
    name: item.name,
    value: item.count,
    percentage: item.percentage,
  }))
})

// 最早上班排行
const firstSpeakerMembers = computed(() => {
  if (!nightOwlData.value) return []
  return nightOwlData.value.firstSpeakerRank.map((item) => ({
    id: String(item.memberId),
    name: item.name,
    value: item.count,
    percentage: item.percentage,
  }))
})
</script>

<template>
  <div class="space-y-6">
    <!-- 标题 -->
    <div>
      <h2 class="text-xl font-bold text-gray-900 dark:text-white">时间规律分析</h2>
      <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">发现群聊的周期性活跃规律</p>
    </div>

    <!-- 统计卡片 -->
    <div class="grid grid-cols-2 gap-4 lg:grid-cols-4">
      <StatCard
        label="最活跃时段"
        :value="`${peakHour?.hour ?? 0}:00`"
        :subtext="`${peakHour?.messageCount ?? 0} 条消息`"
      />
      <StatCard
        label="最活跃星期"
        :value="peakWeekday ? weekdayNames[peakWeekday.weekday - 1] : '-'"
        :subtext="`${peakWeekday?.messageCount ?? 0} 条消息`"
      />
      <StatCard label="夜猫子指数" :value="`${lateNightRatio}%`" value-color="amber" subtext="深夜活跃占比" />
      <StatCard label="周末活跃度" :value="`${weekdayVsWeekend.weekend}%`" value-color="blue" subtext="周末消息占比" />
    </div>

    <!-- 星期分布 -->
    <SectionCard title="星期活跃分布" :show-divider="false">
      <div class="p-5">
        <div v-if="isLoadingWeekday" class="flex h-64 items-center justify-center">
          <UIcon name="i-heroicons-arrow-path" class="h-6 w-6 animate-spin text-pink-500" />
        </div>
        <BarChart v-else :data="weekdayChartData" :height="256" />

        <div class="mt-6 grid grid-cols-2 gap-4">
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">工作日（周一至周五）</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ weekdayVsWeekend.weekday }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-24 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div
                class="h-full rounded-full bg-pink-500 transition-all"
                :style="{ width: `${weekdayVsWeekend.weekday}%` }"
              />
            </div>
          </div>
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">周末（周六、周日）</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ weekdayVsWeekend.weekend }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-24 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div
                class="h-full rounded-full bg-blue-500 transition-all"
                :style="{ width: `${weekdayVsWeekend.weekend}%` }"
              />
            </div>
          </div>
        </div>
      </div>
    </SectionCard>

    <!-- 24小时分布 -->
    <SectionCard title="24小时活跃分布" :show-divider="false">
      <div class="p-5">
        <BarChart
          :data="hourlyChartData"
          :height="256"
          :x-label-filter="(_, index) => (index % 3 === 0 ? `${index}:00` : '')"
        />

        <div class="mt-6 grid grid-cols-4 gap-4">
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">凌晨 0-6点</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ lateNightRatio }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-16 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div class="h-full rounded-full bg-pink-300 transition-all" :style="{ width: `${lateNightRatio}%` }" />
            </div>
          </div>
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">上午 6-12点</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ morningRatio }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-16 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div class="h-full rounded-full bg-pink-400 transition-all" :style="{ width: `${morningRatio}%` }" />
            </div>
          </div>
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">下午 12-18点</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ afternoonRatio }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-16 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div class="h-full rounded-full bg-pink-500 transition-all" :style="{ width: `${afternoonRatio}%` }" />
            </div>
          </div>
          <div class="text-center">
            <div class="text-xs text-gray-500 dark:text-gray-400">晚上 18-24点</div>
            <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">{{ eveningRatio }}%</div>
            <div class="mx-auto mt-2 h-1 w-full max-w-16 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div class="h-full rounded-full bg-pink-600 transition-all" :style="{ width: `${eveningRatio}%` }" />
            </div>
          </div>
        </div>
      </div>
    </SectionCard>

    <!-- 夜猫分析模块 -->
    <SectionCard title="🦉 夜猫分析" :show-divider="false">
      <template #headerRight>
        <span class="text-xs text-gray-400">深夜时段 23:00 - 05:00</span>
      </template>

      <div class="p-5">
        <div v-if="isLoadingNightOwl" class="flex h-32 items-center justify-center">
          <UIcon name="i-heroicons-arrow-path" class="h-6 w-6 animate-spin text-pink-500" />
        </div>

        <template v-else-if="nightOwlData">
          <!-- 修仙王者 TOP 3 -->
          <div v-if="nightOwlData.champions.length > 0" class="mb-6">
            <h4 class="mb-3 text-sm font-medium text-gray-700 dark:text-gray-300">🏆 修仙王者</h4>
            <div class="grid gap-3 sm:grid-cols-3">
              <div
                v-for="(champion, index) in nightOwlData.champions.slice(0, 3)"
                :key="champion.memberId"
                class="relative overflow-hidden rounded-lg p-4"
                :class="[
                  index === 0
                    ? 'bg-gradient-to-br from-amber-50 to-orange-100 dark:from-amber-900/20 dark:to-orange-900/20'
                    : index === 1
                      ? 'bg-gradient-to-br from-gray-50 to-slate-100 dark:from-gray-800/50 dark:to-slate-800/50'
                      : 'bg-gradient-to-br from-orange-50 to-amber-100 dark:from-orange-900/10 dark:to-amber-900/10',
                ]"
              >
                <div class="absolute right-2 top-2 text-3xl opacity-20">
                  {{ index === 0 ? '🥇' : index === 1 ? '🥈' : '🥉' }}
                </div>
                <div class="text-lg font-bold text-gray-900 dark:text-white">{{ champion.name }}</div>
                <div class="mt-1 text-xs text-gray-500 dark:text-gray-400">综合得分 {{ champion.score }}</div>
                <div class="mt-2 space-y-1 text-xs text-gray-600 dark:text-gray-300">
                  <div>🌙 深夜发言 {{ champion.nightMessages }} 条</div>
                  <div>🔚 最晚下班 {{ champion.lastSpeakerCount }} 次</div>
                  <div>🔥 连续修仙 {{ champion.consecutiveDays }} 天</div>
                </div>
              </div>
            </div>
          </div>

          <!-- 修仙排行榜 -->
          <div class="mb-6">
            <h4 class="mb-3 text-sm font-medium text-gray-700 dark:text-gray-300">🌙 修仙排行榜</h4>
            <div v-if="nightOwlData.nightOwlRank.length > 0" class="space-y-2">
              <div
                v-for="(item, index) in nightOwlData.nightOwlRank.slice(0, 10)"
                :key="item.memberId"
                class="flex items-center gap-3 rounded-lg bg-gray-50 p-3 dark:bg-gray-800/50"
              >
                <span class="w-6 text-center text-sm font-bold text-gray-400">{{ index + 1 }}</span>
                <div class="flex-1">
                  <div class="flex items-center gap-2">
                    <span class="font-medium text-gray-900 dark:text-white">{{ item.name }}</span>
                    <span class="rounded-full px-2 py-0.5 text-xs font-medium" :class="titleColors[item.title]">
                      {{ item.title }}
                    </span>
                  </div>
                  <div class="mt-1 flex gap-3 text-xs text-gray-500 dark:text-gray-400">
                    <span>共 {{ item.totalNightMessages }} 条</span>
                    <span>23点:{{ item.hourlyBreakdown.h23 }}</span>
                    <span>0点:{{ item.hourlyBreakdown.h0 }}</span>
                    <span>1点:{{ item.hourlyBreakdown.h1 }}</span>
                    <span>2点:{{ item.hourlyBreakdown.h2 }}</span>
                    <span>3-4点:{{ item.hourlyBreakdown.h3to4 }}</span>
                  </div>
                </div>
                <span class="text-sm font-semibold text-pink-600 dark:text-pink-400">{{ item.percentage }}%</span>
              </div>
            </div>
            <div v-else class="py-8 text-center text-sm text-gray-400">暂无深夜发言数据</div>
          </div>

          <!-- 最晚下班 & 最早上班 -->
          <div class="grid gap-6 lg:grid-cols-2">
            <!-- 最晚下班排名 -->
            <div>
              <RankListPro
                v-if="lastSpeakerMembers.length > 0"
                :members="lastSpeakerMembers"
                title="🔚 最晚下班排名"
                :description="`每天最后一个发言的人（共 ${nightOwlData.totalDays} 天）`"
                unit="次"
              />
              <div v-else class="py-4 text-center text-sm text-gray-400">暂无数据</div>
            </div>

            <!-- 最早上班排名 -->
            <div>
              <RankListPro
                v-if="firstSpeakerMembers.length > 0"
                :members="firstSpeakerMembers"
                title="🌅 最早上班排名"
                :description="`每天第一个发言的人（共 ${nightOwlData.totalDays} 天）`"
                unit="次"
              />
              <div v-else class="py-4 text-center text-sm text-gray-400">暂无数据</div>
            </div>
          </div>

          <!-- 连续修仙记录 -->
          <div v-if="nightOwlData.consecutiveRecords.length > 0" class="mt-6">
            <h4 class="mb-3 text-sm font-medium text-gray-700 dark:text-gray-300">🔥 连续修仙记录</h4>
            <div class="grid gap-2 sm:grid-cols-2 lg:grid-cols-3">
              <div
                v-for="record in nightOwlData.consecutiveRecords.slice(0, 6)"
                :key="record.memberId"
                class="flex items-center justify-between rounded-lg bg-gray-50 p-3 dark:bg-gray-800/50"
              >
                <span class="font-medium text-gray-900 dark:text-white">{{ record.name }}</span>
                <div class="text-right">
                  <div class="text-lg font-bold text-pink-600 dark:text-pink-400">
                    {{ record.maxConsecutiveDays }} 天
                  </div>
                  <div v-if="record.currentStreak > 0" class="text-xs text-green-600 dark:text-green-400">
                    当前连续 {{ record.currentStreak }} 天
                  </div>
                </div>
              </div>
            </div>
          </div>
        </template>
      </div>
    </SectionCard>
  </div>
</template>
