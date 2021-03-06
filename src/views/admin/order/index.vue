<template>
  <el-row v-loading="loading">
    <el-col :span="24">
      <el-row class="search-row">
        <el-col :span="24">
          <el-input v-model="search.condition.merchant_no" placeholder="商户号" size="mini" />
          <el-input v-model="search.condition.platform_order_no" placeholder="平台订单号" size="mini" />
          <el-input v-model="search.condition.marchant_order_no" placeholder="商户订单号" size="mini" />
          <el-date-picker
            v-model="queryDate"
            size="mini"
            type="datetimerange"
            range-separator="至"
            start-placeholder="订单创建开始日期"
            end-placeholder="订单创建结束日期"
          />
          <el-select v-model="queryPayState" size="mini" placeholder="支付状态">
            <el-option v-for="item in orderPayState" :key="item.value" :label="item.label" :value="item.value" />
          </el-select>
          <el-select v-model="queryNotifyState" size="mini" placeholder="通知状态">
            <el-option v-for="item in orderNotifyState" :key="item.value" :label="item.label" :value="item.value" />
          </el-select>
          <el-select v-model="queryChannelId" size="mini" placeholder="支付通道">
            <el-option v-for="item in orderChannel" :key="item.value" :label="item.label" :value="item.value" />
          </el-select>
          <el-button type="primary" size="mini" icon="el-icon-search" round @click="getDataList()">
            搜索
          </el-button>
          <el-button type="primary" size="mini" icon="el-icon-delete" round @click="queryReset()">
            重置
          </el-button>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="24">
          <el-table
            :data="dataList"
            border
            :summary-method="getSummaries"
            show-summary
            header-cell-class-name="header-row"
          >
            <el-table-column prop="marchant.merchantNo" label="商户号" header-align="center" align="center" />
            <el-table-column prop="platformOrderNo" label="平台订单号" header-align="center" align="center" />
            <el-table-column prop="marchantOrderNo" label="商户订单号" header-align="center" align="center" />
            <el-table-column prop="payMoney" label="支付金额" header-align="center" align="center">
              <template slot-scope="scope">
                <span>{{ scope.row.payMoney | CentToDollars }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="balanceMoney" label="结算金额" header-align="center" align="center">
              <template slot-scope="scope">
                <span>{{ scope.row.balanceMoney | CentToDollars }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="poundage" label="手续费" header-align="center" align="center">
              <template slot-scope="scope">
                <span>{{ scope.row.poundage | CentToDollars }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="orderMoney" label="订单金额" header-align="center" align="center">
              <template slot-scope="scope">
                <span>{{ scope.row.orderMoney | CentToDollars }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="createOrderTime" label="创建时间" header-align="center" align="center" />
            <el-table-column prop="payTime" label="支付时间" header-align="center" align="center" />
            <el-table-column prop="channel.name" label="支付通道" header-align="center" align="center" />
            <el-table-column prop="orderType" label="订单类型" header-align="center" align="center">
              <template slot-scope="scope">
                <span :class="orderTypeClass(scope.row.orderType)">{{ orderType(scope.row.orderType) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="balanceState" label="结算状态" header-align="center" align="center">
              <template slot-scope="scope">
                <span :class="balanceStateClass(scope.row.balanceState)">{{ balanceState(scope.row.balanceState) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="payState" label="支付状态" header-align="center" align="center">
              <template slot-scope="scope">
                <span :class="payStateClass(scope.row.payState)">{{ payState(scope.row.payState) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="notifyState" label="通知状态" header-align="center" align="center">
              <template slot-scope="scope">
                <span :class="balanceStateClass(scope.row.notifyState)">{{ notifyState(scope.row.notifyState) }}</span>
              </template>
            </el-table-column>
            <el-table-column label="操作" header-align="center" align="center">
              <template slot-scope="scope">
                <el-button v-show="permission.balance" type="primary" size="mini" @click="startCallback(scope.$index, scope.row)">
                  补单
                </el-button>
              </template>
            </el-table-column>
          </el-table>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="24">
          <el-pagination
            :current-page="search.pageNum"
            :page-sizes="[5, 10, 20, 30, 40, 50, 100,200]"
            :page-size="search.size"
            :total="total"
            background
            layout="total, sizes, prev, pager, next, jumper"
            @size-change="handleSizeChange"
            @current-change="handleCurrentChange"
          />
        </el-col>
      </el-row>
    </el-col>
  </el-row>
</template>

<script>
import { getAllOrderList, getOrderState, balanceOrder } from '@/api/pay/order'
import { getAllChannel } from '@/api/pay/channel'
import { dateFormat, CentToDollars } from '@/common/commonFun'
import store from '@/store'

export default {
  data() {
    return {
      loading: false,
      permission: {
        freeze: false,
        balance: false,
        refund: false
      },
      search: {
        condition: {},
        pageNum: 1,
        size: 10
      },
      queryDate: [],
      total: 0,
      dataList: [],
      orderPayState: [{
        value: 82001,
        label: '待支付'
      }, {
        value: 82002,
        label: '已支付'
      }, {
        value: 82003,
        label: '支付异常'
      }],
      queryPayState: '',
      orderNotifyState: [{
        value: 1,
        label: '待通知'
      }, {
        value: 2,
        label: '已通知'
      }, {
        value: 3,
        label: '通知异常'
      }],
      queryNotifyState: '',
      orderChannel: [],
      queryChannelId: ''
    }
  },
  created() {
    this.pageInit()
  },
  methods: {
    getDataList() {
      this.loading = true
      if (this.queryDate.length === 2) {
        this.search.condition.startTime = dateFormat(this.queryDate[0], 'yyyy-MM-dd hh:mm:ss')
        this.search.condition.endTime = dateFormat(this.queryDate[1], 'yyyy-MM-dd hh:mm:ss')
      }
      if (this.queryPayState) {
        this.search.condition.pay_state = this.queryPayState
      }
      if (this.queryNotifyState) {
        this.search.condition.notify_state = this.queryNotifyState
      }
      if (this.queryChannelId) {
        this.search.condition.channel_id = this.queryChannelId
      }
      getAllOrderList(this.search)
        .then(response => {
          this.total = response.data.total
          this.pageNum = response.data.current
          this.dataList = response.data.rows
          this.loading = false
        })
        .catch(error => {
          this.outputError(error)
        })
    },
    queryReset() {
      this.queryDate = []
      this.queryPayState = ''
      this.queryNotifyState = ''
      this.queryChannelId = ''
      this.search.condition = {}
      this.getDataList()
    },
    handleSizeChange(val) {
      this.search.size = val
      this.getDataList()
    },
    handleCurrentChange(val) {
      this.search.pageNum = val
      this.getDataList()
    },

    async pageInit() {
      this.loading = true
      let resources = store.getters.resources
      let read = resources.find(item => {
        return item.permission === 'admin:order:read'
      })
      if (!read) {
        this.loading = false
        new Promise((resolve, reject) => {
          reject(new Error('没有访问权限'))
        }).catch(reason => {
          this.outputError(reason)
        })
        return
      }
      getAllOrderList(this.search).then(response => {
        let data = response.data
        this.dataList = data.rows
        this.total = data.total
        this.pageNum = data.current
        this.permission.freeze = resources.find(item => {
          return item.permission === 'admin:order:freeze'
        })
        this.permission.refund = resources.find(item => {
          return item.permission === 'admin:order:refund'
        })
        this.permission.balance = resources.find(item => {
          return item.permission === 'admin:order:balance'
        })
        this.loading = false
      }).catch(error => {
        this.outputError(error)
      })
      getAllChannel().then(response => {
        let data = response.data
        if (data.length > 0) {
          for (let i = 0, l = data.length; i < l; i++) {
            let tempObj = {}
            tempObj.value = data[i].id
            tempObj.label = data[i].name
            this.orderChannel.push(tempObj)
          }
        }
      }).catch(error => {
        this.outputError(error)
      })
    },
    async startCallback(index, row) {
      this.loading = true
      let html = ''
      try {
        let [orderState] = await Promise.all([
          getOrderState(row.id)
        ])
        switch (orderState.data) {
          case 82100:
            html = '上游<strong style="color: #ff0000">不存在订单查询接口</strong>'
            break
          case 82101:
            html = '上游订单状态为: <strong style="color: #7b652d">等待支付</strong>'
            break
          case 82102:
            html = '上游订单状态为: <strong style="color: #259c32">支付完成</strong>'
            break
          case 82103:
            html = '上游订单状态为: <strong style="color: #ff0000">支付错误</strong>'
            break
          case 82104:
            html = '上游订单状态为: <strong style="color: #ff0000">其它错误</strong>'
            break
          default:
        }
      } catch (e) {
        this.outputError(e)
      }
      this.$confirm(html, '警告', {
        dangerouslyUseHTMLString: true,
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.loading = false
        balanceOrder(row.id).then(response => {
        }).catch(error => {
          this.outputError(error)
        })
      })
      this.loading = false
    },
    balanceState(val) {
      switch (val) {
        case 1:
          return '待结算'
        case 2:
          return '已结算'
        case 3:
          return '结算异常'
      }
    },
    orderType(val) {
      switch (val) {
        case 83001:
          return '正常'
        case 83002:
          return '补单'
        case 83003:
          return '冻结'
        case 83004:
          return '退款'
        case 83005:
          return '错误'
      }
    },
    orderTypeClass(val) {
      let ot = val
      if (ot === 83001) {
        return 'balance-succes'
      } else if (ot === 83002) {
        return 'balance-wait'
      } else if (ot === 83003) {
        return 'balance-wait'
      } else if (ot === 83004) {
        return 'balance-wait'
      } else if (ot === 83005) {
        return 'balance-error'
      }
    },
    notifyState(val) {
      switch (val) {
        case 1:
          return '待通知'
        case 2:
          return '已通知'
        case 3:
          return '通知异常'
      }
    },
    balanceStateClass(val) {
      switch (val) {
        case 1:
          return 'balance-wait'
        case 2:
          return 'balance-succes'
        case 3:
          return 'balance-error'
      }
    },
    payState(val) {
      switch (val) {
        case 82001:
          return '待支付'
        case 82002:
          return '已支付'
        case 82003:
          return '支付异常'
      }
    },
    payStateClass(val) {
      switch (val) {
        case 82001:
          return 'balance-wait'
        case 82002:
          return 'balance-succes'
        case 82003:
          return 'balance-error'
      }
    },
    getSummaries(param) {
      const { columns, data } = param
      const sums = []
      columns.forEach((column, index) => {
        switch (column.property) {
          case 'platformOrderNo':
            sums[index] = '总     计'
            break
          case 'orderMoney':
            sums[index] = CentToDollars(this.sumData(data, column.property))
            sums[index] += ' 元'
            break
          case 'payMoney':
            sums[index] = CentToDollars(this.sumData(data, column.property))
            sums[index] += ' 元'
            break
          case 'balanceMoney':
            sums[index] = CentToDollars(this.sumData(data, column.property))
            sums[index] += ' 元'
            break
          case 'poundage':
            sums[index] = CentToDollars(this.sumData(data, column.property))
            sums[index] += ' 元'
            break
          case 'platformOrderNo':
            sums[index] = '---'
            break
          case 'marchantOrderNo':
            sums[index] = '---'
            break
          case 'createOrderTime':
            sums[index] = '---'
            break
          case 'channel.name':
            sums[index] = '---'
            break
          case 'balanceState':
            sums[index] = '---'
            break
          case 'payState':
            sums[index] = '---'
            break
          case 'notifyResult':
            sums[index] = '---'
            break
          case 'notifyState':
            sums[index] = '---'
            break
        }
      })
      return sums
    },
    sumData(data, property) {
      let values = data.map(item => Number(item[property]))
      let result = 0
      for (let i = 0, l = values.length; i < l; i++) {
        result += values[i]
      }
      return result
    },
    outputError(error) {
      console.log(error.response ? error.response : error.message)
      this.loading = false
      this.$message({
        showClose: true,
        message: error.message,
        type: 'error'
      })
    }
  }
}
</script>

<style lang="scss" scoped>

  .search-row {
    padding-bottom: 8px;

    .el-input {
      width: 140px;
    }
    .el-date-editor--daterange.el-input,
    .el-date-editor--daterange.el-input__inner,
    .el-date-editor--timerange.el-input,
    .el-date-editor--timerange.el-input__inner{
      width: 280px
    }
  }

  .balance-wait{
    color: #ff9051;
  }
  .balance-succes{
    color: #009933
  }
  .balance-error{
    color: #FF3300;
  }

</style>
