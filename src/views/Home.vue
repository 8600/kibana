<template>
  <div v-if="show" class="home">
    <Datepicker :date="startTime" :option="startTimeOption" @change="startTimeChange" />
    <Chart v-model="mock" :size="{w: 1400, h: 800}"></Chart>
  </div>
</template>

<script>
import 'echarts'
import Chart from 'echarts-middleware'
import Datepicker from 'date-picker-owo'
const elasticsearch = require('elasticsearch')
export default {
  name: 'home',
  data () {
    return {
      show: false,
      startTime: {
        time: ''
      },
      startTimeOption: {
        type: 'day',
        placeholder: '起始时间',
        week: ['yi', 'Tu', 'We', 'Th', 'Fr', 'Sa', 'Su'],
        month: ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'],
        format: 'YYYY-MM-DD',
        inputStyle: {
          'display': 'inline-block',
          'padding': '6px',
          'line-height': '22px',
          'font-size': '16px',
          'border': '2px solid #fff',
          'box-shadow': '0 1px 3px 0 rgba(0, 0, 0, 0.2)',
          'border-radius': '2px',
          'color': '#5F5F5F'
        },
        color: {
          header: '#ccc',
          headerText: '#f00'
        }
      },
      mock: {
        "xAxis": {
          "type": "category", 
          "boundaryGap": false, 
          "data": []
        },
        "tooltip": {
          "trigger": 'axis'
        },
        "yAxis": {
          "name" : '访问量(次)',
          "type": "value", 
          "boundaryGap": [
            0, 
            "100%"
          ]
        },
        "dataZoom": [{
          "start": 0,
          "end": 100
        }],
        "series": [
          {
            "name": "攻击次数", 
            "type": "line", 
            "smooth": true, 
            "symbol": "none", 
            "sampling": "average", 
            "itemStyle": {
              "normal": {
                "color": "rgb(255, 70, 131)"
              }
            },
            "data": []
          }
        ]
      }
    }
  },
  components: {
    Chart,
    Datepicker
  },
  created () {
    const client = new elasticsearch.Client({
      host: 'http://192.168.1.190:9200',
      // log: 'trace',
      country: 'zh-cn',
      apiVersion: '6.2'
    })
    client.msearch({
      body: [
        {},
        { 
          size: 100,
          aggs: {
            data: {
              date_histogram: {
                field: "@timestamp",
                interval: "1d",
                time_zone: "+08:00",
                min_doc_count: 0, // 强制返回空 buckets
                format: "yyyy-MM-dd"
              }
            }
          },
          query: {
            bool: {
              must: [
                {
                  range: {
                    "@timestamp":{
                      gte: 1514736000000,
                      lte: 1530374399999,
                      format: "epoch_millis"
                    }
                  }
                }
              ]
            }
          }
        }
      ]
    }).then(res => {
      const buckets = res.responses[0].aggregations.data.buckets
      let xAxis = []
      let data = []
      buckets.forEach(element => {
        xAxis.push(element.key_as_string)
        data.push({value: element.doc_count})
      })
      this.mock.xAxis.data = xAxis
      this.mock.series[0].data = data
      this.show = true
      console.log(res.responses[0].aggregations.data.buckets)
    })
  },
  methods: {
    startTimeChange (data) {
      console.log(data)
    }
  }
}
</script>
