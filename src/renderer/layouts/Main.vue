<template>
  <div id="main-page">
    <div class="fake-title-bar">
      <div class="fake-title-bar__title">PicList - {{ version }}</div>
      <div v-if="osGlobal !== 'darwin'" class="handle-bar">
        <el-icon
          class="control-icon always-on-top"
          :color="isAlwaysOnTop ? '#409EFF' : '#fff'"
          size="20"
          @click="setAlwaysOnTop"
        >
          <ArrowUpBold />
        </el-icon>
        <el-icon class="control-icon minimize" color="#fff" size="20" @click="minimizeWindow">
          <SemiSelect />
        </el-icon>
        <el-icon class="control-icon mini-window" color="#FFA500" size="20" @click="openMiniWindow">
          <ArrowDownBold />
        </el-icon>
        <el-icon class="control-icon close" color="#fff" size="20" @click="closeWindow">
          <CloseBold />
        </el-icon>
      </div>
    </div>
    <el-progress
      v-if="isShowprogress"
      :percentage="progress"
      :stroke-width="4"
      :text-inside="true"
      :show-text="false"
      status="success"
      class="progress-bar"
    />
    <el-row class="main-content">
      <el-col class="side-bar-menu">
        <el-menu class="picgo-sidebar" :default-active="defaultActive" :unique-opened="true" @select="handleSelect">
          <el-menu-item :index="routerConfig.UPLOAD_PAGE" class="side-menu-item">
            <el-icon class="menu-icon"><UploadFilled /></el-icon>
            <span>{{ $T('UPLOAD_AREA') }}</span>
          </el-menu-item>
          <el-menu-item :index="routerConfig.MANAGE_LOGIN_PAGE" class="side-menu-item">
            <el-icon class="menu-icon"><PieChart /></el-icon>
            <span>{{ $T('MANAGE_PAGE') }}</span>
          </el-menu-item>
          <el-menu-item :index="routerConfig.GALLERY_PAGE" class="side-menu-item">
            <el-icon class="menu-icon"><PictureFilled /></el-icon>
            <span>{{ $T('GALLERY') }}</span>
          </el-menu-item>
          <el-sub-menu index="sub-menu" :show-timeout="0" :hide-timeout="0" :popper-offset="0" class="side-menu-sub">
            <template #title>
              <el-icon class="menu-icon"><Menu /></el-icon>
              <span>{{ $T('PICBEDS_SETTINGS') }}</span>
            </template>
            <template v-for="item in picBedGlobal">
              <el-menu-item
                v-if="item.visible"
                :key="item.type"
                :index="`${routerConfig.UPLOADER_CONFIG_PAGE}-${item.type}`"
                class="side-submenu-item"
              >
                <span>{{ item.name }}</span>
              </el-menu-item>
            </template>
          </el-sub-menu>
          <el-menu-item :index="routerConfig.SETTING_PAGE" class="side-menu-item">
            <el-icon class="menu-icon"><Tools /></el-icon>
            <span>{{ $T('PICLIST_SETTINGS') }}</span>
          </el-menu-item>
          <el-menu-item :index="routerConfig.PLUGIN_PAGE" class="side-menu-item">
            <el-icon class="menu-icon"><Share /></el-icon>
            <span>{{ $T('PLUGIN_SETTINGS') }}</span>
          </el-menu-item>
          <el-menu-item :index="routerConfig.DocumentPage" class="side-menu-item">
            <el-icon class="menu-icon"><Link /></el-icon>
            <span>{{ $T('MANUAL') }}</span>
          </el-menu-item>
        </el-menu>
      </el-col>
      <div class="info-window-container">
        <el-icon class="info-window" @click="openMenu">
          <InfoFilled />
        </el-icon>
      </div>
      <el-col :span="21" :offset="3" class="main-wrapper">
        <router-view v-slot="{ Component }">
          <transition name="picgo-fade" mode="out-in">
            <keep-alive :include="keepAlivePages">
              <component :is="Component" />
            </keep-alive>
          </transition>
        </router-view>
      </el-col>
    </el-row>
    <el-dialog
      v-model="qrcodeVisible"
      class="qrcode-dialog"
      top="3vh"
      width="60%"
      :title="$T('PICBED_QRCODE')"
      :modal-append-to-body="false"
      lock-scroll
      append-to-body
    >
      <el-form label-position="left" label-width="70px" size="small">
        <el-form-item :label="$T('CHOOSE_PICBED')">
          <el-select v-model="choosedPicBedForQRCode" multiple collapse-tags :persistent="false" teleported>
            <el-option v-for="item in picBedGlobal" :key="item.type" :label="item.name" :value="item.type" />
          </el-select>
          <el-button
            v-show="choosedPicBedForQRCode.length > 0"
            type="primary"
            round
            class="copy-picbed-config"
            @click="handleCopyPicBedConfig"
          >
            {{ $T('COPY_PICBED_CONFIG') }}
          </el-button>
        </el-form-item>
      </el-form>
      <div class="qrcode-container">
        <qrcode-vue v-show="choosedPicBedForQRCode.length > 0" :size="280" :value="picBedConfigString" />
      </div>
    </el-dialog>
    <input-box-dialog />
  </div>
