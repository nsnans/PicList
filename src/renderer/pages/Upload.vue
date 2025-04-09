<template>
  <div id="upload-view" class="upload-container">
    <div class="container-inner">
      <div class="view-title">
        <el-tooltip placement="top" effect="light" :content="$T('UPLOAD_VIEW_HINT')" :persistent="false" teleported>
          <span id="upload-view-title" @click="handlePicBedNameClick(picBedName, picBedConfigName)">
            {{ picBedName }} - {{ picBedConfigName || 'Default' }}
          </span>
        </el-tooltip>
        <el-button class="icon-button" type="info" size="small" circle @click="handleChangePicBed">
          <el-icon><CaretBottom /></el-icon>
        </el-button>
        <el-button type="primary" round size="small" class="quick-upload" @click="handleImageProcess">
          <el-icon><Edit /></el-icon>
          {{ $T('UPLOAD_PAGE_IMAGE_PROCESS_NAME') }}
        </el-button>
      </div>

      <div
        id="upload-area"
        class="upload-area"
        :class="{ 'is-dragover': dragover }"
        @drop.prevent="onDrop"
        @dragover.prevent="dragover = true"
        @dragleave.prevent="dragover = false"
      >
        <transition name="scale-fade">
          <div v-if="showProgress" class="upload-overlay">
            <el-progress
              type="dashboard"
              :percentage="progress"
              :status="showError ? 'exception' : 'success'"
              :stroke-width="8"
            />
            <div class="upload-status-text">
              {{ showError ? $T('UPLOAD_FAILED') : $T('UPLOADING') }}
            </div>
          </div>
        </transition>

        <div id="upload-dragger" class="upload-dragger" @click="openUplodWindow">
          <el-icon class="upload-icon"><UploadFilled /></el-icon>
          <div class="upload-dragger__text">
            {{ $T('DRAG_FILE_TO_HERE') }}
            <span class="upload-action-text">{{ $T('CLICK_TO_UPLOAD') }}</span>
          </div>
          <input id="file-uploader" type="file" multiple @change="onChange" />
        </div>
      </div>

      <div class="upload-options">
        <div class="option-card link-format-card">
          <div class="option-header">
            <div class="option-label">
              {{ $T('LINK_FORMAT') }}
            </div>
            <el-switch
              v-model="useShortUrl"
              class="short-url-switch"
              active-text=""
              inactive-text=""
              :active-value="true"
              :inactive-value="false"
              @change="handleUseShortUrlChange"
            />
            <span class="switch-label">{{ useShortUrl ? $T('UPLOAD_SHORT_URL') : $T('UPLOAD_NORMAL_URL') }}</span>
          </div>
          <div class="option-content">
            <el-radio-group
              v-model="pasteStyle"
              size="small"
              class="format-radio-group"
              @change="handlePasteStyleChange"
            >
              <el-radio-button
                v-for="(item, key) in pasteFormatList"
                :key="key"
                :value="key"
                :title="item"
                class="format-radio-btn"
              >
                {{ key }}
              </el-radio-button>
            </el-radio-group>
          </div>
        </div>

        <div class="option-card quick-upload-card">
          <div class="option-label">
            {{ $T('QUICK_UPLOAD') }}
          </div>
          <div class="option-content quick-upload-content">
            <el-button type="primary" class="quick-upload-btn" @click="uploadClipboardFiles">
              <el-icon><Picture /></el-icon>
              {{ $T('CLIPBOARD_PICTURE') }}
            </el-button>
            <el-button type="primary" class="quick-upload-btn" @click="uploadURLFiles">
              <el-icon><Link /></el-icon>
              URL
            </el-button>
          </div>
        </div>
      </div>
    </div>

    <el-dialog
      v-model="imageProcessDialogVisible"
      :title="$T('UPLOAD_PAGE_IMAGE_PROCESS_DIALOG_TITLE')"
      width="50%"
      draggable
      center
      align-center
      append-to-body
    >
      <ImageProcessSetting v-model="imageProcessDialogVisible" />
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import { ipcRenderer, IpcRendererEvent } from 'electron'
import { ElMessage as $message, ElSwitch } from 'element-plus'
import { UploadFilled, CaretBottom, Picture, Link, Edit } from '@element-plus/icons-vue'
import { ref, onBeforeMount, onBeforeUnmount, watch } from 'vue'
import { useRouter } from 'vue-router'
import { useDebounceFn, useThrottleFn } from '@vueuse/core'

