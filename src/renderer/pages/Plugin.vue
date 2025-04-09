<template>
  <div id="plugin-view">
    <div class="view-header">
      <div class="view-title">
        <span>{{ $T('PLUGIN_SETTINGS') }}</span>
        <div class="view-actions">
          <el-tooltip :content="pluginListToolTip" placement="top" :persistent="false" teleported>
            <el-button class="action-button" circle @click="goAwesomeList">
              <el-icon><Goods /></el-icon>
            </el-button>
          </el-tooltip>
          <el-tooltip :content="updateAllToolTip" placement="top" :persistent="false" teleported>
            <el-button class="action-button" circle @click="handleUpdateAllPlugin">
              <el-icon><Refresh /></el-icon>
            </el-button>
          </el-tooltip>
          <el-tooltip :content="importLocalPluginToolTip" placement="top" :persistent="false" teleported>
            <el-button class="action-button" circle @click="handleImportLocalPlugin">
              <el-icon><Download /></el-icon>
            </el-button>
          </el-tooltip>
        </div>
      </div>
      <div class="search-bar">
        <el-input v-model="searchText" :placeholder="$T('PLUGIN_SEARCH_PLACEHOLDER')" size="large" class="search-input">
          <template #prefix>
            <el-icon class="search-icon"><Search /></el-icon>
          </template>
          <template #suffix>
            <el-icon v-if="searchText" class="clear-icon" @click="cleanSearch">
              <Close />
            </el-icon>
          </template>
        </el-input>
      </div>
    </div>

    <div id="pluginList" v-loading="loading" class="plugin-list" element-loading-text="Loading...">
      <el-row :gutter="16">
        <el-col
          v-for="item in pluginList"
          :key="item.fullName"
          class="plugin-item__container"
          :xs="24"
          :sm="12"
          :md="8"
          :lg="8"
          :xl="6"
        >
          <div class="plugin-card" :class="{ disabled: !item.enabled, darwin: osGlobal === 'darwin' }">
            <div v-if="!item.gui" class="cli-only-badge">CLI</div>
            <div class="plugin-header">
              <img class="plugin-logo" :src="item.logo" :onerror="defaultLogo" />
              <div class="plugin-title" @click="openHomepage(item.homepage)">
                <div class="plugin-name">
                  {{ item.name }}
                  <el-tag
                    v-if="latestVersionMap[item.fullName] && latestVersionMap[item.fullName] !== item.version"
                    type="success"
                    size="small"
                    effect="dark"
                    class="version-tag"
                  >
                    new
                  </el-tag>
                </div>
                <div class="plugin-version">v{{ item.version }}</div>
              </div>
            </div>

            <div class="plugin-description" :title="item.description">
              {{ item.description }}
            </div>

            <div class="plugin-footer">
              <div class="plugin-author">
                <el-icon><User /></el-icon>
                {{ item.author.replace(/<.*>/, '') }}
              </div>

              <div class="plugin-actions">
                <template v-if="searchText">
                  <template v-if="!item.hasInstall">
                    <el-button
                      v-if="!item.ing"
                      type="primary"
                      size="small"
                      :loading="item.ing"
                      @click="installPlugin(item)"
                    >
                      {{ $T('PLUGIN_INSTALL') }}
                    </el-button>
                    <el-button v-else type="info" size="small" loading>
                      {{ $T('PLUGIN_INSTALLING') }}
                    </el-button>
                  </template>
                  <el-button v-else type="success" size="small" disabled>
                    {{ $T('PLUGIN_INSTALLED') }}
                  </el-button>
                </template>
                <template v-else>
                  <el-button v-if="item.ing" type="info" size="small" loading>
                    {{ $T('PLUGIN_DOING_SOMETHING') }}
                  </el-button>
                  <template v-else>
                    <el-button v-if="item.enabled" type="primary" size="small" circle @click="buildContextMenu(item)">
                      <el-icon><Tools /></el-icon>
                    </el-button>
                    <el-button v-else type="danger" size="small" circle @click="buildContextMenu(item)">
                      <el-icon><Remove /></el-icon>
                    </el-button>
                  </template>
                </template>
              </div>
            </div>
          </div>
        </el-col>
      </el-row>
    </div>

    <div v-show="needReload" class="reload-mask">
      <el-button type="primary" size="large" @click="reloadApp">
        <el-icon class="reload-icon"><Refresh /></el-icon>
        {{ $T('TIPS_NEED_RELOAD') }}
      </el-button>
    </div>

    <el-dialog
      v-model="dialogVisible"
      :modal-append-to-body="false"
      :title="$T('CONFIG_THING', { c: configName })"
      width="70%"
      class="config-dialog"
      append-to-body
    >
      <config-form :id="configName" ref="$configForm" :config="config" :type="currentType" color-mode="white" />
      <template #footer>
        <el-button @click="dialogVisible = false">
          {{ $T('CANCEL') }}
        </el-button>
        <el-button type="primary" @click="handleConfirmConfig">
          {{ $T('CONFIRM') }}
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import axios from 'axios'
import { ipcRenderer, IpcRendererEvent } from 'electron'
import { ElMessageBox } from 'element-plus'
import { debounce, DebouncedFunc } from 'lodash'
import { Close, Download, Refresh, Goods, Remove, Tools, Search, User } from '@element-plus/icons-vue'
import { computed, ref, onBeforeMount, onBeforeUnmount, watch, onMounted, reactive, toRaw } from 'vue'

