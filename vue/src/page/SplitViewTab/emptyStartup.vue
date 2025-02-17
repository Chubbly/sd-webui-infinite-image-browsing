<script lang="ts" setup>
import { useGlobalStore, type TabPane } from '@/store/useGlobalStore'
import { uniqueId } from 'lodash-es'
import { computed, h, ref } from 'vue'
import { ok } from 'vue3-ts-util'
import { FileDoneOutlined, LockOutlined, PlusOutlined } from '@/icon'
import { t } from '@/i18n'
import { cloneDeep } from 'lodash-es'
import { addScannedPath, removeScannedPath } from '@/api/db'
import { globalEvents } from '@/util'
import { Input, Modal, message } from 'ant-design-vue'
import { open } from '@tauri-apps/api/dialog'
import { checkPathExists } from '@/api'
import { useImgSliStore } from '@/store/useImgSli'
const global = useGlobalStore()
const imgsli = useImgSliStore()
const props = defineProps<{ tabIdx: number; paneIdx: number }>()
const compCnMap: Partial<Record<TabPane['type'], string>> = {
  local: t('local'),
  'tag-search': t('imgSearch'),
  'fuzzy-search': t('fuzzy-search'),
  'global-setting': t('globalSettings'),
  'batch-download': t('batchDownload') + ' / ' + t('archive')
}
const openInCurrentTab = (type: TabPane['type'], path?: string, walkMode = false) => {
  let pane: TabPane
  switch (type) {
    case 'tag-search-matched-image-grid':
    case 'img-sli':
      return
    case 'global-setting':
    case 'tag-search':
    case 'batch-download':
    case 'fuzzy-search':
    case 'empty':
      pane = { type, name: compCnMap[type]!, key: Date.now() + uniqueId() }
      break
    case 'local':
      pane = {
        type,
        name: compCnMap[type]!,
        key: Date.now() + uniqueId(),
        path,
        walkModePath: walkMode ? path : undefined
      }
  }
  const tab = global.tabList[props.tabIdx]
  tab.panes.splice(props.paneIdx, 1, pane)
  tab.key = pane.key
}

const lastRecord = computed(() => global.tabListHistoryRecord?.[1])


const walkModeSupportedDir = computed(() =>
  global.quickMovePaths.filter(
    ({ key: k }) =>
      k === 'outdir_txt2img_samples' ||
      k === 'outdir_img2img_samples'
  )
)
const canpreviewInNewWindow = window.parent !== window
const previewInNewWindow = () => window.parent.open('/infinite_image_browsing' + (window.parent.location.href.includes('theme=dark') ? '?__theme=dark' : ''))

const restoreRecord = () => {
  ok(lastRecord.value)
  global.tabList = cloneDeep(lastRecord.value.tabs)
}

const addToSearchScanPathAndQuickMove = async () => {
  let path: string
  if (import.meta.env.TAURI_ARCH) {
    const ret = await open({ directory: true })
    if (typeof ret === 'string') {
      path = ret
    } else {
      return
    }
  } else {
    path = await new Promise<string>((resolve) => {
      const key = ref('')
      Modal.confirm({
        title: t('inputTargetFolderPath'),
        content: () => {
          return h(Input, {
            value: key.value,
            'onUpdate:value': (v: string) => (key.value = v)
          })
        },
        async onOk () {
          const path = key.value
          const res = await checkPathExists([path])
          if (res[path]) {
            resolve(key.value)
          } else {
            message.error(t('pathDoesNotExist'))
          }
        }
      })
    })
  }
  Modal.confirm({
    content: t('confirmToAddToQuickMove'),
    async onOk () {
      await addScannedPath(path)
      message.success(t('addCompleted'))
      globalEvents.emit('searchIndexExpired')
      globalEvents.emit('updateGlobalSetting')
    }
  })
}