import ImageProcessSetting from '@/components/ImageProcessSetting.vue'
import { T as $T } from '@/i18n'
import { PICBEDS_PAGE } from '@/router/config'
import $bus from '@/utils/bus'
import { sendRPC, triggerRPC } from '@/utils/common'
import { getConfig, saveConfig } from '@/utils/dataSender'
import { picBedGlobal, updatePicBedGlobal } from '@/utils/global'

import { SHOW_INPUT_BOX, SHOW_INPUT_BOX_RESPONSE } from '#/events/constants'
import { IPasteStyle, IRPCActionType } from '#/types/enum'
import { isUrl } from '#/utils/common'
import { configPaths } from '#/utils/configPaths'
import { useDragEventListeners } from '@/utils/drag'

useDragEventListeners()
const $router = useRouter()

// State variables
const imageProcessDialogVisible = ref(false)
const useShortUrl = ref(false)
const dragover = ref(false)
const progress = ref(0)
const showProgress = ref(false)
const showError = ref(false)
const pasteStyle = ref('')
const picBedName = ref('')
const picBedConfigName = ref('')

const pasteFormatList = ref({
  [IPasteStyle.MARKDOWN]: '![alt](url)',
  [IPasteStyle.HTML]: '<img src="url"/>',
  [IPasteStyle.URL]: 'http://test.com/test.png',
  [IPasteStyle.UBB]: '[img]url[/img]',
  [IPasteStyle.CUSTOM]: ''
})

watch(picBedGlobal, () => {
  getDefaultPicBed()
})

watch(progress, onProgressChange)

onBeforeMount(async () => {
  await updatePicBedGlobal()
  ipcRenderer.on('uploadProgress', handleUploadProgress)
  ipcRenderer.on('syncPicBed', getDefaultPicBed)
  $bus.on(SHOW_INPUT_BOX_RESPONSE, handleInputBoxValue)
  await Promise.all([getUseShortUrl(), getPasteStyle(), getDefaultPicBed()])
})

onBeforeUnmount(() => {
  $bus.off(SHOW_INPUT_BOX_RESPONSE)
  ipcRenderer.removeAllListeners('uploadProgress')
  ipcRenderer.removeAllListeners('syncPicBed')
})

const handleUploadProgress = useThrottleFn((_event: IpcRendererEvent, _progress: number) => {
  if (_progress !== -1) {
    showProgress.value = true
    progress.value = _progress
  } else {
    progress.value = 100
    showError.value = true
  }
}, 50)

const handleImageProcess = () => {
  imageProcessDialogVisible.value = true
}

function onProgressChange(val: number) {
  if (val === 100) {
    setTimeout(() => {
      showProgress.value = false
      showError.value = false
    }, 1000)
    setTimeout(() => {
      progress.value = 0
    }, 1200)
  }
}

async function handlePicBedNameClick(_picBedName: string, picBedConfigName: string | undefined) {
  const formatedpicBedConfigName = picBedConfigName || 'Default'
  const currentPicBed = await getConfig<string>(configPaths.picBed.current)
  const currentPicBedConfig = ((await getConfig<any[]>(`uploader.${currentPicBed}`)) as any) || {}
  const configList = await triggerRPC<IUploaderConfigItem>(IRPCActionType.PICBED_GET_CONFIG_LIST, currentPicBed)
  const currentConfigList = configList?.configList ?? []
  const config = currentConfigList.find((item: any) => item._configName === formatedpicBedConfigName)
  $router.push({
    name: PICBEDS_PAGE,
    params: {
      type: currentPicBed,
      configId: config?._id || ''
    },
    query: {
      defaultConfigId: currentPicBedConfig.defaultId || ''
    }
  })
}

