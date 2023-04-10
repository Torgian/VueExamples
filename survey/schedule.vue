<template lang="pug">
.page-administration-survey
  side-nav-menu(menu="survey")

  v-card.main-container(flat)
    v-col.text-right
      v-btn(outlined @click="checkForOngoingSurvey()" width="200px")
        v-icon {{ createIcon }}
        span アンケートを作る
    .list-container
      v-data-table.survey-list(
        :headers="headers"
        :items="surveys"
        :footer-props="{ 'items-per-page-options': [100], 'page-text': `{0}-{1} / {2}` }"
        :loading="isDataTableLoading"
        loading-text="データ取得中"
        :sort-by="['number']"
        :sort-desc="[false]"
        :custom-sort="customSort"
        no-data-text="データがありません"
        no-results-text="該当するデータがありません"
        fixed-header
        must-sort
        height="calc(100% - 60px)"
      )
        template(v-slot:item.number="{ item }")
          p.survey-number(v-if="checkForPeriodicSurvey(item)") 定期{{ item.periodicSurvey.number }} - {{ item.number }}
          p.survey-number(v-else) 単回{{ item.number }}
        template(v-slot:item.targetUserCount="{ item }")
          v-row(no-gutters)
            p.pt-4(v-if="item.targetUserCount") {{ item.answererCount }}/{{ item.targetUserCount }} ({{ item.answerRate }}%)
            p.pt-4(v-else) {{ item.answererCount }}
            span.mt-4.mb-0.ml-1(v-if="!hasNoTargets(item)"): map-icon
              participants(:surveyId="item.id")
            span.mt-4.ml-1(v-if="checkForPeriodicSurvey(item)")
              v-icon(
                :color="suspendedOrPastDate(item) ? 'red' : 'green'"
                v-tippy="tooltipOptions()"
                :content="suspendedOrPastDate(item) ? '停止中' : '進行中'"
              ) {{ clockIcon }}

        template(v-slot:item.targetDestinations="{ item }")
          div(v-if="!item.targetGlobal")
            span.pr-1 {{ item.targetUnits.map(u => u.name).join(', ') }}
            span.pr-1 {{ item.targetByotos.map(b => b.name).join(', ') }}
            span.pr-1 {{ item.targetOccupations.map(o => o.name).join(', ') }}
          div(v-else)
            span.pr-1 全施設

        template(v-slot:item.publicId="{ item }")
          v-row(v-if="item.publicId" no-gutters)
            v-text-field.survey-field(
              hide-details
              color="grey darken-1"
              ref="publicLinkText"
              :value="link ? link + item.publicId : ''"
              readonly
              filled
              dense
              solo
              flat
            )
            v-btn.public-survey-icon(
              @click="copyToClipboard(link + item.publicId)"
              fab
              depressed
              tile
              small
              height=42
              v-tippy="{ placement: 'bottom', size: 'small' }"
              content="コピー"
            )
              v-icon {{ copyIcon }}
            v-btn.public-survey-icon(
              :to="'public/' + item.publicId"
              fab
              depressed
              tile
              small
              height=42
              v-tippy="{ placement: 'bottom', size: 'small' }"
              content="アンケートを閲覧する"
            )
              v-icon {{ eyeIcon }}
          span(v-else) なし

        template(v-slot:item.editButton="{ item }")
          v-btn.no-label(
            depressed
            fab
            x-small
            color="info"
            @click="openSidePanel(item)"
          )
            v-icon {{ editIcon }}
          v-btn.no-label(
            depressed
            fab
            x-small
            color="grey"
            :href="`${$axios.defaults.baseURL}/surveys/${item.id}/export`"
          )
            v-icon(color="white") {{ downloadIcon }}
          v-btn.no-label(
            depressed
            fab
            x-small
            color="error"
            @click="onClickDelete(item)"
          )
            v-icon {{ deleteIcon }}

    template
      v-row(justify="center")
        v-dialog(v-model="dialog" persistent max-width="400")
          v-card.text-center(v-if="dialogType === 'error'")
            v-card-title.headline.justify-center エラー
            v-card-text 現在回答中のアンケートがあるため、新たなアンケートは作成できません。
            v-card-actions
              v-spacer
              v-btn(@click="dialog = false" color="primary" outlined) Ok
          v-card.text-center(v-if="dialogType === 'delete'")
            v-card-title.headline.justify-center アンケートを削除
            v-card-text アンケートを削除します。元に戻せません。
            v-card-actions
              v-spacer
              v-btn(
                @click="confirmDelete"
                :loading="isDeleting"
                :disabled="isDeleting"
                color="error"
                depressed
              ) 削除する
              v-btn(
                @click="dialog = false"
                :disabled="isDeleting"
                color="grey"
                outlined
              ) キャンセル
  v-card.mx-auto.d-flex.flex-column.align-center.creation-container(
    v-if="showSchedulingWindow"
    v-bind:style="`top: ${cardTop}px;`"
    elevation="5"
  )
    .header-container.d-flex.justify-center.flex-shrink-0
      h1.mb-5(v-if="!isNewSchedule") アンケートを登録します
      h1.mb-5(v-else) アンケートを編集します
      div
    schedule-survey(
      :survey="selectedSurvey"
      :units="filteredUnits"
      :byotos="filteredByotos"
      :occupations="occupations"
      @registered-survey="fetchSurveys"
      @updated="fetchSurveys"
      @cancel="showSchedulingWindow = false"
    )
