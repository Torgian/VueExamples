<template lang="pug">
.participants
  template(v-if="participants")
    .title 回答していないユーザー {{ participants.counts.total }}名

    v-tabs(centered dark color="primary" height="32" v-model="currentTab")
      v-tab(href="#list")
        v-icon(small) {{ listIcon }}
        span List
      v-tab(href="#chart")
        v-icon(small) {{ chartBarIcon }}
        span Chart

    v-tabs-items(v-model="currentTab")
      v-tab-item(value="list"): list(:value="participants")

      v-tab-item(value="chart"): chart(:value="participants")

  .spin(v-else): .inner
    spinner(
      :animation-duration="1500"
      :size="64"
      :color="$store.state.view.spin.color"
    )
    .message 読込中...
</template>

<script>
import { mdiViewListOutline, mdiChartBar } from '@mdi/js'
import 'epic-spinners/dist/lib/epic-spinners.min.css'
import { IntersectingCirclesSpinner as Spinner } from 'epic-spinners/dist/lib/epic-spinners.min.js'

import List from '~/pagePartials/profile/shared/List.vue'
import Chart from '~/pagePartials/profile/shared/Chart.vue'

export default {
  name: 'Participants',
  components: { List, Chart, Spinner },

  props: {
    surveyId: {
      type: Number,
      required: true,
    },
  },

  data() {
    return {
      participants: null,
      currentTab: 'list',
    }
  },

  computed: {
    listIcon() {
      return mdiViewListOutline
    },

    chartBarIcon() {
      return mdiChartBar
    },
  },

  created() {
    this.$axios
      .$get(`/surveys/${this.surveyId}/users?submitted=0`)
      .then(res => {
        this.participants = res
      })
  },
}
</script>

<style lang="stylus" scoped>
.participants
  max-height: 400px
  overflow: auto
  .title
    font-size .8rem

  :deep(.v-tabs-bar),
  .v-tabs-items
    background transparent !important
    padding-top 5px

.spin
  display flex
  justify-content center
  align-items center
</style>
