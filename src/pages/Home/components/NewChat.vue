<script setup lang="ts">
import { v4 as uuid } from 'uuid'
import type { VoiceInfo } from 'microsoft-cognitiveservices-speech-sdk'
import Avatar from '@/components/Avatar.vue'
import { supportLanguageMap, voiceStyleMap } from '@/config'
import { useConversationStore } from '@/stores'
import { getAvatarUrl, getInstalledVoices, getMacInstalledVoices } from '@/utils'
const { allVoices } = defineProps<{ allVoices: VoiceInfo[] }>()
const emits = defineEmits(['close'])
// eslint-disable-next-line @typescript-eslint/no-var-requires
const os = require('os')
const modules = import.meta.glob(['../../../assets/avatars/*', '!../../../assets/avatars/self.png'])
const avatarList = ref<string[]>(Object.keys(modules).map(path => path.replace('../../../assets/avatars/', '')))
const currentAvatarIndex = ref(Math.random() * avatarList.value.length | 0)
const { voiceApiName } = useGlobalSetting()
const store = useConversationStore()
const { ssmlToSpeak, isSynthesizing, isPlaying } = useSpeechService({ isFetchAllVoice: false })
const allLanguages = computed(() => [...new Set(allVoices.map(v => v.locale))].filter(l => Object.keys(supportLanguageMap).includes(l)))
const desc = ref('')
const name = ref('')
const rate = ref('1.0')
const previewText = ref('polyglot is awesome!')
const presets = ref('Act as if you are meeting someone for the first time. How would you introduce yourself and start a conversation?')

const voiceValue = ref<string[]>(['en-US', 'en-US-JennyNeural', 'chat'])

const selectLanguage = computed(() => voiceValue.value[0])
const selectVoiceName = computed(() => voiceValue.value[1])
const selectStyle = computed(() => voiceValue.value[2])
const selectPlatform = ref('Azure')
const canAdd = computed(() => !!(selectLanguage.value && selectVoiceName.value && desc.value && name.value))

interface Option {
  label: string
  value: string
  children?: Option[]
}

const options = ref<Option[] >([])

