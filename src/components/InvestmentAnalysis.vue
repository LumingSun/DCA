<template>
  <div class="investment-analysis">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>投资产品分析</span>
        </div>
      </template>
      
      <div v-if="products.length === 0" class="empty-state">
        <el-empty description="暂无投资产品数据" />
      </div>
      
      <div v-else>
        <!-- 产品表现对比 -->
        <div class="analysis-section">
          <h3>产品表现对比</h3>
          <div ref="productChart" style="height: 400px;"></div>
        </div>
        
        <!-- 产品详情表格 -->
        <div class="analysis-section">
          <h3>产品详情统计</h3>
          <el-table :data="productAnalysis" style="width: 100%">
            <el-table-column prop="name" label="产品名称" width="120" />
            
            <el-table-column prop="totalInvested" label="累计投资" width="120">
              <template #default="scope">
                {{ formatMoney(scope.row.totalInvested) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="currentValue" label="当前市值" width="120">
              <template #default="scope">
                {{ formatMoney(scope.row.currentValue) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="profitAmount" label="盈亏金额" width="120">
              <template #default="scope">
                <span :class="{ 'profit': scope.row.profitAmount > 0, 'loss': scope.row.profitAmount < 0 }">
                  {{ formatMoney(scope.row.profitAmount) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="profitRate" label="收益率" width="100">
              <template #default="scope">
                <span :class="{ 'profit': scope.row.profitRate > 0, 'loss': scope.row.profitRate < 0 }">
                  {{ formatPercentage(scope.row.profitRate) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="weight" label="权重" width="100">
              <template #default="scope">
                {{ formatPercentage(scope.row.weight) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="status" label="状态" width="100">
              <template #default="scope">
                <el-tag :type="getStatusType(scope.row.status)">
                  {{ scope.row.status }}
                </el-tag>
              </template>
            </el-table-column>
          </el-table>
        </div>
        
        <!-- 投资策略建议 -->
        <div class="analysis-section">
          <h3>投资策略建议</h3>
          <el-alert
            v-for="(advice, index) in investmentAdvice"
            :key="index"
            :title="advice.title"
            :type="advice.type"
            :description="advice.description"
            show-icon
            :closable="false"
            style="margin-bottom: 10px;"
          />
        </div>
        
        <!-- 风险分析 -->
        <div class="analysis-section">
          <h3>风险分析</h3>
          <el-row :gutter="20">
            <el-col :span="8">
              <el-card class="risk-card">
                <div class="risk-title">最大回撤</div>
                <div class="risk-value">{{ formatPercentage(maxDrawdown) }}</div>
                <div class="risk-desc">历史最大亏损幅度</div>
              </el-card>
            </el-col>
            <el-col :span="8">
              <el-card class="risk-card">
                <div class="risk-title">夏普比率</div>
                <div class="risk-value">{{ sharpeRatio.toFixed(2) }}</div>
                <div class="risk-desc">风险调整后收益</div>
              </el-card>
            </el-col>
            <el-col :span="8">
              <el-card class="risk-card">
                <div class="risk-title">波动率</div>
                <div class="risk-value">{{ formatPercentage(volatility) }}</div>
                <div class="risk-desc">收益波动程度</div>
              </el-card>
            </el-col>
          </el-row>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script>
import * as echarts from 'echarts'

export default {
  name: 'InvestmentAnalysis',
  props: {
    products: {
      type: Array,
      required: true
    },
    history: {
      type: Array,
      required: true
    }
  },
  data() {
    return {
      productChart: null
    }
  },
  computed: {
    productAnalysis() {
      if (this.history.length === 0) {
        return []
      }
      
      // 使用最新的记录数据
      const latestRecord = this.history[this.history.length - 1]
      const totalMarketValue = latestRecord.totalMarketValue
      
      // 由于新格式没有产品详情，我们创建一个简化的分析
      const totalInvested = totalMarketValue - latestRecord.weeklyInvestment
      const profitAmount = totalMarketValue - totalInvested
      const profitRate = totalInvested > 0 ? (profitAmount / totalInvested) * 100 : 0
      
      let status = '正常'
      if (profitRate > 10) status = '优秀'
      else if (profitRate < -10) status = '需关注'
      
      return [{
        name: '投资组合',
        totalInvested: totalInvested,
        currentValue: totalMarketValue,
        profitAmount,
        profitRate,
        weight: 100,
        status
      }]
    },
    investmentAdvice() {
      const advice = []
      
      if (this.productAnalysis.length === 0) return advice
      
      const totalProfitRate = this.productAnalysis.reduce((sum, item) => sum + item.profitRate, 0) / this.productAnalysis.length
      
      if (totalProfitRate > 5) {
        advice.push({
          title: '整体表现良好',
          type: 'success',
          description: '当前投资组合整体收益率为正，建议继续执行定投策略。'
        })
      } else if (totalProfitRate < -5) {
        advice.push({
          title: '市场调整期',
          type: 'warning',
          description: '当前市场处于调整期，建议坚持定投策略，利用市场低位积累更多份额。'
        })
      }
      
      const highProfitProducts = this.productAnalysis.filter(item => item.profitRate > 10)
      if (highProfitProducts.length > 0) {
        advice.push({
          title: '获利了结机会',
          type: 'info',
          description: `发现${highProfitProducts.length}个产品收益率超过10%，可考虑部分获利了结。`
        })
      }
      
      const lowProfitProducts = this.productAnalysis.filter(item => item.profitRate < -10)
      if (lowProfitProducts.length > 0) {
        advice.push({
          title: '加码投资机会',
          type: 'info',
          description: `发现${lowProfitProducts.length}个产品跌幅较大，可考虑适当加码投资。`
        })
      }
      
      return advice
    },
    maxDrawdown() {
      if (this.history.length < 2) return 0
      
      let maxDrawdown = 0
      let peak = 0
      
      this.history.forEach(record => {
        const totalValue = record.totalMarketValue
        if (totalValue > peak) {
          peak = totalValue
        }
        const drawdown = peak > 0 ? ((peak - totalValue) / peak) * 100 : 0
        if (drawdown > maxDrawdown) {
          maxDrawdown = drawdown
        }
      })
      
      return maxDrawdown
    },
    sharpeRatio() {
      if (this.history.length < 2) return 0
      
      const returns = []
      for (let i = 1; i < this.history.length; i++) {
        const prevValue = this.history[i-1].totalMarketValue
        const currValue = this.history[i].totalMarketValue
        const returnRate = prevValue > 0 ? (currValue - prevValue) / prevValue : 0
        returns.push(returnRate)
      }
      
      const avgReturn = returns.reduce((sum, r) => sum + r, 0) / returns.length
      const variance = returns.reduce((sum, r) => sum + Math.pow(r - avgReturn, 2), 0) / returns.length
      const stdDev = Math.sqrt(variance)
      
      return stdDev > 0 ? avgReturn / stdDev : 0
    },
    volatility() {
      if (this.history.length < 2) return 0
      
      const returns = []
      for (let i = 1; i < this.history.length; i++) {
        const prevValue = this.history[i-1].totalMarketValue
        const currValue = this.history[i].totalMarketValue
        const returnRate = prevValue > 0 ? (currValue - prevValue) / prevValue : 0
        returns.push(returnRate)
      }
      
      const avgReturn = returns.reduce((sum, r) => sum + r, 0) / returns.length
      const variance = returns.reduce((sum, r) => sum + Math.pow(r - avgReturn, 2), 0) / returns.length
      
      return Math.sqrt(variance) * 100
    }
  },
  watch: {
    productAnalysis: {
      handler() {
        this.updateChart()
      },
      deep: true
    }
  },
  mounted() {
    this.$nextTick(() => {
      // 延迟初始化图表，确保DOM完全渲染
      setTimeout(() => {
        this.initChart()
      }, 100)
    })
    
    // 监听窗口大小变化
    window.addEventListener('resize', this.handleResize)
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.handleResize)
    if (this.productChart) {
      this.productChart.dispose()
    }
  },
  methods: {
    formatMoney(amount) {
      return new Intl.NumberFormat('zh-CN', {
        style: 'currency',
        currency: 'CNY'
      }).format(amount)
    },
    formatPercentage(rate) {
      return `${rate.toFixed(2)}%`
    },
    getStatusType(status) {
      const types = {
        '优秀': 'success',
        '正常': 'info',
        '需关注': 'warning'
      }
      return types[status] || 'info'
    },
    initChart() {
      if (this.$refs.productChart && this.$refs.productChart.clientWidth > 0) {
        this.productChart = echarts.init(this.$refs.productChart)
        this.updateChart()
      } else {
        // 如果容器还没有尺寸，延迟重试
        setTimeout(() => {
          this.initChart()
        }, 50)
      }
    },
    updateChart() {
      if (!this.productChart || this.productAnalysis.length === 0) return
      
      const names = this.productAnalysis.map(item => item.name)
      const profitRates = this.productAnalysis.map(item => item.profitRate)
      const weights = this.productAnalysis.map(item => item.weight)
      
      const option = {
        title: {
          text: '产品收益率对比',
          left: 'center'
        },
        tooltip: {
          trigger: 'axis',
          axisPointer: {
            type: 'shadow'
          },
          formatter: function(params) {
            const data = params[0]
            const index = data.dataIndex
            return `${data.name}<br/>
                    收益率: ${data.value.toFixed(2)}%<br/>
                    权重: ${weights[index].toFixed(2)}%`
          }
        },
        xAxis: {
          type: 'category',
          data: names,
          axisLabel: {
            rotate: 45
          }
        },
        yAxis: {
          type: 'value',
          name: '收益率 (%)'
        },
        series: [
          {
            name: '收益率',
            type: 'bar',
            data: profitRates,
            itemStyle: {
              color: function(params) {
                const value = params.value
                if (value > 0) return '#67C23A'
                else if (value < 0) return '#F56C6C'
                else return '#909399'
              }
            }
          }
        ],
        animation: false // 关闭动画以提高性能
      }
      
      this.productChart.setOption(option)
    },
    handleResize() {
      if (this.productChart) {
        this.productChart.resize()
      }
    }
  }
}
</script>

<style scoped>
.investment-analysis {
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.empty-state {
  text-align: center;
  padding: 40px;
}

.profit {
  color: #67C23A;
  font-weight: bold;
}

.loss {
  color: #F56C6C;
  font-weight: bold;
}

.analysis-section {
  margin-bottom: 30px;
}

.analysis-section h3 {
  margin-bottom: 20px;
  color: #303133;
  border-bottom: 2px solid #409EFF;
  padding-bottom: 10px;
}

.risk-card {
  text-align: center;
  height: 120px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.risk-title {
  font-size: 14px;
  color: #909399;
  margin-bottom: 10px;
}

.risk-value {
  font-size: 24px;
  font-weight: bold;
  color: #303133;
  margin-bottom: 10px;
}

.risk-desc {
  font-size: 12px;
  color: #C0C4CC;
}
</style> 