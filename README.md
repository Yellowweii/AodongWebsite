```vue
<!-- eslint-disable max-lines-->
<template>
  <div v-loading="isUploading" class="search-by-resource" element-loading-text="正在上传中">
    <div v-show="progressShow" class="progress_pop">
      <el-progress :percentage="progressling"></el-progress>
      <span v-if="progressling < 100" class="progress_size">下载中请稍后······</span>
      <span v-else class="progress_size">下载完成!</span>
    </div>
    <el-form label-width="auto">
      <el-form-item label="创意状态">
        <el-select v-model="creativeStatus" :popper-append-to-body="false">
          <el-option
            v-for="item in CREATIVE_STATUS_OPTIONS"
            :key="item.creativeStatusValue"
            :label="item.creativeStatusLabel"
            :value="item.creativeStatusValue"
          />
        </el-select>
      </el-form-item>
      <el-form-item label="DSP名称">
        <el-select
          v-model="dspName"
          :popper-append-to-body="false"
          :loading="loadingDSP"
          filterable
          @change="updateSwitchDspTime"
        >
          <el-option
            v-for="option in dspNameOptions"
            :key="`${option.dspTypeId}_${option.taskCategory}`"
            :label="option.dspTypeName"
            :value="`${option.dspTypeId}_${option.taskCategory}`"
          />
        </el-select>
      </el-form-item>
      <el-form-item label="资源类型">
        <el-select v-model="resourceType" :popper-append-to-body="false">
          <el-option
            v-for="option in RESOURCE_TYPE_OPTIONS"
            :key="option.type"
            :label="option.label"
            :value="option.type"
          >
          </el-option>
        </el-select>
      </el-form-item>
      <el-form-item label="广告主行业">
        <el-cascader
          v-model="advertiserIndustry"
          :options="advertiserIndustryList"
          :props="EL_CASCADER_PROPS"
          collapse-tags
          clearable
          :disabled="isAdvertiserIndustryDisabled"
          filterable
          :append-to-body="false"
        />
      </el-form-item>
      <el-form-item label="机审行业">
        <el-cascader
          v-model="machineAuditIndustry"
          :options="machineAuditIndustryList"
          :props="EL_CASCADER_PROPS"
          collapse-tags
          clearable
          filterable
          :append-to-body="false"
        />
      </el-form-item>
      <el-form-item label="时间">
        <el-date-picker
          v-model="dateTime"
          type="daterange"
          :disabled-date="isDisabledDates"
          :editable="false"
          :append-to-body="false"
          @calendar-change="handleCanlendarChange"
          @focus="focusDate"
          @change="handleConfirmDatePicker"
        />
      </el-form-item>
      <el-form-item label="品牌名称">
        <el-input v-model="brandName" placeholder="请输入搜索品牌名称" clearable />
      </el-form-item>
      <el-form-item label="title/ocr">
        <el-input v-model="ocrText" placeholder="请输入搜索文本" clearable />
      </el-form-item>
      <el-form-item label="图片/视频">
        <div class="image-or-video-box">
          <!--上传图片组件-->
          <el-upload
            class="upload-demo"
            drag
            :disabled="isUploading"
            :action="UPLOAD_IMAGE_OR_VIDEO_REQUEST_URL"
            :before-upload="beforeUpload"
            :on-success="onSuccess"
            :show-file-list="false"
            :headers="requestHeader"
          >
            <input v-model="tempImageOrVideoUrl" placeholder="请上传搜索图片或视频" class="el-input" readonly />
            <div class="icon-group">
              <el-icon v-if="tempImageOrVideoUrl" class="close-icon" @click="clearImg">
                <el-icon-close />
              </el-icon>
              <el-icon v-if="isSearching" class="loading-icon">
                <el-icon-loading />
              </el-icon>
              <el-icon v-else class="upload-icon">
                <el-icon-camera />
              </el-icon>
            </div>
            <template #tip>
              <div class="el-upload__tip">点击上传图片/视频或拖拽图片/视频到此处</div>
            </template>
          </el-upload>
          <!--搜索按钮-->
          <el-button icon="el-icon-search" type="primary" @click.stop="getCreativeMachineAuditResults">搜索</el-button>
          <!--错误信息提示-->
          <span v-if="errorMsg" class="errorMsg">{{ UPLOAD_ERROR_MESSAGE }}</span>
          <!--图片展示区域-->
          <div v-if="tempImageOrVideoUrl" class="image-show-div" @click="previewImageOrVideo">
            <img v-if="isImageUrl(tempImageOrVideoUrl)" :src="imageOrVideoUrl" class="image-show" />
            <video v-else :src="imageOrVideoUrl" class="image-show" controls />
          </div>
        </div>
      </el-form-item>
    </el-form>

    <div v-if="!machineAuditResultsList.length" class="no-data">
      <img class="no-data-icon" src="/public/static/img/warning.png" />
      <div>暂无数据</div>
    </div>
    <div v-else>
      <!--全选按钮-->
      <div class="select-all-box">
        <el-checkbox v-model="isSelectAllMachineAuditResults" @change="handleAllMachineAuditResultsSelect">
          全选
        </el-checkbox>
        <div class="creative-select-box" @click="handleAllCreativesSelect">
          <el-icon :style="{ color: isSelectedAllCreatives ? '#20b759' : '#bfbfbf' }">
            <el-icon-success-filled />
          </el-icon>
          <span>创意全选</span>
        </div>
      </div>
      <!--机审结果-->
      <el-checkbox-group
        v-model="currentPageSelectedMAResultIds"
        v-loading.fullscreen.lock="machineAuditResultsLoading"
        element-loading-text="拼命加载中"
        element-loading-background="rgba(0, 0, 0, 0.8)"
        element-loading-spinner="el-icon-loading"
        @change="handleCheckboxGroupChange"
      >
        <div class="creative-group-box">
          <div
            v-for="machineAuditResult in machineAuditResultsList"
            :key="machineAuditResult.creativeId"
            class="creative-box"
          >
            <div class="left-box">
              <div
                v-if="machineAuditResult.resourceType.includes('img')"
                class="image"
                @click="previewMachineAuditResult(machineAuditResult)"
              >
                <img title="图片支持点击放大查看" :src="getSafeRequestUrl(machineAuditResult.fdfsPath)" />
              </div>
              <div v-else class="video" @click="previewMachineAuditResult(machineAuditResult)">
                <video controls :src="getSafeRequestUrl(machineAuditResult.fdfsPath)" title="视频支持点击放大查看" />
              </div>
              <div class="review-result" :title="getHoverTitle(machineAuditResult.testResult)">
                审核结果：{{ $filterDisplayFilter(machineAuditResult.testResult, 'testResult') }}
              </div>
              <el-icon
                class="select-icon"
                :style="{ color: machineAuditResult.isSelectCreative ? '#20b759' : '#bfbfbf' }"
                title="选中创意"
                @click="handleCreativeSelect(machineAuditResult)"
              >
                <el-icon-success-filled />
              </el-icon>
            </div>
            <div class="right-box">
              <div :title="machineAuditResult.title">
                {{ machineAuditResult.title }}
              </div>
              <div :title="machineAuditResult.creativeId" class="creative-id">
                <span class="creativeId-box">
                  <span class="creative-id-text">创意ID:</span>
                  <span class="creative-id-content">
                    {{ machineAuditResult.creativeId }}
                  </span>
                </span>
                <i
                  class="deeplink-btn"
                  title="复制创意ID"
                  @click="copyCode(machineAuditResult.creativeId ? machineAuditResult.creativeId : '', '.deeplink-btn')"
                />
              </div>
              <div :title="machineAuditResult.label" class="result-text">
                <div>品牌名称:</div>
                <div>{{ machineAuditResult.label ? machineAuditResult.label : machineAuditResult.brand }}</div>
              </div>
              <div :title="machineAuditResult.positionId" class="result-text">
                <div>展示版位:</div>
                <div>{{ machineAuditResult.positionId }}</div>
              </div>
              <div :title="machineAuditResult.dealIds" class="result-text">
                <div>订单ID:</div>
                <div>{{ machineAuditResult.dealIds }}</div>
              </div>
              <div
                v-if="machineAuditResult.hasOwnProperty('manualScore')"
                :title="machineAuditResult.manualScore"
                class="result-text"
              >
                <div>人审分数:</div>
                <div>{{ machineAuditResult.manualScore }}</div>
              </div>
              <div
                v-if="machineAuditResult.hasOwnProperty('manualReason')"
                :title="machineAuditResult.manualReason"
                class="result-text"
              >
                <div class="machine-audit-text">人审原因:</div>
                <div class="machine-audit-factor">{{ machineAuditResult.manualReason }}</div>
              </div>
              <div
                v-if="!machineAuditResult.hasOwnProperty('manualScore')"
                :title="machineAuditResult.ppsScore"
                class="result-text"
              >
                <div>机审分数:</div>
                <div>{{ machineAuditResult.ppsScore }}</div>
              </div>
              <div
                v-if="!machineAuditResult.hasOwnProperty('manualReason')"
                :title="machineAuditResult.ppsReason"
                class="result-text"
              >
                <div class="machine-audit-text">机审原因:</div>
                <div class="machine-audit-factor">{{ machineAuditResult.ppsReason }}</div>
              </div>
              <div
                v-if="machineAuditResult.advertiserIndustry && !machineAuditResult.hasOwnProperty('manualReason')"
                :title="wideReviewConverting(machineAuditResult.advertiserIndustry)"
                class="result-text"
              >
                <div>广告主行业:</div>
                <div>{{ wideReviewConverting(machineAuditResult.advertiserIndustry) }}</div>
              </div>
              <div
                v-if="machineAuditResult.machineIndustry && !machineAuditResult.hasOwnProperty('manualReason')"
                :title="wideReviewConverting(machineAuditResult.machineIndustry)"
                class="result-text"
              >
                <div>机审行业:</div>
                <div>{{ wideReviewConverting(machineAuditResult.machineIndustry) }}</div>
              </div>
            </div>
            <div class="select-checkbox">
              <el-checkbox title="选中资源" :value="machineAuditResult.id" />
            </div>
          </div>
        </div>
      </el-checkbox-group>
      <!--分页和操作按钮-->
      <div class="footer-content">
        <div class="btn-box">
          <el-button type="primary" :disabled="progressShow" plain @click="downloadMaterials"> 批量下载资源 </el-button>
          <el-button class="reject-button" type="primary" plain @click="reviewComments()">
            {{ creativeStatus === CreativeStatusEnum.FAIL ? '批量通过' : '批量驳回' }}
          </el-button>
        </div>
        <AI-pagination
          v-if="machineAuditResultsTotal > 0"
          :total-num="machineAuditResultsTotal"
          :page-size-options="PAGE_SIZE_OPTIONS"
          :page-size="pageSize"
          :current-page="pageNum"
          @page-num-change="handlePageNumChange"
        />
      </div>
    </div>

    <!--提交确认弹框-->
    <el-dialog v-model="downloadImg" title="提示" width="300px">
      <p class="rejected">请至少选择一条抽样数据！</p>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="downloadImg = false">确定</el-button>
        </div>
      </template>
    </el-dialog>
    <!--提交确认弹框-->
    <el-dialog v-model="commitDialog" title="确认结果" width="350px">
      <p class="rejected">请选择驳回数据！</p>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="commitDialog = false">确定</el-button>
        </div>
      </template>
    </el-dialog>
    <!--提交确认弹框-->
    <el-dialog v-model="commitDialogbo" title="确认结果" width="280px">
      <p class="rejected">确定驳回？</p>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="danger" @click="submitReviewResult">确定</el-button>
          <el-button @click="cancels">取消</el-button>
        </div>
      </template>
    </el-dialog>
    <!--投放媒体详情弹框-->
    <el-dialog v-model="mediaDialog" title="投放媒体" width="350px">
      <p>投放媒体列表</p>
    </el-dialog>
    <!--审核意见弹框-->
    <el-dialog v-model="examineDialog" title="驳回意见" width="785px" top="5vh">
      <div
        style="
          display: flex;
          justify-content: space-between;
          margin-bottom: 20px;
          align-items: center;
          border-top: 1px solid #eee;
          border-bottom: 1px solid #eee;
          padding: 0 20px;
        "
      >
        <!-- 添加意见 -->
        <div style="display: flex; align-items: center; line-height: 22px" @click="addOpinion">
          <el-icon class="add">
            <el-icon-plus />
          </el-icon>
          <span style="font-size: 16px; cursor: pointer">添加意见</span>
        </div>
      </div>
      <div class="display-box" style="height: 500px; overflow-y: scroll">
        <!-- 开始 -->
        <div v-for="(item, index) in openAuditDetail" :key="index + '5'">
          <div style="display: flex; justify-content: space-between; align-items: center; padding-right: 25px">
            <el-select
              v-model="item.materialId"
              placeholder="素材"
              clearable
              style="margin: 0 20px; width: 205px"
              @change="materialTypeChange($event, item)"
            >
              <el-option
                v-for="item in openMaterialOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              ></el-option>
            </el-select>
            <!-- 缩略图 -->
            <img
              v-if="!item.showImgSrc === ''"
              :src="getSafeRequestUrl(item.showImgSrc)"
              style="height: 30px; width: 60px; margin-left: -340px"
            />
            <!-- 删除意见 -->
            <el-icon @click="delOpinion(index)">
              <el-icon-close />
            </el-icon>
          </div>
          <div class="top-content">
            <el-select
              v-model="item.score"
              style="width: 205px"
              placeholder="评分"
              clearable
              @change="auditScoreChange(item)"
            >
              <el-option v-for="(item, i) in scoreOption" :key="i + '3'" :label="item.label" :value="item.value">
              </el-option>
            </el-select>

            <el-select
              v-model="item.firstReason"
              placeholder="一级原因"
              clearable
              style="margin: 0 20px; width: 205px"
              @change="reasonChange(item)"
            >
              <el-option
                v-for="(item, n) in item.firstReasonOption"
                :key="n + '2'"
                :label="item.label"
                :value="item.value"
              >
              </el-option>
            </el-select>

            <el-select
              v-model="item.secondReason"
              placeholder="二级原因"
              clearable
              style="width: 205px"
              @change="secondChange(item)"
            >
              <el-option
                v-for="(item, q) in item.secondReasonOption"
                :key="q + '1'"
                :label="item.label"
                :value="item.value"
              >
              </el-option>
            </el-select>
          </div>
          <div class="input-content">
            <el-select
              v-model="item.thirdReason"
              placeholder="三级原因"
              clearable
              filterable
              style="margin-right: 20px; width: 205px"
            >
              <el-option
                v-for="(item, p) in item.thirdReasonOption"
                :key="p + '0'"
                :label="item.label"
                :value="item.value"
              >
              </el-option>
            </el-select>
            <!-- 意见描述 -->
            <el-select
              v-show="item.isShow"
              v-model="item.selectDesc"
              filterable
              placeholder="意见描述"
              class="opinionDes"
              :disabled="item.score === 40"
              @change="checkOpinoin(item)"
            >
              <el-option v-for="item in opinionList" :key="item" style="width: 380px" :label="item" :value="item">
              </el-option>
            </el-select>
            <el-icon
              v-show="item.score !== 40 ? item.isShow : false"
              style="margin-left: 10px; margin-top: 10px"
              @click="item.isShow = false"
            >
              <el-icon-edit-pen />
            </el-icon>
            <div v-show="!item.isShow" class="display-box">
              <div class="selectReason">
                <div class="selectReason3">
                  <el-input v-model.trim="item.desc" style="width: 380px" placeholder="意见描述"></el-input>
                </div>
                <div class="selectIndex">
                  <el-icon style="margin-left: 43px; margin-top: 6px; cursor: pointer" @click="cleanReason(item)">
                    <el-icon-delete />
                  </el-icon>
                </div>
              </div>
            </div>
          </div>
          <el-divider v-if="openAuditDetail.length > 1"></el-divider>
        </div>
        <!-- 结束 -->
      </div>
      <template #footer>
        <div class="dialog-footer" style="text-align: center">
          <el-button @click="examineDialog = false">取消</el-button>
          <el-button type="primary" @click="saveCurEdit()">提交</el-button>
        </div>
      </template>
    </el-dialog>
    <!--预览图片/视频弹窗-->
    <el-dialog v-model="previewDialog" class="previewDialog" :show-close="false" width="fit-content" top="0vh">
      <div class="preview-box">
        <img
          v-if="previewMediaType.type === MediaTypeEnum.IMAGE"
          ref="mediaTypeRef"
          :src="previewMediaType.mediaTypeUrl"
          @mousewheel="zoomImageOrVideo($event)"
        />
        <video
          v-else
          ref="mediaTypeRef"
          :src="previewMediaType.mediaTypeUrl"
          controls
          @mousewheel="zoomImageOrVideo($event)"
        />
        <div class="image-or-video-percentage">{{ imageOrVideoPercentage }}</div>
        <el-icon class="close-btn" @click="handlePreviewClose">
          <el-icon-circle-close />
        </el-icon>
      </div>
    </el-dialog>
  </div>
</template>

<!-- eslint-disable max-lines-->
<!-- eslint-disable no-mixed-operators -->
<!-- eslint-disable max-lines-per-function -->
<!-- eslint-disable curly -->
<script setup lang="ts">
import { getSafeRequestUrl } from '@hw-security/xss-protect'
import qs from 'qs'
import _axios from '@/service/axios'
import Clipboard from 'clipboard'
import { onMounted, watch, ref, getCurrentInstance, onUnmounted } from 'vue'
import api from '@/service/sampleSystem'
import store from '@/store/index'
import { ElMessage } from 'element-plus'
import {
  COMMODITY_LIBRARY_DSP_OPTION,
  CREATIVE_STATUS_OPTIONS,
  DEFAULT_PAGE_SELECTED_CREATIVEIDS,
  DEFAULT_PAGE_SELECTED_MARESULTIDS,
  DEFAULT_PAGENUM,
  DEFAULT_PAGESIZE,
  DSP_NAME_OPRTIONS,
  EL_CASCADER_PROPS,
  HUAWEI_DSP_OPTION,
  MAX_BRAND_NAME_LENGTH,
  MAX_ORCTEXT_LENGTH,
  MONTH_DAYS,
  ONE_DAY_MILLISECONDS,
  PAGE_SIZE_OPTIONS,
  RESOURCE_TYPE_OPTIONS,
  SUPPORTED_IMAGE_TYPES,
  SUPPORTED_VIDEO_TYPES,
  UPLOAD_ERROR_MESSAGE,
  UPLOAD_IMAGE_OR_VIDEO_REQUEST_URL,
  WEEK_DAYS,
  YEAR_DAYS,
  ZOOM_CONFIG,
} from '@/constants/searchByResource'
import type {
  AdvertiserIndustryProps,
  AllPageSelectedCreativeIdsProps,
  AllPageSelectedMAResultIdsProps,
  DspNameoOptionProps,
  MachineAuditIndutryProps,
  MachineAuditResultsListProps,
  MachineAuditResultsProps,
  MarkPaginationApiResponse,
  PreviewMediaTypeProps,
  SavePageNumSelectedIdsProps,
} from '@/interfaces/search-by-resource'
import {
  CreativeStatusEnum,
  MediaTypeEnum,
  ResourceTypeEnum,
  SelectedIdFieldEnum,
} from '@/interfaces/search-by-resource'
import type { PageNumChangeParams } from '@/interfaces/ai-pagination'

const creativeStatus = ref<CreativeStatusEnum>(CreativeStatusEnum.ALL)
const dspName = ref<string>(`${DSP_NAME_OPRTIONS[0].dspTypeId}_${DSP_NAME_OPRTIONS[0].taskCategory}`)
const loadingDSP = ref<boolean>(false)
const dspNameOptions = ref<Array<DspNameoOptionProps>>(DSP_NAME_OPRTIONS)
const resourceType = ref<ResourceTypeEnum>(ResourceTypeEnum.ALL)
const advertiserIndustry = ref([])
const advertiserIndustryList = ref<Array<AdvertiserIndustryProps>>([])
const isAdvertiserIndustryDisabled = ref<boolean>(false)
const machineAuditIndustry = ref([])
const machineAuditIndustryList = ref<Array<MachineAuditIndutryProps>>([])
const dateTime = ref<[string, string]>(['', ''])
const brandName = ref<string>('')
const ocrText = ref<string>('')
const isUploading = ref<boolean>(false)
const tempImageOrVideoUrl = ref<string>('')
const imageOrVideoUrl = ref<string>('')
const previewDialog = ref<boolean>(false)
const previewMediaType = ref<PreviewMediaTypeProps>({ type: MediaTypeEnum.IMAGE, mediaTypeUrl: '' })
const mediaTypeRef = ref()
const imageOrVideoPercentage = ref<string>(`${ZOOM_CONFIG.default}%`)
const isSearching = ref<boolean>(false)
const errorMsg = ref<boolean>(false)
const firstDate = ref<number>(0)
const timerDate = ref()

const isSelectAllMachineAuditResults = ref<boolean>(false)
const isSelectedAllCreatives = ref<boolean>(false)
const pageNum = ref<number>(DEFAULT_PAGENUM)
const pageSize = ref<number>(DEFAULT_PAGESIZE)
const batchId = ref<string>('') // 当前批号
const machineAuditResultsTotal = ref<number>(0)
const machineAuditResultsList = ref<Array<MachineAuditResultsListProps>>([])
const currentPageAllMAResultIds = ref<Array<string>>([])
const currentPageSelectedMAResultIds = ref<Array<string>>([])
const currentPageSelectedCreativeIds = ref<Array<string>>([])
const machineAuditResultsLoading = ref<boolean>(false)
const previousPageNum = ref<number>(0)
const allPageSelectedMAResultIds = ref<Array<AllPageSelectedMAResultIdsProps>>([DEFAULT_PAGE_SELECTED_MARESULTIDS])
const allPageSelectedCreativeIds = ref<Array<AllPageSelectedCreativeIdsProps>>([DEFAULT_PAGE_SELECTED_CREATIVEIDS])

const checkTotalList = ref([]) // 全部页面选中的id
const downloadImg = ref(false)
const checkTotalErrList = ref([]) // 全部人工判定通过id的整合列表

const taskCategory = ref('')
const isSelect = ref(false)

const mediaDialog = ref(false)
const examineDialog = ref(false)
const commitDialog = ref(false)
const commitDialogbo = ref(false)

const reviewer = ref('') // 当前复核人

const ppsScore = ref(0)
const openAuditDetail = ref([])
const openMaterialOptions = ref([])
const scoreOption = ref([
  { value: 0, label: 0 },
  { value: 5, label: 0.5 },
  { value: 10, label: 1 },
  { value: 20, label: 2 },
  { value: 30, label: 3 },
  { value: 40, label: 4 },
])
const opinionList = ref([]) // 意见列表
const lv1ToLv2 = ref([])
const lv2ToLv3 = ref([])
const openTaskId = ref('')
const openMinScore = ref(0)
const showWidth = ref(0)

const progressShow = ref(false) // 下载进度条显示
const progressling = ref(0)

const timers = ref(null)
const arr = ref([])

const getSafeRequestUrlFn = getSafeRequestUrl

const requestHeader = {
  Country: store.state.activeCountryCode,
  'HiTest-CsrfToken': userToken.csrfToken,
  'HiTest-AccessToken': userToken.accessToken,
  'HiTest-AuthType': 'w3-wise-exempt-tenant',
}
const { proxy } = getCurrentInstance()

const getImageOrVideoUrl = async (tempImageOrVideoUrl: string) => {
  isUploading.value = true
  try {
    const res: { data: Blob; status: number } = await api.inspection.getFile(tempImageOrVideoUrl)
    if (res.status !== 200) {
      ElMessage.error('图片加载失败,请重新上传')
    }
    imageOrVideoUrl.value = window.URL.createObjectURL(res.data)
  } finally {
    isUploading.value = false
  }
}

const isImageUrl = (imageOrVideoUrl: string) => {
  const splitArray = imageOrVideoUrl.split('.')
  const suffix = splitArray[splitArray.length - 1].toLowerCase()
  return SUPPORTED_IMAGE_TYPES.includes(suffix)
}

const beforeUpload = (file: File) => {
  const splitArray = file.type.split('/')
  const suffix = splitArray[splitArray.length - 1].toLowerCase()

  if (SUPPORTED_IMAGE_TYPES.includes(suffix) || SUPPORTED_VIDEO_TYPES.includes(suffix)) {
    errorMsg.value = false
  } else {
    errorMsg.value = true
  }
}

const onSuccess = (response: { result: { data: string } }) => {
  // prettier-ignore
  const { result: { data }} = response
  tempImageOrVideoUrl.value = data
  getImageOrVideoUrl(tempImageOrVideoUrl.value)
}

const zoomImageOrVideo = (event: WheelEvent & { wheelDelta?: number }): boolean => {
  event.preventDefault()
  event.stopPropagation()

  const wheelDelta = event.wheelDelta ?? -event.deltaY * 40
  const currentZoom = Number.parseInt(mediaTypeRef.value.style.zoom) || ZOOM_CONFIG.default
  const newZoom = currentZoom + wheelDelta / ZOOM_CONFIG.step
  const finalZoom = Math.max(ZOOM_CONFIG.min, Math.min(ZOOM_CONFIG.max, newZoom))

  if (finalZoom !== currentZoom) {
    mediaTypeRef.value.style.zoom = `${finalZoom}%`
    imageOrVideoPercentage.value = `${finalZoom}%`
  }

  return false
}

const previewImageOrVideo = () => {
  previewMediaType.value.type = isImageUrl(tempImageOrVideoUrl.value) ? MediaTypeEnum.IMAGE : MediaTypeEnum.VIDEO
  previewMediaType.value.mediaTypeUrl = imageOrVideoUrl.value
  previewDialog.value = true
}

const handlePreviewClose = () => {
  previewDialog.value = false
  if (previewMediaType.value.type === MediaTypeEnum.VIDEO) mediaTypeRef.value?.pause()
}

const getDspNames = async () => {
  loadingDSP.value = true
  // prettier-ignore
  try {
    const { data: { result: { data: { dspType }}}} = await api.inspection.getDspNameOptions() as {data: {result: {data: { dspType: Array<DspNameoOptionProps> }}}}
    dspNameOptions.value = [...dspNameOptions.value, ...dspType]
    loadingDSP.value = false
  }catch (err) {
    ElMessage({
    message: '获取筛选信息失败！',
    type: 'error',
  })
  }
}

const updateDateTime = () => {
  let date = new Date()
  let month = date.getMonth() + 1
  let strDate = date.getDate()
  let startTime
  if (dspName.value.includes('0_')) {
    startTime = getDays(29)
  } else {
    startTime = getDays(6)
  }
  // prettier-ignore
  let endTime = date.getFullYear() + '-' + proxy.$formatTimeFilter(month) + '-' + proxy.$formatTimeFilter(strDate)
  console.log(startTime, endTime)
  return [startTime, endTime] as [string, string]
}

const updateSwitchDspTime = () => {
  console.log(dspName.value)

  if (dspName.value === `${COMMODITY_LIBRARY_DSP_OPTION.dspTypeId}_${COMMODITY_LIBRARY_DSP_OPTION.taskCategory}`) {
    advertiserIndustry.value = []
    isAdvertiserIndustryDisabled.value = true
  } else {
    isAdvertiserIndustryDisabled.value = false
  }

  dateTime.value = updateDateTime()
}

const getIndustryList = async () => {
  // prettier-ignore
  try {
    const { data: { result: { data: { list }}}} = await api.inspection.advertiserIndustryInfo()
    advertiserIndustryList.value = list
    machineAuditIndustryList.value = list
  }catch(error) {
    ElMessage({
      message: '获取筛选信息失败！',
      type: 'error',
    })
  }
}

const isDisabledDates = date => {
  let num = WEEK_DAYS
  if (dspName.value.startsWith(`${HUAWEI_DSP_OPTION.dspTypeId}_`)) {
    num = dspName.value === `${HUAWEI_DSP_OPTION.dspTypeId}_${HUAWEI_DSP_OPTION.taskCategory}` ? YEAR_DAYS : MONTH_DAYS
  }
  const hasFirstDate = Boolean(firstDate.value)
  const timeRange = num * ONE_DAY_MILLISECONDS
  const isFutureDate = date.getTime() > new Date().getTime()
  // prettier-ignore
  const isOutOfRange = hasFirstDate && (date.getTime() > firstDate.value + timeRange || date.getTime() < firstDate.value - timeRange)

  return isFutureDate || isOutOfRange
}

const handleCanlendarChange = ([startDate, endDate]: [Date, Date]) => {
  if (endDate) {
    firstDate.value = 0
    return
  }
  firstDate.value = startDate.getTime()
}

const focusDate = () => clearInterval(timerDate.value)

const handleConfirmDatePicker = () => {
  clearInterval(timerDate.value)

  if (!dateTime.value) {
    timerDate.value = setInterval(() => {
      dateTime.value = updateDateTime()
    }, 1000)
  }
}

const getHoverTitle = testResult => proxy.$filterDisplayFilter(testResult, 'testResult')

const previewMachineAuditResult = (machineAuditResult: MachineAuditResultsListProps) => {
  // prettier-ignore
  previewMediaType.value.type = machineAuditResult.resourceType.includes('img') ? MediaTypeEnum.IMAGE : MediaTypeEnum.VIDEO
  previewMediaType.value.mediaTypeUrl = machineAuditResult.fdfsPath
  previewDialog.value = true
}

const commitSample = () => {
  // 首先将当前页面选择id存到checkAllErrList中，避免用户在当页提交时，当页数据未存
  saveCurrentPageSelectedCreativeIds(pageNum.value)
  // 其次将checkAllErrList中全部的id整理到一个数组里
  checkTotalErrList.value = []
  allPageSelectedCreativeIds.value.forEach(item => {
    if (item.currentPageSelectedCreativeIds.length) {
      checkTotalErrList.value = checkTotalErrList.value.concat(item.currentPageSelectedCreativeIds)
    }
  })
  // 可以去个重
  checkTotalErrList.value = [...new Set(checkTotalErrList.value)]
}

const handleCreativeSelect = (machineAuditResult: MachineAuditResultsListProps) => {
  const { id } = machineAuditResult
  machineAuditResult.isSelectCreative = !machineAuditResult.isSelectCreative
  const isCreativeSelected = machineAuditResult.isSelectCreative
  const selectedSet: Set<string> = new Set(currentPageSelectedCreativeIds.value)

  if (isCreativeSelected) {
    selectedSet.add(id)
  } else {
    selectedSet.delete(id)
  }

  currentPageSelectedCreativeIds.value = Array.from(selectedSet)
}

const handleAllCreativesSelect = () => {
  isSelectedAllCreatives.value = !isSelectedAllCreatives.value

  // prettier-ignore
  currentPageSelectedCreativeIds.value = machineAuditResultsList.value.map(machineAuditResult => {
    machineAuditResult.isSelectCreative = isSelectedAllCreatives.value;
    return machineAuditResult.id;
  }).filter(() => isSelectedAllCreatives.value);
}

const handleAllMachineAuditResultsSelect = isSelectedAll => {
  currentPageSelectedMAResultIds.value = isSelectedAll ? currentPageAllMAResultIds.value : []
}

const handleCheckboxGroupChange = currentSelectedMAResultIds => {
  isSelectAllMachineAuditResults.value = currentSelectedMAResultIds.length === currentPageAllMAResultIds.value.length
}

// prettier-ignore
const convertIndustryName2Id = (industryName: [string, string][]) => industryName.map((industry: [string, string]) => industry[industry.length - 1]).join(',')

const checkIsFieldsValid = () => {
  if (!dateTime.value) {
    ElMessage.error({ message: '请填写时间！' })
    return false
  }

  if (
    !tempImageOrVideoUrl.value &&
    !ocrText.value.trim() &&
    !brandName.value.trim() &&
    !convertIndustryName2Id(advertiserIndustry.value) &&
    !convertIndustryName2Id(machineAuditIndustry.value)
  ) {
    ElMessage.error({ message: '请先上传图片/视频或输入文本后再进行搜索！' })
    return false
  }

  if (brandName.value.length > MAX_BRAND_NAME_LENGTH) {
    ElMessage.error({ message: '品牌名称长度必须在0~50之间' })
    return false
  }

  if (ocrText.value.length > MAX_ORCTEXT_LENGTH) {
    ElMessage.error({ message: 'title/orc长度必须在0~100之间' })
    return false
  }

  return true
}

const requestMachineAuditResults = async () => {
  const params = {
    dspType: dspName.value.split('_')[0],
    taskCategory: dspName.value.split('_')[1],
    taskResult: creativeStatus.value,
    fileName: tempImageOrVideoUrl.value,
    paging: {
      pageNumber: pageNum.value,
      pageSize: pageSize.value,
    },
    sourceType: resourceType.value,
    text: ocrText.value,
    startDay: dateTime.value[0],
    endDay: dateTime.value[1]!,
    brand: brandName.value,
    advertiserIndustry: convertIndustryName2Id(advertiserIndustry.value),
    machineIndustry: convertIndustryName2Id(machineAuditIndustry.value),
  }
  try {
    // prettier-ignore
    const {data: { result: { data }}} = await api.searchByImage.getMaterialsList(params)
    console.log(data)

    const machineAuditResults = data as MachineAuditResultsProps
    if (machineAuditResults?.attachment) {
      tempImageOrVideoUrl.value = machineAuditResults.attachment
    }
    if (machineAuditResults.list.length > 0) {
      batchId.value = machineAuditResults.list[0].batchId
      machineAuditResultsTotal.value = machineAuditResults.total
      machineAuditResultsList.value = machineAuditResults.list
      currentPageAllMAResultIds.value = machineAuditResultsList.value.map(machineAuditResult => machineAuditResult.id)
    }
  } catch (error) {
    ElMessage.error({ message: '搜素失败！' })
  } finally {
    isSearching.value = false
  }
}

const getCreativeMachineAuditResults = () => {
  if (!checkIsFieldsValid()) return

  // prettier-ignore
  // 重新抽样时，重置所有预存的值，避免影响第二次抽样
  (() => {
    isSearching.value = true
    machineAuditResultsList.value = []
    currentPageSelectedCreativeIds.value = []
    allPageSelectedCreativeIds.value = []
    allPageSelectedMAResultIds.value = []
    currentPageAllMAResultIds.value = []
    currentPageSelectedMAResultIds.value = []
    machineAuditResultsTotal.value = 0
    pageNum.value = 1
  })()

  requestMachineAuditResults()
}

const getCurrentPageMAResults = async () => {
  if (machineAuditResultsLoading.value) return
  machineAuditResultsLoading.value = true
  const params = {
    batchId: batchId.value,
    paging: {
      pageNumber: pageNum.value,
      pageSize: pageSize.value,
    },
    sourceType: resourceType.value,
    text: ocrText.value,
  }

  try {
    // prettier-ignore
    const { data: { result: { data: { list }}}} = await api.inspection.markPagination(params) as MarkPaginationApiResponse
    const selectedCreativeIdsSet = new Set(currentPageSelectedCreativeIds.value)
    const processedList = list.map(item => {
      item.isSelectCreative = selectedCreativeIdsSet.has(item.id)
      return item
    })
    machineAuditResultsList.value = processedList
    currentPageAllMAResultIds.value = processedList.map(item => item.id)
  } catch (error) {
    ElMessage.error({ message: '查询失败！' })
  } finally {
    machineAuditResultsLoading.value = false
  }
}

const savePageNumSelectedIds = (params: SavePageNumSelectedIdsProps) => {
  const { currentPageNum, previousPageNum, currentPageSelectedIds, allPageSelectedTargetIds, selectedIdField } = params
  const prevPageRecord = allPageSelectedTargetIds.value.find(item => item.pageNum === previousPageNum.value)
  // 上一页有记录 → 更新选中状态
  if (prevPageRecord && previousPageNum.value) {
    prevPageRecord[selectedIdField] = currentPageSelectedIds.value
  }
  // 上一页无记录且有上一页页码 → 新增选中状态记录
  else if (previousPageNum.value) {
    allPageSelectedTargetIds.value.push({
      pageNum: previousPageNum.value,
      [selectedIdField]: currentPageSelectedIds.value,
    })
  }

  const curPageRecord = allPageSelectedTargetIds.value.find(item => item.pageNum === currentPageNum)
  // 当前页有记录 → 取已保存的选中ID；无记录 → 置为空数组
  currentPageSelectedIds.value = curPageRecord?.[selectedIdField] || []
}

const saveCurrentPageSelectedMAResultIds = (currentPageNum: number) => {
  savePageNumSelectedIds({
    currentPageNum,
    previousPageNum,
    currentPageSelectedIds: currentPageSelectedMAResultIds,
    allPageSelectedTargetIds: allPageSelectedMAResultIds,
    selectedIdField: SelectedIdFieldEnum.MARESULTID,
  })
}

const saveCurrentPageSelectedCreativeIds = (currentPageNum: number) => {
  savePageNumSelectedIds({
    currentPageNum,
    previousPageNum,
    currentPageSelectedIds: currentPageSelectedCreativeIds,
    allPageSelectedTargetIds: allPageSelectedCreativeIds,
    selectedIdField: SelectedIdFieldEnum.CREATIVEID,
  })
}

const handlePageNumChange = (pageParams: PageNumChangeParams) => {
  previousPageNum.value = pageNum.value
  saveCurrentPageSelectedCreativeIds(pageParams.pageNum)
  saveCurrentPageSelectedMAResultIds(pageParams.pageNum)
  pageNum.value = pageParams.pageNum
  pageSize.value = pageParams.pageSize
  getCurrentPageMAResults()
}

const submitReviewResult = () => {
  commitDialogbo.value = false
  // 审核意见
  let result = []
  let batchIdVal = ''
  let errorCodes = null
  machineAuditResultsList.value.forEach(item => {
    let materialDetail = []
    batchIdVal = item.batchId
    for (let n = 0; n < checkTotalErrList.value.length; n++) {
      if (checkTotalErrList.value[n] === item.id) {
        item.auditDetail.forEach(obj => {
          if (obj.materialId !== '') {
            let s = {
              desc: obj.desc,
              firstLevel: obj.firstReason,
              materialId: item.resourceId,
              score: obj.score,
              secondLevel: obj.secondReason,
              threeLevel: obj.thirdReason,
              type: obj.materialType,
            }
            materialDetail.push(s)
          }
        })
      }
    }
    if (item.score === 0) {
      errorCodes = 1
    } else {
      errorCodes = 0
    }
    if (materialDetail.length > 0) {
      result.push({
        taskId: item.taskId,
        batchId: item.batchId,
        minScore: item.score,
        date: item.testTime,
        hashId: item.testId,
        creativeId: item.creativeId,
        taskCategory: dspName.value ? Number(dspName.value.split('_')[1]) : '',
        errorCode: errorCodes,
        testResult: item.testResult,
        materialDetail,
      })
    }
  })
  let param = {
    reviewer: reviewer.value,
    batchId: batchId.value,
    falsePositiveIds: checkTotalErrList.value,
    auditDetails: result,
  }
  api.inspection
    .submitReviewResult(param)
    .then(res => {
      if (res.status === 200) {
        ElMessage({
          message: '驳回成功！',
          type: 'success',
        })
        // 重新抽样一批
        setTimeout(() => {
          getCreativeMachineAuditResults(pageNum.value)
        }, 500)
      }
    })
    .catch(error => {
      ElMessage({
        message: '驳回失败！',
        type: 'error',
      })
    })
}

const cancels = () => {
  commitDialogbo.value = false
  machineAuditResultsList.value.forEach(item => {
    item.isSelectCreative = false
  })
  currentPageSelectedCreativeIds.value = arr.value
}

const downloadMaterials = () => {
  // 首先将当前页面选择id存到checkAllList中，避免用户在当页提交时，当页数据未存
  saveCurrentPageSelectedMAResultIds(pageNum.value)
  // 其次将checkAllList中全部的id整理到一个数组里
  checkTotalList.value = []
  allPageSelectedMAResultIds.value.forEach(item => {
    if (item.currentPageSelectedMAResultIds.length) {
      checkTotalList.value = checkTotalList.value.concat(item.currentPageSelectedMAResultIds)
    }
  })
  // 可以去个重
  checkTotalList.value = [...new Set(checkTotalList.value)]
  // 判断checkTotalList是否有值
  if (!checkTotalList.value.length) {
    downloadImg.value = true
  } else {
    api.inspection.downloadMaterials(checkTotalList.value).then(res => {
      allPageSelectedMAResultIds.value = []
      progressShow.value = true
      downloadProgress(res.data.result.data)
    })
    isSelectAllMachineAuditResults.value = false
  }
}

// 下载进度
const downloadProgress = uuidStr => {
  progressling.value = 0
  api.inspection
    .downLoadProgress(uuidStr)
    .then(res => {
      if (res.status === 200) {
        if (res.data.result.data) {
          progressling.value = parseInt(res.data.result.data)
        }
        timers.value = setInterval(() => {
          api.inspection
            .downLoadProgress(uuidStr)
            .then(res => {
              if (res.data.result.data) {
                progressling.value = parseInt(res.data.result.data)
              }
              if (progressling.value === 100) {
                downLoadInfos(uuidStr)
                ElMessage({
                  message: '下载成功！',
                  type: 'success',
                })
                clearIntervals()
                setTimeout(() => {
                  progressShow.value = false
                }, 2000)
              }
            })
            .catch(error => {
              ElMessage({
                message: '下载失败！',
                type: 'error',
              })
              clearIntervals()
              setTimeout(() => {
                progressShow.value = false
              }, 2000)
            })
        }, 5000)
      }
    })
    .catch(error => {
      ElMessage({
        message: '下载失败！',
        type: 'error',
      })
      clearIntervals()
      setTimeout(() => {
        progressShow.value = false
      }, 2000)
    })
}

// 文件下载
const downLoadInfos = uuidStr => {
  _axios
    .get('/PPSMASamplingService/v2/review/taskRecord/downLoadInfo?' + qs.stringify({ uuidStr }), {
      responseType: 'blob',
      headers: {
        'Content-Type': 'multipart/form-data',
        Country: store.state.activeCountryCode,
        'HiTest-CsrfToken': userToken.csrfToken,
        'HiTest-AccessToken': userToken.accessToken,
        'HiTest-AuthType': 'w3-wise-exempt-tenant',
      },
    })
    .then(res => {
      const blob = new Blob([res.data], { type: 'application/zip' })
      const url = window.URL.createObjectURL(blob)
      const link = document.createElement('a')
      link.href = url
      link.download = 'file'
      link.click()
      URL.revokeObjectURL(url)
    })
}

// 清除定时器
const clearIntervals = () => {
  clearInterval(timers.value)
  timers.value = null
}

const getDays = i => {
  let oneDay = 24 * 60 * 60 * 1000
  return formatterDate(new Date(Date.now() - i * oneDay), 'yyyy-MM-dd')
}

const formatterDate = (date, fmt) => {
  let nowDate = {
    yyyy: date.getFullYear(), // 年
    MM: date.getMonth() + 1, // 月份
    dd: date.getDate(), // 日
  }
  if (/(y+)/.test(fmt)) {
    fmt = fmt.replace(RegExp.$1, String(date.getFullYear()).substr(4 - RegExp.$1.length))
  }
  for (let k in nowDate) {
    if (new RegExp('(' + k + ')').test(fmt)) {
      fmt = fmt.replace(
        RegExp.$1,
        RegExp.$1.length === 1 ? nowDate[k] : ('00' + nowDate[k]).substr(String(nowDate[k]).length),
      )
    }
  }
  return fmt
}

// 审核意见弹出框
const reviewComments = () => {
  if (currentPageSelectedCreativeIds.value.length > 0) {
    if (currentPageSelectedCreativeIds.value.length > 500) {
      ElMessage.warning('单次批量驳回不能超过500条！')
    } else {
      openMaterialOptions.value = []
      // 添加一个弹窗的数据
      let auditDetail = [
        {
          isShow: true,
          materialId: '',
          materialType: '',
          score: '',
          firstReason: '',
          secondReason: '',
          thirdReason: '',
          selectDesc: '',
          desc: '',
          firstReasonOption: [],
          secondReasonOption: [],
          thirdReasonOption: [],
          showImgSrc: '',
        },
      ]
      openAuditDetail.value = JSON.parse(JSON.stringify(auditDetail))
      if (resourceType.value === 'all') {
        openMaterialOptions.value.push({ value: '图片/视频', type: 'all' }, { value: '文案', type: 'title' })
      } else if (resourceType.value === 'img') {
        openMaterialOptions.value.push({ value: '图片', type: 'img' }, { value: '文案', type: 'title' })
      } else {
        openMaterialOptions.value.push({ value: '视频', type: 'video' }, { value: '文案', type: 'title' })
      }
      examineDialog.value = true
    }
  } else {
    commitDialog.value = true
  }
}

// 添加意见
const addOpinion = () => {
  openAuditDetail.value.unshift({
    isShow: true,
    materialId: '',
    materialType: '',
    score: '',
    firstReason: '',
    secondReason: '',
    thirdReason: '',
    selectDesc: '',
    desc: '',
    firstReasonOption: [],
    secondReasonOption: [],
    thirdReasonOption: [],
    showImgSrc: '',
  })
}

// 删除意见
const delOpinion = index => {
  openAuditDetail.value.forEach((item, i) => {
    if (index === i) {
      openAuditDetail.value.splice(i, 1)
    }
  })
}

// 选择素材
const materialTypeChange = (v, item) => {
  item.materialType = ''
  item.showImgSrc = '' // 缩略图
  let typeOptions = openMaterialOptions.value
  for (let i = 0; i < typeOptions.length; i++) {
    let op = typeOptions[i]
    if (op.value === v) {
      item.materialType = op.type
      item.materialId = op.value
      if (op.type === 'pic') {
        item.showImgSrc = op.url // 缩略图
      }
    }
  }
}

// 评分事件
const auditScoreChange = item => {
  item.firstReason = ''
  item.secondReason = ''
  item.thirdReason = ''
  item.selectDesc = ''
  item.desc = ''
  item.firstReasonOption = []
  item.secondReasonOption = []
  item.thirdeasonOption = []
  let firstReasonTmp = []
  item.isShow = true
  if (item.score !== 40) {
    for (let i = 0; i < lv2ToLv3.value.length; i++) {
      let reasonObj = lv2ToLv3.value[i]
      let lv1 = reasonObj.lv1Reason
      if (firstReasonTmp.includes(lv1)) {
        continue
      }
      let lv3 = JSON.parse(reasonObj.lv3Reason)
      for (let j = 0; j < lv3.length; j++) {
        let lv3Tmp = lv3[j]
        if (lv3Tmp.score === item.score && !firstReasonTmp.includes(lv1)) {
          item.firstReasonOption.push({ value: lv1, label: lv1 })
          firstReasonTmp.push(lv1)
          break
        }
      }
    }
  }
  // 取最低分
  if (openAuditDetail.value.length >= 1) {
    openMinScore.value = Math.min.apply(
      Math,
      openAuditDetail.value.map((item, index) => item.score),
    )
  }
}

// 选一级原因触发事件
const reasonChange = item => {
  item.secondReason = ''
  item.thirdReason = ''
  item.secondReasonOption = []
  item.thirdReasonOption = []
  let secondReasonTmp = []
  for (let i = 0; i < lv2ToLv3.value.length; i++) {
    let reasonObj = lv2ToLv3.value[i]
    let lv1 = reasonObj.lv1Reason
    let lv2 = reasonObj.lv2Reason
    if (item.firstReason !== lv1 || secondReasonTmp.includes(lv2)) {
      continue
    }
    let lv3 = JSON.parse(reasonObj.lv3Reason)
    for (let j = 0; j < lv3.length; j++) {
      let lv3Tmp = lv3[j]
      if (lv3Tmp.score === item.score && !secondReasonTmp.includes(lv2)) {
        item.secondReasonOption.push({ value: lv2, label: lv2 })
        secondReasonTmp.push(lv2)
        break
      }
    }
  }
}

// 选二级原因触发事件
const secondChange = item => {
  item.thirdReason = ''
  item.thirdReasonOption = []
  let thirdReasonTmp = []
  for (let i = 0; i < lv2ToLv3.value.length; i++) {
    let reasonObj = lv2ToLv3.value[i]
    let lv1 = reasonObj.lv1Reason
    let lv2 = reasonObj.lv2Reason
    if (item.firstReason !== lv1 || item.secondReason !== lv2) {
      continue
    }
    let lv3 = JSON.parse(reasonObj.lv3Reason)
    for (let j = 0; j < lv3.length; j++) {
      let lv3Tmp = lv3[j]
      let { lv3Reason } = lv3Tmp
      if (lv3Tmp.score === item.score && !thirdReasonTmp.includes(lv3Reason)) {
        item.thirdReasonOption.push({ value: lv3Reason, label: lv3Reason })
        thirdReasonTmp.push(lv3Reason)
      }
    }
  }
}

// 选择意见描述
const checkOpinoin = val => {
  val.desc = val.selectDesc
  val.isShow = false
}

// 删除选择的意见描述
const cleanReason = item => {
  item.desc = ''
  item.selectDesc = ''
  item.isShow = true
}

// 保存弹窗数据
const saveCurEdit = () => {
  for (let i = 0; i < machineAuditResultsList.value.length; i++) {
    let saveItem = machineAuditResultsList.value[i]
    // 意见描述框删完不能点保存
    if (openAuditDetail.value.length === 0) {
      ElMessage({
        message: '请添加审核意见！',
        type: 'warning',
      })
      return
    }
    let desc = ''
    for (let i = 0; i < openAuditDetail.value.length; i++) {
      if (openAuditDetail.value[i].materialId === '') {
        ElMessage({ message: '请选择“素材”！', type: 'warning' })
        return
      } else if (openAuditDetail.value[i].score === '') {
        ElMessage({ message: '请选择“评分”！', type: 'warning' })
        return
      } else if (openAuditDetail.value[i].score !== 40 && !openAuditDetail.value[i].thirdReason) {
        ElMessage({
          message: '请选择一、二、三级原因！',
          type: 'warning',
        })
        return
      }

      if (openAuditDetail.value[i].firstReason !== '') {
        desc += openAuditDetail.value[i].firstReason + ';'
      }
      if (openAuditDetail.value[i].secondReason !== '') {
        desc += openAuditDetail.value[i].secondReason + ';'
      }
      if (openAuditDetail.value[i].thirdReason !== '') {
        desc += openAuditDetail.value[i].thirdReason + ';'
      }
      if (openAuditDetail.value[i].desc !== '') {
        desc += openAuditDetail.value[i].desc + ';'
      }
      desc += '\n'
    }
    saveItem.score = openMinScore.value
    saveItem.auditDetail = openAuditDetail.value
    examineDialog.value = false
    if (currentPageSelectedCreativeIds.value.length > 0) {
      commitSample()
    }
    commitDialogbo.value = true
  }
}

// 获取机审原因，意见描述
const getMachineReason = () => {
  api.inspection
    .getMachineReason()
    .then(res => {
      if (res.status === 200) {
        lv1ToLv2.value = res.data.result.data.lv1ToLv2
        lv2ToLv3.value = res.data.result.data.lv2ToLv3
        opinionList.value = res.data.result.data.opinion
      } else {
        ElMessage({
          message: '获取意见描述列表失败！',
          type: 'error',
        })
      }
    })
    .catch(() => {
      ElMessage({
        message: '获取意见描述列表失败！',
        type: 'error',
      })
    })
}

const reset = () => {
  machineAuditResultsList.value.forEach(item => {
    item.visible = false
  })
}

const clearImg = e => {
  e.stopPropagation()
  tempImageOrVideoUrl.value = ''
  machineAuditResultsList.value = []
}

// 广告审计数据数据转换
const wideReviewConverting = num => {
  let wideReviewDate = []
  let wideReviewList = []
  for (let i = 0; i < machineAuditIndustryList.value.length; i++) {
    wideReviewDate.push(machineAuditIndustryList.value[i].children)
  }
  let wideReviewDateList = wideReviewDate.reduce((a, b) => {
    return a.concat(b)
  })
  if (num !== null) {
    let numlist = num.split(',')
    for (let n = 0; n < numlist.length; n++) {
      for (let m = 0; m < wideReviewDateList.length; m++) {
        if (numlist[n] === wideReviewDateList[m].value) {
          wideReviewList.push(wideReviewDateList[m].fullName)
        }
      }
    }
  }
  return wideReviewList.toString()
}

// 复制
const copyCode = async (url, type) => {
  if (url) {
    let clipboard = new Clipboard(type, {
      text() {
        return url
      },
    })
    await clipboard.on('success', e => {
      ElMessage({
        message: '复制成功！',
        type: 'success',
      })
      clipboard.destroy()
    })
    await clipboard.on('error', e => {
      ElMessage({
        message: '复制失败！',
        type: 'error',
      })
      clipboard.destroy()
    })
  } else {
    ElMessage({
      message: '没有复制内容！',
      type: 'warning',
    })
  }
}

watch(
  () => [currentPageSelectedMAResultIds.value, currentPageAllMAResultIds.value],
  ([newSelectedMAResultIds, newAllMAResultIds]: Array<string[]>) => {
    isSelectAllMachineAuditResults.value =
      newSelectedMAResultIds.length > 0 &&
      newAllMAResultIds.length > 0 &&
      newSelectedMAResultIds.length === newAllMAResultIds.length
  },
)

watch(
  () => [currentPageSelectedCreativeIds.value, currentPageAllMAResultIds.value],
  ([newSelectedCreativeIds, newAllMAResultIds]: Array<string[]>) => {
    // prettier-ignore
    isSelectedAllCreatives.value = newSelectedCreativeIds.length > 0 && newSelectedCreativeIds.length === newAllMAResultIds.length
    getMachineReason()
  },
)

// mounted 生命周期
onMounted(() => {
  // 获取动态left
  const resizeHandler = () => {
    showWidth.value = document.body.clientWidth
  }
  window.addEventListener('resize', resizeHandler)
  // 组件卸载时移除监听
  onUnmounted(() => {
    window.removeEventListener('resize', resizeHandler)
  })

  getDspNames()
  getIndustryList()
  dateTime.value = updateDateTime()
})

onMounted(() => {
  showWidth.value = document.body.clientWidth
  getMachineReason()
})
</script>

<style lang="scss" scoped>
@import url('./SearchByResource.scss');

.search-by-resource {
  .el-input {
    border: none;
    cursor: pointer;
  }

  .el-input:focus {
    border: none;
    outline: none;
  }
}
</style>

```
