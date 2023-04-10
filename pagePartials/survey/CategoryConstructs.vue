<template lang="pug">
tr
  td.text-left
    slot(name="teamMenuScores")
    slot(name="personnelScoresBtn")
    slot(name="avatarImage")
    slot(name="answererName")
    slot(name="answererByotoAndTeam")

  td.text-left
    span {{ overall.grade }}
    span
      v-icon(:class="['arrow', progressionClass(overall.progression)]")
        | {{ progressionArrowIcon(overall.progression) }}

  td(v-for="cat in categories")
    span.mark(:style="markStyle(cat.mark)") {{ cat.mark }}
    span: v-icon(:class="['arrow', progressionClass(cat.progression)]")
      | {{ progressionArrowIcon(cat.progression) }}

  td(v-if="nominationCount !== null")
    span {{ nominationCount }}
</template>
<script>
import { mdiArrowLeftBold } from '@mdi/js'
import { ScoreStylingMixin } from '~/utilities/surveys'

export default {
  name: 'CategoryConstructs',

  mixins: [ScoreStylingMixin],

  props: {
    overall: {
      type: Object,
      default: null,
    },
    categories: {
      type: Array,
      required: true,
    },
    nominationCount: {
      type: Number,
      default: null,
    },
  },

  computed: {
    backButtonIcon() {
      return mdiArrowLeftBold
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
</style>