</template>

<script lang="ts" setup>
import {
  Tools,
  UploadFilled,
  PictureFilled,
  Menu,
  Share,
  InfoFilled,
  SemiSelect,
  ArrowDownBold,
  CloseBold,
  PieChart,
  Link,
  ArrowUpBold
} from '@element-plus/icons-vue'
import { ipcRenderer, IpcRendererEvent, clipboard } from 'electron'
import { ElMessage as $message, ElMessageBox } from 'element-plus'
import pick from 'lodash/pick'
import QrcodeVue from 'qrcode.vue'
import { ref, onBeforeUnmount, Ref, onBeforeMount, watch, nextTick, reactive } from 'vue'
import { onBeforeRouteUpdate, useRouter } from 'vue-router'

import InputBoxDialog from '@/components/InputBoxDialog.vue'
import { T as $T } from '@/i18n/index'
import * as config from '@/router/config'
import { getConfig, saveConfig } from '@/utils/dataSender'
import { sendRPC } from '@/utils/common'
import { osGlobal, picBedGlobal, updatePicBedGlobal } from '@/utils/global'

import { SHOW_MAIN_PAGE_QRCODE } from '#/events/constants'
import { configPaths, manualPageOpenType } from '#/utils/configPaths'
import { II18nLanguage, IRPCActionType } from '#/types/enum'

import pkg from 'root/package.json'

const version = ref(process.env.NODE_ENV === 'production' ? pkg.version : 'Dev')
const routerConfig = reactive(config)
const defaultActive = ref(routerConfig.UPLOAD_PAGE)
const $router = useRouter()
const qrcodeVisible = ref(false)
const picBedConfigString = ref('')
const choosedPicBedForQRCode: Ref<string[]> = ref([])
const isAlwaysOnTop = ref(false)
const keepAlivePages = $router
  .getRoutes()
  .filter(item => item.meta.keepAlive)
  .map(item => item.name as string)

const isShowprogress = ref(false)
const progress = ref(0)

onBeforeMount(() => {
  updatePicBedGlobal()
  ipcRenderer.on(SHOW_MAIN_PAGE_QRCODE, () => {
    qrcodeVisible.value = true
  })
  ipcRenderer.on('updateProgress', (_event: IpcRendererEvent, data: { progress: number }) => {
    isShowprogress.value = data.progress !== 100 && data.progress !== 0
    progress.value = data.progress
  })
})

watch(
  () => choosedPicBedForQRCode,
  val => {
    if (val.value.length > 0) {
      nextTick(async () => {
        const picBedConfig = await getConfig('picBed')
        const config = pick(picBedConfig, ...choosedPicBedForQRCode.value)
        picBedConfigString.value = JSON.stringify(config)
      })
    }
  },
  { deep: true }
)

