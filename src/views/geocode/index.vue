<template>
  <div class="geocode-container">
    <div class="geocode-text">从地名地址获取经纬度坐标工具，采用天地图API(仅支持广西)</div>
    <div class="geocode-data">
      <el-row :gutter="5">
        <el-col :span="10">
          <upload-excel-component :on-success="handleSuccess" :before-upload="beforeUpload" />
        </el-col>
        <el-col :span="12">
          <el-form ref="form" label-width="80px">
            <el-row :gutter="24">
              <el-col :span="12">
                <el-form-item label="首选地址">
                  <el-select v-model="address1" placeholder="请选择解析地址字段">
                    <template v-for="(add1,index) in tableHeader">
                      <el-option :key="index" :label="add1" :value="add1" />
                    </template>
                  </el-select>
                </el-form-item>
              </el-col>
              <el-col :span="12">
                <el-form-item label="备选地址">
                  <el-select v-model="address2" placeholder="请选择备选解析地址字段">
                    <template v-for="(add2,index) in tableHeader">
                      <el-option :key="index" :label="add2" :value="add2" />
                    </template>
                  </el-select>
                </el-form-item>
              </el-col>
            </el-row>
          </el-form>
          <el-form ref="form" label-width="80px">
            <el-row :gutter="24">
              <el-col :span="12">
                <el-form-item label="行政区划">
                  <el-select v-model="areacode" placeholder="请选择行政区划字段">
                    <template v-for="(ar,index) in tableHeader">
                      <el-option :key="index" :label="ar" :value="ar" />
                    </template>
                  </el-select>
                </el-form-item>
              </el-col>
              <el-col :span="6">
                <el-button type="primary" :disabled="exporting" :loading="encoding" @click="start">开始解析</el-button>
              </el-col>
              <el-col :span="6">
                <el-button type="success" :disabled="encoding" :loading="exporting" @click="exportData">导出结果</el-button>
              </el-col>
            </el-row>
          </el-form>
        </el-col>
        <el-col :span="2">
          <el-progress type="circle" :percentage="percentage" :color="percentage==100?'#13ce66':'#20a0ff'" />
        </el-col>
      </el-row>
      <el-table :key="tableKey" v-loading="encoding" :data="tableData" border highlight-current-row height="800" style="width: 100%;margin-top:0px;">
        <el-table-column v-for="item of tableHeader" :key="item" :prop="item" :label="item" />
      </el-table>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import UploadExcelComponent from '@/components/UploadExcel/index.vue'
import axios from 'axios'
import cityCodeJson from './citycode.json'
import sleep from 'sleep-promise'