const onDrop = useDebounceFn((e: DragEvent) => {
  dragover.value = false
  if (e.dataTransfer?.files?.length) {
    ipcSendFiles(e.dataTransfer.files)
  } else if (e.dataTransfer?.items) {
    const items = e.dataTransfer.items
    if (items.length === 2 && items[0].type === 'text/uri-list') {
      handleURLDrag(items, e.dataTransfer)
    } else if (items[0].type === 'text/plain') {
      const str = e.dataTransfer.getData(items[0].type)
      if (isUrl(str)) {
        sendRPC(IRPCActionType.UPLOAD_CHOOSED_FILES, [{ path: str }])
      } else {
        $message.error($T('TIPS_DRAG_VALID_PICTURE_OR_URL'))
      }
    }
  }
}, 200)

function handleURLDrag(items: DataTransferItemList, dataTransfer: DataTransfer) {
  const urlString = dataTransfer.getData(items[1].type)
  const urlMatch = urlString.match(/<img.*src="(.*?)"/)
  if (urlMatch) {
    sendRPC(IRPCActionType.UPLOAD_CHOOSED_FILES, [
      {
        path: urlMatch[1]
      }
    ])
  } else {
    $message.error($T('TIPS_DRAG_VALID_PICTURE_OR_URL'))
  }
}

function openUplodWindow() {
  document.getElementById('file-uploader')!.click()
}

function onChange(e: any) {
  ipcSendFiles(e.target.files)
  ;(document.getElementById('file-uploader') as HTMLInputElement).value = ''
}

function ipcSendFiles(files: FileList) {
  const sendFiles: IFileWithPath[] = []
  Array.from(files).forEach(item => {
    const obj = {
      name: item.name,
      path: item.path
    }
    sendFiles.push(obj)
  })
  sendRPC(IRPCActionType.UPLOAD_CHOOSED_FILES, sendFiles)
}

async function getPasteStyle() {
  pasteStyle.value = (await getConfig(configPaths.settings.pasteStyle)) || IPasteStyle.MARKDOWN
  pasteFormatList.value.Custom = (await getConfig(configPaths.settings.customLink)) || '![$fileName]($url)'
}

async function getUseShortUrl() {
  useShortUrl.value = (await getConfig(configPaths.settings.useShortUrl)) || false
}

async function handleUseShortUrlChange() {
  saveConfig({
    [configPaths.settings.useShortUrl]: useShortUrl.value
  })
}

function handlePasteStyleChange(val: string | number | boolean | undefined) {
  saveConfig({
    [configPaths.settings.pasteStyle]: val || IPasteStyle.MARKDOWN
  })
}

function uploadClipboardFiles() {
  sendRPC(IRPCActionType.UPLOAD_CLIPBOARD_FILES_FROM_UPLOAD_PAGE)
}

async function uploadURLFiles() {
  const str = await navigator.clipboard.readText()
  $bus.emit(SHOW_INPUT_BOX, {
    value: isUrl(str) ? str : '',
    title: $T('TIPS_INPUT_URL'),
    placeholder: $T('TIPS_HTTP_PREFIX')
  })
}

function handleInputBoxValue(val: string) {
  if (val === '') return
  if (isUrl(val)) {
    sendRPC(IRPCActionType.UPLOAD_CHOOSED_FILES, [
      {
        path: val
      }
    ])
  } else {
    $message.error($T('TIPS_INPUT_VALID_URL'))
  }
}

async function getDefaultPicBed() {
  const currentPicBed = await getConfig<string>(configPaths.picBed.current)
  picBedGlobal.value.forEach(item => {
    if (item.type === currentPicBed) {
      picBedName.value = item.name
    }
  })
  picBedConfigName.value = (await getConfig<string>(`picBed.${currentPicBed}._configName`)) || ''
}