const handleSelect = async (index: string) => {
  defaultActive.value = index
  if (index === routerConfig.DocumentPage) {
    const manualPageOpenSetting = await getConfig<manualPageOpenType>(configPaths.settings.manualPageOpen)
    const lang = (await getConfig(configPaths.settings.language)) || II18nLanguage.ZH_CN
    const openManual = () => sendRPC(IRPCActionType.OPEN_MANUAL_WINDOW)
    const openExternal = () =>
      sendRPC(
        IRPCActionType.OPEN_URL,
        lang === II18nLanguage.ZH_CN ? 'https://piclist.cn/app.html' : 'https://piclist.cn/en/app.html'
      )

    if (!manualPageOpenSetting) {
      ElMessageBox.confirm($T('MANUAL_PAGE_OPEN_TIP'), $T('MANUAL_PAGE_OPEN_TIP_TITLE'), {
        confirmButtonText: $T('MANUAL_PAGE_OPEN_BY_BROWSER'),
        cancelButtonText: $T('MANUAL_PAGE_OPEN_BY_BUILD_IN'),
        type: 'info',
        center: true
      })
        .then(() => {
          saveConfig(configPaths.settings.manualPageOpen, 'browser')
          openExternal()
        })
        .catch(() => {
          saveConfig(configPaths.settings.manualPageOpen, 'window')
          openManual()
        })
    } else {
      manualPageOpenSetting === 'window' ? openManual() : openExternal()
    }
    return
  }
  const type = index.match(routerConfig.UPLOADER_CONFIG_PAGE)
  if (type === null) {
    $router.push({
      name: index
    })
  } else {
    const type = index.replace(`${routerConfig.UPLOADER_CONFIG_PAGE}-`, '')
    $router.push({
      name: routerConfig.UPLOADER_CONFIG_PAGE,
      params: {
        type
      }
    })
  }
}

function minimizeWindow() {
  sendRPC(IRPCActionType.MINIMIZE_WINDOW)
}

function closeWindow() {
  sendRPC(IRPCActionType.CLOSE_WINDOW)
}

function openMenu() {
  sendRPC(IRPCActionType.SHOW_MAIN_PAGE_MENU)
}

function openMiniWindow() {
  sendRPC(IRPCActionType.OPEN_MINI_WINDOW)
}

function handleCopyPicBedConfig() {
  clipboard.writeText(picBedConfigString.value)
  $message.success($T('COPY_PICBED_CONFIG_SUCCEED'))
}

function setAlwaysOnTop() {
  isAlwaysOnTop.value = !isAlwaysOnTop.value
  sendRPC(IRPCActionType.MAIN_WINDOW_ON_TOP)
}

onBeforeRouteUpdate(async to => {
  if (to.params.type) {
    defaultActive.value = `${routerConfig.UPLOADER_CONFIG_PAGE}-${to.params.type}`
  } else {
    defaultActive.value = to.name as string
  }
})

onBeforeUnmount(() => {
  ipcRenderer.removeAllListeners(SHOW_MAIN_PAGE_QRCODE)
  ipcRenderer.removeAllListeners('updateProgress')
})
</script>
<script lang="ts">
export default {
  name: 'MainPage'
}
</script>

