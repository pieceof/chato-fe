<script setup lang="ts">
import SpaceRightsFreeExpUpgrate from '@/components/Space/SpaceRightsFreeExpUpgrate.vue'
import SpaceRightsMask from '@/components/Space/SpaceRightsMask.vue'
import useGlobalProperties from '@/composables/useGlobalProperties'
import useSpaceRights from '@/composables/useSpaceRights'
import { currentEnvConfig } from '@/config'
import { FreeCommercialTypeExperienceDay } from '@/constant/space'
import { EAppletcStatus } from '@/enum/release'
import { ESpaceCommercialType, ESpaceRightsType } from '@/enum/space'
import { useBase } from '@/stores/base'
import { useDomainStore } from '@/stores/domain'
import { useSpaceStore } from '@/stores/space'
import { copyPaste } from '@/utils/help'
import { getSpecifiedDateSinceNowDay } from '@/utils/timeRange'
import {
  CirclePlus,
  CopyDocument,
  Document,
  Tools,
  UploadFilled,
  View
} from '@element-plus/icons-vue'
import { useSessionStorage } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import {
  computed,
  defineAsyncComponent,
  onMounted,
  onUnmounted,
  reactive,
  ref,
  toRefs,
  watch,
  type Ref
} from 'vue'
import { useI18n } from 'vue-i18n'
import { useRoute } from 'vue-router'
import ReleaseBox from '../releaseView/components/ReleaseBox.vue'

const ApplicationForm = defineAsyncComponent(
  () => import('../releaseView/components/application/ApplicationForm.vue')
)
const BrandDomain = defineAsyncComponent(
  () => import('./components/brandDomain/BrandDomainIndex.vue')
)
const DrawerSite = defineAsyncComponent(() => import('./components/implantJs/DrawerSite.vue'))
const SetEffectSite = defineAsyncComponent(() => import('./components/implantJs/SetEffectSite.vue'))
const Copylink = defineAsyncComponent(() => import('./components/webPage/Copylink.vue'))
const CreatePoster = defineAsyncComponent(() => import('./components/gzhPoster/SharePoster.vue'))
const ExperienceApplet = defineAsyncComponent(
  () => import('./components/applet/ExperienceApplet.vue')
)
const CreateApplet = defineAsyncComponent(() => import('./components/applet/CreateApplet.vue'))
const VerificationTxt = defineAsyncComponent(
  () => import('./components/applet/VerificationTxt.vue')
)

const route = useRoute()
const { t } = useI18n()
const base = useBase()
const spaceStoreI = useSpaceStore()
const domainStoreI = useDomainStore()
const { userInfo, orgInfo, userCommercialType } = storeToRefs(base)
const { currentRights } = storeToRefs(spaceStoreI)
const { domainInfo } = storeToRefs(domainStoreI)
const botSlug = computed(() => domainInfo.value.slug)
const botId = computed(() => domainInfo.value.id)
const chatWebPageBaseURL = `${currentEnvConfig.baseURL}`
const chatReleaseURL = computed(() => {
  return {
    chatWebPage: `${currentEnvConfig.wwwBaseURL}/b/${botSlug.value}`,
    chatAPI: `${chatWebPageBaseURL}/chato/api-public/domains/${botSlug.value}/chat`
  }
})
const chatScript = `${currentEnvConfig.scriptURL}/assets/iframe.min.js`
const defaultAppletcStatus = ref<EAppletcStatus>()
const releaseChannel = useSessionStorage('releaseChannel', '')

const specifiedBetweenDay = getSpecifiedDateSinceNowDay(orgInfo.value.created)
const rightsMaskVisible = computed(
  () =>
    currentRights.value.type === ESpaceCommercialType.free &&
    specifiedBetweenDay > FreeCommercialTypeExperienceDay
)

const features = reactive({
  showCopyLinkVisbile: false, // 网页-复制链接
  siteListVisible: false, // 站点列表
  createSiteVisible: false, // 创建站点
  brandDomainVisible: false, // 域名部署
  createPoster: false, // 海报
  createAppletVisible: false, // 小程序-扫码授权
  drawerAppletVisible: false, // 小程序-查看授权结果
  linkAppletVisible: false, // 小程序-链接小程序
  empowerAppletVisible: false, // 小程序-覆盖已有的
  verificationAppletVisible: false // 小程序-嵌入已有
})