async function handleChangePicBed() {
  sendRPC(IRPCActionType.SHOW_UPLOAD_PAGE_MENU)
}
</script>

<script lang="ts">
export default {
  name: 'UploadPage'
}
</script>

<style lang="stylus">
.upload-container
  position absolute
  left 142px
  right 0
  height 100%
  padding 0
  transition all 0.3s ease
  overflow-y auto
  overflow-x hidden
  display flex
  align-items center
  justify-content center

.container-inner
  max-width 800px
  width 100%
  padding 20px
  max-height 90vh // Limit maximum height
  display flex
  flex-direction column
  justify-content center // Center content vertically
  margin auto // Center the container
  gap 20px // Add consistent spacing between elements

.view-title
  display flex
  color #f2f2f2
  font-size 20px
  text-align center
  margin-bottom 0
  align-items center
  justify-content center
  flex-wrap wrap
  padding 12px
  border-radius 10px
  max-width 800px

  @media (max-width: 640px), (max-height: 700px)
    font-size 16px
    padding 10px

  .icon-button
    margin-left 8px
    transition transform 0.2s ease
    &:hover
      transform scale(1.1)

  .quick-upload
    margin-left 12px
    display flex
    align-items center
    gap 4px
    transition all 0.3s ease
    &:hover
      transform translateY(-2px)
      box-shadow 0 4px 12px rgba(0, 0, 0, 0.2)

    @media (max-width: 640px)
      margin-top 8px
      margin-left 0

#upload-view-title
  transition all 0.2s ease
  position relative
  overflow hidden
  display inline-block

  &:after
    content ''
    position absolute
    left 0
    bottom -2px
    width 0
    height 2px
    background #409EFF
    transition width 0.3s ease

  &:hover
    cursor pointer
    color #409EFF

    &:after
      width 100%

.upload-area
  flex 1
  min-height 250px
  max-height 550px
  border 2px dashed rgba(255, 255, 255, 0.2)
  border-radius 16px
  text-align center
  width 100%
  margin 0 auto
  color #eee
  cursor pointer
  transition all 0.3s cubic-bezier(0.25, 0.1, 0.25, 1.0)
  position relative
  display flex
  align-items center
  justify-content center
  backdrop-filter blur(5px)
  background linear-gradient(135deg, rgba(50, 50, 50, 0.1) 0%, rgba(30, 30, 30, 0.2) 100%)
  box-shadow 0 8px 20px rgba(0, 0, 0, 0.1)

  @media (max-height: 600px)
    min-height 180px
    max-height 250px

  @media (min-height: 800px)
    min-height 350px
    max-height 550px

  &.is-dragover,
  &:hover
    border 2px dashed #409EFF
    background-color rgba(64, 158, 255, 0.1)
    transform scale(1.01)
    box-shadow 0 10px 25px rgba(0, 0, 0, 0.15)
    color #fff

.upload-overlay
  position absolute
  top 0
  left 0
  right 0
  bottom 0
  background rgba(0, 0, 0, 0.7)
  display flex
  flex-direction column
  justify-content center
  align-items center
  z-index 10
  border-radius 15px
  backdrop-filter blur(5px)

  .upload-status-text
    margin-top 16px
    color #ffffff
    font-size 16px
    font-weight 500

.upload-dragger
  height 100%
  width 100%
  display flex
  flex-direction column
  justify-content center
  align-items center

  .upload-icon
    font-size 55px
    margin-bottom 20px
    color rgba(255, 255, 255, 0.8)
    transition all 0.3s ease
    filter drop-shadow(0 4px 6px rgba(0, 0, 0, 0.1))

    @media (max-height: 600px)
      font-size 40px
      margin-bottom 12px

  &__text
    font-size 16px
    color rgba(255, 255, 255, 0.9)
    margin-bottom 8px
    max-width 80%
    text-align center
    line-height 1.5

    @media (max-height: 600px)
      font-size 14px
      line-height 1.3

    .upload-action-text
      color #409EFF
      font-weight 500
      margin-left 6px
      position relative
      display inline-block

      &:after
        content ''
        position absolute
        left 0
        bottom -2px
        width 100%
        height 1px
        background #409EFF

