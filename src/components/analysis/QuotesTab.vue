<script setup lang="ts">
import { ref, watch } from 'vue'
import type { CatchphraseAnalysis } from '@/types/chat'
import { ListPro } from '@/components/charts'
import { SectionCard, EmptyState, LoadingState } from '@/components/UI'
import { KeywordAnalysis } from './quotes'

interface TimeFilter {
  startTs?: number
  endTs?: number
}

const props = defineProps<{
  sessionId: string
  timeFilter?: TimeFilter
}>()

// ==================== 口头禅分析 ====================
const catchphraseAnalysis = ref<CatchphraseAnalysis | null>(null)
const isLoadingCatchphrase = ref(false)

async function loadCatchphraseAnalysis() {
  if (!props.sessionId) return
  isLoadingCatchphrase.value = true
  try {
    catchphraseAnalysis.value = await window.chatApi.getCatchphraseAnalysis(props.sessionId, props.timeFilter)
  } catch (error) {
    console.error('加载口头禅分析失败:', error)
  } finally {
    isLoadingCatchphrase.value = false
  }
}

function truncateContent(content: string, maxLength = 30): string {
  if (content.length <= maxLength) return content
  return content.slice(0, maxLength) + '...'
}

// 监听 sessionId 和 timeFilter 变化
watch(
  () => [props.sessionId, props.timeFilter],
  () => {
    loadCatchphraseAnalysis()
  },
  { immediate: true, deep: true }
)
</script>

<template>
  <div class="space-y-6">
    <!-- 口头禅分析模块 -->
    <LoadingState v-if="isLoadingCatchphrase" text="正在分析口头禅数据..." />

    <ListPro
      v-else-if="catchphraseAnalysis && catchphraseAnalysis.members.length > 0"
      :items="catchphraseAnalysis.members"
      title="💬 口头禅分析"
      :description="`分析了 ${catchphraseAnalysis.members.length} 位成员的高频发言`"
      countTemplate="共 {count} 位成员"
    >
      <template #item="{ item: member }">
        <div class="flex items-start gap-4">
          <div class="w-28 shrink-0 pt-1 font-medium text-gray-900 dark:text-white">
            {{ member.name }}
          </div>

          <div class="flex flex-1 flex-wrap items-center gap-2">
            <div
              v-for="(phrase, index) in member.catchphrases"
              :key="index"
              class="flex items-center gap-1.5 rounded-lg px-3 py-1.5"
              :class="
                index === 0
                  ? 'bg-amber-50 dark:bg-amber-900/20'
                  : index === 1
                    ? 'bg-gray-100 dark:bg-gray-800'
                    : 'bg-gray-50 dark:bg-gray-800/50'
              "
            >
              <span
                class="text-sm"
                :class="
                  index === 0 ? 'font-medium text-amber-700 dark:text-amber-400' : 'text-gray-700 dark:text-gray-300'
                "
                :title="phrase.content"
              >
                {{ truncateContent(phrase.content, 20) }}
              </span>
              <span class="text-xs text-gray-400">{{ phrase.count }}次</span>
            </div>
          </div>
        </div>
      </template>
    </ListPro>

    <SectionCard v-else title="💬 口头禅分析">
      <EmptyState text="暂无口头禅数据" />
    </SectionCard>

    <!-- 关键词分析 -->
    <KeywordAnalysis :session-id="sessionId" :time-filter="timeFilter" />
  </div>
</template>