export default {
  name: 'GeoCode',
  computed: {
    ...mapGetters([
      'name'
    ]),
    percentage: function() {
      return parseFloat((this.done / this.all * 100).toFixed(2))
    }
  },
  components: { UploadExcelComponent },
  data() {
    return {
      filename: '',
      address1: '',
      address2: '',
      areacode: '',
      tableData: [],
      tableHeader: [],
      searchApiUrl: 'http://api.tianditu.gov.cn/v2/search',
      hasAlternative: false,
      cityCode: cityCodeJson,
      encoding: false,
      exporting: false,
      all: 1,
      done: 0,
      tableKey: Math.random()
    }
  },
  methods: {
    exportData() {
      if (this.tableData.length > 0) {
        this.exporting = true
        import('@/vendor/Export2Excel').then(excel => {
          const tHeader = this.tableHeader
          const filterVal = this.tableHeader
          const list = this.tableData
          const data = this.formatJson(filterVal, list)
          const filename = this.filename.split('.')[0]
          excel.export_json_to_excel({
            header: tHeader,
            data,
            filename: '解析结果-' + filename,
            autoWidth: false,
            bookType: 'csv'
          })
          this.exporting = false
        })
      } else {
        this.$message.warning('暂无数据！')
      }
    },
    formatJson(filterVal, jsonData) {
      return jsonData.map(v => filterVal.map(j => {
        return v[j]
      }))
    },
    start() {
      if (this.address1 === '' || this.tableHeader.indexOf(this.address1) === -1) {
        this.$message.error('解析地址错误！')
        return
      }
      this.hasAlternative = this.address2 !== '' && this.tableHeader.indexOf(this.address2) > -1 && this.address1 !== this.address2
      if (this.areacode === '' || this.tableHeader.indexOf(this.areacode) === -1 || this.address1 === this.areacode) {
        this.$message.warning('未指定所属行政区划！')
        return
      }
      this.encoding = true
      this.all = this.tableData.length
      this.done = 0
      let index = 1
      for (let i = 0; i < this.tableData.length; i++) {
        const row = this.tableData[i]
        row.index = i
        this.geoSearch(row, false)
        index = index + 1
        if (index > 10) {
          (async() => {
            await sleep(3000)
            index = 1
          })()
        }
      }
    },
    updateTableData(index, data) {
      this.tableData[index]['wgs84_x'] = data.wgs84_x
      this.tableData[index]['wgs84_y'] = data.wgs84_y
      this.tableData[index]['an_address'] = data.an_address
      this.tableData[index]['an_name'] = data.an_name
      this.tableData[index]['httpcode'] = data.httpcode
      this.done = this.done + 1
      const _this = this
      if (this.done >= this.all) {
        setTimeout(function() {
          _this.encoding = false
          _this.tableKey = Math.random()
        }, 3000)
      }
    },
    geoSearch(row, alternative) {
      let keyWork = alternative ? row[this.address2] : row[this.address1]
      const arCodeList = this.cityCode.filter((item) => { return item.name === row[this.areacode] })
      let areacode = 0
      const result = {
        wgs84_x: 0,
        wgs84_y: 0,
        an_address: '',
        an_name: '',
        httpcode: '500'
      }
      if (keyWork.replaceAll(' ', '') === '') {
        this.updateTableData(row.index, result)
        return
      }
      if (arCodeList.length > 0) {
        areacode = '156' + arCodeList[0].code
      } else {
        this.updateTableData(row.index, result)
        return
      }
      keyWork = keyWork.replaceAll('\\t', '').replaceAll(' ', '')
      const params = {
        postStr: { keyWord: keyWork, queryType: 12, start: 0, count: 1, specify: areacode },
        type: 'query',
        tk: '85c9d12d5d691d168ba5cb6ecaa749eb'
      }
      axios.get(this.searchApiUrl, { timeout: 1500000, params }).then(res => {
        if (res.status === 200) {
          const data = res.data
          if (data.resultType === 1) {
            const poi = data.pois[0]
            const position = poi.lonlat.split(',')
            const r = {
              wgs84_x: position[0],
              wgs84_y: position[1],
              an_address: poi.address,
              an_name: poi.name,
              httpcode: '200'
            }
            this.updateTableData(row.index, r)
          } else {
            if (this.hasAlternative && !alternative) {
              this.geoSearch(row, true)
            }
          }
        } else {
          this.updateTableData(row.index, result)
        }
      }).catch(err => {
        console.log(err)
        this.updateTableData(row.index, result)
      })
    },
    beforeUpload(file) {
      const isLt1M = file.size / 1024 / 1024 < 1
      if (isLt1M) {
        return true
      }
      this.$message({
        message: 'Please do not upload files larger than 1m in size.',
        type: 'warning'
      })
      return false
    },
    handleSuccess({ results, header, filename }) {
      this.tableData = results
      this.tableHeader = header
      this.tableHeader.push('wgs84_x')
      this.tableHeader.push('wgs84_y')
      this.tableHeader.push('an_address')
      this.tableHeader.push('an_name')
      this.tableHeader.push('httpcode')
      this.tableHeader.push('index')
      this.filename = filename
    }
  }
}
</script>

<style lang="scss" scoped>
.geocode {
  &-container {
    padding: 15px;
  }
  &-text {
    font-size: 26px;
    line-height: 40px;
    background-color: #dad6d0a6;
  }
  &-data {
    margin-top: 10px;
  }
}
</style>