</template>

<script>
import {
  mdiDownload,
  mdiTrashCanOutline,
  mdiPencilOutline,
  mdiClipboardMultipleOutline,
  mdiClockTimeFourOutline,
  mdiPlusThick,
  mdiEyeOutline,
} from '@mdi/js'
import moment from 'moment'
import cloneDeep from 'lodash/cloneDeep'

import ScheduleSurvey from '~/pagePartials/survey/ScheduleSurvey.vue'

import SideNavMenu from '~/components/Layouts/SideNavMenu/SideNavMenu.vue'

import Participants from '~/pagePartials/survey/Participants.vue'
import MapIcon from '~/pagePartials/profile/shared/Map.vue'

import PageMixin from '~/mixins/page.js'

export default {
  name: 'SurveySchedule',
  components: {
    ScheduleSurvey,
    SideNavMenu,
    Participants,
    MapIcon,
  },

  mixins: [PageMixin],

  middleware: ['admin', 'disableOnCorpUser'],
  async asyncData({ $axios, store }) {
    const [surveys, byotos, units, occupations] = await Promise.all([
      $axios.$get('/surveys', {
        params: {
          page: 'monolith',
        },
      }),
      store.dispatch('byotos/fetch'),
      store.dispatch('units/fetch'),
      store.dispatch('occupations/fetch'),
    ])

    const filteredByotos = byotos.filter(b => b.isDontBelong === false)
    const filteredUnits = units.filter(u => u.isDontBelong === false)

    return { surveys, filteredByotos, filteredUnits, occupations }
  },

  data() {
    const cardTop = 0
    const dialog = false
    const dialogType = null
    const surveyIdToBeDeleted = null
    const showSchedulingWindow = false
    const selectedSurvey = {}
    const hasUserSurveyIds = false
    const isDataTableLoading = false
    const isDeleting = false
    const totalItemLength = 0
    const link = ''
    return {
      cardTop,
      dialog,
      dialogType,
      selectedSurvey,
      hasUserSurveyIds,
      isDataTableLoading,
      isDeleting,
      totalItemLength,
      showSchedulingWindow,
      surveyIdToBeDeleted,
      link,
    }
  },

  fetch() {
    return this.$store.dispatch('survey/retrievePendingSurveys')
  },
  computed: {
    headers() {
      return [
        { text: '番号', value: 'number' },
        { text: '配布日', value: 'distributeAt' },
        { text: '回答期限', value: 'expireAt' },
        { text: '回答状況', value: 'status' },
        { text: '回答者数', value: 'targetUserCount' },
        { text: '配布先', value: 'targetDestinations', sortable: false },
        { text: '公開リンク', value: 'publicId' },
        { text: '編集', value: 'editButton', sortable: false },
      ]
    },
    isNewSchedule() {
      return this.selectedSurvey.id
    },
    checkPublic() {
      return !!this.selectedSurvey.publicId
    },

    hasOngoingSurvey() {
      return !!this.surveys.find(s => s.status === '回答中')
    },

    eyeIcon() {
      return mdiEyeOutline
    },

    editIcon() {
      return mdiPencilOutline
    },

    deleteIcon() {
      return mdiTrashCanOutline
    },

    createIcon() {
      return mdiPlusThick
    },

    clockIcon() {
      return mdiClockTimeFourOutline
    },

    copyIcon() {
      return mdiClipboardMultipleOutline
    },

    downloadIcon() {
      return mdiDownload
    },
  },

  mounted() {
    this.setLink()
  },
  methods: {
    setLink() {
      this.link = window.location.origin + '/survey/public/'
    },

    customSort(items, index, isDesc) {
      /**
       * Get a sort key for number.
       * Use padded strings as sort keys as we can hardly do any better with
       * the bult-in `Array.sort` function.
       */
      function numberSortKey(item) {
        if (item.periodicSurvey) {
          const number1 = String(item.periodicSurvey.number).padStart(20)
          const number2 = String(item.number).padStart(20)
          // Sort periodic survey first, so prepend 0
          return `0${number1}${number2}`
        } else {
          const number = String(item.number).padStart(20)
          return `1${number}`
        }
      }

      items.sort((a, b) => {
        if (index[0] === 'number') {
          const aKey = numberSortKey(a)
          const bKey = numberSortKey(b)

          if (aKey === bKey) return 0
          if (!isDesc[0]) {
            return aKey < bKey ? -1 : 1
          } else {
            return bKey < aKey ? -1 : 1
          }
        } else if (index[0] === 'targetUserCount') {
          if (a.answererCount === b.answererCount) return 0
          if (!isDesc[0]) {
            return a.answererCount < b.answererCount ? -1 : 1
          } else {
            return b.answererCount < a.answererCount ? -1 : 1
          }
        } else {
          if (a[index[0]] === b[index[0]]) return 0
          if (!isDesc[0]) {
            return a[index[0]] < b[index[0]] ? -1 : 1
          } else {
            return b[index[0]] < a[index[0]] ? -1 : 1
          }
        }
      })
      return items
    },

    checkForOngoingSurvey() {
      if (this.hasOngoingSurvey) {
        this.dialog = true
        this.dialogType = 'error'
      } else {
        this.openSidePanel()
      }
    },

    openSidePanel(survey) {
      if (survey) {
        this.selectedSurvey = cloneDeep(survey)
      } else {
        this.selectedSurvey = {
          id: null,
          distributeAt: '',
          expireAt: '',
          targetGlobal: true,
          targetByotoIds: [],
        }
      }
      this.showSchedulingWindow = true
      this.cardTop = window.pageYOffset
    },

    async fetchSurveys() {
      this.showSchedulingWindow = false
      this.isDataTableLoading = true
      await Promise.all([
        this.getSurveys().then(() => {
          this.isDataTableLoading = false
        }),
        this.$store.dispatch('survey/retrievePendingSurveys', {
          force: true,
        }),
      ])
    },
    async getSurveys() {
      this.surveys = await this.$axios.$get('/surveys', {
        params: {
          page: 'monolith',
        },
      })
    },
    copyToClipboard(linkText) {
      navigator.clipboard.writeText(linkText)
    },
    hasNoTargets(item) {
      return (
        !item.targetGlobal &&
        item.targetUnits.length === 0 &&
        item.targetByotos.length === 0 &&
        item.targetOccupations.length === 0
      )
    },

    onClickDelete(survey) {
      this.dialog = true
      this.dialogType = 'delete'
      this.surveyIdToBeDeleted = survey.id
    },

    async confirmDelete() {
      this.isDeleting = true
      const currentlySelectedSurvey =
        this.$store.getters['survey/currentlySelectedSurvey']
      if (
        currentlySelectedSurvey &&
        this.surveyIdToBeDeleted === currentlySelectedSurvey.id
      ) {
        this.$store.commit('survey/setSelectedSurvey', null)
      }
      await this.$axios.$delete(`/surveys/${this.surveyIdToBeDeleted}`)
      await this.fetchSurveys()
      this.isDeleting = false
      this.dialog = false
      this.surveyIdToBeDeleted = null
    },

    checkForPeriodicSurvey(survey) {
      return !!survey.periodicSurvey
    },

    /**
     * Tooltip options
     * Cannot be shared by computed
     * @return Object
     */
    tooltipOptions() {
      return { zIndex: 999, size: 'small', placement: 'right' }
    },

    suspendedOrPastDate(item) {
      const today = moment().format('YYYY-MM-DD')

      return item.periodicSurvey.suspended || today > item.expireAt
    },
  },
}
</script>
<style lang="stylus" scoped>
.page-administration-survey
  position: relative
  height: 100%

  .main-container
    .list-container
      .survey-list
        height: 100%
        :deep(.row)
          flex-wrap nowrap
        thead th
          background-color: #f3f3f3

  .creation-container
    position: absolute
    right: 0
    z-index: 10
    width: 500px
    .header-container
      width: 100%
      margin: 30px 0
      .close
        font-size: 1.8em

  .over-flow-container
    overflow-y: auto

  .flex-grow-1
    min-width: 0
    min-height: 0

  .survey-field
    border: 1px solid #dbdbdb
    border-radius: 0px
    background: #fafafa
  .public-survey-icon
    border-top: 1px solid #dbdbdb
    border-bottom: 1px solid #dbdbdb
    border-right: 1px solid #dbdbdb
    padding: 20px

  .survey-number
    white-space nowrap
</style>
