<template>
  <div v-if="show" class="home">
    <Chart v-model="mock" :size="{w: 1400, h: 800}"></Chart>
  </div>
</template>

<script>
import 'echarts'
import Chart from 'echarts-middleware'
const elasticsearch = require('elasticsearch')
export default {
  name: 'home',
  data () {
    return {
      show: false,
      mock: {
        "xAxis": {
          "type": "category", 
          "boundaryGap": false, 
          "data": [
            'sd'
          ]
        }, 
        "yAxis": {
          "type": "value", 
          "boundaryGap": [
            0, 
            "100%"
          ]
        }, 
        "series": [
          {
            "name": "模拟数据", 
            "type": "line", 
            "smooth": true, 
            "symbol": "none", 
            "sampling": "average", 
            "itemStyle": {
              "normal": {
                "color": "rgb(255, 70, 131)"
              }
            }, 
            "data": [
              {"name": 'sdsd', "value": 12},
              {"name": 'df', "value": 112}
            ]
          }
        ]
      }
    }
  },
  components: {
    Chart
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
        // match all query, on all indices and types
        {},
        { 
          size: 100,
          aggs: {
            "data": {
              "date_histogram": {
                field: "@timestamp",
                interval: "12h",
                time_zone: "Asia/Shanghai",
                min_doc_count: 1
              }
            }
          },
          query: {
            bool: {
              must: [
                {
                  range: {
                    "@timestamp":{
                      gte: 1525795200000,
                      lte: 1530374399999,
                      format:"epoch_millis"
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
  }
}
</script>
