<template>
  <div class="inbox-search-box">
    <div class="inbox-search-bar">
      <el-date-picker
          v-model="searchForm.dateRange"
          type="daterange"
          range-separator="至"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
          value-format="YYYY-MM-DD"
          clearable
          class="search-date"
      />
      <el-input
          v-model="searchForm.code"
          placeholder="邮件标"
          clearable
          class="search-input"
      />
      <el-input
          v-model="searchForm.recipient"
          placeholder="收件人"
          clearable
          class="search-input"
      />
      <el-button type="primary" @click="resetSearch">重置</el-button>
    </div>

    <emailScroll ref="scroll"
                 :cancel-success="cancelStar"
                 :star-success="addStar"
                 :getEmailList="getEmailList"
                 :emailDelete="emailDelete"
                 :star-add="starAdd"
                 :star-cancel="starCancel"
                 :time-sort="params.timeSort"
                 :email-read="emailRead"
                 :show-unread="true"
                 :search-filters="searchFilters"
                 actionLeft="4px"
                 @jump="jumpContent"
    >
      <template #first>
        <Icon class="icon" @click="changeTimeSort" icon="material-symbols-light:timer-arrow-down-outline"
              v-if="params.timeSort === 0" width="28" height="28"/>
        <Icon class="icon" @click="changeTimeSort" icon="material-symbols-light:timer-arrow-up-outline" v-else
              width="28" height="28"/>
      </template>

    </emailScroll>
  </div>
</template>

<script setup>
import {useAccountStore} from "@/store/account.js";
import {useEmailStore} from "@/store/email.js";
import {useSettingStore} from "@/store/setting.js";
import emailScroll from "@/components/email-scroll/index.vue"
import {emailList, emailDelete, emailLatest, emailRead} from "@/request/email.js";
import {starAdd, starCancel} from "@/request/star.js";
import {computed, defineOptions, h, onMounted, reactive, ref, watch} from "vue";
import {sleep} from "@/utils/time-utils.js";
import router from "@/router/index.js";
import {Icon} from "@iconify/vue";
import { useRoute } from 'vue-router'

defineOptions({
  name: 'email'
})

const route = useRoute();
const emailStore = useEmailStore();
const accountStore = useAccountStore();
const settingStore = useSettingStore();
const scroll = ref({})
const searchForm = reactive({
  dateRange: [],
  code: '',
  recipient: '',
})
const searchFilters = computed(() => ({
  startDate: searchForm.dateRange?.[0] || '',
  endDate: searchForm.dateRange?.[1] || '',
  code: searchForm.code || '',
  recipient: searchForm.recipient || '',
}))
const params = reactive({
  timeSort: 0,
})

function resetSearch() {
  searchForm.dateRange = []
  searchForm.code = ''
  searchForm.recipient = ''
}

onMounted(() => {
  emailStore.emailScroll = scroll;
  latest()
})


watch(() => accountStore.currentAccountId, () => {
  scroll.value.refreshList();
})

function changeTimeSort() {
  params.timeSort = params.timeSort ? 0 : 1
  scroll.value.refreshList();
}

function jumpContent(email) {
  emailStore.contentData.email = emailStore.toContentEmail(email)
  emailStore.contentData.delType = 'logic'
  emailStore.contentData.showUnread = true
  emailStore.contentData.showStar = true
  emailStore.contentData.showReply = true
  router.push('/mail')
}

const existIds = new Set();

async function latest() {
  while (true) {

    let autoRefresh = settingStore.settings.autoRefresh;
    await sleep(autoRefresh > 1 ? autoRefresh * 1000 : 3000);

    if (route.name !== 'email') {
      continue;
    }

    const latestId = scroll.value.latestEmail?.emailId

    if (!scroll.value.firstLoad && autoRefresh > 1) {
      try {
        const accountId = accountStore.currentAccountId
        const allReceive = scroll.value.latestEmail?.allReceive
        const curTimeSort = params.timeSort
        let list = []

        //确保发起请求时最后一个邮件是当前账号的,或者
        if (accountId === scroll.value.latestEmail?.reqAccountId) {
          list = await emailLatest(latestId, accountId, allReceive);
        }

        //确保请求回来后，账号没有切换，时间排序没有改变，全部邮件类型没变
        if (accountId === accountStore.currentAccountId && params.timeSort === curTimeSort && allReceive === accountStore.currentAccount.allReceive) {
          if (list.length > 0) {
            emailStore.applyFullList(list)

            for (let email of list) {

              email.reqAccountId = accountId;
              email.allReceive = allReceive;

              if (!existIds.has(email.emailId)) {

                existIds.add(email.emailId)
                scroll.value.addItem(email)

                await sleep(50)
              }

            }

          }

        }
      } catch (e) {
        if (e.code === 401 || e.code === 403) {
          settingStore.settings.autoRefresh = 0;
        }
        console.error(e)
      }
    }
  }
}

function addStar(email) {
  emailStore.starScroll?.addItem(email)
}

function cancelStar(email) {
  emailStore.starScroll?.deleteEmail([email.emailId])
}

function getEmailList(emailId, size) {
  const accountId =  accountStore.currentAccountId;
  const allReceive = accountStore.currentAccount.allReceive;
  return emailStore.fetchList(full =>
    emailList(accountId, allReceive, emailId, params.timeSort, size, 0, full)
  ).then(data => {
    data.latestEmail.reqAccountId = accountId;
    data.latestEmail.allReceive = allReceive;
    return data;
  })
}

</script>
<style>
.inbox-search-box {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.inbox-search-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px 8px;
  border-bottom: 1px solid var(--el-border-color-lighter);
  background: var(--el-bg-color);
  flex-wrap: wrap;
}

.search-date {
  width: 260px;
}

.search-input {
  width: 180px;
}

.icon {
  cursor: pointer;
}
</style>