import ConfigForm from '@/components/ConfigFormForPlugin.vue'
import { T as $T } from '@/i18n/index'
import { sendRPC } from '@/utils/common'
import { getConfig, saveConfig } from '@/utils/dataSender'
import { osGlobal, updatePicBedGlobal } from '@/utils/global'

import {
  PICGO_CONFIG_PLUGIN,
  PICGO_HANDLE_PLUGIN_ING,
  PICGO_TOGGLE_PLUGIN,
  PICGO_HANDLE_PLUGIN_DONE
} from '#/events/constants'
import { IRPCActionType } from '#/types/enum'
import { handleStreamlinePluginName } from '#/utils/common'
import { configPaths } from '#/utils/configPaths'

const $confirm = ElMessageBox.confirm
const searchText = ref('')
const pluginList = ref<IPicGoPlugin[]>([])
const config = ref<any[]>([])
const currentType = ref<'plugin' | 'uploader' | 'transformer'>('plugin')
const configName = ref('')
const dialogVisible = ref(false)
const pluginNameList = ref<string[]>([])
const loading = ref(true)
const needReload = ref(false)
const latestVersionMap = reactive<{ [key: string]: string }>({})
const pluginListToolTip = $T('PLUGIN_LIST')
const importLocalPluginToolTip = $T('PLUGIN_IMPORT_LOCAL')
const updateAllToolTip = $T('PLUGIN_UPDATE_ALL')
const defaultLogo = ref(`this.src="file://${__static.replace(/\\/g, '/')}/roundLogo.png"`)
const $configForm = ref<InstanceType<typeof ConfigForm> | null>(null)
const npmSearchText = computed(() => {
  return searchText.value.match('picgo-plugin-')
    ? searchText.value
    : searchText.value !== ''
      ? `picgo-plugin-${searchText.value}`
      : searchText.value
})
let getSearchResult: DebouncedFunc<(val: string) => void>

watch(npmSearchText, (val: string) => {
  if (val) {
    loading.value = true
    pluginList.value = []
    getSearchResult(val)
  } else {
    getPluginList()
  }
})

watch(dialogVisible, (val: boolean) => {
  if (val) {
    // @ts-expect-error ts-migrate(2531) FIXME: Object is possibly 'null'.
    document.querySelector('.main-content.el-row').style.zIndex = 101
  } else {
    // @ts-expect-error ts-migrate(2531) FIXME: Object is possibly 'null'.
    document.querySelector('.main-content.el-row').style.zIndex = 10
  }
})

