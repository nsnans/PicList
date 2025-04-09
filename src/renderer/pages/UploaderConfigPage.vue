<template>
  <div id="config-list-view">
    <div class="view-title">
      {{ $T('SETTINGS') }}
    </div>
    <div class="config-list-container">
      <el-row :gutter="20" justify="start" class="config-list">
        <el-col
          v-for="item in curConfigList"
          :key="item._id"
          class="config-item-col"
          :xs="24"
          :sm="curConfigList.length === 1 ? 24 : 12"
          :md="curConfigList.length === 1 ? 24 : 12"
          :lg="curConfigList.length === 1 ? 12 : 8"
          :xl="curConfigList.length === 1 ? 12 : 6"
        >
          <div
            :class="`config-card ${defaultConfigId === item._id ? 'selected' : ''}`"
            @click="() => selectItem(item._id)"
          >
            <div class="config-card-content">
              <div class="config-name">{{ item._configName }}</div>
              <div class="config-update-time">{{ formatTime(item._updatedAt) }}</div>
            </div>
            <div v-if="defaultConfigId === item._id" class="default-badge">
              {{ $T('SELECTED_SETTING_HINT') }}
            </div>
            <div class="config-card-actions">
              <el-button class="action-btn edit-btn" circle type="primary" @click.stop="openEditPage(item._id)">
                <el-icon><Edit /></el-icon>
              </el-button>
              <el-button
                class="action-btn delete-btn"
                circle
                type="danger"
                :disabled="curConfigList.length <= 1"
                @click.stop="() => deleteConfig(item._id)"
              >
                <el-icon><Delete /></el-icon>
              </el-button>
            </div>
          </div>
        </el-col>
        <el-col
          class="config-item-col"
          :xs="24"
          :sm="curConfigList.length === 1 ? 24 : 12"
          :md="curConfigList.length === 1 ? 24 : 12"
          :lg="curConfigList.length === 1 ? 12 : 8"
          :xl="curConfigList.length === 1 ? 12 : 6"
        >
          <div class="config-card add-card" @click="addNewConfig">
            <el-icon class="add-icon"><Plus /></el-icon>
            <span class="add-text">{{ $T('SETTINGS') }}</span>
          </div>
        </el-col>
      </el-row>
    </div>
    <div class="set-default-container">
      <el-button
        class="set-default-btn"
        type="success"
        round
        :disabled="store?.state.defaultPicBed === type"
        @click="setDefaultPicBed(type)"
      >
        {{ $T('SETTINGS_SET_DEFAULT_PICBED') }}
      </el-button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import dayjs from 'dayjs'
import { Edit, Delete, Plus } from '@element-plus/icons-vue'
import { onBeforeMount, ref } from 'vue'
import { useRouter, useRoute, onBeforeRouteUpdate } from 'vue-router'
import { saveConfig } from '@/utils/dataSender'

import { T as $T } from '@/i18n/index'
import { useStore } from '@/hooks/useStore'
import { PICBEDS_PAGE, UPLOADER_CONFIG_PAGE } from '@/router/config'
import { sendRPC, triggerRPC } from '@/utils/common'

import { IRPCActionType } from '#/types/enum'
import { configPaths } from '#/utils/configPaths'

const router = useRouter()
const route = useRoute()

const type = ref('')
const curConfigList = ref<IStringKeyMap[]>([])
const defaultConfigId = ref('')
const store = useStore()

async function selectItem(id: string) {
  await triggerRPC<void>(IRPCActionType.UPLOADER_SELECT, type.value, id)
  if (store?.state.defaultPicBed === type.value) {
    sendRPC(
      IRPCActionType.TRAY_SET_TOOL_TIP,
      `${type.value} ${curConfigList.value.find(item => item._id === id)?._configName || ''}`
    )
  }
  defaultConfigId.value = id
}

onBeforeRouteUpdate((to, _, next) => {
  if (to.params.type && to.name === UPLOADER_CONFIG_PAGE) {
    type.value = to.params.type as string
    getCurrentConfigList()
  }
  next()
})

onBeforeMount(() => {
  type.value = route.params.type as string
  getCurrentConfigList()
})

async function getCurrentConfigList() {
  const configList = await triggerRPC<IUploaderConfigItem>(IRPCActionType.PICBED_GET_CONFIG_LIST, type.value)
  curConfigList.value = configList?.configList ?? []
  defaultConfigId.value = configList?.defaultId ?? ''
}

function openEditPage(configId: string) {
  router.push({
    name: PICBEDS_PAGE,
    params: {
      type: type.value,
      configId
    },
    query: {
      defaultConfigId: defaultConfigId.value
    }
  })
}

function formatTime(time: number): string {
  return dayjs(time).format('YY-MM-DD HH:mm')
}

