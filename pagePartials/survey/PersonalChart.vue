<script>
// eslint-disable-next-line import/named
import { Radar } from 'vue-chartjs'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'SurveyPersonalChart',
  extends: Radar,

  props: {
    surveyScoreData: {
      type: Array,
      required: true,
    },

    targetUser: {
      type: Object,
      default: null,
    },
  },

  mounted() {
    const colorCycle = ColorCycle()
    // First color of the cycle is for 総合スコア,
    // as displayed in the PeriodicScoreChart
    // As 総合スコア is not shown on the PersonalChart, we skip it.
    colorCycle.next()

    const labels = []
    const data = []
    const colors = []
    for (const score of this.surveyScoreData) {
      labels.push(score.label)
      data.push(score.mark)
      colors.push(colorCycle.next())
    }

    const datasets = [
      {
        data,
        borderColor: '#1f77b460',
        label: this.targetUser
          ? `${this.targetUser.name}のスコア`
          : '私のスコア',
      },
    ]

    this.renderChart(
      // Data
      {
        labels,
        datasets,
      },

      // Options
      {
        maintainAspectRatio: false,
        scale: {
          ticks: {
            suggestedMin: 0,
            suggestedMax: 100,
          },
        },
        tooltips: {
          callbacks: {
            label(tooltip) {
              return labels[tooltip.index]
            },
          },
        },

        elements: {
          point: {
            backgroundColor(context) {
              return colors[context.dataIndex]
            },
          },
        },
      }
    )
  },
}
</script>
