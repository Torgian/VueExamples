<template lang="pug">
.px-0.mx-0.mt-2
  v-simple-table.personnel-table(fixed-header height="calc(100% - 60px)")
    thead
      tr
        th 名前
        th
          div(v-if="selectedHeader === null") 総合スコア
          survey-back-btn(
            v-else
            :selectedHeader="selectedHeader"
            @goBack="goBack"
          )
        th.text-left(v-for="header in renderedHeaders" :key="header.text")
          div(v-if="selectedHeader === null")
            v-btn.header-border(
              @click="setHeader(header.text)"
              small
              v-tippy="{ placement: 'bottom', size: 'small' }"
              content="詳しく見る"
              :style="`border-color: ${header.color}`"
            ) {{ header.text }}
          div(v-else) {{ header.text }}
        th
          v-btn(@click="toggleNominationSort()" small) 指名回数
            v-icon(small) {{ arrowIcon }}

    tbody
      category-constructs(
        v-for="answerer in surveyDataForTable"
        :key="`answerer-${answerer.user ? `user${answerer.user.id}` : answerer.id}`"
        :overall="getOverall(selectedHeader, answerer.score.all)"
        :categories="getCategories(selectedHeader, answerer.score.all.perCategory)"
        :nomination-count="answerer.score.nominationCount"
      )
        template(v-slot:avatarImage)
          link-wrapper.user-link(:to="link(answerer)")
            v-img.mt-2(:src="userAvatar(answerer.user)" width="50" height="50")
            span {{ answerer.answererName }}
            .chips
              v-chip(v-if="!!answerer.answererRealByoto" x-small) {{ answerer.answererRealByoto.name }}
              v-chip(v-else) --
              v-chip(
                v-if="!!answerer.answererRealTeam && !answerer.answererRealTeam.isDontBelong && !checkIsDontBelong(answerer.answererRealByoto)"
                x-small
              ) {{ answerer.answererRealTeam.name }}

    tfoot(v-if="!allScoresLoaded")
      tr
        td(:colspan="currentHeaders.length")
          observer(@intersect="handleIntersect()")
</template>
<script>
import sortBy from 'lodash/sortBy'
import { mdiArrowUp, mdiArrowDown, mdiMinus } from '@mdi/js'

import CategoryConstructs from '~/pagePartials/survey/CategoryConstructs.vue'
import SurveyBackBtn from '~/pagePartials/survey/SurveyBackBtn.vue'
import { WatchSetCategoryMixin } from '~/utilities/surveys'
import Observer from '~/components/Elements/Observer.vue'
import LinkWrapper from '~/components/Containers/LinkWrapper.vue'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'PersonnelScoresTable',

  components: {
    CategoryConstructs,
    SurveyBackBtn,
    Observer,
    LinkWrapper,
  },

  mixins: [WatchSetCategoryMixin],

  props: {
    surveyScoreData: {
      type: Array,
      required: true,
    },

    nominatedUsers: {
      type: Array,
      default: () => [],
    },

    tableHeaders: {
      type: Array,
      required: true,
    },
  },

  data() {
    return {
      currentPage: 1,
      allScoresLoaded: false,
      sortNominationCount: false,
    }
  },

  computed: {
    arrowIcon() {
      let icon = mdiMinus
      if (this.sortNominationCount) {
        icon = this.sortNominationCount === 'down' ? mdiArrowDown : mdiArrowUp
      }
      return icon
    },

    renderedHeaders() {
      const colorCycle = ColorCycle()
      // First color of the cycle is for 総合スコア,
      // as displayed in the PeriodicScoreChart
      // As 総合スコア is not shown on the PersonnelScoresTable, we skip it.
      colorCycle.next()

      return this.currentHeaders.map(header => {
        return {
          ...header,
          color: colorCycle.next(),
        }
      })
    },

    surveyDataForTable() {
      // 20 users per page
      const lastUserIndex = this.currentPage * 20

      const surveyScoreData = [...this.surveyScoreData, ...this.nominatedUsers]

      let sortedData = null
      if (this.sortNominationCount) {
        sortedData =
          this.sortNominationCount === 'down'
            ? sortBy(surveyScoreData, x => -x.score.nominationCount)
            : sortBy(surveyScoreData, x => x.score.nominationCount)
      } else {
        sortedData = surveyScoreData
      }
      return sortedData.slice(0, lastUserIndex)
    },
  },

  watch: {
    surveyScoreData() {
      this.currentPage = 1
      this.allScoresLoaded = false
    },
  },

  methods: {
    handleIntersect() {
      if (this.surveyScoreData.length > this.surveyDataForTable.length) {
        this.currentPage += 1
      } else {
        this.allScoresLoaded = true
      }
    },

    checkIsDontBelong(byotoObject) {
      // Checks if byotoObject is null / undefined
      if (!!byotoObject) {
        return byotoObject.isDontBelong
      } else {
        return true
      }
    },

    toggleNominationSort() {
      switch (this.sortNominationCount) {
        case false:
          this.sortNominationCount = 'down'
          break
        case 'down':
          this.sortNominationCount = 'up'
          break
        default:
          this.sortNominationCount = false
          break
      }
    },

    /**
     * Link to user's profile
     * @return string | undefined
     */
    link(answerer) {
      if (answerer.user) {
        return `/survey/my-score?userId=${answerer.user.id}`
      } else {
        return undefined
      }
    },
  },
}
</script>
<style lang="stylus" scoped>
.header-border
  border 2px solid

.chips
  margin-bottom: 5px
  .v-chip
    margin-right 3px

.user-link
  // Do not show underscore/green color on user links, as it makes the UI ugly
  text-decoration inherit
  color inherit

.personnel-table
  :deep(table)
    min-width max-content
</style>