async function getLatestVersionOfPlugIn(pluginName: string) {
  try {
    const res = await axios.get(`https://registry.npmjs.com/${pluginName}`)
    latestVersionMap[pluginName] = res.data['dist-tags'].latest
  } catch (err) {
    console.error(err)
  }
}

onBeforeMount(async () => {
  ipcRenderer.on('hideLoading', () => {
    loading.value = false
  })
  ipcRenderer.on(PICGO_HANDLE_PLUGIN_DONE, (_: IpcRendererEvent, fullName: string) => {
    pluginList.value.forEach(item => {
      if (item.fullName === fullName || item.name === fullName) {
        item.ing = false
      }
    })
    loading.value = false
  })
  ipcRenderer.on('pluginList', (_: IpcRendererEvent, list: IPicGoPlugin[]) => {
    pluginList.value = list
    pluginNameList.value = list.map(item => item.fullName)
    for (const item of pluginList.value) {
      getLatestVersionOfPlugIn(item.fullName)
    }
    loading.value = false
  })
  ipcRenderer.on(
    'installPlugin',
    (
      _: IpcRendererEvent,
      {
        success,
        body
      }: {
        success: boolean
        body: string
      }
    ) => {
      loading.value = false
      pluginList.value.forEach(item => {
        if (item.fullName === body) {
          item.ing = false
          item.hasInstall = success
        }
      })
    }
  )
  ipcRenderer.on('updateSuccess', (_: IpcRendererEvent, plugin: string) => {
    loading.value = false
    pluginList.value.forEach(item => {
      if (item.fullName === plugin) {
        item.ing = false
        item.hasInstall = true
      }
      updatePicBedGlobal()
    })
    handleReload()
    getPluginList()
  })
  ipcRenderer.on('uninstallSuccess', (_: IpcRendererEvent, plugin: string) => {
    loading.value = false
    pluginList.value = pluginList.value.filter(item => {
      if (item.fullName === plugin) {
        // restore Uploader & Transformer after uninstalling
        if (item.config.transformer.name) {
          handleRestoreState('transformer', item.config.transformer.name)
        }
        if (item.config.uploader.name) {
          handleRestoreState('uploader', item.config.uploader.name)
        }
        updatePicBedGlobal()
      }
      return item.fullName !== plugin
    })
    pluginNameList.value = pluginNameList.value.filter(item => item !== plugin)
  })
  ipcRenderer.on(
    PICGO_CONFIG_PLUGIN,
    (_: IpcRendererEvent, _currentType: 'plugin' | 'transformer' | 'uploader', _configName: string, _config: any) => {
      currentType.value = _currentType
      configName.value = _configName
      config.value = _config
      dialogVisible.value = true
    }
  )
  ipcRenderer.on(PICGO_HANDLE_PLUGIN_ING, (_: IpcRendererEvent, fullName: string) => {
    pluginList.value.forEach(item => {
      if (item.fullName === fullName || item.name === fullName) {
        item.ing = true
      }
    })
    loading.value = true
  })
  ipcRenderer.on(PICGO_TOGGLE_PLUGIN, (_: IpcRendererEvent, fullName: string, enabled: boolean) => {
    const plugin = pluginList.value.find(item => item.fullName === fullName)
    if (plugin) {
      plugin.enabled = enabled
      updatePicBedGlobal()
      needReload.value = true
    }
  })
  getPluginList()
  getSearchResult = debounce(_getSearchResult, 50)
  needReload.value = (await getConfig<boolean>(configPaths.needReload)) || false
})

async function buildContextMenu(plugin: IPicGoPlugin) {
  sendRPC(IRPCActionType.SHOW_PLUGIN_PAGE_MENU, plugin)
}