<style lang="stylus">
$darwinBg = transparentify(#172426, #000, 0.7)
$menuActiveBg = rgba(64, 158, 255, 0.15)
$menuHoverBg = rgba(255, 255, 255, 0.08)
$borderRadius = 8px
$transitionDefault = all 0.25s cubic-bezier(0.4, 0, 0.2, 1)

.setting-list-scroll
  height 800px
  overflow-y auto
  overflow-x hidden
  margin-right 0!important

.picgo-fade
  &-enter,
  &-leave,
  &-leave-active
    opacity 0
  &-enter-active,
  &-leave-active
    transition all 200ms cubic-bezier(0.4, 0, 0.2, 1)

.view-title
  color #eee
  font-size 20px
  text-align center
  margin 10px auto

#main-page
  height 100%
  background linear-gradient(135deg, rgba(40, 44, 52, 0.85) 0%, rgba(30, 34, 42, 0.9) 100%)
  display flex
  flex-direction column
  overflow hidden

  .info-window-container
    position fixed
    bottom 4px
    left 4px
    z-index 50
    width 40px
    height 40px

    .info-window
      cursor pointer
      color #878d99
      padding 6px
      background rgba(40, 44, 52, 0.5)
      border-radius 50%
      box-shadow 0 2px 8px rgba(0, 0, 0, 0.15)
      transition $transitionDefault

      &:hover
        color #409EFF
        background rgba(64, 158, 255, 0.15)
        transform translateY(-2px)
        box-shadow 0 4px 12px rgba(0, 0, 0, 0.25)

  .qrcode-dialog
    .qrcode-container
      display flex
      justify-content center
    .el-dialog__body
      padding-top 10px
    .copy-picbed-config
      margin-left 10px
      transition $transitionDefault
      &:hover
        transform translateY(-2px)
        box-shadow 0 4px 12px rgba(0, 0, 0, 0.2)

  .fake-title-bar
    -webkit-app-region drag
    height h = 28px
    width 100%
    text-align center
    color #eee
    font-size 13px
    font-weight 500
    line-height h
    position fixed
    z-index 100
    backdrop-filter blur(5px)
    background rgba(30, 34, 42, 0.7)
    box-shadow 0 1px 3px rgba(0, 0, 0, 0.2)

    &.darwin
      background transparent
      background-image linear-gradient(
        to right,
        transparent 0%,
        transparent 167px,
        $darwinBg 167px,
        $darwinBg 100%
      )
      .fake-title-bar__title
        padding-left 167px

    .fake-title-bar__title
      letter-spacing 0.5px

    .handle-bar
      position absolute
      top 4px
      right 8px
      z-index 10000
      -webkit-app-region no-drag
      display flex
      gap 12px

      .control-icon
        cursor pointer
        font-size 16px
        transition $transitionDefault
        border-radius 4px
        padding 2px

        &.always-on-top
          &:hover
            color #409EFF
            background rgba(64, 158, 255, 0.15)

        &.minimize
          &:hover
            color #409EFF
            background $menuHoverBg

        &.close
          &:hover
            color #F15140
            background rgba(241, 81, 64, 0.15)

        &.mini-window
          &:hover
            color #FFC107
            background rgba(255, 193, 7, 0.15)

  .progress-bar
    position fixed
    top 28px
    width 100%
    z-index 90
    transition opacity 0.3s ease

  .main-wrapper
    &.darwin
      background $darwinBg
    padding 10px
    border-radius $borderRadius
    background rgba(30, 34, 42, 0.5)

  .side-bar-menu
    position fixed
    height calc(100vh - 28px)
    overflow-x hidden
    overflow-y auto
    width 142px
    padding-top 35px
    padding-bottom 40px
    transition $transitionDefault
    background rgba(30, 34, 42, 0.5)
    backdrop-filter blur(10px)

    &:hover
      width 145px

  .el-menu
    border-right none
    background transparent
    width 142px

    &-item
      color #eee
      position relative
      border-radius $borderRadius
      margin 4px 8px
      height 40px
      line-height 40px
      transition $transitionDefault

      &:focus,
      &:hover
        color #fff
        background $menuHoverBg

      &.is-active
        color active-color = #409EFF
        background $menuActiveBg
        font-weight 500

        .menu-icon
          color active-color

  .side-menu-item
    padding-left 16px!important

    .menu-icon
      margin-right 8px
      transition $transitionDefault

  .el-sub-menu__title
    color #eee
    border-radius $borderRadius
    margin 4px 8px
    height 40px
    line-height 40px
    transition $transitionDefault

    &:hover
      background $menuHoverBg
      span
        color #fff

    .menu-icon
      margin-right 8px

  .el-sub-menu
    .el-menu-item
      min-width 142px

      &.is-active
        background $menuActiveBg

  .side-submenu-item
    margin-left 0!important
    border-radius $borderRadius
    padding-left 38px!important

  .main-content
    padding-top 28px
    position relative
    height calc(100vh - 28px)
    z-index 10

  .el-dialog__body
    padding 20px

  .support
    text-align center
    &-title
      text-align center
      color #878d99

  .align-center
    input
      text-align center

  *::-webkit-scrollbar
    width 4px
    height 8px

  *::-webkit-scrollbar-thumb
    border-radius 10px
    background rgba(111, 111, 111, 0.5)

    &:hover
      background rgba(140, 140, 140, 0.7)

  *::-webkit-scrollbar-track
    background-color transparent
</style>