async function deleteConfig(id: string) {
  const res = await triggerRPC<IUploaderConfigItem>(IRPCActionType.PICBED_DELETE_CONFIG, type.value, id)
  if (!res) return
  curConfigList.value = res.configList
  defaultConfigId.value = res.defaultId
}

function addNewConfig() {
  router.push({
    name: PICBEDS_PAGE,
    params: {
      type: type.value,
      configId: ''
    }
  })
}

function setDefaultPicBed(type: string) {
  saveConfig({
    [configPaths.picBed.current]: type,
    [configPaths.picBed.uploader]: type
  })

  store?.setDefaultPicBed(type)
  const currentConfigName = curConfigList.value.find(item => item._id === defaultConfigId.value)?._configName
  sendRPC(IRPCActionType.TRAY_SET_TOOL_TIP, `${type} ${currentConfigName || ''}`)
  const successNotification = new Notification($T('SETTINGS_DEFAULT_PICBED'), {
    body: $T('TIPS_SET_SUCCEED')
  })
  successNotification.onclick = () => {
    return true
  }
}
</script>
<script lang="ts">
export default {
  name: 'UploaderConfigPage'
}
</script>
<style lang="stylus">
$transitionDefault = all 0.25s cubic-bezier(0.4, 0, 0.2, 1)
$borderRadius = 8px
$cardBg = rgba(55, 60, 70, 0.55)
$hoverBg = rgba(50, 54, 62, 0.7)
$selectedBorder = #409EFF
$cardShadow = 0 3px 8px rgba(0, 0, 0, 0.25)
$cardShadowHover = 0 5px 15px rgba(0, 0, 0, 0.35)
$normalBorder = rgba(150, 150, 150, 0.25)

#config-list-view
  position absolute
  min-height 100%
  left 162px
  right 0
  overflow-x hidden
  overflow-y auto
  padding 0 20px 90px
  box-sizing border-box
  display flex
  flex-direction column

  .view-title
    color #eee
    font-size 24px
    font-weight 500
    margin 20px 0
    text-align center
    letter-spacing 0.5px

  .config-list-container
    flex 1
    width 100%
    box-sizing border-box

  .config-list
    margin 0
    width 100%

  .config-item-col
    margin-bottom 20px

  .config-card
    position relative
    height 110px
    border-radius $borderRadius
    background $cardBg
    backdrop-filter blur(5px)
    box-shadow $cardShadow
    transition $transitionDefault
    cursor pointer
    overflow hidden
    display flex
    align-items center
    justify-content space-between
    padding 0 15px 0 20px
    border 1px solid $normalBorder

    &:hover
      transform translateY(-3px)
      box-shadow $cardShadowHover
      background $hoverBg

    &.selected
      border 1px solid $selectedBorder
      background rgba(64, 158, 255, 0.15) // Slightly more visible selected state

    .default-badge
      position absolute
      right 0
      top 0
      font-size 11px
      font-weight 600
      padding 4px 10px
      background $selectedBorder
      color #fff
      z-index 5
      border-bottom-left-radius $borderRadius

    .config-card-content
      flex-grow 1
      padding 15px 0
      overflow hidden
      position relative

      .config-name
        color #fff
        font-size 16px
        font-weight 500
        margin-bottom 10px
        white-space nowrap
        overflow hidden
        text-overflow ellipsis

      .config-update-time
        color #aaa
        font-size 14px

    .config-card-actions
      display flex
      flex-direction row
      gap 10px
      align-items center

      .action-btn
        transition $transitionDefault
        border none
        width 32px
        height 32px
        display flex
        align-items center
        justify-content center

        &.edit-btn
          &:hover
            transform scale(1.1)

        &.delete-btn
          &:hover:not(:disabled)
            transform scale(1.1)

          &:disabled
            opacity 0.5
            cursor not-allowed

  .add-card
    display flex
    flex-direction column
    align-items center
    justify-content center
    background rgba(50, 54, 62, 0.4)
    border 1px dashed rgba(255, 255, 255, 0.4)

    &:hover
      border-color rgba(255, 255, 255, 0.8)
      background rgba(64, 158, 255, 0.15)

      .add-icon
        transform rotate(90deg)

    .add-icon
      font-size 32px
      color #eee
      margin-bottom 8px
      transition $transitionDefault

    .add-text
      color #eee
      font-size 14px

  .set-default-container
    position fixed
    bottom 0
    left 162px
    right 0
    display flex
    justify-content center
    align-items center
    padding 15px 0
    height 70px
    background linear-gradient(to top, rgba(30, 34, 42, 0.95) 0%, rgba(30, 34, 42, 0.8) 60%, rgba(30, 34, 42, 0) 100%)
    z-index 10

    .set-default-btn
      width 250px
      height 40px
      font-weight 500
      transition $transitionDefault
      margin-bottom 0

      &:not(:disabled):hover
        transform translateY(-2px)
        box-shadow 0 4px 12px rgba(0, 0, 0, 0.3)
</style>