function handleResize() {
  const myDiv = document.getElementById('pluginList') as HTMLElement
  const windowHeight = window.innerHeight
  const headerHeight = 120 // Adjusted for new header layout
  const newHeight = windowHeight - headerHeight - 30
  myDiv.style.height = newHeight + 'px'
}

onMounted(() => {
  window.addEventListener('resize', handleResize)
})

function getPluginList() {
  sendRPC(IRPCActionType.PLUGIN_GET_LIST)
}

function installPlugin(item: IPicGoPlugin) {
  if (!item.gui) {
    $confirm($T('TIPS_PLUGIN_NOT_GUI_IMPLEMENT'), $T('TIPS_NOTICE'), {
      confirmButtonText: $T('CONFIRM'),
      cancelButtonText: $T('CANCEL'),
      type: 'warning'
    })
      .then(() => {
        item.ing = true
        sendRPC(IRPCActionType.PLUGIN_INSTALL, item.fullName)
      })
      .catch(() => {
        console.log('Install canceled')
      })
  } else {
    item.ing = true
    sendRPC(IRPCActionType.PLUGIN_INSTALL, item.fullName)
  }
}

function reloadApp() {
  sendRPC(IRPCActionType.RELOAD_APP)
}

async function handleReload() {
  saveConfig({
    needReload: true
  })
  needReload.value = true
  const successNotification = new Notification($T('PLUGIN_UPDATE_SUCCEED'), {
    body: $T('TIPS_NEED_RELOAD')
  })
  successNotification.onclick = () => {
    reloadApp()
  }
}

function cleanSearch() {
  searchText.value = ''
}

async function handleConfirmConfig() {
  const result = (await $configForm.value?.validate()) || false
  if (result !== false) {
    switch (currentType.value) {
      case 'plugin':
        saveConfig({
          [`${configName.value}`]: result
        })
        break
      case 'uploader':
        saveConfig({
          [`picBed.${configName.value}`]: result
        })
        break
      case 'transformer':
        saveConfig({
          [`transformer.${configName.value}`]: result
        })
        break
    }
    const successNotification = new Notification($T('SETTINGS_RESULT'), {
      body: $T('TIPS_SET_SUCCEED')
    })
    successNotification.onclick = () => {
      return true
    }
    dialogVisible.value = false
    getPluginList()
  }
}

function _getSearchResult(val: string) {
  axios
    .get(`https://registry.npmjs.com/-/v1/search?text=${val}`)
    .then((res: INPMSearchResult) => {
      pluginList.value = res.data.objects
        .filter((item: INPMSearchResultObject) => {
          return item.package.name.includes('picgo-plugin-')
        })
        .map((item: INPMSearchResultObject) => {
          return handleSearchResult(item)
        })
      loading.value = false
    })
    .catch(err => {
      console.log(err)
      loading.value = false
    })
}

function handleSearchResult(item: INPMSearchResultObject) {
  const pkg = item.package
  const name = handleStreamlinePluginName(pkg.name)
  let gui = false
  if (pkg.keywords && pkg.keywords.length > 0) {
    if (pkg.keywords.includes('picgo-gui-plugin')) {
      gui = true
    }
  }
  return {
    name,
    fullName: pkg.name,
    author: pkg.author?.name || pkg.publisher?.username || 'unknown',
    description: pkg.description,
    logo: `https://cdn.jsdelivr.net/npm/${pkg.name}/logo.png`,
    config: {},
    homepage: pkg.links ? pkg.links.homepage : '',
    hasInstall: pluginNameList.value.some(plugin => plugin === pkg.name),
    version: pkg.version,
    gui,
    ing: false // installing or uninstalling
  }
}

