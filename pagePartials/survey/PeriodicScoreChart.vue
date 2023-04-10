<script>
// eslint-disable-next-line import/named
import { Line } from 'vue-chartjs'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'PeriodicScoreChart',
  extends: Line,

  props: {
    surveyScoreData: {
      type: Object,
      required: true,
    },

    currentSurveyId: {
      type: Number,
      required: true,
    },

    filter: {
      type: String,
      default: null,
    },
  },

  computed: {
    parsedScoreData() {
      const colorCycle = ColorCycle()
      const data = []

      if (this.filter) {
        // score-summary
        for (const d of this.surveyScoreData.scores) {
          if (d.data.overall.data.length > 1) {
            data.push({
              data: d.data.overall.data,
              borderColor: `${colorCycle.next()}70`,
              radius: this.definePointRads(d.data.overall.data),
              pointStyle: this.definePointStyle(d.data.overall.data),
              pointHoverRadius: 10,
              fill: false,
              label:
                this.filter === 'all'
                  ? d.data.overall.label
                  : d[this.filter.toLowerCase()].name,
            })
          }
        }
      } else {
        const overallColor = `${colorCycle.next()}70`
        // my-score
        data.push({
          data: this.surveyScoreData.overall.data,
          borderColor: overallColor,
          borderWidth: 7,
          hoverBorderWidth: 7,
          pointBackgroundColor: overallColor,
          pointStyle: this.definePointStyle(this.surveyScoreData.overall.data),
          radius: this.definePointRads(this.surveyScoreData.overall.data),
          pointHoverRadius: 10,
          fill: false,
          label: this.surveyScoreData.overall.label,
        })

        for (const category of this.surveyScoreData.categories) {
          const color = colorCycle.next()
          data.push({
            data: category.data,
            fill: false,
            radius: this.definePointRads(this.surveyScoreData.overall.data),
            pointBackgroundColor: `${color}70`,
            pointStyle: this.definePointStyle(
              this.surveyScoreData.overall.data
            ),
            pointHoverRadius: 10,
            borderColor: `${color}70`,
            label: category.label,
          })
        }
      }

      return data
    },
  },

  mounted() {
    const datasets = this.parsedScoreData

    this.renderChart(
      // Data
      {
        datasets,
      },

      // Options
      {
        maintainAspectRatio: false,
        spanGaps: true,
        layout: {
          padding: 10,
        },

        scales: {
          xAxes: [
            {
              type: 'time',
              time: {
                displayFormats: {
                  month: 'YYYY年M月',
                },
                unit: 'month',
              },
              gridLines: { zeroLineColor: 'rgba(0, 0, 0, 0.1)' },
            },
          ],
          yAxes: [
            {
              ticks: {
                beginAtZero: true,
                max: 100,
              },
            },
          ],
        },
        elements: {
          line: {
            tension: 0,
          },
        },
      }
    )
  },

  methods: {
    definePointRads(data) {
      const arrayOfRads = data.map(x => {
        return x.surveyId === this.currentSurveyId ? 10 : 3
      })
      return arrayOfRads
    },

    definePointStyle(data) {
      const arrayOfStyles = data.map(x => {
        return x.surveyId === this.currentSurveyId ? 'rectRot' : 'circle'
      })
      return arrayOfStyles
    },
  },
}
</script>