const {
  showCopyLinkVisbile,
  siteListVisible,
  createSiteVisible,
  brandDomainVisible,
  createAppletVisible,
  createPoster,
  drawerAppletVisible,
  linkAppletVisible,
  empowerAppletVisible,
  verificationAppletVisible
} = toRefs(features)

const { checkRightsTypeNeedUpgrade } = useSpaceRights()

// 蒙层
const maskVisible = computed(() => {
  return !base.orgInfo.visible && ESpaceCommercialType.free === userCommercialType.value
})

const handleUpdateOrgInfo = () => {
  base.updateOrgInfo({
    ...orgInfo.value,
    visible: true
  })
}

// 预览体验
const handlePreview = (e: string) => {
  window.open(e)
}

const commonVisible = (visibleRef: Ref<boolean>, show: boolean = false) => {
  if (show) {
    checkRightsTypeNeedUpgrade(ESpaceRightsType.default, true)
  }
  visibleRef.value = !show
}

// 域名部署
const brandDomain = async () => {
  if (!currentRights.value.dns) {
    checkRightsTypeNeedUpgrade(ESpaceRightsType.brand)
    return
  }
  brandDomainVisible.value = true
}

const { $sensors } = useGlobalProperties()
const releaseList = [
  {
    icon: 'wangye',
    title: t('网页'),
    desc: t('用户在此链接可以直接和您的机器人聊天'),
    setList: [
      {
        icon: CopyDocument,
        label: t('复制链接'),
        scriptId: 'Chato-copy-link',
        click: () => commonVisible(showCopyLinkVisbile)
      },
      {
        icon: View,
        label: t('预览体验'),
        scriptId: 'Chato-preview-chat',
        click: () => handlePreview(chatReleaseURL.value.chatWebPage)
      },
      {
        icon: UploadFilled,
        label: t('域名部署'),
        scriptId: 'Chato-brand-domain',
        click: brandDomain
      }
    ]
  },
  {
    icon: 'applet',
    title: t('微信小程序'),
    desc: t('支持企业通过以下三种不同的方式，将机器人应用在微信小程序中'),
    setList: [
      {
        icon: Tools,
        label: t('获取小程序链接'),
        scriptId: 'Chato-applet-link',
        click: () => commonVisible(linkAppletVisible)
      },
      {
        icon: View,
        label: t('嵌入已有小程序'),
        scriptId: 'Chato-applet-iframe',
        click: () => commonVisible(verificationAppletVisible, rightsMaskVisible.value)
      },
      {
        icon: Document,
        label: t('覆盖已有小程序'),
        scriptId: 'Chato-applet-cover',
        click: () => commonVisible(empowerAppletVisible, rightsMaskVisible.value)
      }
    ]
  },
  {
    icon: 'wechat-pyq',
    title: t('朋友圈'),
    desc: t('用户扫码后，可直接和您的机器人聊天'),
    setList: [
      {
        icon: View,
        label: t('生成海报'),
        scriptId: 'Chato-preview-chat',
        click: () => {
          commonVisible(createPoster)
          $sensors.track('bot_poster_click', {
            bot_id: botId.value
          })
        }
      }
    ]
  },
  {
    icon: 'js-qianru',
    title: t('JS嵌入'),
    desc: t('可添加到网站的任何位置，将此 iframe 添加到 html 代码中'),
    setList: [
      {
        icon: CirclePlus,
        label: t('创建站点'),
        scriptId: 'Chato-setted-effect-site',
        click: () => commonVisible(createSiteVisible)
      },
      {
        icon: View,
        label: t('查看代码'),
        scriptId: 'Chato-copy-code',
        click: () => commonVisible(siteListVisible)
      }
    ]
  }
]

const upgradationName = ['wangye', 'applet']

const handleCopyButtonClick = (e: MouseEvent) => {
  e.stopPropagation()
  copyPaste((e.target as HTMLElement).querySelector('.text').innerHTML)
}