// restore Uploader & Transformer
async function handleRestoreState(item: string, name: string) {
  if (item === 'uploader') {
    const current = await getConfig(configPaths.picBed.current)
    if (current === name) {
      saveConfig({
        [configPaths.picBed.current]: 'smms',
        [configPaths.picBed.uploader]: 'smms'
      })
    }
  }
  if (item === 'transformer') {
    const current = await getConfig(configPaths.picBed.transformer)
    if (current === name) {
      saveConfig({
        [configPaths.picBed.transformer]: 'path'
      })
    }
  }
}

function openHomepage(url: string) {
  if (url) {
    sendRPC(IRPCActionType.OPEN_URL, url)
  }
}

function goAwesomeList() {
  sendRPC(IRPCActionType.OPEN_URL, 'https://github.com/PicGo/Awesome-PicGo')
}

function handleImportLocalPlugin() {
  sendRPC(IRPCActionType.PLUGIN_IMPORT_LOCAL)
  loading.value = true
}

function handleUpdateAllPlugin() {
  sendRPC(IRPCActionType.PLUGIN_UPDATE_ALL, toRaw(pluginNameList.value))
}

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  ipcRenderer.removeAllListeners('pluginList')
  ipcRenderer.removeAllListeners('installPlugin')
  ipcRenderer.removeAllListeners('uninstallSuccess')
  ipcRenderer.removeAllListeners('updateSuccess')
  ipcRenderer.removeAllListeners('hideLoading')
  ipcRenderer.removeAllListeners(PICGO_HANDLE_PLUGIN_DONE)
})
</script>
<script lang="ts">
export default {
  name: 'PluginPage'
}
</script>
<style lang="stylus">
$primaryColor = #409EFF
$dangerColor = #F56C6C
$successColor = #67C23A
$warningColor = #E6A23C
$infoColor = #909399
$textColor = #FFFFFF
$subtextColor = #E8EAED
$disabledColor = #A0A0A0
$cardBg = rgba(55, 60, 70, 0.55)
$headerBg = rgba(35, 40, 50, 0.9)
$hoverBg = rgba(65, 70, 85, 0.9)
$borderRadius = 8px
$transitionEase = all 0.3s cubic-bezier(0.4, 0, 0.2, 1)
$boxShadow = 0 4px 12px rgba(0, 0, 0, 0.2)
$hoverShadow = 0 8px 16px rgba(0, 0, 0, 0.3)

