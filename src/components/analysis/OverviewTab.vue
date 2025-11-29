<script setup lang="ts">
import { computed } from 'vue'
import type { AnalysisSession, MemberActivity, HourlyActivity, MessageType } from '@/types/chat'
import { getMessageTypeName } from '@/types/chat'
import { DoughnutChart, ProgressBar } from '@/components/charts'
import type { DoughnutChartData } from '@/components/charts'
import { SectionCard, StatCard } from '@/components/UI'

const props = defineProps<{
  session: AnalysisSession
  memberActivity: MemberActivity[]
  topMembers: MemberActivity[]
  bottomMembers: MemberActivity[]
  messageTypes: Array<{ type: MessageType; count: number }>
  hourlyActivity: HourlyActivity[]
  timeRange: { start: number; end: number } | null
  selectedYear: number | null
  filteredMessageCount: number
  filteredMemberCount: number
}>()

// 时间跨度
const durationDays = computed(() => {
  if (props.selectedYear) {
    const isLeapYear =
      (props.selectedYear % 4 === 0 && props.selectedYear % 100 !== 0) || props.selectedYear % 400 === 0
    return isLeapYear ? 366 : 365
  }
  if (!props.timeRange) return 0
  return Math.ceil((props.timeRange.end - props.timeRange.start) / 86400)
})

// 显示的消息数和成员数
const displayMessageCount = computed(() => {
  return props.selectedYear ? props.filteredMessageCount : props.session.messageCount
})

const displayMemberCount = computed(() => {
  return props.selectedYear ? props.filteredMemberCount : props.session.memberCount
})

// 消息类型图表数据
const typeChartData = computed<DoughnutChartData>(() => {
  return {
    labels: props.messageTypes.map((t) => getMessageTypeName(t.type)),
    values: props.messageTypes.map((t) => t.count),
  }
})

// 最活跃时段
const peakHour = computed(() => {
  if (!props.hourlyActivity.length) return null
  const peak = props.hourlyActivity.reduce(
    (max, h) => (h.messageCount > max.messageCount ? h : max),
    props.hourlyActivity[0]
  )
  return peak
})

// 图片消息数量
const imageCount = computed(() => {
  const imageType = props.messageTypes.find((t) => t.type === 1)
  return imageType?.count || 0
})

// 日均消息数
const dailyAvgMessages = computed(() => {
  if (durationDays.value === 0) return 0
  return Math.round(displayMessageCount.value / durationDays.value)
})

// 获取排名徽章
function getRankBadge(index: number): string {
  const badges = ['🥇', '🥈', '🥉']
  return badges[index] || `${index + 1}`
}
</script>

<template>
  <div class="space-y-6">
    <!-- 群聊身份卡 -->
    <div class="rounded-2xl bg-linear-to-br from-pink-400 via-pink-500 to-pink-600 p-6 text-white shadow-lg">
      <div class="flex items-start justify-between">
        <div>
          <h2 class="text-2xl font-bold">{{ session.name }}</h2>
          <p class="mt-1 text-white/80">
            平台: {{ session.platform.toUpperCase() }} · {{ session.memberCount > 2 ? '群聊' : '私聊' }}
          </p>
        </div>
        <div class="flex h-12 w-12 items-center justify-center rounded-xl bg-white/20 backdrop-blur">
          <UIcon name="i-heroicons-chat-bubble-left-right" class="h-6 w-6" />
        </div>
      </div>

      <div class="mt-6 grid grid-cols-3 gap-4">
        <div class="rounded-xl bg-white/10 px-4 py-3 backdrop-blur">
          <p class="text-2xl font-bold">{{ displayMessageCount }}</p>
          <p class="text-sm text-white/70">{{ selectedYear ? '筛选消息' : '消息总数' }}</p>
        </div>
        <div class="rounded-xl bg-white/10 px-4 py-3 backdrop-blur">
          <p class="text-2xl font-bold">{{ displayMemberCount }}</p>
          <p class="text-sm text-white/70">{{ selectedYear ? '活跃成员' : '成员' }}</p>
        </div>
        <div class="rounded-xl bg-white/10 px-4 py-3 backdrop-blur">
          <p class="text-2xl font-bold">{{ durationDays }}</p>
          <p class="text-sm text-white/70">天</p>
        </div>
      </div>
    </div>

    <!-- 关键指标卡片 -->
    <div class="grid grid-cols-1 gap-4 sm:grid-cols-3">
      <!-- 龙王 -->
      <StatCard label="龙王" :value="topMembers[0]?.name || '-'" icon="🏆" icon-bg="amber">
        <template #subtext>
          <span class="text-2xl font-bold text-amber-500">{{ topMembers[0]?.messageCount || 0 }}</span>
          <span class="text-sm text-gray-500">条</span>
          <span class="ml-2 text-sm text-gray-400">({{ topMembers[0]?.percentage || 0 }}%)</span>
        </template>
      </StatCard>

      <!-- 日均消息 -->
      <StatCard label="日均消息" :value="`${dailyAvgMessages} 条`" icon="📊" icon-bg="blue">
        <template #subtext>
          <span class="text-sm text-gray-500">共 {{ durationDays }} 天</span>
        </template>
      </StatCard>

      <!-- 图片/表情 -->
      <StatCard label="图片消息" :value="`${imageCount} 张`" icon="📸" icon-bg="pink">
        <template #subtext>
          <span class="text-sm text-gray-500">最活跃时段:</span>
          <span class="font-semibold text-pink-500">{{ peakHour?.hour || 0 }}:00</span>
        </template>
      </StatCard>
    </div>

    <!-- 图表区域 -->
    <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
      <!-- 消息类型分布 -->
      <SectionCard title="消息类型分布" :show-divider="false">
        <div class="p-5">
          <DoughnutChart :data="typeChartData" :height="256" />
        </div>
      </SectionCard>

      <!-- Top 成员预览 -->
      <SectionCard title="活跃榜 Top 5" :show-divider="false">
        <div class="space-y-3 p-5">
          <div
            v-for="(member, index) in memberActivity.slice(0, 5)"
            :key="member.memberId"
            class="flex items-center gap-3"
          >
            <span class="w-6 text-center text-lg">{{ getRankBadge(index) }}</span>
            <div class="flex-1">
              <div class="flex items-center justify-between">
                <span class="font-medium text-gray-900 dark:text-white">{{ member.name }}</span>
                <span class="text-sm text-gray-500">{{ member.messageCount }}</span>
              </div>
              <ProgressBar :percentage="member.percentage" color="from-pink-400 to-pink-600" />
            </div>
          </div>
        </div>
      </SectionCard>
    </div>
  </div>
</template>
