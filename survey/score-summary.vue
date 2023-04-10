<template lang="pug">
.page-administration-survey
  side-nav-menu(menu="survey")

  v-card.flex-shrink-1.d-flex.flex-column.main-container(
    flat
    v-if="surveys.length > 0"
  )
    .d-flex.flex-column.flex-md-row(:class="{ 'row-width': !isMobile }")
      survey-selector(
        :surveys="surveys"
        :class="isMobile ? '' : 'px-1'"
        @update-score="getScores"
      )

      div(:class="isMobile ? 'pb-1 pt-0' : 'px-1'")
        v-select(
          v-model="selectedFilterKey"
          :items="filterTypes"
          filled
          dense
          label="フィルター"
          item-value="key"
          item-text="label"
          @change="setFilteredData"
        )

    v-row.pb-5
      v-col.d-flex.pb-0(cols="12")
        v-progress-linear.loading-bar(
          :active="isDataTableLoading || isPeriodicChartLoading"
          :indeterminate="isDataTableLoading || isPeriodicChartLoading"
        )

    .list-container.flex-grow-1(v-if="scoresAvailable")
      .text-center
        .text-center(v-if="notEnoughDataPoints")
          p 時系列データを表示するためには、定期アンケートを2回以上実施する必要があります
        div(v-else)
          periodic-score-chart.chart(
            v-if="periodicSurveyScoreData"
            :filter="selectedFilter.type"
            :surveyScoreData="periodicSurveyScoreData"
            :current-survey-id="currentDisplayedSurvey.id"
            :key="'periodic-' + periodicSurveyScoreKey"
          )

      summary-chart.chart(
        :surveyScoreData="surveyScoreDataRows"
        :key="scoreDataKey"
      )

      score-table(
        :surveyScoreData="surveyScoreDataRows"
        :tableHeaders="headers.mainHeaders"
        :teamHeaders="headers.teamHeaders"
        :surveyId="currentDisplayedSurvey.id"
        :selectedFilter="selectedFilter.type"
      )

    .flex-shrink-1.flex-grow-1.d-flex.flex-column.main-container(flat v-else)
      .none アンケートスコアーがありません
  .flex-shrink-1.flex-grow-1.d-flex.flex-column.main-container(flat v-else)
    .none アンケートがありません
</template>

<script>
import SummaryChart from '~/pagePartials/survey/SummaryChart.vue'

import SideNavMenu from '~/components/Layouts/SideNavMenu/SideNavMenu.vue'

import ScoreTable from '~/pagePartials/survey/ScoreTable.vue'
import PageMixin from '~/mixins/page.js'
import SurveySelector from '~/pagePartials/survey/SurveySelector.vue'
import PeriodicScoreChart from '~/pagePartials/survey/PeriodicScoreChart.vue'

import {
  ScoreDataFunctionsMixin,
  PeriodicScoreChartDataMixin,
} from '~/utilities/surveys'

/**
 * Retrieve survey score of the survey with the provided ID, from the backend API.
 * We then treat the data to only keep authorized units and byotos.
 */
async function getSurveyScore($axios, surveyId, me) {
  const surveyScoreData = await $axios
    .$get(`/surveys/${surveyId}/score`)
    .then(res => {
      res.score.all.values = { ...res.score.all }
      res.score.all.type = 'all'
      res.score.all.name = '全'
      return res.score
    })

  // Only keep authorized units and byotos
  const authorizedUnits = surveyScoreData.perUnit.filter(unit => {
    if (me.privilege && me.privilege.general.all) {
      return true
    }

    if (me.unit && me.privilege.general.sameUnit && me.unit.id === unit.id) {
      return true
    }

    if (me.privilege && me.privilege.general.unitIds[unit.id]) {
      return true
    }

    return false
  })

  const authorizedByotos = surveyScoreData.perByoto.filter(byoto => {
    if (me.privilege && me.privilege.general.all) {
      return true
    }

    if (
      me.byoto &&
      me.privilege.general.sameByoto &&
      me.byoto.id === byoto.id
    ) {
      return true
    }

    if (me.privilege && me.privilege.general.byotoIds[byoto.id]) {
      return true
    }

    return false
  })

  surveyScoreData.perUnit = authorizedUnits
  surveyScoreData.perByoto = authorizedByotos

  return surveyScoreData
}

/**
 * Logic to retrieve available filter keys for the logged user (me)
 */
