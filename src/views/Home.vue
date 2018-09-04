<template lang="pug">
  .home(v-if="show")
    .left-bar
      .left-bar-item(v-for="item in indices", :key="item.uuid", @click="changeDataIndex(item)") {{item.index}}
    .right-bar
      .empty(v-if="isEmpty") 当前分类下没有数据
      template(v-else)
        .top-bar
          Datepicker.data-check(:date="startTime" @change="startTimeChange")
          span.connector -
          Datepicker.data-check(:date="endTime" @change="endTimeChange")
        .chart-box
          Chart(v-model="mock")
</template>

<script>
import 'echarts'
import moment from 'moment'
import Chart from 'echarts-middleware'
import Datepicker from 'date-picker-owo'
const elasticsearch = require('elasticsearch')

export default {
  name: 'home',
  data () {
    return {
      show: false,
      isEmpty: true,
      client: null,
      indices: null,
      endTime: {
        time: '2018-6-25'
      },
      startTime: {
        time: '2018-1-1'
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
      },
      searchData: {
        body: [
          { index: [] },
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
      }
    }
  },
  components: {
    Chart,
    Datepicker
  },
  created () {
    this.client = new elasticsearch.Client({
      host: 'http://192.168.1.190:9200',
      country: 'zh-cn',
      apiVersion: '6.2'
    })
    this.client.cat.indices({format: 'json'}).then(data => {
      console.log('获取到indices列表:', data)
      this.indices = data
    })
    this.getSearchData()
  },
  methods: {
    getSearchData () {
      this.client.msearch(this.searchData).then(res => {
        const buckets = res.responses[0].aggregations.data.buckets
        let xAxis = []
        let data = []
        console.log('获取到数据:', buckets)
        if (buckets.length > 0) {
          buckets.forEach(element => {
            xAxis.push(element.key_as_string)
            data.push({value: element.doc_count})
          })
          this.mock.xAxis.data = xAxis
          this.mock.series[0].data = data
          this.show = true
          this.isEmpty = false
        } else {
          this.isEmpty = true
        }
        // console.log(res.responses[0].aggregations.data.buckets)
      })
    },
    startTimeChange (data) {
      const time = moment(data, 'YYYY-MM-DD').valueOf()
      this.searchData.body[1].query.bool.must[0].range['@timestamp'].gte = time
      this.getSearchData()
      // console.log(time)
    },
    endTimeChange (data) {
      const time = moment(data, 'YYYY-MM-DD').valueOf()
      this.searchData.body[1].query.bool.must[0].range['@timestamp'].lte = time
      this.getSearchData()
    },
    changeDataIndex (item) {
      // console.log(item)
      this.searchData.body[0].index[0] = item.index
      this.getSearchData()
    }
  }
}
</script>

<style lang="less" scoped>
  .home {
    width: 100%;
    height: 100%;
    overflow: hidden;
    display: flex;
  }
  .left-bar {
    width: 400px;
    height: 100%;
    line-height: 30px;
    overflow-x: hidden;
    overflow-y: auto;
    .left-bar-item {
      cursor: pointer;
      text-align: left;
      border-bottom: 1px solid #ccc;
      margin: 0 10px;
    }
    .left-bar-item:hover {
      color: white;
      background-color: skyblue;
    }
  }
  .right-bar {
    position: relative;
    width: calc(100% - 400px);
  }
  .top-bar {
    height: 40px;
    text-align: right;
  }
  .chart-box {
    height: calc(100% - 40px);
  }
  .empty {
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    margin: auto;
    width: 300px;
    height: 40px;
    font-size: 2rem;
    color: #ccc;
  }
</style>