const onRemoveQuickPathClick = (path: string) => {
  Modal.confirm({
    content: t('confirmDelete'),
    closable: true,
    async onOk () {
      await removeScannedPath(path)
      message.success(t('removeCompleted'))
      globalEvents.emit('searchIndexExpired')
      globalEvents.emit('updateGlobalSetting')
    }
  })
}
</script>
<template>
  <div class="container">
    <div class="header">
      <h1>{{ $t('welcome') }}</h1>
      <div v-if="global.conf?.enable_access_control && global.dontShowAgain" style="margin-left: 16px;font-size: 1.5em;">
        <LockOutlined title="Access Control mode" style="vertical-align: text-bottom;" />
      </div>
      <div flex-placeholder />
      <a href="https://github.com/zanllp/sd-webui-infinite-image-browsing" target="_blank" class="last-record">{{
        $t('document') }}</a>
      <a href="https://github.com/zanllp/sd-webui-infinite-image-browsing/issues/131" target="_blank"
        class="last-record">{{ $t('changlog') }}</a>
      <a href="https://github.com/zanllp/sd-webui-infinite-image-browsing/issues/90" target="_blank"
        class="last-record">{{ $t('faq') }}</a>
    </div>
    <a-alert show-icon v-if="global.conf?.enable_access_control && !global.dontShowAgain">
      <template #message>
        <div class="access-mode-message">
          <div>
            {{ $t('accessControlModeTips') }}
          </div>
          <div flex-placeholder />
          <a @click.prevent="global.dontShowAgain = true">{{ $t('dontShowAgain') }}</a>
        </div>
      </template>
      <template #icon>
        <LockOutlined></LockOutlined>
      </template>
    </a-alert>
    <a-alert show-icon v-if="!global.dontShowAgainNewImgOpts">
      <template #message>
        <div class="access-mode-message">
          <div>
            {{ $t('majorUpdateCustomCellSizeTips') }}
          </div>
          <div flex-placeholder />
          <a @click.prevent="global.dontShowAgainNewImgOpts = true">{{ $t('dontShowAgain') }}</a>
        </div>
      </template>
    </a-alert>
    <div class="content">
      <div class="feature-item" v-if="walkModeSupportedDir.length">
        <h2>{{ $t('walkMode') }}</h2>
        <ul>
          <li v-for="item in walkModeSupportedDir" :key="item.dir" class="item">
            <AButton @click="openInCurrentTab('local', item.dir, true)" ghost type="primary" block>{{ item.zh }}</AButton>
          </li>
        </ul>
      </div>
      <div class="feature-item" v-if="global.quickMovePaths.length">
        <h2>{{ $t('launchFromQuickMove') }}</h2>
        <ul>
          <li @click="addToSearchScanPathAndQuickMove" class="item" style="text-align: ;">
            <span class="text line-clamp-1">
              <PlusOutlined /> {{ $t('add') }}
            </span>
          </li>
          <li v-for="dir in global.quickMovePaths" :key="dir.key" class="item rem"
            @click.prevent="openInCurrentTab('local', dir.dir)">
            <span class="text line-clamp-1">{{ dir.zh }}</span>
            <AButton v-if="dir.can_delete" type="link" @click.stop="onRemoveQuickPathClick(dir.dir)">{{ $t('remove') }}
            </AButton>
          </li>
        </ul>
      </div>
      <div class="feature-item">
        <h2>{{ $t('launch') }}</h2>
        <ul>
          <li v-for="comp in Object.keys(compCnMap) as TabPane['type'][]" :key="comp" class="item"
            @click.prevent="openInCurrentTab(comp)">
            <span class="text line-clamp-1">{{ compCnMap[comp] }}</span>
          </li>
          <li class="item" @click="imgsli.opened = true">
            <span class="text line-clamp-1">{{ $t('imgCompare') }}</span>
          </li>
          <li class="item" v-if="canpreviewInNewWindow" @click="previewInNewWindow">
            <span class="text line-clamp-1">{{ $t('openInNewWindow') }}</span>
          </li>
          <li class="item" v-if="lastRecord?.tabs.length" @click="restoreRecord">
            <span class="text line-clamp-1">{{ $t('restoreLastRecord') }}</span>
          </li>
        </ul>
      </div>
      <div class="feature-item recent" v-if="global.recent.length">
        <div class="title">
          <h2>{{ $t('recent') }}</h2>
          <AButton @click="global.recent = []" type="link">{{ $t('clear') }}</AButton>
        </div>
        <ul>
          <li v-for="item in global.recent" :key="item.key" class="item"
            @click.prevent="openInCurrentTab('local', item.path)">
            <FileDoneOutlined class="icon" />
            <span class="text line-clamp-1">{{ item.path }}</span>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.access-mode-message {
  display: flex;
  flex-direction: row;
  align-items: center;

  a {
    margin-left: 16px;
  }
}

.container {
  padding: 20px;
  background-color: var(--zp-secondary-background);
  height: 100%;
  overflow: auto;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header h1 {
  font-size: 28px;
  font-weight: bold;
  color: var(--zp-primary);
  margin: 0;
}

.last-record {
  margin-left: 16px;
  font-size: 14px;
  color: var(--zp-secondary);
}

.last-record a {
  text-decoration: none;
  color: var(--zp-secondary);
}

.last-record a:hover {
  color: var(--zp-primary);
}

.content {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  grid-gap: 20px;
  margin-top: 16px;
}

.feature-item {
  background-color: var(--zp-primary-background);
  border-radius: 8px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  padding: 20px;

  ul {
    list-style: none;
    padding: 4px;
    max-height: 70vh;
    overflow-y: auto;
  }

  &.recent {
    .title {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 20px;

      h2 {
        margin: 0;
      }
    }
  }

  .item {
    margin-bottom: 10px;
    padding: 4px 8px;
    display: flex;
    align-items: center;

    &.rem {
      display: flex;
      align-items: center;
      justify-content: space-between;
    }

    &:hover {
      background: var(--zp-secondary-background);
      border-radius: 4px;
      color: var(--primary-color);
      cursor: pointer;
    }
  }

  .icon {
    margin-right: 8px;
  }
}

.feature-item h2 {
  margin-top: 0;
  margin-bottom: 20px;
  font-size: 20px;
  font-weight: bold;
  color: var(--zp-primary);
}


.text {
  flex: 1;
  font-size: 16px;
}
</style>