function availableFilterKeys(me) {
  const filterKeys = []

  const general = me.privilege.general

  // We can only filter by units if we have read-access to some units
  const hasSameUnit = me.unit && !me.unit.isDontBelong && general.sameUnit
  const hasUnitIds = Object.keys(general.unitIds).length > 0
  if (general.all || hasSameUnit || hasUnitIds) {
    filterKeys.push('perUnit')
  }

  // We can only filter by byotos if we have read-access to some byotos
  const hasSameByoto = me.byoto && !me.byoto.isDontBelong && general.sameByoto
  const hasByotoIds = Object.keys(general.byotoIds).length > 0
  if (general.all || hasSameByoto || hasByotoIds) {
    filterKeys.push('perByoto')
  }

  // We can always filter by occupation
  filterKeys.push('perOccupation')

  return filterKeys
}

export default {
  name: 'SurveyScore',
  components: {
    SummaryChart,
    SideNavMenu,
    ScoreTable,
    SurveySelector,
    PeriodicScoreChart,
  },

  mixins: [PageMixin, ScoreDataFunctionsMixin, PeriodicScoreChartDataMixin],

  middleware: ['generalAdmin', 'disableOnCorpUser'],

  async asyncData({ store, $auth, $axios }) {
    let surveyScoreData = null
    const surveys = await $axios.$get('/surveys', {
      params: {
        page: 'monolith',
      },
    })

    const me = $auth.user
    if (surveys.length > 0) {
      const currentlySelectedSurvey =
        store.getters['survey/currentlySelectedSurvey'] || surveys[0]

      surveyScoreData = await getSurveyScore(
        $axios,
        currentlySelectedSurvey.id,
        me
      )
    }

    const filterKeys = availableFilterKeys(me)
    // Select the first of the available filter keys as default
    const selectedFilterKey = filterKeys[0]

    return { surveys, surveyScoreData, selectedFilterKey }
  },

  fetch() {
    return this.$store.dispatch('survey/retrievePendingSurveys')
  },

  computed: {
    selectedFilter() {
      return this.filterTypes.find(type => type.key === this.selectedFilterKey)
    },

    scoresAvailable() {
      return this.surveyScoreData
        ? !!this.surveyScoreData.all.overall.mark
        : null
    },

    /**
     * Return the available filter types for the logged user (me)
     */
    filterTypes() {
      const allFilterTypes = [
        {
          key: 'perUnit',
          type: 'unit',
          label: '部署',
        },
        {
          key: 'perByoto',
          type: 'byoto',
          label: '所属',
        },
        {
          key: 'perOccupation',
          type: 'occupation',
          label: '職種',
        },
      ]

      const filterKeys = availableFilterKeys(this.$me)

      return filterKeys.map(filterKey => {
        const filterType = allFilterTypes.find(
          filterType => filterType.key === filterKey
        )
        if (!filterType) {
          throw new Error(`Unhandled filter key: '${filterType}'`)
        }

        return filterType
      })
    },

    surveyScoreDataRows() {
      const scoreData = [
        this.surveyScoreData.all,
        ...this.surveyScoreData[this.selectedFilter.key],
      ]

      return scoreData
    },

    /**
     * '960' is the same width used for the breakpoint used in `flex-md-row`.
     */
    isMobile() {
      return this.viewportWidth < 960
    },
  },

  mounted() {
    window.addEventListener('resize', this.updateViewportWidth)
    this.updateViewportWidth()
    this.setFilteredData()
  },

  destroyed() {
    window.removeEventListener('resize', this.updateViewportWidth)
  },

  methods: {
    async getScores(survey) {
      this.isDataTableLoading = true

      const promises = []
      if (survey.periodicSurvey) {
        const periodicSurveyPromise = this.retrieveAggregatedPeriodicScores(
          survey.periodicSurvey.id
        )
        promises.push(periodicSurveyPromise)
      } else {
        this.periodicSurveyScoreData = null
      }

      const surveyPromise = getSurveyScore(
        this.$axios,
        survey.id,
        this.$me
      ).then(surveyScoreData => {
        this.surveyScoreData = surveyScoreData
        this.scoreDataKey = survey.id
      })

      promises.push(surveyPromise)

      await Promise.all(promises)

      this.isDataTableLoading = false
    },

    setFilteredData() {
      if (this.currentDisplayedSurvey?.periodicSurvey) {
        this.retrieveAggregatedPeriodicScores(
          this.currentDisplayedSurvey.periodicSurvey.id
        )
      }
      this.scoreDataKey += 1
    },

    updateViewportWidth() {
      this.viewportWidth = window.innerWidth
    },
  },
}
</script>
<style lang="stylus" scoped>
.page-administration-survey
  position: relative
  height: 100%
  & > .v-card
    padding: 12px

  .main-container
    .list-container
      .survey-score-list
        height: 100%
        thead th
          background-color: #f3f3f3

.chart
  height 300px
  width 100%

.loading-bar
  position absolute
  left 0

.row-width
  max-width 75%
  align-self flex-end
</style>
