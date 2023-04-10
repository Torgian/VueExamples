<template lang="pug">
.page-my-score
  side-nav-menu(menu="survey")

  v-row
    v-col.d-flex.pb-3(cols="12")
      v-progress-linear(
        :active="isDataTableLoading"
        :indeterminate="isDataTableLoading"
      )

  v-card.flex-shrink-1.flex-grow-1.d-flex.flex-column.main-container(
    flat
    v-if="surveyScoreData"
  )
    survey-selector(
      :surveys="surveyListForSelector"
      @update-score="returnSelectedSurveyScore"
    )

    .list-container.flex-grow-1
      .text-center
        p.caption(v-if="isPeriodicChartLoading") ロード中
        v-progress-circular(
          v-if="isPeriodicChartLoading"
          color="primary"
          :active="isPeriodicChartLoading"
          :indeterminate="isPeriodicChartLoading"
        )

        periodic-score-chart.chart(
          v-if="periodicSurveyScoreData && !onlyOneDataPoint"
          :surveyScoreData="periodicSurveyScoresForChart"
          :current-survey-id="currentDisplayedSurvey.id"
          :key="'periodic-' + periodicSurveyScoreKey"
        )

        p(v-if="onlyOneDataPoint") 時系列データを表示するためには、定期アンケートを2回以上実施する必要があります

        v-divider.my-5(v-if="isPeriodicChartLoading")

      template(v-if="targetUser")
        .d-flex.justify-center: v-img(
          :src="userAvatar(targetUser)"
          max-width="55"
        )
      personal-chart.chart(
        v-if="scoreDataForCharts"
        :surveyScoreData="scoreDataForCharts"
        :key="`personal-chart${scoreDataKey}`"
        :target-user="targetUser"
      )

      individual-score(
        :surveyScoreData="surveyScoreData"
        :tableHeaders="[headers]"
        @updateCharts="updateCharts"
      )
  v-card.flex-shrink-1.flex-grow-1.d-flex.flex-column.main-container(
    flat
    v-else
  )
    .none
      template(v-if="isDataTableLoading") ロード中
      template(v-else) {{ targetUser ? targetUser.name : 'あなた' }}はアンケートに回答したことがありません
</template>
<script>
import SideNavMenu from '~/components/Layouts/SideNavMenu/SideNavMenu.vue'

import SurveySelector from '~/pagePartials/survey/SurveySelector.vue'
import PersonalChart from '~/pagePartials/survey/PersonalChart.vue'
import PeriodicScoreChart from '~/pagePartials/survey/PeriodicScoreChart.vue'
import IndividualScore from '~/components/Elements/IndividualScore.vue'
import {
  ScoreDataFunctionsMixin,
  PeriodicScoreChartDataMixin,
} from '~/utilities/surveys'
import PageMixin from '~/mixins/page.js'

export default {
  name: 'MyScore',
  components: {
    SideNavMenu,
    SurveySelector,
    PersonalChart,
    IndividualScore,
    PeriodicScoreChart,
  },

  mixins: [PageMixin, ScoreDataFunctionsMixin, PeriodicScoreChartDataMixin],

  middleware: ['disableOnCorpUser'],

  async asyncData({ query, $auth, $axios }) {
    let targetUser = null

    if (query.userId && Number(query.userId) !== $auth.$state.user.id) {
      targetUser = await $axios.$get(`/users/${query.userId}`)
    }

    return { targetUser }
  },

  data() {
    const periodicSurveyId = null
    const surveyListForSelector = null
    const scoreDataForCharts = null

    return {
      periodicSurveyId,
      surveyListForSelector,
      scoreDataForCharts,
    }
  },

  computed: {
    onlyOneDataPoint() {
      return this.periodicSurveyScoreData?.overall.data.length < 2
    },
  },

  async mounted() {
    this.isDataTableLoading = true

    await this.$axios
      .$get('/survey-submissions', {
        params: {
          userId: this.targetUser ? this.targetUser.id : this.$me.id,
          page: 'monolith',
        },
      })
      .then(res => {
        this.submissions = res
      })

    if (this.submissions.length > 0) {
      this.surveyListForSelector = this.submissions.map(s => s.survey)

      /**
       * IDs from survey-submissions is different from surveys endpoint.
       * The below variables will find the current state's selected survey
       *  in the current user's list of submissions, if available.
       */
      const currentlySelectedSurvey =
        this.$store.getters['survey/currentlySelectedSurvey']

      const findSurvey = currentlySelectedSurvey
        ? this.submissions.find(s => s.surveyId === currentlySelectedSurvey.id)
        : null

      /**
       * If currentlySelectedSurvey is set, but findSurvey is not,
       *  then selectedSurvey will be initialized to the first survey in the list.
       *
       * This value is then used to set the selectedSurvey state.
       */
      const selectedSurvey = findSurvey || this.submissions[0]

      if (!findSurvey) {
        this.$store.commit('survey/setSelectedSurvey', selectedSurvey.survey)
      }

      if (selectedSurvey.survey.periodicSurvey) {
        this.getPeriodicSurveyScoreData(selectedSurvey.survey.periodicSurvey.id)
      }

      this.surveyScoreData = selectedSurvey.score
      this.scoreDataKey = selectedSurvey.id
    }
    this.isDataTableLoading = false
  },

  methods: {
    async getPeriodicSurveyScoreData(id) {
      this.isPeriodicChartLoading = true
      const url = this.targetUser
        ? `/periodic-surveys/${id}/score/users/${this.targetUser.id}`
        : `/periodic-surveys/${id}/my-score`

      await this.$axios
        .$get(url)
        .then(res => {
          this.periodicSurveyScoreData = res
          this.periodicSurveyId = id
        })
        .catch(() => {
          this.$toast.error(
            '定期アンケートスコアのロード時、予期せぬエラーが発生しました。申し訳ありません。'
          )
          this.periodicSurveyScoreData = null
          this.periodicSurveyId = null
        })
        .finally(() => {
          this.periodicSurveyScoreKey += 1
        })
      this.isPeriodicChartLoading = false
    },

    returnSelectedSurveyScore(selectedSurvey) {
      const submission = this.submissions.find(
        i => i.surveyId === selectedSurvey.id
      )

      if (submission.survey.periodicSurvey) {
        if (submission.survey.periodicSurvey.id !== this.periodicSurveyId) {
          this.getPeriodicSurveyScoreData(submission.survey.periodicSurvey.id)
        } else {
          this.periodicSurveyScoreKey += 1
        }
      } else {
        this.periodicSurveyScoreData = null
        this.periodicSurveyId = null
      }

      this.surveyScoreData = submission.score

      if (this.currentHeader) {
        const constructs = submission.score.all.perCategory.find(
          c => c.label === this.currentHeader
        ).perConstruct
        this.scoreDataForCharts = constructs
      } else {
        this.scoreDataForCharts = submission.score.all.perCategory
      }
      this.scoreDataKey = submission.id
    },

    updateCharts(scoreData, currentHeader) {
      this.scoreDataForCharts = scoreData
      this.currentHeader = currentHeader
      this.scoreDataKey += 1
      this.periodicSurveyScoreKey += 1
    },
  },
}
</script>
<style lang="stylus" scoped>
.page-my-score
  position: relative
  height: 100%
  & > .v-card
    padding: 12px

.chart {
  height 300px
  width 100%
}
</style>
