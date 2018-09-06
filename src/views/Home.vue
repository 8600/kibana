<template lang="pug">
  .home(v-if="show")
    .top-bar
      .search-bar
        Search(@input="searchText")
      Datepicker.data-check(:date="startTime" @change="getSearchData()")
      span.connector -
      Datepicker.data-check(:date="endTime" @change="getSearchData()")
    .chart-box
      Chart(v-model="mock")
    .data-select
      .left-bar
        select(v-model="searchList[0]", @change="getSearchData()")
          option(value="") *
          option(v-for="item in indices", :key="item.uuid", :value="item.index") {{item.index}}
        //- 选择左侧栏
        .left-bar-item-box
          .left-bar-label 未选择
      .right-bar
        .empty(v-if="hits.length === 0") 当前筛选条件下没有数据
        .table-box(v-else)
          .table-title-bar
            .time 时间
            .log 日志
          .table-body
            .table-body-bar(v-for="item in hits")
              .time {{item._source['@timestamp']}}
              .log(v-html="item._source.message")
</template>

<script>
import 'echarts'
import Search from './Search'
import Chart from 'echarts-middleware'
import Datepicker from 'date-picker-owo'
const elasticsearch = require('elasticsearch')

export default {
  name: 'home',
  data () {
    return {
      show: false,
      hits: null,
      client: null,
      indices: null,
      activeIndex: null,
      searchIndices: '',
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
        "series": []
      },
      sourceList: [],
      searchList: [""]
    }
  },
  components: {
    Chart,
    Search,
    Datepicker
  },
  computed: {
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
      let searchData = {
        body: []
      }
      // 根据搜索列表生成搜索条件
      if (this.searchList.length > 0) {
        // 如果选择的个数大于零则按照key搜索
        for (let i = 0; i < this.searchList.length; i++) {
          // 空搜索
          searchData.body.push({ index: this.searchList[i] })
          searchData.body.push({
            size: 100,
            aggs: {
              requestCount: {
                date_histogram: {
                  field: "@timestamp",
                  interval: "1d",
                  time_zone: "+08:00",
                  min_doc_count: 0, // 强制返回空 buckets
                  format: "yyyy-MM-dd",
                  // 强制返回整年
                  extended_bounds: {
                    "min" : this.startTime.time,
                    "max" : this.endTime.time
                  }
                }
              }
            },
            query: {
              bool: {
                must: {
                  range: {
                    "@timestamp":{
                      gte: this.startTime.time,
                      lte: this.endTime.time,
                      format: "yyyy-MM-dd"
                    }
                  }
                }
              }
            }
          })
        }
      } else {
        // 空搜索
        searchData.body.push({ index: '' })
        searchData.body.push({
          size: 100,
          aggs: {
            requestCount: {
              date_histogram: {
                field: "@timestamp",
                interval: "1d",
                time_zone: "+08:00",
                min_doc_count: 0, // 强制返回空 buckets
                format: "yyyy-MM-dd",
                // 强制返回整年
                extended_bounds: {
                  "min" : this.startTime.time,
                  "max" : this.endTime.time
                }
              }
            }
          },
          query: {
            bool: {
              must: {
                range: {
                  "@timestamp":{
                    gte: this.startTime.time,
                    lte: this.endTime.time,
                    format: "yyyy-MM-dd"
                  }
                }
              }
            }
          }
        })
      }
      this.client.msearch(searchData).then(res => {
        let xAxis = []
        let data = []
        let chartData = JSON.parse(JSON.stringify(this.mock))
        console.log('获取到数据:', res.responses)
        res.responses.forEach((element, ind) => {
          // 图表数据
          const buckets = element.aggregations.requestCount.buckets
          this.hits = element.hits.hits
          if (buckets.length > 0) {
            // 初始化数据列表
            data[ind] = []
            buckets.forEach(bucketsElement => {
              // 坐标轴只用添加一次
              if (ind === 0) xAxis.push(bucketsElement.key_as_string)
              data[ind].push({value: bucketsElement.doc_count})
            })
            chartData.xAxis.data = xAxis
            // 清除图表中已有数据
            chartData.series = []
            // 填充图表数据
            data.forEach((dataElement, ind) => {
              chartData.series.push({
                "name": this.searchList[ind], 
                "type": "line", 
                "smooth": true, 
                "symbol": "none", 
                "sampling": "average", 
                "itemStyle": {
                  "normal": {
                    "color": "rgb(255, 70, 131)"
                  }
                },
                "data": dataElement
              })
            })
            this.mock = chartData
            this.show = true
          }
        })
      })
    },
    changeDataIndex () {
      this.getSearchData()
    },
    beforeEnter (el) {
      el.style.opacity = 0
      el.style.height = 0
    },
    searchText (value) {
      console.log(value)
      if (value) {
        this.client.search({
          q: value,
          size: 100,
          index: this.searchList[0]
        }).then((res) => {
          // 高亮搜索内容
          let hits = res.hits.hits
          console.log(hits)
          for (let key in hits) {
            hits[key]._source.message = hits[key]._source.message.replace(new RegExp(value, 'gm'), `<highlight>${value}</highlight>`)
          }
          this.hits = hits
        })
      } else {
        alert('没有输入搜索条件!')
      }
    }
  }
}
</script>

<style lang="less" scoped>
  .home {
    height: 100%;
    padding: 0 10px;
    overflow: hidden;
    width: calc(100% - 20px);
    background-color: #eff0f4;
  }
  .left-bar {
    width: 400px;
    margin-bottom: 5px;
    background-color: white;
    line-height: 30px;
    overflow: hidden;
    border-radius: 2px;
    .left-bar-item-box {
      overflow: auto;
      height: calc(100% - 40px);
    }
    .left-bar-item {
      cursor: pointer;
      text-align: left;
      margin: 0 10px;
      padding: 0 5px;
    }
    .left-bar-item:hover {
      color: #333;
      background-color: beige;
    }
  }
  .right-bar {
    position: relative;
    background-color: white;
    margin-left: 5px;
    margin-bottom: 5px;
    width: calc(100% - 405px);
  }
  .top-bar {
    height: 49px;
    margin: 5px 0;
    border-radius: 2px;
    line-height: 49px;
    text-align: right;
    background-color: white;
  }
  .chart-box {
    height: 400px;
    margin: 5px 0;
    border-radius: 2px;
    background-color: white;
  }
  .empty {
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    margin: auto;
    width: 500px;
    height: 40px;
    font-size: 2rem;
    color: #ccc;
  }
  .data-select {
    display: flex;
    height: calc(100% - 464px);
  }
  .table-box {
    height: 100%;
    overflow: hidden;
    .table-title-bar {
      background-color: #eff0f4;
      margin: 10px;
      height: 50px;
      line-height: 50px;
      text-align: left;
      display: flex;
    }
    .table-body {
      height: calc(100% - 70px);
      overflow: auto;
    }
    .table-body-bar {
      display: flex;
      margin: 10px;
      overflow: hidden;
      border-bottom: 1px solid #ccc;
    }
    .time {
      text-align: left;
      width: 260px;
      padding: 0 10px;
    }
    .log {
      text-align: left;
      word-break: break-all;
      word-wrap: break-word;
      width: calc(100% - 280px);
    }
  }
</style>
