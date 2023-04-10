<template lang="pug">
.px-0.mx-0.mt-2
  v-simple-table.table-panel(fixed-header height="calc(100% - 60px)")
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
        th.text-left(v-for="header in currentHeaders" :key="header.text")
          div(v-if="selectedHeader === null")
            v-btn.header-border(
              @click="setHeader(header.text), color.reset()"
              small
              v-tippy="{ placement: 'bottom', size: 'small' }"
              content="詳しく見る"
              :style="`border-color: ${color.next()}`"
            ) {{ header.text }}
          div(v-else) {{ header.text }}

    tbody
      category-constructs(
        v-for="item in surveyScoreData"
        :key="`${selectedFilter}-${item.id}`"
        :overall="getOverall(selectedHeader, item.values)"
        :categories="getCategories(selectedHeader, item.values.perCategory)"
      )
        template(v-if="selectedFilter === 'byoto'" v-slot:teamMenuScores)
          shift-groups-menu(:perItem="item" :teamHeaders="teamHeaders")

        template(
          v-if="item.values.overall.mark && item.type !== 'all'"
          v-slot:personnelScoresBtn
        )
          personnel-scores-btn(
            :filteredType="selectedFilter"
            :filteredTypeId="item.id"
            :tableHeaders="tableHeaders"
            :surveyId="surveyId"
          )

        template(v-slot:answererName)
          span(v-if="item.type === 'all'") 全
          span(v-else) {{ item.name }}
</template>
<script>
import CategoryConstructs from '~/pagePartials/survey/CategoryConstructs.vue'
import ShiftGroupsMenu from '~/pagePartials/survey/ShiftGroupsMenu.vue'
import SurveyBackBtn from '~/pagePartials/survey/SurveyBackBtn.vue'
import PersonnelScoresBtn from '~/pagePartials/survey/PersonnelScoresBtn.vue'
import { WatchSetCategoryMixin } from '~/utilities/surveys'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'ScoreTable',

  components: {
    CategoryConstructs,
    ShiftGroupsMenu,
    SurveyBackBtn,
    PersonnelScoresBtn,
  },

  mixins: [WatchSetCategoryMixin],

  props: {
    surveyScoreData: {
      type: Array,
      required: true,
    },
    tableHeaders: {
      type: Array,
      required: true,
    },
    teamHeaders: {
      type: Array,
      required: true,
    },
    personnelScores: {
      type: Boolean,
      default: false,
    },
    surveyId: {
      type: Number,
      default: null,
    },
    selectedFilter: {
      type: String,
      default: null,
    },
  },

  data() {
    return {
      color: ColorCycle(),
    }
  },

  watch: {
    surveyScoreData() {
      this.color.reset()
    },
  },

  methods: {
    checkIsDontBelong(byotoObject) {
      // Checks if byotoObject is null / undefined
      if (!!byotoObject) {
        return byotoObject.isDontBelong
      } else {
        return true
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

.table-panel
  :deep(table)
    min-width max-content
</style>
