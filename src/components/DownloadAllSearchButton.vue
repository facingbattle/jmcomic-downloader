<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useStore } from '../store.ts'
import { commands, DownloadAllSearchEvent, events, SearchSort } from '../bindings.ts'
import { MessageReactive, useMessage } from 'naive-ui'

const store = useStore()

const props = defineProps<{
  keyword: string
  sort: SearchSort
  pageCount: number
}>()

const popConfirmShowing = ref<boolean>(false)
const running = ref<boolean>(false)
const pageLimit = ref<number>(props.pageCount)

const rejectCooldown = ref<number>(0)
const rejectButtonDisabled = computed(() => rejectCooldown.value > 0)

const countdownInterval = ref<ReturnType<typeof setInterval>>(setInterval(() => {}, 1000))

type ProgressData = Extract<DownloadAllSearchEvent, { event: 'StartCreateDownloadTasks' }>['data'] & {
  progressMessage: MessageReactive
}

const message = useMessage()

const progresses = ref<Map<number, ProgressData>>(new Map())
let prepareMessage: MessageReactive | undefined

onMounted(async () => {
  await events.downloadAllSearchEvent.listen(({ payload }) => {
    if (payload.event === 'GetSearchResultsStart') {
      running.value = true
      prepareMessage = message.loading('正在获取搜索结果', { duration: 0 })
    } else if (payload.event === 'Cancelled') {
      running.value = false
      prepareMessage?.destroy()
      prepareMessage = undefined
      progresses.value.forEach((progress) => {
        progress.progressMessage.destroy()
      })
      progresses.value.clear()
      message.warning('已取消批量下载搜索结果')
    } else if (payload.event === 'GetComicsProgress' && prepareMessage !== undefined) {
      const { current, total } = payload.data
      prepareMessage.content = `正在获取搜索结果中的漫画(${current}/${total})`
    } else if (payload.event === 'StartCreateDownloadTasks') {
      const { comicId, comicTitle, current, total } = payload.data
      progresses.value.set(comicId, {
        comicId,
        comicTitle,
        current,
        total,
        progressMessage: message.loading(
          () => {
            const progressData = progresses.value.get(comicId)
            if (progressData === undefined) return ''
            return `${progressData.comicTitle} 正在创建下载任务(${progressData.current}/${progressData.total})`
          },
          { duration: 0 },
        ),
      })
    } else if (payload.event === 'CreatingDownloadTask') {
      const { comicId, current } = payload.data
      const progressData = progresses.value.get(comicId)
      if (progressData) {
        progressData.current = current
      }
    } else if (payload.event === 'EndCreateDownloadTasks') {
      const { comicId } = payload.data
      const progressData = progresses.value.get(comicId)
      if (progressData) {
        progressData.progressMessage.type = 'success'
        progressData.progressMessage.content = `${progressData.comicTitle} 创建下载任务完成(${progressData.current}/${progressData.total})`
        setTimeout(() => {
          progressData.progressMessage.destroy()
          progresses.value.delete(comicId)
        }, 3000)
      }
    } else if (payload.event === 'GetComicsEnd' && prepareMessage !== undefined) {
      running.value = false
      prepareMessage.type = 'success'
      prepareMessage.content = '成功获取搜索结果中所有的漫画'
      setTimeout(() => {
        prepareMessage?.destroy()
        prepareMessage = undefined
      }, 3000)
    }
  })
})

async function agree() {
  if (store.config === undefined) {
    return
  }

  // 1秒下载5张
  store.config.imgDownloadIntervalSec = Math.max(1, Math.floor(store.config.imgConcurrency / 5))
  store.config.chapterDownloadIntervalSec = Math.min(10, Math.floor(store.config.imgConcurrency * 3))

  popConfirmShowing.value = false

  const result = await commands.downloadAllSearchResults(props.keyword, props.sort, pageLimit.value)
  if (result.status === 'error') {
    console.error(result.error)
    running.value = false
    prepareMessage?.destroy()
    progresses.value.forEach((progress) => {
      progress.progressMessage.destroy()
    })
    progresses.value.clear()
    return
  }
}

async function reject() {
  popConfirmShowing.value = false
  const result = await commands.downloadAllSearchResults(props.keyword, props.sort, pageLimit.value)
  if (result.status === 'error') {
    console.error(result.error)
    running.value = false
    prepareMessage?.destroy()
    progresses.value.forEach((progress) => {
      progress.progressMessage.destroy()
    })
    progresses.value.clear()
    return
  }
}

async function cancel() {
  const result = await commands.cancelDownloadAllSearchResults()
  if (result.status === 'error') {
    console.error(result.error)
  }
}

function handleDownloadClick() {
  pageLimit.value = props.pageCount
  // 清理可能存在的旧计时器
  if (countdownInterval.value) {
    clearInterval(countdownInterval.value)
  }
  rejectCooldown.value = 10

  countdownInterval.value = setInterval(() => {
    rejectCooldown.value -= 1
    if (rejectCooldown.value <= 0) {
      clearInterval(countdownInterval.value)
    }
  }, 1000)
}
</script>

<template>
  <n-popconfirm :positive-text="null" :negative-text="null" v-model:show="popConfirmShowing">
    <div class="flex flex-col">
      <div>下载整个搜索结果是个大任务</div>
      <div>为了减轻禁漫服务器压力</div>
      <div>将自动调整配置中的下载间隔</div>
      <div>
        <span>之后你随时可以在右上角的</span>
        <span class="bg-gray-2 px-1">配置</span>
        <span>调整</span>
      </div>
      <n-input-group class="mt-2">
        <n-input-group-label size="small">下载前</n-input-group-label>
        <n-input-number
          class="w-full"
          v-model:value="pageLimit"
          size="small"
          :min="1"
          :max="props.pageCount"
          :parse="(x: string) => Number(x)" />
        <n-input-group-label size="small">页(共{{ props.pageCount }}页)</n-input-group-label>
      </n-input-group>
    </div>

    <template #action>
      <n-button size="small" :disabled="rejectButtonDisabled" @click="reject">
        <span v-if="rejectButtonDisabled">不调整直接下载 ({{ rejectCooldown }})</span>
        <span v-else>不调整直接下载</span>
      </n-button>
      <n-button size="small" type="primary" @click="agree">调整并下载</n-button>
    </template>

    <template #trigger>
      <n-button type="primary" size="small" @click="handleDownloadClick">下载全部搜索结果</n-button>
    </template>
  </n-popconfirm>
  <n-button v-if="running" class="ml-2" size="small" type="error" secondary @click="cancel">取消</n-button>
</template>
