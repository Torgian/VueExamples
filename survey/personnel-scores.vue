<template lang="pug">
.page-personnel-scores
  side-nav-menu(menu="survey")

  v-row
    v-col.d-flex.pb-5(cols="12")
      v-progress-linear(
        :active="isDataTableLoading"
        :indeterminate="isDataTableLoading"
      )

  nav.search
    v-layout.pb-2(wrap justify-space-between)
      v-flex(xs12)
        v-col.col-md-4.px-0(cols="12")
          survey-selector(
            :surveys="surveys"
            :class="isMobile ? '' : 'px-1'"
            @update-score="setSurveyId"
          )

      v-flex(xs12 md4)
        personnel-filter(
          v-if="filterData.answererUnits.length"
          :filtersLoading="filtersLoading"
          :items="filterData.answererUnits"
          label="部署"
          v-model="selectedUnitIds"
          :rules="[validateUnits]"
        )
        personnel-filter(
          v-if="filterData.answererByotos.length"
          :filtersLoading="filtersLoading"
          :items="filterData.answererByotos"
          label="所属"
          v-model="selectedByotoIds"
          :rules="[validateByotos]"
        )

      v-flex.align-center(xs12 md4)
        v-select(
          dense
          multiple
          chips
          counter
          clearable
          label="役職"
          v-model="selectedPositionIds"
          item-text="name"
          item-value="id"
          :items="positions"
        )
          template(v-slot:selection="{ item, index }")
            v-chip(v-if="index === 0")
              span {{ item.name }}
            span.grey--text.caption(v-if="index === 1")
              | (+{{ selectedPositionIds.length - 1 }})

          order-by-button(
            slot="prepend"
            kind="positions.sort"
            v-model="orderBy"
          )

        personnel-filter(
          :filtersLoading="filtersLoading"
          :items="filterData.answererOccupations"
          is-occupation
          label="職種"
          v-model="selectedOccupationIds"
        )

      v-flex(xs12 md4)
        .label 臨床年数
        v-row.align-center(no-gutters)
          v-col.d-flex.align-end(cols=6)
            v-select(
              v-model="selectedYearsClinicalStartedGte"
              :items="yearsClinicalGteRange"
              item-text="name"
              item-value="value"
              dense
              hide-details
            )
            span.year-suffix 年以上

          v-col.d-flex.align-end(cols=6)
            v-select.ml-1(
              v-model="selectedYearsClinicalStartedLte"
              :items="yearsClinicalLteRange"
              item-text="name"
              item-value="value"
              dense
              hide-details
            )
            span.year-suffix 年以下

        .mt-2.label 当施設勤続年数
        v-row.align-center(no-gutters)
          v-col.d-flex.align-end(cols=6)
            v-select.year-selector(
              v-model="selectedYearsJoinedGte"
              :items="yearsJoinedGteRange"
              item-text="name"
              item-value="value"
              dense
              hide-details
            )
            span.year-suffix 年以上

          v-col.d-flex.align-end(cols=6)
            v-select.ml-1(
              v-model="selectedYearsJoinedLte"
              :items="yearsJoinedLteRange"
              item-text="name"
              item-value="value"
              dense
              hide-details
            )
            span.year-suffix 年以下

  v-card.flex-shrink-1.d-flex.flex-column.main-container(
    flat
    v-if="surveys.length > 0"
  )
    .list-container.flex-grow-1(v-if="scoresAvailable")
      personnel-chart.chart(:surveyScoreData="surveyData" :key="scoreDataKey")

      personnel-scores-table(
        :survey-score-data="surveyData"
        :nominated-users="nominatedUsers"
        :table-headers="headers"
      )

    .list-container.flex-grow-1(v-else-if="!isDataTableLoading")
      .none アンケートのスコアがありません

  .list-container.flex-grow-1(v-else)
    .none アンケートがありません
</template>
<script>
import PageMixin from '~/mixins/page.js'

import SideNavMenu from '~/components/Layouts/SideNavMenu/SideNavMenu.vue'

