<template>
  <data-view :title="title" :title-id="titleId" :date="date">
    <template v-slot:description>
      <slot name="description" />
    </template>
    <ul
      :class="$style.GraphLegend"
      :style="{ display: canvas ? 'block' : 'none' }"
    >
      <li
        v-for="(dataLabel, i) in dataLabels"
        :key="i"
        @click="onClickLegend(i)"
      >
        <button>
          <div
            v-if="i >= 2"
            :style="{
              backgroundColor: '#CC7004',
              borderColor: '#CC7004'
            }"
          />
          <div
            v-else
            :style="{
              backgroundColor: colors[i].fillColor,
              borderColor: colors[i].strokeColor
            }"
          />
          <span
            :style="{
              textDecoration: displayLegends[i] ? 'none' : 'line-through'
            }"
            >{{ dataLabel }}</span
          >
        </button>
      </li>
    </ul>
    <h4 :id="`${titleId}-graph`" class="visually-hidden">
      {{ $t(`{title}のグラフ`, { title }) }}
    </h4>
    <div :ref="'EveChart'" class="LegendStickyChart">
      <div class="scrollable" :style="{ display: canvas ? 'block' : 'none' }">
        <div :style="{ width: `${chartWidth}px` }">
          <bar
            :ref="'barChart'"
            :chart-id="chartId"
            :chart-data="displayData"
            :options="displayOption"
            :plugins="scrollPlugin"
            :display-legends="displayLegends"
            :height="240"
            :width="chartWidth"
          />
        </div>
      </div>
      <div>
        <bar
          class="sticky-legend"
          :style="{ display: canvas ? 'block' : 'none' }"
          :chart-id="`${chartId}-header-right`"
          :chart-data="displayDataHeader"
          :options="displayOptionHeader"
          :plugins="yAxesBgRightPlugin"
          :display-legends="displayLegends"
          :height="240"
          :width="width"
        />
      </div>
    </div>
    <template v-slot:additionalDescription>
      <slot name="additionalDescription" />
    </template>
    <template v-slot:dataTable>
      <v-data-table
        :headers="tableHeaders"
        :items="tableData"
        :items-per-page="-1"
        :hide-default-footer="true"
        :height="240"
        :fixed-header="true"
        :disable-sort="true"
        :mobile-breakpoint="0"
        class="cardTable"
        item-key="name"
      >
        <template v-slot:body="{ items }">
          <tbody>
            <tr v-for="item in items" :key="item.text">
              <th scope="row">{{ item.text }}</th>
              <td class="text-end">{{ item['0'] }}</td>
              <td class="text-end">{{ item['1'] }}</td>
              <td class="text-end">{{ item['2'] }}</td>
              <td class="text-end">{{ item['3'] }}</td>
            </tr>
          </tbody>
        </template>
      </v-data-table>
    </template>
    <template v-slot:infoPanel>
      <data-view-basic-info-panel
        :l-text="displayInfo.lText"
        :s-text="displayInfo.sText"
        :unit="displayInfo.unit"
      />
    </template>
  </data-view>
</template>

<script lang="ts">
import Vue from 'vue'
import { ThisTypedComponentOptionsWithRecordProps } from 'vue/types/options'
import { TranslateResult } from 'vue-i18n'
import { Chart } from 'chart.js'
import dayjs from 'dayjs'
import DataView from '@/components/DataView.vue'
import DataSelector from '@/components/DataSelector.vue'
import DataViewBasicInfoPanel from '@/components/DataViewBasicInfoPanel.vue'
import {
  DisplayData,
  yAxesBgPlugin,
  yAxesBgRightPlugin,
  scrollPlugin
} from '@/plugins/vue-chart'
import { getGraphSeriesStyle, SurfaceStyle } from '@/utils/colors'

interface HTMLElementEvent<T extends HTMLElement> extends MouseEvent {
  currentTarget: T
}
type Data = {
  canvas: boolean
  displayLegends: boolean[]
  colors: SurfaceStyle[]
  chartWidth: number | null
  width: number
}
type Methods = {
  pickLastNumber: (chartDataArray: number[][]) => number[]
  pickLastSecondNumber: (chartDataArray: number[][]) => number[]
  onClickLegend: (i: number) => void
  formatDayBeforeRatio: (dayBeforeRatio: number) => string
}

