<script setup lang="ts">
import { ref, onMounted } from 'vue'
import DarkMode from '@/components/DarkMode.vue'
import {
  Select,
  SelectContent,
  SelectGroup,
  SelectItem,
  SelectLabel,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import {
  TagsInput,
  TagsInputInput,
  TagsInputItem,
  TagsInputItemDelete,
  TagsInputItemText,
} from '@/components/ui/tags-input'
import {
  useClipboard,
  useStorage,
} from '@vueuse/core'
import { Button } from '@/components/ui/button'
import { marked } from 'marked'
const isSupport = ref(false)
const isLoading = ref(false)
const writer = ref<any>(null)
const output = ref<string>('')
const parsedOutput = ref<string>('')


const { copy } = useClipboard()

const toneMap = {
  formal: '正式',
  neutral: '中性',
  casual: '随意',
}
const lengthMap = {
  short: '短',
  medium: '中',
  long: '长',
}
const formatMap = {
  'plain-text': '纯文本',
  markdown: 'Markdown',
}

const config = ref({
  sharedContext: '根据用户输入的关键词，写一篇工作周报，包含具体工作事项、总结、下周计划三个部分。',
  tone: 'formal',
  length: 'medium',
  format: 'plain-text',
})
const works = ref<string[]>([])
const controller = ref<AbortController | null>(null)

async function createWriter() {
  if (writer.value) {
    await writer.value.destroy()
  }
  writer.value = await ai?.writer?.create?.(config.value)
  console.log('createWriter', writer.value)
}

async function generate() {
  output.value = ''
  parsedOutput.value = ''
  if (!writer.value || !works.value.length) return
  if (controller.value) {
    controller.value.abort()
  }
  controller.value = new AbortController()
  const stream = writer.value.writeStreaming(works.value.join(', '), {
    signal: controller.value.signal,
  })
  for await (const chunk of stream) {
    output.value = chunk
    parsedOutput.value = marked.parse(chunk)

  }
}

onMounted(async () => {
  isLoading.value = true
  try {
    await createWriter()
    isSupport.value = true
  } catch (error) {
    console.error(error)
    isSupport.value = false
  } finally {
    isLoading.value = false
  }
})
</script>

<template>
  <div class="dark:bg-[rgb(2,8,23)] min-h-screen box-border p-6">
    <DarkMode class="fixed right-2 top-2" />
    <div v-if="isLoading">Loading...</div>
    <div v-else-if="!isSupport">Your browser does not support AI Writer</div>
    <div v-else>
      <h1 class="text-2xl font-bold dark:text-white mb-4">🐔 周报鸡</h1>
      <div class="select-wrapper flex items-center gap-4">
        <div class="flex items-center gap-2">
          <span class="text-sm text-gray-500 dark:text-white">语气：</span>
        <Select v-model="config.tone" @update:model-value="createWriter">
          <SelectTrigger class="w-[120px] text-gray-500 dark:text-white">
          <SelectValue placeholder="请选择语气" />
        </SelectTrigger>
        <SelectContent>
          <SelectGroup>
            <SelectItem v-for="tone in Object.keys(toneMap)" :value="tone">
              {{ toneMap[tone] }}
            </SelectItem>
          </SelectGroup>
          </SelectContent>
        </Select>
        </div>
        <div class="flex items-center gap-2">
          <span class="text-sm text-gray-500 dark:text-white">长度：</span>
          <Select v-model="config.length" @update:model-value="createWriter">
            <SelectTrigger class="w-[120px] text-gray-500 dark:text-white">
              <SelectValue placeholder="请选择长度" />
            </SelectTrigger>
            <SelectContent>
              <SelectGroup>
                <SelectItem v-for="length in Object.keys(lengthMap)" :value="length">
                  {{ lengthMap[length] }}
                </SelectItem>
              </SelectGroup>
            </SelectContent>
          </Select>
        </div>
        <div class="flex items-center gap-2">
          <span class="text-sm text-gray-500 dark:text-white">格式：</span>
          <Select v-model="config.format" @update:model-value="createWriter">
            <SelectTrigger class="w-[120px] text-gray-500 dark:text-white">
              <SelectValue placeholder="请选择格式" />
            </SelectTrigger>
            <SelectContent>
              <SelectGroup>
                <SelectItem v-for="format in Object.keys(formatMap)" :value="format">
                  {{ formatMap[format] }}
                </SelectItem>
              </SelectGroup>
            </SelectContent>
          </Select>
        </div>
      </div>
      <div class="flex items-center gap-2">
        <div class="text-lg text-gray-500 dark:text-white my-4">本周干了啥：</div>
        <TagsInput v-model="works" class="text-gray-500 dark:text-white max-w-[80vw]">
          <TagsInputItem v-for="item in works" :key="item" :value="item">
            <TagsInputItemText />
            <TagsInputItemDelete />
          </TagsInputItem>
          <TagsInputInput placeholder="输入工作事项，回车确认" />
        </TagsInput>
        <Button variant="outline" class="text-gray-500 dark:text-white border-gray-500 dark:border-white" @click="generate">生成</Button>
        <Button v-if="parsedOutput" variant="outline" class="text-gray-500 dark:text-white border-gray-500 dark:border-white" @click="copy(output)">复制</Button>
      </div>
      <div
        v-if="parsedOutput"
        class="output pretty-scrollbar text-lg dark:text-white h-[calc(100vh-325px)] whitespace-pre-line overflow-y-auto pr-[18px] leading-normal border border-gray-200 rounded-md dark:border-gray-800 p-2"
        v-html="parsedOutput"></div>
    </div>
  </div>
</template>

<style scoped>
.pretty-scrollbar {
  scrollbar-width: thin;
  scrollbar-color: rgba(156, 163, 175, 0.5) transparent;
}

.pretty-scrollbar::-webkit-scrollbar {
  width: 8px;
}

.pretty-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}

.pretty-scrollbar::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.5);
  border-radius: 4px;
}

.pretty-scrollbar::-webkit-scrollbar-thumb:hover {
  background-color: rgba(156, 163, 175, 0.7);
}
</style>