watch(
  () => route.query,
  (v) => {
    if (v.releaseChannel && v.releaseChannel !== releaseChannel.value) {
      releaseChannel.value = v.releaseChannel as string
      defaultAppletcStatus.value = EAppletcStatus.empowerSuccess
      createAppletVisible.value = true
    }
  },
  { immediate: true }
)

onMounted(() => {
  const observer = new MutationObserver((mutationsList) => {
    for (let mutation of mutationsList) {
      if (mutation.type === 'childList') {
        const copyBtns = document.querySelectorAll('.copy-btn-code')
        // 移除旧的点击事件监听器
        copyBtns.forEach((btn) => {
          btn.removeEventListener('click', handleCopyButtonClick)
        })

        // 添加新的点击事件监听器
        copyBtns.forEach((btn) => {
          btn.addEventListener('click', handleCopyButtonClick)
        })
      }
    }
  })

  observer.observe(document.body, { childList: true, subtree: true })

  onUnmounted(() => {
    observer.disconnect()
  })
})
</script>

<template>
  <div class="release-drawer-container">
    <template v-if="!maskVisible">
      <div class="mx-auto my-0">
        <div class="overflow-hidden flex flex-wrap justify-between">
          <ReleaseBox
            v-for="item in releaseList"
            :key="item.icon"
            :svgName="item.icon"
            :title="item.title"
            :desc="item.desc"
            class="relative"
          >
            <div
              class="icon-set-container text-[#b5bed0] cursor-pointer gap-2 text-xs flex items-center justify-center mr-[16px] md:mr-[6px] md:mb-2"
              @click="ic.click"
              v-for="ic in item.setList"
              :key="ic.label"
            >
              <el-icon>
                <component :is="ic.icon" />
              </el-icon>
              {{ ic.label }}
            </div>
            <SpaceRightsMask :visible="rightsMaskVisible && !upgradationName.includes(item.icon)">
              <SpaceRightsFreeExpUpgrate upgrade-link upgrade-text="该功能为付费权益" />
            </SpaceRightsMask>
          </ReleaseBox>
        </div>
      </div>
    </template>
    <ApplicationForm
      v-else
      @handleUpdateOrgInfo="handleUpdateOrgInfo"
      :orgId="userInfo.org.id"
      :orgUserId="userInfo.id"
    />
    <BrandDomain :slug="botSlug" v-model:value="brandDomainVisible" />
    <DrawerSite
      v-model:value="siteListVisible"
      :slug="botSlug"
      :chatScript="chatScript"
      :chatWebPage="chatReleaseURL.chatWebPage"
      :wwwBaseURL="currentEnvConfig.wwwBaseURL"
      :baseURL="chatWebPageBaseURL"
    >
    </DrawerSite>
    <SetEffectSite
      v-model:value="createSiteVisible"
      :slug="domainInfo.slug"
      @showDrawerSite="() => (siteListVisible = true)"
    />
    <Copylink v-model:value="showCopyLinkVisbile" :chatWebPage="chatReleaseURL.chatWebPage" />
    <CreatePoster v-model:value="createPoster" :domainId="botId" />
    <DrawerApplet
      v-model:value="drawerAppletVisible"
      :domainId="domainInfo.id"
      @handleRetry="createAppletVisible = true"
    />
    <ExperienceApplet v-model:value="linkAppletVisible" :slug="domainInfo.slug" />
    <CreateApplet
      v-model:value="empowerAppletVisible"
      :domainId="domainInfo.id"
      :defaultAppletcStatus="defaultAppletcStatus"
      @handleView="drawerAppletVisible = true"
    />
    <VerificationTxt
      v-model:value="verificationAppletVisible"
      :chatAPI="chatReleaseURL.chatWebPage"
    />
  </div>
</template>

<style lang="scss" scoped>
.icon-set-container {
  &:hover {
    @apply text-[#7c5cfc];
  }
}
</style>

<style lang="scss">
.release-drawer-container {
  .el-drawer__header {
    @apply mb-5 text-[#303133] font-medium;
  }
  .el-drawer__body {
    @apply pt-0;
  }
  .el-collapse {
    @apply border-t-0;
  }
  .el-form-item__label {
    @apply text-sm text-[#303133];
  }
}
</style>
