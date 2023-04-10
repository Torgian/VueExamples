<template lang="pug">
span
  v-btn.public-survey-icon(
    :to="href"
    fab
    depressed
    tile
    small
    v-tippy="{ placement: 'bottom', size: 'small' }"
    content="個人スコア"
  )
    v-icon {{ icon }}
</template>
<script>
import { mdiAccountBoxMultiple } from '@mdi/js'

export default {
  name: 'PersonnelScoresBtn',
  props: {
    surveyId: {
      type: Number,
      required: true,
    },
    filteredType: {
      type: String,
      required: true,
    },
    filteredTypeId: {
      type: Number,
      required: true,
    },
  },

  data() {
    const icon = mdiAccountBoxMultiple
    return {
      icon,
    }
  },

  computed: {
    href() {
      return `/survey/personnel-scores?${this.filteredType}Id=${this.filteredTypeId}&surveyId=${this.surveyId}`
    },
  },
}
</script>
