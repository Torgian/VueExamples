<template lang="pug">
v-card.tycard.text-center
  h2 アンケートが終わりました。
    br
    | ありがとうございました。
  v-btn.mt-4(
    v-if="!isPublic"
    @click="redirectToFrontPage"
    color="primary"
    depressed
  ) Ok
  individual-score(
    v-if="scores"
    :surveyScoreData="scores"
    :tableHeaders="headers"
  )
</template>

<script>
import IndividualScore from '~/components/Elements/IndividualScore.vue'
import { mainTableHeaders } from '~/utilities/surveys'

export default {
  name: 'ThankYou',

  components: {
    IndividualScore,
  },

  props: {
    isPublic: {
      type: Boolean,
      required: true,
    },

    scores: {
      type: Object,
      required: true,
    },
  },

  computed: {
    headers() {
      const mainHeaders = mainTableHeaders(this.scores)
      return mainHeaders
    },
  },

  methods: {
    redirectToFrontPage() {
      this.$router.push('/survey')
    },
  },
}
</script>
<style lang="stylus" scoped>
.tycard
  width: 100%
  min-height: 300px
  h2
    padding-top: 10%
    padding-left: 20px
    padding-right: 20px
</style>
