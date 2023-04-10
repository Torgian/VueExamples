<template lang="pug">
v-row
  v-col.d-flex(cols="12")
    v-select(
      v-model="currentlyDisplayedSurvey"
      :items="sortedSurveys"
      filled
      item-value="id"
      @change="changeSelectedSurvey(currentlyDisplayedSurvey)"
      label="アンケート"
      return-object
      dense
    )
      template(v-slot:selection="data") {{ displaySurveyInfo(data.item) }}
      template(v-slot:item="data") {{ displaySurveyInfo(data.item) }}
</template>
<script>
import sortBy from 'lodash/sortBy'

export default {
  name: 'SurveySelector',

  props: {
    surveys: {
      type: Array,
      required: true,
    },
  },

  computed: {
    currentlyDisplayedSurvey: {
      get() {
        return this.$store.getters['survey/currentlySelectedSurvey']
      },
      set(value) {
        return this.$store.commit('survey/setSelectedSurvey', value)
      },
    },

    sortedSurveys() {
      return sortBy(this.surveys, [
        // periodic survey last
        s => (s.periodicSurvey ? 1 : 0),
        s => (s.periodicSurvey ? s.periodicSurvey.number : -1),
        s => s.number,
      ])
    },
  },

  created() {
    if (!this.currentlyDisplayedSurvey) {
      // Initialize selected survey if it is not set
      this.$store.commit('survey/setSelectedSurvey', this.surveys[0])
    }
  },

  methods: {
    changeSelectedSurvey(survey) {
      this.$emit('update-score', survey)
    },

    displaySurveyInfo(surveyData) {
      const data = surveyData.survey ? surveyData.survey : surveyData
      const periodicOrRegularTag = data.periodicSurvey
        ? `定期 No.${data.periodicSurvey.number} - `
        : '単回 No.'

      return `${periodicOrRegularTag} ${data.number} (配布日): ${data.distributeAt}`
    },
  },
}
</script>