#plugin-view
  position absolute
  left 142px
  right 0
  height 100%
  padding 0 16px
  box-sizing border-box
  overflow hidden
  color $textColor

  .view-header
    position sticky
    top 0
    z-index 90
    padding 16px 0
    background linear-gradient(to bottom, rgba(30, 34, 42, 0.95), rgba(30, 34, 42, 0.85))
    backdrop-filter blur(8px)
    border-bottom 1px solid rgba(255, 255, 255, 0.1)

  .view-title
    display flex
    justify-content space-between
    align-items center
    margin-bottom 16px
    font-size 24px
    font-weight 600

    span
      flex 1

    .view-actions
      display flex
      gap 8px

      .action-button
        background rgba(255, 255, 255, 0.1)
        border none
        color $textColor
        transition $transitionEase

        &:hover
          background rgba(255, 255, 255, 0.2)
          transform translateY(-2px)

  .search-bar
    margin-bottom 8px

    .search-input
      .el-input__wrapper
        background rgba(255, 255, 255, 0.1)
        border-radius $borderRadius
        border none
        box-shadow none !important
        transition $transitionEase

        &:hover, &:focus-within
          background rgba(255, 255, 255, 0.15)

      .el-input__inner
        color $textColor

      .search-icon
        color $disabledColor

      .clear-icon
        color $disabledColor
        cursor pointer
        transition $transitionEase

        &:hover
          color $textColor

  .plugin-list
    height calc(100vh - 175px)
    overflow-y auto
    overflow-x hidden
    padding 16px 0  // Increased padding from 8px to 16px
    scroll-behavior smooth

    .el-row
      margin-bottom 0 !important  // Ensure el-row doesn't add its own margins

    .el-loading-mask
      background-color rgba(0, 0, 0, 0.8)
      backdrop-filter blur(4px)

  .plugin-item__container
    margin-bottom 24px  // Increased from 16px to provide more space between rows

    .plugin-card
      background $cardBg
      border-radius $borderRadius
      padding 16px
      min-height 220px
      box-shadow $boxShadow
      transition $transitionEase
      display flex
      flex-direction column
      position relative
      overflow hidden
      border 1px solid rgba(255, 255, 255, 0.1)  // Added subtle border

      &:hover
        transform translateY(-3px)
        box-shadow $hoverShadow
        background $hoverBg
        border-color rgba(255, 255, 255, 0.2)  // Brighter border on hover

    &.disabled
      opacity 0.7

      &:before
        content ""
        position absolute
        top 0
        left 0
        right 0
        bottom 0
        background rgba(0, 0, 0, 0.3)
        z-index 1
        pointer-events none

    .cli-only-badge
      position absolute
      right 0
      top 0
      font-size 11px
      font-weight 600
      padding 4px 10px
      background $warningColor
      color #fff
      z-index 5
      border-bottom-left-radius $borderRadius

  .plugin-header
    display flex
    align-items center
    margin-bottom 12px

    .plugin-logo
      width 48px
      height 48px
      border-radius 12px
      object-fit cover
      background rgba(0, 0, 0, 0.2)
      box-shadow 0 2px 6px rgba(0, 0, 0, 0.2)

    .plugin-title
      margin-left 12px
      cursor pointer
      transition $transitionEase

      &:hover
        color $primaryColor

      .plugin-name
        font-size 16px
        font-weight 600
        display flex
        align-items center
        max-width 200px
        overflow hidden
        text-overflow ellipsis
        white-space nowrap

        .version-tag
          margin-left 8px
          font-size 10px
          height 18px
          line-height 1
          padding 2px 6px
          border-radius 10px

      .plugin-version
        font-size 12px
        color $disabledColor
        margin-top 2px

  .plugin-description
    flex 1
    font-size 14px
    line-height 1.5
    margin-bottom 16px
    display -webkit-box
    -webkit-line-clamp 3
    -webkit-box-orient vertical
    overflow hidden
    color rgba(255, 255, 255, 0.95)
    font-weight 400

  .plugin-footer
    display flex
    justify-content space-between
    align-items center
    margin-top auto
    padding-top 12px
    border-top 1px solid rgba(255, 255, 255, 0.1)

    .plugin-author
      font-size 13px
      color $disabledColor
      display flex
      align-items center
      max-width 60%
      overflow hidden
      text-overflow ellipsis
      white-space nowrap

      .el-icon
        margin-right 6px
        font-size 14px

    .plugin-actions
      display flex
      gap 8px

  .reload-mask
    position fixed
    bottom 0
    left 142px
    right 0
    padding 16px
    background rgba(0, 0, 0, 0.7)
    display flex
    justify-content center
    align-items center
    backdrop-filter blur(8px)
    z-index 100
    box-shadow 0 -2px 10px rgba(0, 0, 0, 0.3)
    animation slideUp 0.3s cubic-bezier(0.4, 0, 0.2, 1)

    .reload-icon
      margin-right 8px

  .config-dialog
    .el-dialog__header
      border-bottom 1px solid rgba(255, 255, 255, 0.1)
      padding-bottom 16px

    .el-dialog__footer
      border-top 1px solid rgba(255, 255, 255, 0.1)
      padding-top 16px

  @keyframes slideUp
    from
      transform translateY(100%)
    to
      transform translateY(0)

  // Scrollbar styling
  *::-webkit-scrollbar
    width 6px
    height 6px

  *::-webkit-scrollbar-thumb
    border-radius 10px
    background rgba(255, 255, 255, 0.2)

    &:hover
      background rgba(255, 255, 255, 0.3)

  *::-webkit-scrollbar-track
    background-color transparent
</style>