import SurveySelector from '~/pagePartials/survey/SurveySelector.vue'
import { mainTableHeaders } from '~/utilities/surveys'
import PersonnelFilter from '~/pagePartials/survey/PersonnelFilter.vue'
import PersonnelChart from '~/pagePartials/survey/PersonnelChart.vue'
import PersonnelScoresTable from '~/pagePartials/survey/PersonnelScoresTable.vue'

import OrderByButton from '~/components/Elements/OrderByButton.vue'

import {
  buildGteRange,
  buildLteRange,
} from '~/pagePartials/roster/mixins/search.js'

/**
 * Helper functions to retrieve filter data
 */
const FilterData = {
  new() {
    return {
      answererByotos: [],
      answererOccupations: [],
      answererUnits: [],
    }
  },

  /**
   * Retrieve filter data for a specific survey
   *
   * @param {Object} $axios Axios instance
   * @param {Object} me Logged user's instance
   * @param {number} surveyId
   * @return {Promise}
   */
  retrieve($axios, me, surveyId) {
    return $axios.$get(`/surveys/${surveyId}`).then(res => {
      return {
        answererByotos: res.answererByotos.filter(byoto => {
          if (me.privilege && me.privilege.general.all) {
            return true
          }

          if (
            me.byoto &&
            !me.byoto.isDontBelong &&
            me.privilege.general.sameByoto &&
            me.byoto.id === byoto.id
          ) {
            return true
          }

          if (me.privilege && me.privilege.general.byotoIds[byoto.id]) {
            return true
          }

          return false
        }),
        answererUnits: res.answererUnits.filter(unit => {
          if (me.privilege && me.privilege.general.all) {
            return true
          }

          if (
            me.unit &&
            !me.unit.isDontBelong &&
            me.privilege.general.sameUnit &&
            me.unit.id === unit.id
          ) {
            return true
          }

          if (me.privilege && me.privilege.general.unitIds[unit.id]) {
            return true
          }
        }),
        answererOccupations: res.answererOccupations,
      }
    })
  },
}

/**
 * Helper to parse ids into arrays in query parameters
 */
function parseQueryIdParam(param) {
  if (!param) {
    return []
  }

  // Convert to array
  const ids = Array.isArray(param) ? param : [param]

  return (
    ids
      // Convert to ints
      .map(id => parseInt(id))
      // Get rid of NaN values (which were not valid ints)
      .filter(id => !isNaN(id))
  )
}

/**
 * Helper to return selected year as a number for query parameters
 */
function parseYearsParams(param) {
  const yearId = parseInt(param)

  if (Number.isInteger(yearId)) {
    return yearId
  }
}

