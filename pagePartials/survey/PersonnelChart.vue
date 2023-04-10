<script>
// eslint-disable-next-line import/named
import { Scatter } from 'vue-chartjs'

import { ColorCycle } from '~/utilities/viz'

export default {
  name: 'SurveyPersonnelChart',
  extends: Scatter,

  props: {
    surveyScoreData: {
      type: Array,
      required: true,
    },
  },

  data() {
    const datasets = []
    const tooltipEl = null
    return {
      datasets,
      tooltipEl,
    }
  },

  computed: {
    options() {
      const self = this
      return {
        responsive: true,
        maintainAspectRatio: false,
        layout: {
          padding: {
            right: 10,
          },
        },
        legend: {
          display: false,
        },
        scales: {
          yAxes: [
            {
              position: 'right',
              id: 'y-axis-1',
              scaleLabel: {
                display: true,
                labelString: 'エンゲージメント',
              },
              gridLines: {
                zeroLineWidth: 2,
                zeroLineColor: 'black',
              },
              ticks: {
                display: false,
                beginAtZero: true,
                suggestedMin: -50,
                suggestedMax: 50,
              },
            },
          ],
          xAxes: [
            {
              id: 'x-axis-1',
              position: 'top',
              scaleLabel: {
                display: true,
                labelString: 'モチベーション',
              },
              gridLines: {
                zeroLineWidth: 2,
                zeroLineColor: 'black',
              },
              ticks: {
                beginAtZero: true,
                display: false,
                suggestedMin: -50,
                suggestedMax: 50,
              },
            },
          ],
        },
        tooltips: {
          enabled: false,

          /**
           * Implement custom tooltip
           * Based on API explanations and examples
           * @link https://www.chartjs.org/docs/latest/configuration/tooltip.html#external-custom-tooltips
           */
          custom(tooltipModel) {
            const tooltipEl = self.tooltipEl
            const colorCycle = ColorCycle()

            // Hide if no tooltip
            if (tooltipModel.opacity === 0) {
              tooltipEl.style.opacity = 0
              return
            }

            let innerHTML = '<div class="personnel-chart-tooltip">'
            const dataPoints = tooltipModel.dataPoints
            for (const dataPoint of dataPoints) {
              const originalData = self.datasets[dataPoint.datasetIndex]
              innerHTML += `<div class="chart-tooltip-content"><p>${originalData.label}</p>`
              if (originalData.avatar) {
                innerHTML += `<img width="50" height="50" src="${originalData.avatar}">`
              }
              innerHTML += '</div>'
              innerHTML += '<div class="chart-tooltip-scores">'
              if (originalData.grade) {
                innerHTML += `<p>${originalData.grade}</p>`
              }
              if (originalData.perCategory) {
                colorCycle.reset()
                for (const cat of originalData.perCategory) {
                  innerHTML += `<span class="mark-border" style="border-color: ${colorCycle.next()}">${
                    cat.mark
                  }</span>`
                }
              }
              innerHTML += '</div>'
            }
            innerHTML += '</div>'

            tooltipEl.innerHTML = innerHTML

            // `this` will be the overall tooltip
            const position = this._chart.canvas.getBoundingClientRect()

            // Display, position, and set styles for font
            tooltipEl.style.opacity = 1
            tooltipEl.style.left =
              position.left + window.pageXOffset + tooltipModel.caretX + 'px'
            tooltipEl.style.top =
              position.top + window.pageYOffset + tooltipModel.caretY + 'px'
            tooltipEl.style.fontFamily = tooltipModel._bodyFontFamily
            tooltipEl.style.fontSize = tooltipModel.bodyFontSize + 'px'
            tooltipEl.style.fontStyle = tooltipModel._bodyFontStyle
            tooltipEl.style.padding =
              tooltipModel.yPadding + 'px ' + tooltipModel.xPadding + 'px'
          },
        },
      }
    },
  },

  mounted() {
    const dataSet = new Map()
    const colorCycle = ColorCycle()

    for (const item of this.surveyScoreData) {
      const xValue =
        item.score.all.perCategory[0].mark + item.score.all.perCategory[1].mark
      const yValue =
        item.score.all.perCategory[2].mark + item.score.all.perCategory[3].mark
      const coordinates = {
        // Linear transformation to center the data
        x: xValue / 2 - 50,
        y: yValue / 2 - 50,
      }

      if (!dataSet.has(item.answererName)) {
        dataSet.set(item.answererName, {
          points: [],
          avatar: this.userAvatar(item.user),
          grade: item.score.all.overall.grade,
          perCategory: item.score.all.perCategory,
        })
      }
      dataSet.get(item.answererName).points.push(coordinates)
    }

    for (const data of dataSet) {
      const color = colorCycle.next()
      this.datasets.push({
        label: data[0],
        xAxisID: 'x-axis-1',
        yAxisID: 'y-axis-1',
        backgroundColor: `${color}40`, // Add alpha channel
        borderColor: color,
        borderWidth: 1,
        data: data[1].points,
        maxBarThickness: 1,

        // Append avatar data for use in tooltip
        avatar: data[1].avatar,
        grade: data[1].grade,
        perCategory: data[1].perCategory,
      })
    }

    this.renderChart(
      // Data
      { datasets: this.datasets },
      // Options
      this.options
    )

    // Initialize tooltip
    const tooltipEl = document.createElement('div')
    // Hide tooltip on initialize
    tooltipEl.style.position = 'absolute'
    tooltipEl.style.opacity = 0

    tooltipEl.style.pointerEvents = 'none'
    tooltipEl.style.borderColor = 'black'
    tooltipEl.style.borderStyle = 'solid'
    tooltipEl.style.borderWidth = '1px'
    tooltipEl.style.borderRadius = '10px'
    tooltipEl.style.backgroundColor = 'white'
    tooltipEl.style.zIndex = 1001
    document.body.insertAdjacentElement('beforeend', tooltipEl)

    this.tooltipEl = tooltipEl
  },

  beforeDestroy() {
    // Clean up tooltip
    document.body.removeChild(this.tooltipEl)
    this.tooltipEl = null
  },
}
</script>

<style lang="stylus">
.personnel-chart-tooltip
  display: flex
  max-width: 100px
  flex-wrap: wrap
  justify-content: space-around

  .chart-tooltip-content
    width: 100%
    text-align center

    img
      max-width: 50px

  .chart-tooltip-scores
    text-align center

    .mark-border
      border 2px solid
      padding 3px
</style>