#file-uploader
  display none

.upload-options
  display flex
  justify-content space-between
  gap 15px
  margin-bottom 0
  width 100%
  max-width: 800px

  @media (max-width: 768px)
    flex-direction column
    gap 12px

.option-card
  background rgba(255, 255, 255, 0.05)
  border-radius 12px
  padding 12px
  backdrop-filter blur(5px)
  box-shadow 0 5px 15px rgba(0, 0, 0, 0.1)
  transition all 0.3s ease

  &:hover
    transform translateY(-3px)
    box-shadow 0 8px 20px rgba(0, 0, 0, 0.15)

  @media (max-height: 600px)
    padding 10px

  .option-header
    display flex
    align-items center
    margin-bottom 12px
    flex-wrap wrap
    gap 8px

  .option-label
    font-size 14px
    color #f2f2f2
    font-weight 500
    border-left 3px solid #409EFF
    padding-left 10px
    margin-right auto

    @media (max-height: 600px)
      font-size 13px

  .option-content
    display flex
    flex-wrap wrap
    gap 10px

.link-format-card
  flex 3
  min-width 0
  max-width 580px

.quick-upload-card
  flex 1
  min-width 200px
  max-width 280px
  display flex
  flex-direction column

  .option-label
    margin-bottom 12px

.short-url-switch
  margin-left auto

.switch-label
  font-size 12px
  color #e0e0e0
  white-space nowrap

.format-radio-group
  display flex
  flex-wrap wrap
  gap 5px
  width 100%

.format-radio-btn
  flex-grow 1
  min-width 0

  .el-radio-button__inner
    padding 8px 10px
    width 100%
    font-size 13px
    border-radius 4px !important
    white-space nowrap
    overflow hidden
    text-overflow ellipsis
    box-shadow 0 2px 4px rgba(0, 0, 0, 0.1)

.quick-upload-content
  display flex
  gap 10px
  flex 1

  @media (max-width: 768px) and (min-width: 641px)
    flex-direction column
    gap 8px

  @media (max-width: 640px)
    flex-direction row

.quick-upload-btn
  flex 1
  padding 8px 12px
  border-radius 8px
  transition all 0.3s ease
  display flex
  align-items center
  justify-content center
  gap 5px
  font-size 13px
  height 40px

  @media (max-width: 768px) and (min-width: 641px)
    height 36px

  &:hover
    transform translateY(-2px)
    box-shadow 0 4px 12px rgba(0, 0, 0, 0.2)

// Animations
.scale-fade-enter-active,
.scale-fade-leave-active
  transition all 0.4s cubic-bezier(0.4, 0, 0.2, 1)

.scale-fade-enter-from,
.scale-fade-leave-to
  opacity 0
  transform scale(0.9)

// Radio button styling
.el-radio-button
  transition all 0.2s ease

  &__inner
    border-radius 8px !important
    box-shadow 0 2px 4px rgba(0, 0, 0, 0.1)
    margin 0 2px

  &:first-child
    .el-radio-button__inner
      border-radius 8px 0 0 8px !important

  &:last-child
    .el-radio-button__inner
      border-radius 0 8px 8px 0 !important

// Custom scrollbar for the container
.upload-container::-webkit-scrollbar
  width 5px

.upload-container::-webkit-scrollbar-thumb
  background rgba(255, 255, 255, 0.2)
  border-radius 5px

  &:hover
    background rgba(255, 255, 255, 0.3)

.upload-container::-webkit-scrollbar-track
  background transparent

// Add responsive adjustments for larger screens
@media (min-height: 900px)
  .container-inner
    gap 30px // Increase gap for larger screens
    justify-content center

  .upload-area
    flex 1 1 auto
    min-height 400px

  .view-title, .upload-options
    flex-shrink 0 // Prevent these elements from shrinking
</style>