export default {
  name: 'PersonnelScores',

  components: {
    SideNavMenu,
    SurveySelector,
    PersonnelFilter,
    PersonnelChart,
    PersonnelScoresTable,
    OrderByButton,
  },

  mixins: [PageMixin],

  middleware: ['generalAdmin', 'disableOnCorpUser'],

  async asyncData({ $axios, query, store }) {
    const me = store.getters['user/me']

    const positions = await store.dispatch('positions/fetch')

    const surveys = await $axios.$get('/surveys', {
      params: {
        page: 'monolith',
      },
    })

    let surveyId = null

    let filterData = FilterData.new()

    if (surveys.length > 0) {
      const currentlySelectedSurvey =
        store.getters['survey/currentlySelectedSurvey'] || surveys[0]
      surveyId = query.surveyId ? query.surveyId : currentlySelectedSurvey.id

      filterData = await FilterData.retrieve($axios, me, surveyId)
    }

    // TODO: Validate that the logged user has read access to the item selected
    // by the query
    const selectedByotoIds = parseQueryIdParam(query.byotoId)
    const selectedUnitIds = parseQueryIdParam(query.unitId)
    const selectedOccupationIds = parseQueryIdParam(query.occupationId)
    const selectedPositionIds = parseQueryIdParam(query.positionId)

    const orderBy = query.orderBy
    const selectedYearsJoinedLte = parseYearsParams(query.yearsJoinedLte)
    const selectedYearsJoinedGte = parseYearsParams(query.yearsJoinedGte)
    const selectedYearsClinicalStartedLte = parseYearsParams(
      query.yearsClinicalStartedLte
    )
    const selectedYearsClinicalStartedGte = parseYearsParams(
      query.yearsClinicalStartedGte
    )

    // Set default value for selected byoto ids and unit ids for general users
    if (!me.privilege.general.all) {
      if (!selectedByotoIds.length && filterData.answererByotos.length) {
        selectedByotoIds.push(filterData.answererByotos[0].id)
      }
      if (!selectedUnitIds.length && filterData.answererUnits.length) {
        selectedUnitIds.push(filterData.answererUnits[0].id)
      }
    }

    return {
      positions,
      surveys,
      surveyId,
      filterData,

      selectedByotoIds,
      selectedUnitIds,
      selectedOccupationIds,
      selectedPositionIds,

      orderBy,
      selectedYearsJoinedLte,
      selectedYearsJoinedGte,
      selectedYearsClinicalStartedLte,
      selectedYearsClinicalStartedGte,
    }
  },

  data() {
    const isDataTableLoading = false
    const surveyData = null
    const nominatedUsers = []
    const filtersLoading = false
    const fetchCancelTokenSource = null
    const scoreDataKey = null
    const viewportWidth = 0

    return {
      isDataTableLoading,
      surveyData,
      nominatedUsers,
      filtersLoading,
      scoreDataKey,
      /**
       * Source of the cancel token for the currently running axios request.
       * Value is `null` if there is no running axios request.
       */
      fetchCancelTokenSource,
      viewportWidth,
    }
  },

  async fetch() {
    const { surveyId } = this

    if (surveyId) {
      const requestParam = {
        surveyId,
        ...this.selectedFilters,
      }
      await this.retrieveSurveySubmissions(requestParam)
    }
  },

  computed: {
    headers() {
      const mainHeaders = mainTableHeaders(this.surveyData[0].score)
      return mainHeaders
    },

    scoresAvailable() {
      return this.surveyData ? this.surveyData.length > 0 : null
    },

    selectedSurvey() {
      const surveyToPass =
        this.surveys.find(s => s.id === Number(this.surveyId)) ||
        this.surveys[0]
      return surveyToPass
    },

    selectedFilters() {
      return {
        byotoId: this.selectedByotoIds,
        unitId: this.selectedUnitIds,
        occupationId: this.selectedOccupationIds,
        positionId: this.selectedPositionIds,
        orderBy: this.orderBy,
        yearsJoinedLte: parseYearsParams(this.selectedYearsJoinedLte),
        yearsJoinedGte: parseYearsParams(this.selectedYearsJoinedGte),
        yearsClinicalStartedLte: parseYearsParams(
          this.selectedYearsClinicalStartedLte
        ),
        yearsClinicalStartedGte: parseYearsParams(
          this.selectedYearsClinicalStartedGte
        ),
      }
    },

    /**
     * For users who cannot see all units and byotos, unit and byoto filtering
     * cannot be left empty. Else, they would see users outside of their access
     * rights!
     */
    unitsByotosAreRequired() {
      return !this.$me.privilege.general.all
    },

    /**
     * '960' is the same width used for the breakpoint used in `flex-md-row`.
     */
    isMobile() {
      return this.viewportWidth < 960
    },

    yearsJoinedGteRange() {
      let end = this.selectedYearsJoinedLte

      if (!parseInt(end)) {
        end = 15
      }
      return buildGteRange(end)
    },

    yearsJoinedLteRange() {
      const start = this.selectedYearsJoinedGte || 0
      return buildLteRange(start)
    },

    yearsClinicalGteRange() {
      let end = this.selectedYearsClinicalStartedLte

      if (!parseInt(end)) {
        end = 15
      }
      return buildGteRange(end)
    },

    yearsClinicalLteRange() {
      const start = this.selectedYearsClinicalStartedGte || 0
      return buildLteRange(start)
    },
  },

  watch: {
    surveyId() {
      this.selectedByotoIds = []
      this.selectedUnitIds = []
      this.selectedOccupationIds = []
      this.selectedPositionIds = []
      this.selectedYearsJoinedLte = null
      this.selectedYearsJoinedGte = null
      this.selectedYearsClinicalStartedLte = null
      this.selectedYearsClinicalStartedGte = null
    },

    selectedFilters() {
      this.getFilteredResults()
    },
  },

  mounted() {
    window.addEventListener('resize', this.updateViewportWidth)
    this.updateViewportWidth()
  },

  destroyed() {
    window.removeEventListener('resize', this.updateViewportWidth)
  },

  methods: {
    setSurveyId(survey) {
      this.surveyId = survey.id
    },

    /**
     * Retrieve survey submissions from backend, with given parameters
     *
     * This function sets `surveyData`.
     */
    retrieveSurveySubmissions(requestParam) {
      // Abort search for users who cannot see all units and byotos,
      // But still did not select any byoto nor unit.
      if (this.unitsByotosAreRequired) {
        if (!requestParam.unitId.length && !requestParam.byotoId.length) {
          return Promise.resolve()
        }
      }

      this.isDataTableLoading = true

      // If there is a previous request currently being run, we first cancel it
      if (this.fetchCancelTokenSource) {
        this.fetchCancelTokenSource.cancel('New request issued')
      }

      const source = this.$axios.CancelToken.source()
      this.fetchCancelTokenSource = source

      const surveySubmissionsPromise = this.$axios.$get('/survey-submissions', {
        params: { ...requestParam, page: 'monolith' },
        cancelToken: source.token,
      })

      // In addition, retrieve nominated users who did not submit this survey
      const nominatedUsersUrl = `/surveys/${requestParam.surveyId}/nominated-users`
      const nominatedUsersPromise = this.$axios.$get(nominatedUsersUrl, {
        params: { ...requestParam, submitted: false },
        cancelToken: source.token,
      })

      return Promise.all([surveySubmissionsPromise, nominatedUsersPromise])
        .then(([submissions, nominatedUsers]) => {
          this.surveyData = submissions
          this.nominatedUsers = nominatedUsers
          this.scoreDataKey += 1
          // As request is completed, we set to null
          this.fetchCancelTokenSource = null
        })
        .catch(err => {
          // If request was cancelled,
          if (this.$axios.isCancel(err)) {
            // We finish with success
            return Promise.resolve()

            // Otherwise we make the error propagate
          } else {
            // As request is completed, we set to null
            this.fetchCancelTokenSource = null

            return Promise.reject(err)
          }
        })
        .finally(() => {
          this.isDataTableLoading = false
        })
    },

    async getFilteredResults() {
      const requestParam = {
        surveyId: this.surveyId,
        ...this.selectedFilters,
      }

      this.$router.replace({ query: requestParam })

      await this.retrieveSurveySubmissions(requestParam)
    },

    async getFilterData() {
      this.filtersLoading = true

      this.filterData = await FilterData.retrieve(
        this.$axios,
        this.$me,
        this.surveyId
      )

      this.filtersLoading = false
    },

    /**
     * For users who have no read access to *all* units, at least one unit
     * must be selected.
     */
    validateUnits(unitIds) {
      if (this.unitsByotosAreRequired) {
        if (unitIds.length < 1) {
          return '部署を必ず入力してください'
        }
      }
      return true
    },

    /**
     * For users who have no read access to *all* byotos, at least one byoto
     * must be selected.
     */
    validateByotos(byotoIds) {
      if (this.unitsByotosAreRequired) {
        if (byotoIds.length < 1) {
          return '所属を必ず入力してください'
        }
      }
      return true
    },

    updateViewportWidth() {
      this.viewportWidth = window.innerWidth
    },
  },
}
</script>
<style lang="stylus" scoped>
.page-personnel-scores
  position relative
  height 100%
  overflow-x hidden
  & > .v-card
    padding: 12px

.search
  border-radius 3px
  box-shadow 0 2px 5px rgba(black, .7) inset
  padding 20px
  font-size 80%
  margin-bottom 20px
  .flex
    padding 0px 12px !important

.chart
  height 300px
  width 100%

.row-width
  max-width 75%
  align-self flex-end

.year-suffix
  white-space nowrap
</style>
