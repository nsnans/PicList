<template>
  <el-form
    label-position="top"
    require-asterisk-position="right"
    size="default"
    :model="waterMarkForm"
    class="image-process-form"
  >
    <div class="settings-grid">
      <!-- Left Column -->
      <div class="settings-column">
        <!-- Watermark Settings Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><Picture /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISADDWM') }}</h3>
          </div>

          <div class="section-content">
            <el-form-item class="toggle-control">
              <el-switch v-model="waterMarkForm.isAddWatermark" :style="switchStyle" />
              <span class="toggle-label">{{ waterMarkForm.isAddWatermark ? $T('ENABLE') : $T('DISABLE') }}</span>
            </el-form-item>

            <div v-show="waterMarkForm.isAddWatermark" class="subsection">
              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMTYPE')">
                <el-radio-group v-model="waterMarkForm.watermarkType" class="radio-group">
                  <el-radio value="text">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_WMTYPE_TEXT') }}</el-radio>
                  <el-radio value="image">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_WMTYPE_IMAGE') }}</el-radio>
                </el-radio-group>
              </el-form-item>

              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_ISFULLSCREEN_WM')" class="toggle-control">
                <el-switch v-model="waterMarkForm.isFullScreenWatermark" :style="switchStyle" />
              </el-form-item>

              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMPOSITION')">
                <el-radio-group v-model="waterMarkForm.watermarkPosition" class="position-grid">
                  <el-radio v-for="item in waterMarkPositionMap" :key="item[0]" :value="item[0]">
                    {{ item[1] }}
                  </el-radio>
                </el-radio-group>
              </el-form-item>

              <div class="form-row">
                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMDEGREE')">
                  <el-input-number v-model="waterMarkForm.watermarkDegree" :step="1" controls-position="right" />
                </el-form-item>

                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMRATIO')">
                  <el-input-number
                    v-model="waterMarkForm.watermarkScaleRatio"
                    :min="0"
                    :max="1"
                    :step="0.01"
                    controls-position="right"
                  />
                </el-form-item>
              </div>

              <!-- Text Watermark Options -->
              <div v-show="waterMarkForm.watermarkType === 'text'" class="subsection variant-subsection">
                <h4 class="subsection-title">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_WMTYPE_TEXT') }}</h4>
                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMTEXT')">
                  <el-input v-model="waterMarkForm.watermarkText" />
                </el-form-item>

                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMTEXT_FONT_PATH')">
                  <el-input v-model="waterMarkForm.watermarkFontPath" placeholder="/path/to/font.ttf" />
                </el-form-item>

                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMCOLOR')">
                  <div class="color-picker-container">
                    <el-color-picker v-model="waterMarkForm.watermarkColor" show-alpha />
                    <span class="color-value">{{ waterMarkForm.watermarkColor }}</span>
                  </div>
                </el-form-item>
              </div>

              <!-- Image Watermark Options -->
              <div v-show="waterMarkForm.watermarkType === 'image'" class="subsection variant-subsection">
                <h4 class="subsection-title">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_WMTYPE_IMAGE') }}</h4>
                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_WMPATH')">
                  <el-input v-model="waterMarkForm.watermarkImagePath" placeholder="/path/to/watermark.png" />
                </el-form-item>
              </div>
            </div>
          </div>
        </section>

        <!-- Format Conversion Settings Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><Document /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISCONVERT') }}</h3>
          </div>

          <div class="section-content">
            <el-form-item class="toggle-control">
              <el-switch v-model="compressForm.isConvert" :style="switchStyle" />
              <span class="toggle-label">{{ compressForm.isConvert ? $T('ENABLE') : $T('DISABLE') }}</span>
            </el-form-item>

            <div v-show="compressForm.isConvert" class="subsection">
              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_CONVERTFORMAT')">
                <el-select v-model="compressForm.convertFormat" :persistent="false" teleported class="full-width">
                  <el-option v-for="item in availableFormat" :key="item" :label="item" :value="item" />
                </el-select>
              </el-form-item>

              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_CONVERTFORMAT_SPECIFIC')">
                <el-input
                  v-model="formatConvertObj"
                  placeholder='{"jpg": "png", "png": "jpg"}'
                  type="textarea"
                  :autosize="{ minRows: 2, maxRows: 4 }"
                  class="code-input"
                />
              </el-form-item>
            </div>
          </div>
        </section>
      </div>

      <!-- Right Column -->
      <div class="settings-column">
        <!-- Image Processing Settings Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><Edit /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_QUALITY') }}</h3>
          </div>

          <div class="section-content">
            <el-form-item class="toggle-control">
              <el-switch v-model="compressForm.isRemoveExif" :style="switchStyle" />
              <span class="toggle-label">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISREMOVEEXIF') }}</span>
            </el-form-item>

            <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_QUALITY')">
              <el-slider v-model="compressForm.quality" :min="1" :max="100" :step="1" show-input class="full-width" />
            </el-form-item>
          </div>
        </section>

        <!-- Resize Settings Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><ZoomIn /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISRESIZE') }}</h3>
          </div>

          <div class="section-content">
            <el-form-item class="toggle-control">
              <el-switch v-model="compressForm.isReSize" :style="switchStyle" />
              <span class="toggle-label">{{ compressForm.isReSize ? $T('ENABLE') : $T('DISABLE') }}</span>
            </el-form-item>

            <div v-show="compressForm.isReSize" class="subsection">
              <div class="form-row">
                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_RESIZEWIDTH')">
                  <el-input-number v-model="compressForm.reSizeWidth" :min="0" controls-position="right" />
                </el-form-item>

                <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_RESIZEHEIGHT')">
                  <el-input-number v-model="compressForm.reSizeHeight" :min="0" controls-position="right" />
                </el-form-item>
              </div>

              <el-form-item
                v-show="compressForm.reSizeHeight! > 0 && compressForm.reSizeWidth === 0"
                :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_SKIPRESIZEOfSMALLIMG_HEIGHT')"
                class="toggle-control"
              >
                <el-switch v-model="compressForm.skipReSizeOfSmallImg" :style="switchStyle" />
              </el-form-item>

              <el-form-item
                v-show="compressForm.reSizeWidth! > 0 && compressForm.reSizeHeight === 0"
                :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_SKIPRESIZEOfSMALLIMG_WIDTH')"
                class="toggle-control"
              >
                <el-switch v-model="compressForm.skipReSizeOfSmallImg" :style="switchStyle" />
              </el-form-item>
            </div>
          </div>
        </section>

        <!-- Resize by Percent Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><ScaleToOriginal /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISRESIZEBYPERCENT') }}</h3>
          </div>

          <div class="section-content">
            <el-form-item class="toggle-control">
              <el-switch v-model="compressForm.isReSizeByPercent" :style="switchStyle" />
              <span class="toggle-label">{{ compressForm.isReSizeByPercent ? $T('ENABLE') : $T('DISABLE') }}</span>
            </el-form-item>

            <div v-show="compressForm.isReSizeByPercent" class="subsection">
              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_RESIZEPERCENT')">
                <el-slider
                  v-model="compressForm.reSizePercent"
                  :min="1"
                  :max="100"
                  :step="1"
                  show-input
                  class="full-width"
                />
              </el-form-item>
            </div>
          </div>
        </section>

        <!-- Flip and Rotation Section -->
        <section class="settings-section">
          <div class="section-header">
            <el-icon><Refresh /></el-icon>
            <h3>{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_TRANSFORMATION') }}</h3>
          </div>

          <div class="section-content">
            <div class="form-row">
              <el-form-item class="toggle-control">
                <el-switch v-model="compressForm.isFlip" :style="switchStyle" />
                <span class="toggle-label">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISFLIP') }}</span>
              </el-form-item>

              <el-form-item class="toggle-control">
                <el-switch v-model="compressForm.isFlop" :style="switchStyle" />
                <span class="toggle-label">{{ $T('UPLOAD_PAGE_IMAGE_PROCESS_ISFLOP') }}</span>
              </el-form-item>
            </div>

            <el-form-item class="toggle-control">
              <el-switch v-model="compressForm.isRotate" :style="switchStyle" />
              <span class="toggle-label">{{
                compressForm.isRotate
                  ? $T('UPLOAD_PAGE_IMAGE_PROCESS_ISROTATE')
                  : $T('UPLOAD_PAGE_IMAGE_PROCESS_ISROTATE')
              }}</span>
            </el-form-item>

            <div v-show="compressForm.isRotate" class="subsection">
              <el-form-item :label="$T('UPLOAD_PAGE_IMAGE_PROCESS_ROTATEDEGREE')">
                <div class="rotation-control">
                  <el-input-number v-model="compressForm.rotateDegree" :step="1" controls-position="right" />
                  <div class="rotation-buttons">
                    <el-button size="small" @click="compressForm.rotateDegree = 0">0°</el-button>
                    <el-button size="small" @click="compressForm.rotateDegree = 90">90°</el-button>
                    <el-button size="small" @click="compressForm.rotateDegree = 180">180°</el-button>
                    <el-button size="small" @click="compressForm.rotateDegree = 270">270°</el-button>
                  </div>
                </div>
              </el-form-item>
            </div>
          </div>
        </section>
      </div>
    </div>

    <!-- Action Buttons -->
    <div class="form-actions">
      <el-button type="primary" @click="handleSaveConfig">
        {{ $T('UPLOAD_PAGE_IMAGE_PROCESS_CONFIRM') }}
      </el-button>
      <el-button @click="closeDialog">
        {{ $T('UPLOAD_PAGE_IMAGE_PROCESS_CANCEL') }}
      </el-button>
    </div>
  </el-form>
</template>

<script lang="ts" setup>
import { IBuildInCompressOptions, IBuildInWaterMarkOptions } from 'piclist'
import { onBeforeMount, reactive, ref, toRaw } from 'vue'
import { Picture, Edit, Document, ZoomIn, ScaleToOriginal, Refresh } from '@element-plus/icons-vue'

import { T as $T } from '@/i18n/index'
import { getConfig, saveConfig } from '@/utils/dataSender'
import { configPaths } from '#/utils/configPaths'

const imageProcessDialogVisible = defineModel<boolean>()

const waterMarkPositionMap = new Map([
  ['northwest', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_TOP_LEFT')],
  ['north', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_TOP')],
  ['northeast', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_TOP_RIGHT')],
  ['west', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_LEFT')],
  ['centre', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_CENTER')],
  ['east', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_RIGHT')],
  ['southwest', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_BOTTOM_LEFT')],
  ['south', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_BOTTOM')],
  ['southeast', $T('UPLOAD_PAGE_IMAGE_PROCESS_POSITION_BOTTOM_RIGHT')]
])
const imageExtList = ['jpg', 'jpeg', 'png', 'webp', 'bmp', 'tiff', 'tif', 'svg', 'ico', 'avif', 'heif', 'heic']
const availableFormat = [
  'avif',
  'dz',
  'fits',
  'gif',
  'heif',
  'input',
  'jpeg',
  'jpg',
  'jp2',
  'jxl',
  'magick',
  'openslide',
  'pdf',
  'png',
  'ppm',
  'raw',
  'svg',
  'tiff',
  'tif',
  'v',
  'webp'
]

const waterMarkForm = reactive<IBuildInWaterMarkOptions>({
  isAddWatermark: false,
  watermarkType: 'text',
  isFullScreenWatermark: false,
  watermarkDegree: 0,
  watermarkText: '',
  watermarkFontPath: '',
  watermarkScaleRatio: 0.15,
  watermarkColor: '#CCCCCC73',
  watermarkImagePath: '',
  watermarkPosition: 'southeast'
})
const compressForm = reactive<IBuildInCompressOptions>({
  quality: 100,
  isConvert: false,
  convertFormat: 'jpg',
  isReSize: false,
  reSizeWidth: 500,
  reSizeHeight: 500,
  skipReSizeOfSmallImg: false,
  isReSizeByPercent: false,
  reSizePercent: 50,
  isRotate: false,
  rotateDegree: 0,
  isRemoveExif: false,
  isFlip: false,
  isFlop: false
})
const formatConvertObj = ref('{}')

const waterMarkFormKeys = Object.keys(waterMarkForm) as (keyof typeof waterMarkForm)[]
const compressFormKeys = Object.keys(compressForm) as (keyof typeof compressForm)[]

const switchStyle = '--el-switch-on-color: #13ce66; --el-switch-off-color: #ff4949;'

function handleSaveConfig() {
  let iformatConvertObj = {}
  try {
    iformatConvertObj = JSON.parse(formatConvertObj.value)
  } catch (error) {}
  const formatConvertObjEntries = Object.entries(iformatConvertObj)
  const formatConvertObjEntriesFilter = formatConvertObjEntries.filter((item: any) => {
    return imageExtList.includes(item[0]) && availableFormat.includes(item[1])
  })
  const formatConvertObjFilter = Object.fromEntries(formatConvertObjEntriesFilter)
  formatConvertObj.value = JSON.stringify(formatConvertObjFilter)
  compressForm.formatConvertObj = formatConvertObjFilter
  saveConfig(configPaths.buildIn.compress, toRaw(compressForm))
  saveConfig(configPaths.buildIn.watermark, toRaw(waterMarkForm))
  closeDialog()
}

async function initData() {
  const compress = await getConfig<any>(configPaths.buildIn.compress)
  const watermark = await getConfig<any>(configPaths.buildIn.watermark)
  if (compress) {
    compressFormKeys.forEach(key => {
      compressForm[key] = compress[key] ?? compressForm[key]
    })
    try {
      if (typeof compress.formatConvertObj === 'object') {
        formatConvertObj.value = JSON.stringify(compress.formatConvertObj)
      } else {
        formatConvertObj.value = compress.formatConvertObj ?? '{}'
      }
    } catch (error) {
      formatConvertObj.value = '{}'
    }
  }
  if (watermark) {
    waterMarkFormKeys.forEach(key => {
      waterMarkForm[key] = watermark[key] ?? waterMarkForm[key]
    })
    waterMarkForm.watermarkColor = watermark.watermarkColor === '' ? '#CCCCCC73' : watermark.watermarkColor
  }
}

function closeDialog() {
  imageProcessDialogVisible.value = false
}

onBeforeMount(async () => {
  await initData()
})
</script>

<style lang="stylus" scoped>
.image-process-form {
  max-height: 85vh;
  overflow-y: auto;
  padding: 16px;
  scrollbar-width: thin;

  &::-webkit-scrollbar {
    width: 6px;
  }

  &::-webkit-scrollbar-thumb {
    background: rgba(144, 147, 153, 0.3);
    border-radius: 6px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
  }
}

.settings-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
}

.settings-column {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.settings-section {
  background: var(--el-bg-color-overlay, #ffffff);
  border-radius: 10px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.06);
  overflow: hidden;
  transition: all 0.25s ease;
  border: 1px solid var(--el-border-color-lighter, #EBEEF5);

  &:hover {
    box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.08);
    transform: translateY(-2px);
  }
}

.section-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 18px;
  background: linear-gradient(90deg, var(--el-color-primary-light-9, #ecf5ff) 0%, transparent 100%);
  border-bottom: 1px solid var(--el-border-color-light, #E4E7ED);
  border-left: 4px solid var(--el-color-primary, #409eff);

  .el-icon {
    color: var(--el-color-primary, #409eff);
    font-size: 18px;
  }

  h3 {
    margin: 0;
    font-size: 16px;
    font-weight: 600;
    color: var(--el-text-color-primary, #303133);
  }
}

.section-content {
  padding: 18px;
}

.subsection {
  background: var(--el-fill-color-light, #f5f7fa);
  border-radius: 8px;
  padding: 16px;
  margin-top: 10px;
  animation: fade-in 0.3s ease;
}

.variant-subsection {
  background: var(--el-color-primary-light-9, #ecf5ff);
  border-left: 3px solid var(--el-color-primary-light-5, #a0cfff);
}

.subsection-title {
  font-size: 14px;
  font-weight: 500;
  margin: 0 0 12px 0;
  color: var(--el-color-primary, #409eff);
  border-bottom: 1px dashed var(--el-border-color-light, #E4E7ED);
  padding-bottom: 8px;
}

.toggle-control {
  display: flex;
  align-items: center;
  margin-bottom: 14px;

  :deep(.el-form-item__content) {
    display: flex;
    align-items: center;
  }

  .toggle-label {
    margin-left: 10px;
    font-size: 14px;
    color: var(--el-text-color-regular, #606266);
  }
}

.form-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 16px;
}

.position-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;

  :deep(.el-radio) {
    margin-right: 0;
    margin-bottom: 8px;

    .el-radio__label {
      font-size: 13px;
    }
  }
}

.color-picker-container {
  display: flex;
  align-items: center;
  gap: 12px;
}

.color-value {
  font-family: monospace;
  background: var(--el-fill-color-lighter, #f5f7fa);
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 13px;
}

.rotation-control {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
}

.rotation-buttons {
  display: flex;
  gap: 8px;

  .el-button {
    transition: transform 0.2s;

    &:hover {
      transform: scale(1.05);
    }
  }
}

.code-input {
  font-family: monospace;
  font-size: 13px;
}

.helper-text {
  font-size: 12px;
  color: var(--el-text-color-secondary, #909399);
  margin-top: 4px;
  padding-left: 2px;
}

.full-width {
  width: 100%;
}

.form-actions {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-top: 28px;
  padding: 16px 0;
  background: linear-gradient(to top, var(--el-bg-color, #ffffff) 60%, transparent);
  position: sticky;
  bottom: 0;
  z-index: 10;

  .el-button {
    min-width: 120px;
    transition: all 0.3s;

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    }
  }
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(-5px); }
  to { opacity: 1; transform: translateY(0); }
}

@media (max-width: 768px) {
  .settings-grid {
    grid-template-columns: 1fr;
  }

  .position-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .form-actions {
    flex-direction: column;
    align-items: stretch;

    .el-button {
      width: 100%;
    }
  }
}
</style>
