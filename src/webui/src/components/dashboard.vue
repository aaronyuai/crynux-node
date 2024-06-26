<script setup>
import { computed, createVNode, h, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import {
  PauseCircleOutlined,
  LogoutOutlined,
  PlayCircleOutlined,
  ExclamationCircleOutlined
} from '@ant-design/icons-vue'
import { Grid, Modal } from 'ant-design-vue'
import EditAccount from './edit-account.vue'

import systemAPI from '../api/v1/system'
import nodeAPI from '../api/v1/node'
import taskAPI from '../api/v1/task'
import accountAPI from '../api/v1/account'
import config from "../config.json"

const appVersion = APP_VERSION

const accountEditor = ref(null)

const systemInfo = reactive({
  gpu: {
    usage: 0,
    model: '',
    vram_used_mb: 0,
    vram_total_mb: 0
  },
  cpu: {
    usage: 0,
    num_cores: 0,
    frequency_mhz: 0,
    description: '',
  },
  memory: {
    available_mb: 0,
    total_mb: 0
  },
  disk: {
    base_models: 0,
    lora_models: 0,
    logs: 0
  }
})

const nodeStatus = reactive({
  status: '',
  message: '',
  tx_status: '',
  tx_error: ''
})

const accountStatus = reactive({
  address: '',
  eth_balance: 0,
  cnx_balance: 0
})

const taskStatus = reactive({
  status: 'waiting',
  num_today: 0,
  num_total: 0
})

const shortAddress = computed(() => {
  if (accountStatus.address === '') {
    return 'N/A'
  } else {
    return (
      accountStatus.address.substring(0, 6) +
      '...' +
      accountStatus.address.substring(accountStatus.address.length - 4)
    )
  }
})

const toEtherValue = (bigNum) => {
  if (bigNum === 0) return 0

  const decimals = (bigNum / BigInt(1e18)).toString()

  let fractions = ((bigNum / BigInt(1e16)) % 100n).toString()

  if (fractions.length === 1) fractions += '0'

  return decimals + '.' + fractions
}

const ethEnough = () => {
  if (accountStatus.eth_balance === 0) return false
  return accountStatus.eth_balance >= 1e16
}

const cnxEnough = () => {
  if (accountStatus.cnx_balance === 0) return false
  return accountStatus.cnx_balance >= 4e20
}

let systemUpdateInterval = null
onMounted(async () => {
  await updateSystemInfo()

  if (systemUpdateInterval == null) {
    systemUpdateInterval = setInterval(async () => {
      if (isTxSending.value) return
      await updateSystemInfo()
    }, 5000)
  }

  if (accountStatus.address === '') {
    accountEditor.value.showModal()
  }
})
onBeforeUnmount(() => {
  clearInterval(systemUpdateInterval)
  systemUpdateInterval = null
})

let isTxSending = ref(false)

const updateSystemInfo = async () => {
  const systemResp = await systemAPI.getSystemInfo()
  Object.assign(systemInfo, systemResp)

  const nodeResp = await nodeAPI.getNodeStatus()
  Object.assign(nodeStatus, nodeResp)

  const accountResp = await accountAPI.getAccountInfo()
  Object.assign(accountStatus, accountResp)

  const taskResp = await taskAPI.getTaskRunningStatus()
  Object.assign(taskStatus, taskResp)
}

const sendNodeAction = async (action) => {
  if (isTxSending.value) {
    return
  }

  if (action === 'pause') {
    await sendNodeActionAfterConfirmation(
      action,
      'Are you sure to pause the node? You could resume the running later.'
    )
  } else if (action === 'stop') {
    await sendNodeActionAfterConfirmation(
      action,
      'Are you sure to stop the node? The staked tokens will be returned. You could start the node later.'
    )
  } else {
    await doSendNodeAction(action)
  }
}

const sendNodeActionAfterConfirmation = async (action, message) => {
  Modal.confirm({
    title: 'Confirm node operation',
    icon: createVNode(ExclamationCircleOutlined),
    content: message,
    onOk() {
      return doSendNodeAction(action)
    },

    onCancel() {}
  })
}

const doSendNodeAction = async (action) => {
  isTxSending.value = true
  try {
    await nodeAPI.sendNodeAction(action)
    await updateSystemInfo()
  } finally {
    isTxSending.value = false
  }
}

const useBreakpoint = Grid.useBreakpoint
const screens = useBreakpoint()

const topRowClasses = computed(() => {
  let classes = ['top-row']
  for (let v in screens.value) {
    if (screens.value[v]) {
      classes.push(v)
    }
  }

  return classes
})
</script>

<template>
  <a-row :class="topRowClasses"> </a-row>
  <a-row>
    <a-col :span="14" :offset="5">
      <a-alert
        message="System is initializing. This may take a while, please be patient..."
        class="top-alert"
        v-if="nodeStatus.status === nodeAPI.NODE_STATUS_INITIALIZING"
      ></a-alert>
      <a-alert
        type="error"
        :message="'Node error: ' + nodeStatus.message + '. Please restart the Node.'"
        class="top-alert"
        v-if="nodeStatus.status === nodeAPI.NODE_STATUS_ERROR"
      ></a-alert>
      <a-alert
        type="error"
        :message="'Transaction error: ' + nodeStatus.tx_error + '. Please try again later.'"
        class="top-alert"
        v-if="nodeStatus.tx_status === nodeAPI.TX_STATUS_ERROR"
      ></a-alert>
      <a-alert
        message="Waiting for the Blockchain confirmation..."
        class="top-alert"
        v-if="nodeStatus.tx_status === nodeAPI.TX_STATUS_PENDING"
      ></a-alert>
      <a-alert
        message="Node will stop after finishing the current task"
        class="top-alert"
        v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PENDING_STOP"
      ></a-alert>
      <a-alert
        message="Node will pause after finishing the current task"
        class="top-alert"
        v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PENDING_PAUSE"
      ></a-alert>
      <a-alert
        type="error"
        message="Not enough gas tokens in the wallet. At least 0.01 gas token is required."
        class="top-alert"
        v-if="accountStatus.address !== '' && !ethEnough() && cnxEnough()"
      >
        <template #action>
          <a-button size="small" type="primary" :href="config.discord_link" target="_blank">Crynux Discord</a-button>
        </template>
        <template #description>
        Get the test tokens for free from: <a-typography-link :href="config.discord_link" target="_blank">{{ config.discord_link }}</a-typography-link>
      </template>
      </a-alert>
      <a-alert
        type="error"
        message="Not enough test CNX in the wallet. At least 400 test CNX is required."
        class="top-alert"
        v-if="
          nodeStatus.status === nodeAPI.NODE_STATUS_STOPPED &&
          accountStatus.address !== '' &&
          !cnxEnough() && ethEnough()
        "
      >
        <template #action>
          <a-button size="small" type="primary" :href="config.discord_link" target="_blank">Crynux Discord</a-button>
        </template>
        <template #description>
        Get the test tokens for free from: <a-typography-link :href="config.discord_link" target="_blank">{{ config.discord_link }}</a-typography-link>
      </template>
      </a-alert>
      <a-alert
        type="error"
        message="Not enough test tokens in the wallet."
        class="top-alert"
        v-if="accountStatus.address !== '' && !ethEnough() && !cnxEnough()"
      >
        <template #action>
          <a-button size="small" type="primary" :href="config.discord_link" target="_blank">Crynux Discord</a-button>
        </template>
        <template #description>
        Get the test tokens for free from: <a-typography-link :href="config.discord_link" target="_blank">{{ config.discord_link }}</a-typography-link>
      </template>
      </a-alert>
    </a-col>
  </a-row>
  <a-row :gutter="[16, 16]">
    <a-col
      :xs="{ span: 24, offset: 0, order: 1 }"
      :sm="{ span: 12, offset: 0, order: 1 }"
      :md="{ span: 12, offset: 0, order: 1 }"
      :lg="{ span: 5, offset: 0, order: 1 }"
      :xl="{ span: 5, offset: 1, order: 1 }"
      :xxl="{ span: 4, offset: 3, order: 1 }"
    >
      <a-card title="Node Status" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="12">
            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              v-if="nodeStatus.status === nodeAPI.NODE_STATUS_RUNNING"
            />
            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              status="exception"
              v-if="nodeStatus.status === nodeAPI.NODE_STATUS_ERROR"
            >
            </a-progress>
            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              :stroke-color="'lightgray'"
              v-if="
                [
                  nodeAPI.NODE_STATUS_PAUSED,
                  nodeAPI.NODE_STATUS_STOPPED,
                  nodeAPI.NODE_STATUS_PENDING
                ].indexOf(nodeStatus.status) !== -1
              "
            >
              <template #format="percent">
                <span style="font-size: 14px; color: lightgray">
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_INITIALIZING"
                    >Preparing</span
                  >
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PAUSED">Paused</span>
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_STOPPED">Stopped</span>
                </span>
              </template>
            </a-progress>
            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              :stroke-color="'cornflowerblue'"
              v-if="
                [
                  nodeAPI.NODE_STATUS_PENDING_PAUSE,
                  nodeAPI.NODE_STATUS_PENDING_STOP,
                  nodeAPI.NODE_STATUS_INITIALIZING
                ].indexOf(nodeStatus.status) !== -1
              "
            >
              <template #format="percent">
                <span style="font-size: 14px; color: cornflowerblue">
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PENDING_PAUSE"
                    >Pausing</span
                  >
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PENDING_STOP"
                    >Stopping</span
                  >
                  <span v-if="nodeStatus.status === nodeAPI.NODE_STATUS_INITIALIZING"
                    >Preparing</span
                  >
                </span>
              </template>
            </a-progress>
          </a-col>
          <a-col :span="12">
            <div class="node-op-btn" v-if="nodeStatus.status === nodeAPI.NODE_STATUS_RUNNING">
              <a-button
                :icon="h(PauseCircleOutlined)"
                @click="sendNodeAction('pause')"
                :loading="isTxSending || nodeStatus.tx_status === nodeAPI.TX_STATUS_PENDING"
                :disabled="!ethEnough()"
                >Pause</a-button
              >
            </div>
            <div
              class="node-op-btn"
              style="margin-top: 8px"
              v-if="nodeStatus.status === nodeAPI.NODE_STATUS_RUNNING"
            >
              <a-button
                :icon="h(LogoutOutlined)"
                @click="sendNodeAction('stop')"
                :loading="isTxSending || nodeStatus.tx_status === nodeAPI.TX_STATUS_PENDING"
                :disabled="!ethEnough()"
                >Stop</a-button
              >
            </div>
            <div class="node-op-btn" v-if="nodeStatus.status === nodeAPI.NODE_STATUS_STOPPED">
              <a-button
                type="primary"
                :icon="h(PlayCircleOutlined)"
                @click="sendNodeAction('start')"
                :loading="isTxSending || nodeStatus.tx_status === nodeAPI.TX_STATUS_PENDING"
                :disabled="!ethEnough() || !cnxEnough()"
                >Start</a-button
              >
            </div>
            <div class="node-op-btn" v-if="nodeStatus.status === nodeAPI.NODE_STATUS_PAUSED">
              <a-button
                type="primary"
                :icon="h(PlayCircleOutlined)"
                @click="sendNodeAction('resume')"
                :loading="isTxSending || nodeStatus.tx_status === nodeAPI.TX_STATUS_PENDING"
                :disabled="!ethEnough()"
                >Resume</a-button
              >
            </div>
            <div class="node-op-btn" v-if="nodeStatus.status === nodeAPI.NODE_STATUS_INITIALIZING">
              <a-button type="primary" :icon="h(PlayCircleOutlined)" disabled>Start</a-button>
            </div>
          </a-col>
        </a-row>
      </a-card>
    </a-col>

    <a-col
      :xs="{ span: 24, order: 3 }"
      :sm="{ span: 24, order: 3 }"
      :md="{ span: 24, order: 3 }"
      :lg="{ span: 12, order: 2 }"
      :xl="{ span: 11, order: 2 }"
      :xxl="{ span: 9, order: 2 }"
    >
      <a-card title="Wallet" :bordered="false" style="height: 100%; opacity: 0.9">
        <template #extra>
          <edit-account
            ref="accountEditor"
            :account-status="accountStatus"
            @private-key-updated="updateSystemInfo"
          ></edit-account>
        </template>
        <a-row>
          <a-col :span="10">
            <a-tooltip>
              <template #title>{{ accountStatus.address }}</template>
              <a-statistic title="Address" :value="shortAddress"></a-statistic>
            </a-tooltip>
          </a-col>
          <a-col :span="7">
            <a-statistic title="Gas token" :value="toEtherValue(accountStatus.eth_balance)"></a-statistic>
          </a-col>
          <a-col :span="7">
            <a-statistic title="Test CNX" :value="toEtherValue(accountStatus.cnx_balance)"></a-statistic>
          </a-col>
        </a-row>
      </a-card>
    </a-col>

    <a-col
      :xs="{ span: 24, order: 2 }"
      :sm="{ span: 12, order: 2 }"
      :md="{ span: 12, order: 2 }"
      :lg="{ span: 7, order: 3 }"
      :xl="{ span: 6, order: 3 }"
      :xxl="{ span: 5, order: 3 }"
    >
      <a-card title="Task Execution" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="8">
            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              :stroke-color="'cornflowerblue'"
              v-if="taskStatus.status === 'idle'"
            >
              <template #format="percent">
                <span style="font-size: 14px; color: cornflowerblue">Idle</span>
              </template>
            </a-progress>

            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              :stroke-color="'lightgray'"
              v-if="taskStatus.status === 'stopped'"
            >
              <template #format="percent">
                <span style="font-size: 14px; color: lightgray">Stopped</span>
              </template>
            </a-progress>

            <a-progress
              type="circle"
              :size="70"
              :percent="100"
              status="success"
              v-if="taskStatus.status === 'running'"
            >
              <template #format="percent">
                <span style="font-size: 14px">Running</span>
              </template>
            </a-progress>
          </a-col>
          <a-col :span="8">
            <a-statistic title="Today" :precision="0" :value="taskStatus.num_today"></a-statistic>
          </a-col>
          <a-col :span="8">
            <a-statistic title="Total" :precision="0" :value="taskStatus.num_total"></a-statistic>
          </a-col>
        </a-row>
      </a-card>
    </a-col>
  </a-row>
  <a-row :gutter="[16, 16]" style="margin-top: 16px">
    <a-col
      :xs="{ span: 24, offset: 0 }"
      :sm="{ span: 16, offset: 0 }"
      :md="{ span: 16, offset: 0 }"
      :lg="{ span: 9, offset: 0 }"
      :xl="{ span: 7, offset: 1 }"
      :xxl="{ span: 6, offset: 3 }"
    >
      <a-card title="GPU" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="8">
            <a-progress type="dashboard" :size="80" :percent="systemInfo.gpu.usage" />
          </a-col>
          <a-col :span="16">
            <a-row>
              <a-col :span="24">
                <a-statistic :value="systemInfo.gpu.model" :value-style="{ 'font-size': '14px' }">
                  <template #title><span style="font-size: 12px">Card Model</span></template>
                </a-statistic>
              </a-col>
            </a-row>
            <a-row style="margin-top: 12px">
              <a-col :span="12">
                <a-statistic
                  :value="systemInfo.gpu.vram_used_mb"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">VRAM Used</span></template>
                  <template #suffix>MB</template>
                </a-statistic>
              </a-col>
              <a-col :span="12">
                <a-statistic
                  :value="systemInfo.gpu.vram_total_mb"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">VRAM Total</span></template>
                  <template #suffix>MB</template>
                </a-statistic>
              </a-col>
            </a-row>
          </a-col>
        </a-row>
      </a-card>
    </a-col>
    <a-col :xs="12" :sm="8" :md="8" :lg="5" :xxl="4">
      <a-card title="CPU" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="12">
            <a-progress type="dashboard" :size="80" :percent="systemInfo.cpu.usage" />
          </a-col>
          <a-col :span="12">
            <a-row>
              <a-col :span="24">
                <a-statistic
                  :value="systemInfo.cpu.num_cores"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">Num of Cores</span></template>
                </a-statistic>
              </a-col>
            </a-row>
            <a-row style="margin-top: 12px">
              <a-col :span="24">
                <a-statistic
                  :value="systemInfo.cpu.frequency_mhz"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">Frequency</span></template>
                  <template #suffix>MHz</template>
                </a-statistic>
              </a-col>
            </a-row>
          </a-col>
        </a-row>
      </a-card>
    </a-col>
    <a-col :xs="12" :sm="12" :md="12" :lg="5" :xxl="4">
      <a-card title="Memory" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="12">
            <a-progress
              type="dashboard"
              :size="80"
              :percent="
                Math.round(
                  ((systemInfo.memory.total_mb - systemInfo.memory.available_mb) /
                    systemInfo.memory.total_mb) *
                    100
                )
              "
            />
          </a-col>
          <a-col :span="12">
            <a-row>
              <a-col :span="24">
                <a-statistic
                  :value="systemInfo.memory.total_mb - systemInfo.memory.available_mb"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">RAM Used</span></template>
                  <template #suffix>MB</template>
                </a-statistic>
              </a-col>
            </a-row>
            <a-row style="margin-top: 12px">
              <a-col :span="24">
                <a-statistic
                  :value="systemInfo.memory.total_mb"
                  :value-style="{ 'font-size': '14px' }"
                >
                  <template #title><span style="font-size: 12px">RAM Total</span></template>
                  <template #suffix>MB</template>
                </a-statistic>
              </a-col>
            </a-row>
          </a-col>
        </a-row>
      </a-card>
    </a-col>
    <a-col :xs="12" :sm="12" :md="12" :lg="5" :xxl="4">
      <a-card title="Disk" :bordered="false" style="height: 100%; opacity: 0.9">
        <a-row>
          <a-col :span="12">
            <a-statistic
              :value="systemInfo.disk.base_models"
              :value-style="{ 'font-size': '14px' }"
            >
              <template #title><span style="font-size: 12px">Base Models</span></template>
              <template #suffix>GB</template>
            </a-statistic>
          </a-col>
          <a-col :span="12">
            <a-statistic
              :value="systemInfo.disk.lora_models"
              :value-style="{ 'font-size': '14px' }"
            >
              <template #title><span style="font-size: 12px">Lora Models</span></template>
              <template #suffix>MB</template>
            </a-statistic>
          </a-col>
        </a-row>
        <a-row style="margin-top: 12px">
          <a-col :span="12">
            <a-statistic :value="systemInfo.disk.logs" :value-style="{ 'font-size': '14px' }">
              <template #title><span style="font-size: 12px">Logs</span></template>
              <template #suffix>KB</template>
            </a-statistic>
          </a-col>
        </a-row>
      </a-card>
    </a-col>
  </a-row>
  <div class="bottom-bar">
    <a-space class="footer-links">
      <a-typography-link href="https://crynux.ai" target="_blank">Home</a-typography-link>
      &nbsp;|&nbsp;
      <a-typography-link href="https://docs.crynux.ai" target="_blank">Docs</a-typography-link>
      &nbsp;|&nbsp;
      <a-typography-link href="https://github.com/crynux-ai" target="_blank"
        >GitHub</a-typography-link
      >
      &nbsp;|&nbsp;
      <a-typography-link href="https://blog.crynux.ai" target="_blank">Blog</a-typography-link>
      &nbsp;|&nbsp;
      <a-typography-link href="https://twitter.com/crynuxai" target="_blank"
        >Twitter</a-typography-link
      >
      &nbsp;|&nbsp;
      <a-typography-link :href="config.discord_link" target="_blank"
        >Discord</a-typography-link
      >
      &nbsp;|&nbsp;
      <a-typography-text :style="{'color':'white'}">v{{ appVersion }}</a-typography-text>
    </a-space>
    <img class="footer-logo" src="./logo-full-white.png" width="140" alt="Crynux logo" />
  </div>
</template>

<style lang="stylus">
.ant-row
    margin-left 0!important
    margin-right 0!important
</style>
<style scoped lang="stylus">
.top-alert
    margin-bottom 16px
.bottom-bar
    position fixed
    width 100%
    height 60px
    bottom 0
    left 0
    padding 0 32px

.footer-links
    color #fff
    opacity 0.8
    line-height 60px

    a
        color #fff
        &:hover
            text-decoration underline

.footer-logo
    opacity 0.8
    float right

.top-row
    &.xs
        height 16px
    &.sm
        height 20px
    &.md
        height 40px
    &.lg
        height 40px
    &.xl
        height 80px
    &.xxl
        height 100px
</style>
