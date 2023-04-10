<template lang="pug">
v-menu(
  v-if="perItem.perTeam && perItem.perTeam.length > 0"
  :close-on-content-click="false"
  :nudge-width="200"
  offset-x
)
  template(v-slot:activator="{ on, attrs }")
    v-btn.mr-1(
      fab
      depressed
      tile
      small
      v-bind="attrs"
      v-on="on"
      v-tippy="{ placement: 'right', size: 'small' }"
      content="チーム"
    )
      v-icon(large) {{ eyeIcon }}

  v-card
    v-simple-table.table-panel.px-4.py-4(fixed-header)
      thead
        tr
          th {{ perItem.name }}
          th
            div(v-if="selectedTeamHeader === null") 総合スコア
            div(v-else)
              survey-back-btn(
                :selectedHeader="selectedTeamHeader"
                @goBack="selectedTeamHeader = null"
              )

          th.text-left(v-for="header in currentTeamHeaders" :key="header.text")
            v-btn(
              v-if="selectedTeamHeader === null"
              @click="selectedTeamHeader = header.text"
              small
              v-tippy="{ placement: 'bottom', size: 'small' }"
              content="詳しく見る"
            ) {{ header.text }}
            div(v-else) {{ header.text }}

      tbody
        tr.team-row(v-for="team in perItem.perTeam")
          .team-names
            p.caption.mb-0 {{ returnTeamShiftGroupName(team.shiftGroup) }}
            p.text-right.mr-5.mb-0 {{ team.name }}

          td
            span(v-if="!selectedTeamHeader") {{ team.values.overall.grade }}
              v-icon(
                :class="['arrow', progressionClass(team.values.overall.progression)]"
              )
                | {{ progressionArrowIcon(team.values.overall.progression) }}
            span(v-else) {{ overallScore(team.values.perCategory).grade }}
              v-icon(
                :class="['arrow', progressionClass(overallScore(team.values.perCategory).progression)]"
              )
                | {{ progressionArrowIcon(overallScore(team.values.perCategory).progression) }}

          td(
            v-for="category in setTeamCategories(team.values.perCategory)"
            :key="category.label"
          )
            span.mark(:style="markStyle(category.mark)") {{ category.mark }}
            span: v-icon(
              :class="['arrow', progressionClass(category.progression)]"
            )
              | {{ progressionArrowIcon(category.progression) }}
</template>
<script>
import { mdiEyeOutline } from '@mdi/js'
import SurveyBackBtn from '~/pagePartials/survey/SurveyBackBtn.vue'
import {
  WatchSetTeamCategoryMixin,
  ScoreStylingMixin,
} from '~/utilities/surveys'

export default {
  name: 'ShiftGroupsMenu',

  components: {
    SurveyBackBtn,
  },

  mixins: [WatchSetTeamCategoryMixin, ScoreStylingMixin],

  props: {
    perItem: {
      type: Object,
      required: true,
    },
    teamHeaders: {
      type: Array,
      required: true,
    },
  },

  computed: {
    eyeIcon() {
      return mdiEyeOutline
    },
  },

  methods: {
    returnTeamShiftGroupName(shiftGroup) {
      return shiftGroup?.name
    },
  },
}
</script>
<style lang="stylus" scoped>
.mark
  padding: 5px
  border-style: solid
  border-width: 2px
  border-color: rgba(0,0,0,0.0)

.progression-positive
  &.arrow
    color: green

.progression-zero
  &.arrow
    color: gray

.progression-negative
  &.arrow
    color: red

.team-row:nth-child(n+2)
  .team-names
    p
      margin-top 12px

.table-panel
  :deep(table)
    min-width max-content
</style>