const initOptions = () => {
  options.value = []

  if (selectPlatform.value === 'Azure') {
    allLanguages.value.forEach((item) => {
      const children: Option[] = []
      allVoices.forEach((v) => {
        if (v.locale === item) {
          children.push({
            value: v.shortName,
            label: `${v.gender === 1 ? '🧒🏻' : '👦🏻'} ${v.localName}`,
            children: v.styleList?.map(x => ({ label: voiceStyleMap[x], value: x })) || [],
          })
        }
      })
      options.value.push({
        value: item,
        label: supportLanguageMap[item],
        children,
      })
    })
  }
  if (selectPlatform.value === 'MAC TTS') {
    getMacInstalledVoices((res) => {
      if (res.length > 0) {
        res.forEach((item: string[]) => {
          // 0 - Chinese DisplayName 1 - zh-CN 2 - Gender 3 - Age 4 - Name
          const name = item[4]
          // 合并同一语言的不同声音
          const index = options.value.findIndex(x => x.value === item[1])
          if (index === -1) {
            options.value.push({
              value: item[1],
              label: supportLanguageMap[item[1]] ? supportLanguageMap[item[1]] : supportLanguageMap[item[1].split('-')[0]],
              children: [
                {
                  value: item[4],
                  label: `${item[2] === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
                },
              ],
            })
          }
          else {
            options.value[index].children.push({
              value: item[4],
              label: `${item[2] === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
            })
          }
        })
      }
      else {
        alert('请检查Mac语音配置')
      }
    })
  }
  if (selectPlatform.value === 'Windows TTS') {
    getInstalledVoices((res) => {
      if (res.length > 0) {
        res.forEach((item: string[]) => {
          // 0 - Chinese DisplayName 1 - zh-CN 2 - Gender 3 - Age 4 - Name
          const name = item[4].split(' ')[1]
          // 合并同一语言的不同声音
          const index = options.value.findIndex(x => x.value === item[1])
          if (index === -1) {
            options.value.push({
              value: item[1],
              label: supportLanguageMap[item[1]],
              children: [
                {
                  value: item[4],
                  label: `${item[2] === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
                },
              ],
            })
          }
          else {
            options.value[index].children.push({
              value: item[4],
              label: `${item[2] === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
            })
          }
        })
      }
      else {
        alert('请检查Windows语音配置')
      }
    })
  }
  if (selectPlatform.value === 'Electron TTS') {
    const res = window.speechSynthesis.getVoices()

    res.forEach((item) => {
      let name = ''
      if (os.type().indexOf('Windows') >= 0)
        name = item.name.split('-')[0].trim().split(' ')[1]
      else
        name = item.name
      item.gender = 'Unknown'
      const index = options.value.findIndex(x => x.value === item.lang)
      if (index === -1) {
        options.value.push({
          value: item.lang,
          label: supportLanguageMap[item.lang] ? supportLanguageMap[item.lang] : supportLanguageMap[item.lang.split('-')[0]],
          children: [
            {
              value: item.name,
              label: `${item.gender === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
            },
          ],
        })
      }
      else {
        options.value[index].children.push({
          value: item.name,
          label: `${item.gender === 'Male' ? '🧒🏻' : '👦🏻'} ${name}`,
        })
      }
    })
  }
}

onMounted(() => {
  initOptions()
})

watch(selectPlatform, () => {
  initOptions()
})

const randomAvatar = getAvatarUrl(avatarList.value[Math.random() * avatarList.value.length | 0]) // 随机默认选择一个头像
const imageUrl = ref(randomAvatar)

const addChat = (event: any) => {
  event.preventDefault()
  const uid = uuid()
  store.addConversation({
    language: selectLanguage.value,
    voice: selectVoiceName.value,
    desc: desc.value,
    name: name.value,
    key: uid,
    avatar: imageUrl.value,
    rate: +rate.value,
    isDefault: false,
    voiceStyle: selectStyle.value,
    voicePlatform: selectPlatform.value,
  }, presets.value)
  store.changeCurrentKey(uid)
  emits('close')
}

// 使用本地内置头像
const changeAvatar = () => {
  currentAvatarIndex.value = avatarList.value.length - 1 === currentAvatarIndex.value ? 0 : currentAvatarIndex.value + 1
}
const previewSpeech = () => {
  ssmlToSpeak(previewText.value, { voice: selectVoiceName.value, lang: selectLanguage.value, voiceRate: +rate.value, voiceStyle: selectStyle.value }, selectPlatform.value)
}
</script>

<template>
  <div class="wrapper" flex="~ col gap-3" items-center>
    <div flex>
      <Avatar v-model:image-url="imageUrl" />
    </div>
    <div flex>
      <label for="">姓名</label>
      <input v-model="name" type="text">
    </div>
    <div flex>
      <label for="">备注</label>
      <input v-model="desc" type="text">
    </div>
    <div
      class="flex ml-[-30px]"
    >
      <label for="">
        场景预设</label>
      <textarea v-model="presets" :rows="4" placeholder="system prompt..." />
    </div>
    <div class="flex ml-[-30px]">
      <label for="">语音平台</label>
      <div w-55 flex>
        <select
          v-model="selectPlatform"
        >
          <!--          Windows TTS ~ Electron 调用 powershell 命令实现 Text To Speech,添加更多语音请前往 [Windows设置]-[时间和语言]-[语言]-[添加语音] -->
          <!--          MAC TTS ~  Electron 调用 say 命令实现 Text To Speech -->
          <!--          Electron TTS ~ Electron 调用 Html5 SpeechSynthesisUtterance 接口实现 Text To Speech -->
          <option value="Azure">
            Azure
          </option>
          <option v-if="os.type().indexOf('Windows') >= 0" value="Windows TTS">
            Windows TTS
          </option>
          <option v-if="os.type().indexOf('Darwin') >= 0" value="MAC TTS">
            MAC TTS
          </option>
          <option value="Electron TTS">
            Electron TTS
          </option>
        </select>
      </div>
    </div>
    <div flex>
      <label for="">语音</label>
      <div w-55 flex>
        <ElCascader v-model="voiceValue" filterable placeholder="select voice" style="width: 220px;" :options="options" />
      </div>
    </div>

    <div class="ml-[-30px] relative center-y">
      <div flex>
        <label for="">语音预览</label>
        <input v-model="previewText" placeholder="输入文字预览语音" type="text">
      </div>
      <div absolute left-73>
        <button v-if="!isSynthesizing && !isPlaying " cursor-pointer :disabled="!previewText" center-y ml-2 @click="previewSpeech()">
          <i icon-btn rotate-90 i-ic:sharp-wifi />
        </button>
        <button v-else center-y ml-2>
          <i rotate-90 icon-btn i-svg-spinners:wifi-fade />
        </button>
      </div>
    </div>
    <div flex>
      <label for="">语速</label>
      <div w-55 flex>
        <input v-model="rate" flex-1 type="range" step="0.1" min="0.1" max="2.0">
        <span w-4 ml-1>{{ Number(rate).toFixed(1) }}</span>
      </div>
    </div>
    <!-- todo -->

    <div center-y text-sm text-gray-500>
      <i inline-block w-4 h-4 m-1 cursor-auto i-ic:baseline-lightbulb />
      点击头像可更换头像
    </div>
    <button w-20 btn center p-2 mt2 :disabled="!canAdd" @click="addChat($event)">
      <i mr-1 i-ic:outline-add-circle />
      submit
    </button>
  </div>
</template>

<style scoped>
  label{
    @apply center-y justify-center mr-2
  }
  input{
    @apply w-50 p-2 text-[#222]
  }
  select{
    @apply w-55 select-settings text-[#222]
  }
  textarea{
    @apply resize-none w-50 font-sans block p-2  text-[#222]
  }
  textArea::-webkit-scrollbar{
    width: 0;
    height: 0;
  }
  .wrapper :deep(.el-input__wrapper){
    box-shadow: 0 0 0 1px #777 inset;
    padding-top: 3px;
    padding-bottom: 3px;
  }
  .wrapper :deep(.el-input__inner){
    color: #222;
  }
  .wrapper :deep(.el-input__wrapper):hover {
    box-shadow: 0 0 0 1px #777 inset;
  }
</style>
