<template>
  <div v-loading="loading">
    <div
      v-if="
        !pagination.page_count &&
        DocSelectStatus === LearningStatesPerformanceType.all &&
        !searchInput
      "
      class="flex flex-col items-center justify-center mt-[20vh]"
    >
      <img
        :src="emptydocImg"
        class="w-[332px] h-[196px] object-cover mb-12 lg:w-[249px] lg:h-[147px] lg:mb-8"
      />
      <p class="text-[#9DA3AF] text-xs mb-6">
        {{ $t('您还没有录入文档，快去录入吧！') }}
      </p>
      <el-button plain @click="() => (dialogVisible = true)">
        {{ $t('录入文档') }}
      </el-button>
    </div>
    <div v-else class="pt-14 relative -mt-[67px] lg:mt-0 lg:pt-0">
      <div
        class="absolute top-0 right-0 flex justify-end items-center gap-4 lg:relative lg:top-auto lg:right-auto lg:justify-start lg:mb-6"
      >
        <el-button @click="handleTriggerMate">{{ $t('批量操作') }}</el-button>
        <el-button
          v-if="batchRemove && EAllRole.superman === userInfo.role"
          :disabled="!multipleSelection.length"
          @click="onDocToQA"
          link
          class="ml-4 text-[#303133]"
        >
          <el-icon class="mr-1"><Notification /></el-icon>
          {{ $t('生成训练问答') }}
        </el-button>
        <el-button
          :disabled="!multipleSelection.length"
          v-if="batchRemove"
          @click="handleBatchRemove"
          link
          class="ml-[16px] text-[#303133]"
        >
          <el-icon class="mr-[4px]"><Delete /></el-icon>
          {{ $t(' 删除') }}
        </el-button>
        <SearchInput v-model:value="searchInput" />
        <el-button type="primary" @click="() => (dialogVisible = true)">
          {{ $t('录入文档') }}
        </el-button>
      </div>
      <DocLearnTable
        :batchRemove="batchRemove"
        v-model:selectStatus="DocSelectStatus"
        v-model:loading="loading"
        v-model:multipleSelection="multipleSelection"
        :domainId="domainId"
        :learnCount="learnCount"
        :doc-list="(tableData as IDocumentList[])"
        v-model:pagination="pagination"
        @remove-doc="onRemoveDoc"
        @edit-preview-doc="onEditPreviewDoc"
        @generateQA="onGenerateQA"
        @generateQARouter="onGenerateQARouter"
      />
    </div>
    <EnterDoc
      :domainId="domainId"
      :defaultForm="currentEdit"
      :sizeLimit="sizeLimit"
      :qtyLimit="qtyLimit"
      :apiUpload="apiUpload"
      :dialogVisible="dialogVisible"
      @setSuccess="onCloseDialog"
      @reloadList="initDocList"
      @closeDialogVisble="onCloseDialog"
    />
    <DocToQAModal v-model:visible="docToQAModalVisible" :ids="multipleIds" />
  </div>
</template>
<script lang="ts" setup>
import { deleteRetryFileMate, getFilesByDomainId, postGenerateDocAPI } from '@/api/file'
import EnterDoc from '@/components/EnterAnswer/EnterDoc.vue'
import SearchInput from '@/components/Input/SearchInput.vue'
import useImagePath from '@/composables/useImagePath'
import useSpaceRights from '@/composables/useSpaceRights'
import { currentEnvConfig } from '@/config'
import { USER_ROLES } from '@/constant/common'
import { KnowledgeLearningFinalStatus } from '@/constant/knowledge'
import {
  DeleteRetryFileMateStatusType,
  EDocumentOperateType,
  EKnowledgeBusinessType,
  LearningStatesPerformanceType
} from '@/enum/knowledge'
import { ESpaceRightsType } from '@/enum/space'
import { EAllRole } from '@/enum/user'
import type { IPage } from '@/interface/common'
import type { GetFilesByDomainIdType, IDocumentForm, IDocumentList } from '@/interface/knowledge'
import { RoutesMap } from '@/router'
import { useBase } from '@/stores/base'
import { useDomainStore } from '@/stores/domain'
import * as url from '@/utils/url'
import { debouncedWatch } from '@vueuse/core'
import { ElLoading, ElMessageBox, ElNotification } from 'element-plus'
import { storeToRefs } from 'pinia'
import { computed, onUnmounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRoute, useRouter } from 'vue-router'
import DocLearnTable from './DocLearnTable.vue'
import DocToQAModal from './DocToQAModal.vue'

const { t } = useI18n()
const { ImagePath: emptydocImg } = useImagePath('empty-doc')
const { checkRightsTypeNeedUpgrade } = useSpaceRights()

let timer = null
const base = useBase()
const route = useRoute()
const router = useRouter()
const domainStoreI = useDomainStore()
const { domainInfo } = storeToRefs(domainStoreI)
const { userInfo } = storeToRefs(base)
const domainId = computed(() => domainInfo.value.id || (route.params.botId as string))
const qtyLimit = base.userInfo.role === USER_ROLES.SUPERMAN ? 1000 : 20 // 同时上传的文件数量限制
const sizeLimit = 30 // 单个文件的体积限制（MB）
const apiUpload = computed(() =>
  url.join(
    currentEnvConfig.uploadBaseURL,
    `/chato/api/domains/${domainId.value}/files/upload/document`
  )
)
const learnCount = ref<{
  success: number
  total: number
}>({
  success: 0,
  total: 0
})
const serachStatus = ref(false)
const searchInput = ref<string>('')
const multipleSelection = ref<IDocumentList[]>([])
const DocSelectStatus = ref(LearningStatesPerformanceType.all)
const batchRemove = ref<boolean>(false)
const loading = ref(true)
const tableData = ref<IDocumentList[]>([])
const pagination = ref<IPage>({
  page: 1,
  page_size: 10,
  total: 0,
  page_count: 0
})