type Computed = {
  displayTransitionRatio: string
  displayInfo: {
    lText: string
    sText: string
    unit: string
  }
  displayData: DisplayData
  displayOption: Chart.ChartOptions
  displayDataHeader: DisplayData
  displayOptionHeader: Chart.ChartOptions
  scaledTicksYAxisMax: number
  scaledTicksYAxisMaxRight: number
  tableHeaders: {
    text: TranslateResult
    value: string
  }[]
  tableData: {
    [key: number]: number
  }[]
}

type Props = {
  title: string
  titleId: string
  chartId: string
  chartData: number[][]
  date: string
  labels: string[]
  dataLabels: string[] | TranslateResult[]
  tableLabels: string[] | TranslateResult[]
  unit: string
  scrollPlugin: Chart.PluginServiceRegistrationOptions[]
  yAxesBgPlugin: Chart.PluginServiceRegistrationOptions[]
  yAxesBgRightPlugin: Chart.PluginServiceRegistrationOptions[]
}

const options: ThisTypedComponentOptionsWithRecordProps<
  Vue,
  Data,
  Methods,
  Computed,
  Props
> = {
  created() {
    this.canvas = process.browser
  },
  components: { DataView, DataSelector, DataViewBasicInfoPanel },
  props: {
    title: {
      type: String,
      default: ''
    },
    titleId: {
      type: String,
      required: false,
      default: ''
    },
    chartId: {
      type: String,
      default: 'PositiveRateMixedChart'
    },
    chartData: {
      type: Array,
      required: false,
      default: () => []
    },
    date: {
      type: String,
      required: true,
      default: ''
    },
    labels: {
      type: Array,
      default: () => []
    },
    dataLabels: {
      type: Array,
      default: () => []
    },
    tableLabels: {
      type: Array,
      default: () => []
    },
    unit: {
      type: String,
      default: ''
    },
    scrollPlugin: {
      type: Array,
      default: () => scrollPlugin
    },
    yAxesBgPlugin: {
      type: Array,
      default: () => yAxesBgPlugin
    },
    yAxesBgRightPlugin: {
      type: Array,
      default: () => yAxesBgRightPlugin
    }
  },
  data: () => ({
    displayLegends: [true, true, true],
    colors: getGraphSeriesStyle(2),
    chartWidth: null,
    canvas: true,
    width: 300
  }),
  computed: {
    displayTransitionRatio() {
      const lastDay = this.pickLastNumber(this.chartData)[2]
      const lastDayBefore = this.pickLastSecondNumber(this.chartData)[2]
      return this.formatDayBeforeRatio(lastDay - lastDayBefore)
    },
    displayInfo() {
      return {
        lText: this.pickLastNumber(this.chartData)[2].toLocaleString(),
        sText: `${this.$t('{date}の数値', {
          date: dayjs(this.labels[this.labels.length - 1]).format('M/D')
        })}（${this.$t('前日比')}: ${this.displayTransitionRatio} ${
          this.unit
        }）`,
        unit: this.unit
      }
    },
    displayData() {
      const graphSeries = getGraphSeriesStyle(2)
      return {
        labels: this.labels,
        datasets: [
          {
            type: 'bar',
            yAxisID: 'y-axis-1',
            label: this.dataLabels[0],
            data: this.chartData[0],
            backgroundColor: graphSeries[0].fillColor,
            borderColor: graphSeries[0].strokeColor,
            borderWidth: 1,
            order: 1
          },
          {
            type: 'bar',
            yAxisID: 'y-axis-1',
            label: this.dataLabels[1],
            data: this.chartData[1],
            backgroundColor: graphSeries[1].fillColor,
            borderColor: graphSeries[1].strokeColor,
            borderWidth: 1,
            order: 2
          },
          {
            type: 'line',
            yAxisID: 'y-axis-2',
            label: this.dataLabels[2],
            data: this.chartData[2],
            pointBackgroundColor: 'rgba(0,0,0,0)',
            pointBorderColor: 'rgba(0,0,0,0)',
            borderColor: '#CC7004',
            borderWidth: 3,
            fill: false,
            order: 0,
            lineTension: 0
          }
        ]
      }
    },
    tableHeaders() {
      return [
        { text: this.$t('日付'), value: 'text' },
        ...(this.tableLabels as string[]).map((text, i) => {
          return { text, value: String(i), align: 'end' }
        })
      ]
    },
    tableData() {
      return this.labels
        .map((label, i) => {
          return Object.assign(
            { text: dayjs(label).format('M/D') },
            ...this.tableHeaders.map((_, j) => {
              if (j <= 1) {
                const data = this.chartData[j]
                return {
                  [j]: data[i].toLocaleString()
                }
              }
              if (j === 2) {
                const positive = this.chartData[0]
                const negative = this.chartData[1]
                return {
                  [j]: (positive[i] + negative[i]).toLocaleString()
                }
              }
              if (j === 3) {
                const data = this.chartData[2]
                return {
                  [j]: data[i].toLocaleString()
                }
              }
            })
          )
        })
        .sort((a, b) => dayjs(a.text).unix() - dayjs(b.text).unix())
        .reverse()
    },
    displayOption() {
      const unit = this.unit
      const data = this.chartData
      const options: Chart.ChartOptions = {
        tooltips: {
          displayColors: false,
          callbacks: {
            label: tooltipItem => {
              const cases = data[tooltipItem.datasetIndex!][
                tooltipItem.index!
              ].toLocaleString()
              let label = `${
                this.dataLabels[tooltipItem.datasetIndex!]
              } : ${cases} ${this.$t('人')}`
              if (this.dataLabels[tooltipItem.datasetIndex!] === '陽性率') {
                label = `${
                  this.dataLabels[tooltipItem.datasetIndex!]
                } : ${cases} ${unit}`
              }
              return label
            },
            title(tooltipItem, data) {
              const date = dayjs(
                data.labels![tooltipItem[0].index!].toString()
              ).format('M/D')
              return String(date)
            }
          }
        },
        responsive: false,
        maintainAspectRatio: false,
        legend: {
          display: false
        },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: '#808080',
                maxRotation: 0,
                callback: (label: string) => {
                  return dayjs(label).format('D')
                }
              }
              // #2384: If you set "type" to "time", make sure that the bars at both ends are not hidden.
              // #2384: typeをtimeに設定する時はグラフの両端が見切れないか確認してください
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: true,
                drawBorder: false,
                tickMarkLength: 10
              },
              ticks: {
                fontSize: 11,
                fontColor: '#808080',
                padding: 3,
                fontStyle: 'bold'
              },
              type: 'time',
              time: {
                unit: 'month',
                displayFormats: {
                  month: 'MMM'
                }
              }
            }
          ],
          yAxes: [
            {
              id: 'y-axis-1',
              position: 'left',
              stacked: true,
              gridLines: {
                display: true,
                drawOnChartArea: true,
                color: '#E5E5E5' // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax
              }
            },
            {
              id: 'y-axis-2',
              position: 'right',
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5' // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMaxRight,
                callback(value) {
                  return value + '%'
                }
              }
            }
          ]
        }
      }
      if (this.$route.query.ogp === 'true') {
        Object.assign(options, { animation: { duration: 0 } })
      }
      return options
    },
    displayDataHeader() {
      let n = 0
      let max = 0
      for (const i in this.displayData.datasets[0].data) {
        const current =
          this.displayData.datasets[0].data[i] +
          this.displayData.datasets[1].data[i]
        if (current > max) {
          max = current
          n = Number(i)
        }
      }
      const test = ['2020/1/1']
      this.labels.map((label, _) => {
        test.push(label)
      })
      return {
        labels: test,
        datasets: [
          {
            data: [this.displayData.datasets[0].data[n]],
            backgroundColor: 'transparent',
            yAxisID: 'y-axis-1',
            borderWidth: 0
          },
          {
            data: [this.displayData.datasets[1].data[n]],
            backgroundColor: 'transparent',
            yAxisID: 'y-axis-1',
            borderWidth: 0
          },
          {
            data: [this.displayData.datasets[2].data[n]],
            backgroundColor: 'transparent',
            yAxisID: 'y-axis-2',
            borderWidth: 0
          }
        ]
      }
    },
    displayOptionHeader() {
      const options: Chart.ChartOptions = {
        responsive: false,
        maintainAspectRatio: false,
        legend: {
          display: false
        },
        tooltips: { enabled: false },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: 'transparent',
                maxRotation: 0,
                minRotation: 0,
                callback: (label: string) => {
                  return dayjs(label).format('D')
                }
              }
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: false, // true -> false
                drawBorder: false,
                tickMarkLength: 10
              },
              ticks: {
                fontSize: 11,
                fontColor: 'transparent', // #808080
                padding: 13, // 3 + 10(tickMarkLength)
                fontStyle: 'bold'
              },
              type: 'time',
              time: {
                unit: 'month'
              }
            }
          ],
          yAxes: [
            {
              id: 'y-axis-1',
              type: 'linear',
              position: 'left',
              stacked: true,
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5' // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax
              }
            },
            {
              id: 'y-axis-2',
              type: 'linear',
              position: 'right',
              stacked: true,
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5' // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMaxRight,
                callback(value) {
                  return value + '%'
                }
              }
            }
          ]
        },
        animation: { duration: 0 }
      }
      return options
    },
    scaledTicksYAxisMax() {
      let max = 0
      for (const i in this.chartData[0]) {
        max = Math.max(max, this.chartData[0][i] + this.chartData[1][i])
      }
      return max
    },
    scaledTicksYAxisMaxRight() {
      let max = 0
      for (const i in this.chartData[2]) {
        max = Math.max(max, this.chartData[2][i])
      }
      return max
    }
  },
  methods: {
    onClickLegend(i) {
      this.displayLegends[i] = !this.displayLegends[i]
      this.displayLegends = this.displayLegends.slice()
    },
    pickLastNumber(chartDataArray: number[][]) {
      return chartDataArray.map(array => {
        return array[array.length - 1]
      })
    },
    pickLastSecondNumber(chartDataArray: number[][]) {
      return chartDataArray.map(array => {
        return array[array.length - 2]
      })
    },
    formatDayBeforeRatio(dayBeforeRatio: number): string {
      const dayBeforeRatioLocaleString = dayBeforeRatio.toLocaleString()
      switch (Math.sign(dayBeforeRatio)) {
        case 1:
          return `+${dayBeforeRatioLocaleString}`
        case -1:
          return `${dayBeforeRatioLocaleString}`
        default:
          return `${dayBeforeRatioLocaleString}`
      }
    }
  },
  mounted() {
    if (this.$el) {
      this.chartWidth =
        ((this.$el!.clientWidth - 22 * 2 - 38) / 60) * this.labels.length + 38
      this.chartWidth = Math.max(
        this.$el!.clientWidth - 22 * 2,
        this.chartWidth
      )
    }
    const barChart = this.$refs.barChart as Vue
    const barElement = barChart.$el
    const canvas = barElement.querySelector('canvas')
    const labelledbyId = `${this.titleId}-graph`

    if (canvas) {
      canvas.setAttribute('role', 'img')
      canvas.setAttribute('aria-labelledby', labelledbyId)
    }
    const chartelem = this.$refs.EveChart as Element
    const { width } = getComputedStyle(chartelem)
    this.width = Number(width.replace('px', '')) + 0.5
  }
}

export default Vue.extend(options)
</script>

<style module lang="scss">
.Graph {
  &Legend {
    text-align: center;
    list-style: none;
    padding: 0 !important;
    li {
      display: inline-block;
      margin: 0 3px;
      div {
        height: 12px;
        margin: 2px 4px;
        width: 40px;
        display: inline-block;
        vertical-align: middle;
        border-width: 1px;
        border-style: solid;
      }
      button {
        color: $gray-3;
        @include font-size(12);
      }
    }
  }
}
</style>
