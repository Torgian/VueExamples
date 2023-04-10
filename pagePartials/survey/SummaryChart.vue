<script>
// eslint-disable-next-line import/named
import { Bar } from 'vue-chartjs'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'SurveySummaryChart',
  extends: Bar,

  props: {
    surveyScoreData: {
      type: Array,
      required: true,
    },
  },

  mounted() {
    const labels = this.surveyScoreData.map(item => item.name)

    const datasetLabels = new Map()
    for (const item of this.surveyScoreData) {
      for (const category of item.values.perCategory) {
        if (!datasetLabels.has(category.label)) {
          datasetLabels.set(category.label, [])
        }

        datasetLabels.get(category.label).push(category.mark)
      }
    }

    const colorCycle = ColorCycle()
    const datasets = []
    for (const [label, data] of datasetLabels) {
      const color = colorCycle.next()
      datasets.push({
        label,
        backgroundColor: `${color}40`, // Add alpha channel
        borderColor: color,
        borderWidth: 1,
        data,
        maxBarThickness: 50,
      })
    }

    this.renderChart(
      // Data
      {
        labels,
        datasets,
      },

      // Options
      {
        maintainAspectRatio: false,
        scales: {
          xAxes: [
            {
              stacked: true,
            },
          ],
          yAxes: [
            {
              stacked: true,
              scaleLabel: {
                display: true,
                labelString: '点数',
              },
            },
          ],
        },
      }
    )
  },
}
</script>