const currentEdit = ref<IDocumentForm>({})
const dialogVisible = ref(false)

const multipleIds = computed(() => multipleSelection.value.map((item) => item.id))

const onEditPreviewDoc = (value: any, type: EDocumentOperateType) => {
  if (type === EDocumentOperateType.retry) {
    deteleOrRetryFile(DeleteRetryFileMateStatusType.pending, [value.id])
    return
  }
  dialogVisible.value = true
  currentEdit.value = {
    title: value.title,
    content_html: value.content_html,
    id: value.id,
    status: type
  }
}

const onCloseDialog = (refresh = true) => {
  dialogVisible.value = false
  refresh && initDocList()
  currentEdit.value = {}
}

const onRemoveDoc = async (fileId: number, message: string) => {
  await initDocList()
  ElNotification({
    type: 'success',
    message
  })
}

const initDocList = async () => {
  loading.value = true
  try {
    const params: GetFilesByDomainIdType = {
      page: pagination.value.page,
      business_type: EKnowledgeBusinessType.document,
      keyword: searchInput.value
    }
    if (DocSelectStatus.value !== LearningStatesPerformanceType.all) {
      params.status = DocSelectStatus.value
    }
    const {
      data: { data, meta }
    } = await getFilesByDomainId(domainId.value, params)
    tableData.value = data
    pagination.value.page_count = meta.pagination.page_count
  } catch (err) {
  } finally {
    loading.value = false
    serachStatus.value = false
  }
}

const handleTriggerMate = () => {
  batchRemove.value = !batchRemove.value
  if (batchRemove.value) {
    clearInterval(timer)
  } else {
    timer = setInterval(() => {
      initDocList()
    }, 10000)
  }
}

const deteleOrRetryFile = async (type: DeleteRetryFileMateStatusType, ids: number[]) => {
  const loading = ElLoading.service({
    lock: true,
    text: type === DeleteRetryFileMateStatusType.deleted ? t('删除中...') : t('重试中...'),
    background: 'rgba(0, 0, 0, 0.7)'
  })
  const data = {
    status: type,
    ids: ids
  }
  const res = await deleteRetryFileMate(domainId.value, data)
  loading.close()

  return {
    code: res.data.code,
    message: res.data.msg || res.data.message
  }
}

const docToQAModalVisible = ref(false)
const onDocToQA = () => {
  docToQAModalVisible.value = true
}

const handleBatchRemove = () => {
  if (!multipleSelection.value.length) return
  ElMessageBox.confirm(t('您确认确删除这些文件吗？'), t('温馨提示'), {
    confirmButtonText: t('确认'),
    cancelButtonText: t('取消'),
    type: 'warning'
  })
    .then(async () => {
      const ids = multipleSelection.value.map((item) => item.id)
      const res = await deteleOrRetryFile(DeleteRetryFileMateStatusType.deleted, ids)
      const code = res.code === 200
      const message = res.message
      initDocList()
      ElNotification({
        type: code ? 'success' : 'error',
        message: code ? t('删除成功') : message
      })
    })
    .catch(() => {})
}

const onGenerateQA = async (id: number) => {
  const res = await postGenerateDocAPI(id)
  loading.value = false
  if (!res.data.data) {
    return checkRightsTypeNeedUpgrade(ESpaceRightsType.default)
  }
  ElNotification.success(t('问答生成成功'))
  initDocList()
}

const onGenerateQARouter = (id: number) => {
  router.push({
    name: RoutesMap.tranning.knowledgeGenerate,
    query: { docId: id }
  })
}

debouncedWatch(searchInput, () => initDocList(), { debounce: 300 })

watch(
  tableData,
  () => {
    if (!tableData.value.length) {
      clearInterval(timer)
      return
    }

    const startTagStudy = tableData.value.some(
      (item) => !KnowledgeLearningFinalStatus.includes(item.status)
    )

    const startTagStudyConvert = tableData.value.some((item) => item.qa_status === 1)
    if (startTagStudy || startTagStudyConvert) {
      if (!timer) {
        timer = setInterval(() => {
          initDocList()
        }, 10000)
      }
    } else {
      clearInterval(timer)
    }
  },
  { deep: true }
)

watch(
  [() => pagination.value.page, domainId, DocSelectStatus],
  ([page, v1, v2]) => {
    if ((page || v1 || v2) && domainId.value) {
      initDocList()
    }
  },
  { immediate: true }
)

onUnmounted(() => {
  clearInterval(timer)
})
</script>
<style lang="scss" scoped>
:deep(.el-message-box) {
  --el-messagebox-width: 461px;
}

.tab-describe {
  font-size: 12px;
  color: #9da3af;
  line-height: 20px;

  a {
    color: #7c5cfc;
  }
}
</style>
