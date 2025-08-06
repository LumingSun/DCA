<template>
  <div class="investment-history">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>定投历史记录</span>
          <div class="header-actions">
            <el-button type="success" @click="exportData">
              <el-icon><Download /></el-icon>
              导出数据
            </el-button>
            <el-button type="danger" @click="clearHistory">
              <el-icon><Delete /></el-icon>
              清空历史
            </el-button>
          </div>
        </div>
      </template>
      
      <div v-if="history.length === 0" class="empty-state">
        <el-empty description="暂无历史记录" />
      </div>
      
      <div v-else>
        <el-table :data="sortedHistory" style="width: 100%">
          <el-table-column prop="week" label="周数" width="80" />
          
          <el-table-column prop="date" label="日期" width="180">
            <template #default="scope">
              {{ formatDate(scope.row.date) }}
            </template>
          </el-table-column>
          
          <el-table-column prop="weeklyInvestment" label="本次投资" width="120">
            <template #default="scope">
              {{ formatMoney(scope.row.weeklyInvestment) }}
            </template>
          </el-table-column>
          
          <el-table-column prop="totalInvested" label="累计投资" width="120">
            <template #default="scope">
              {{ formatMoney(scope.row.totalInvested) }}
            </template>
          </el-table-column>
          
          <el-table-column prop="totalMarketValue" label="总市值" width="120">
            <template #default="scope">
              {{ formatMoney(scope.row.totalMarketValue) }}
            </template>
          </el-table-column>
          
          <el-table-column prop="profitRate" label="收益率" width="100">
            <template #default="scope">
              <span :class="{ 'profit': scope.row.profitRate > 0, 'loss': scope.row.profitRate < 0 }">
                {{ formatPercentage(scope.row.profitRate) }}
              </span>
            </template>
          </el-table-column>
          
          <el-table-column prop="actions" label="操作" width="200">
            <template #default="scope">
              <div style="display: flex; gap: 8px;">
                <el-button size="small" @click="viewDetails(scope.row)">
                  查看详情
                </el-button>
                <el-button size="small" type="danger" @click="deleteRecord(scope.row)">
                  删除
                </el-button>
              </div>
            </template>
          </el-table-column>
        </el-table>
        
        <div class="summary-chart">
          <h3>投资趋势图</h3>
          <div ref="chartContainer" style="height: 400px; width: 100%; min-width: 100%; max-width: none; box-sizing: border-box;"></div>
        </div>
      </div>
    </el-card>
    
    <!-- 详情对话框 -->
    <el-dialog v-model="detailDialogVisible" title="周度详情" width="80%">
      <div v-if="selectedRecord">
        <h4>第{{ selectedRecord.week }}周定投详情</h4>
        <el-descriptions :column="2" border>
          <el-descriptions-item label="记录时间">
            {{ formatDate(selectedRecord.date) }}
          </el-descriptions-item>
          <el-descriptions-item label="本次投资">
            {{ formatMoney(selectedRecord.weeklyInvestment) }}
          </el-descriptions-item>
          <el-descriptions-item label="累计投资">
            {{ formatMoney(selectedRecord.totalInvested) }}
          </el-descriptions-item>
          <el-descriptions-item label="总市值">
            {{ formatMoney(selectedRecord.totalMarketValue) }}
          </el-descriptions-item>
          <el-descriptions-item label="收益率">
            <span :class="{ 'profit': selectedRecord.profitRate > 0, 'loss': selectedRecord.profitRate < 0 }">
              {{ formatPercentage(selectedRecord.profitRate) }}
            </span>
          </el-descriptions-item>
          <el-descriptions-item label="投资状态">
            <el-tag :type="getInvestmentStatusType(selectedRecord.weeklyInvestment)">
              {{ getInvestmentStatus(selectedRecord.weeklyInvestment) }}
            </el-tag>
          </el-descriptions-item>
        </el-descriptions>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { Download, Delete } from '@element-plus/icons-vue'
import * as echarts from 'echarts'

export default {
  name: 'InvestmentHistory',
  components: {
    Download,
    Delete
  },
  props: {
    history: {
      type: Array,
      required: true
    },
    currentWeek: {
      type: Number,
      required: true
    }
  },
  data() {
    return {
      detailDialogVisible: false,
      selectedRecord: null,
      chart: null
    }
  },
  computed: {
    sortedHistory() {
      // 按周数分组记录
      const weekGroups = {}
      this.history.forEach(record => {
        // 使用当前周数，或者从记录中推断周数
        let week = this.currentWeek
        if (this.history.length > 1) {
          // 如果有多个记录，按时间顺序分配周数
          const sortedRecords = [...this.history].sort((a, b) => new Date(a.date) - new Date(b.date))
          const recordIndex = sortedRecords.findIndex(r => r.date === record.date)
          week = recordIndex + 1
        }
        
        if (!weekGroups[week]) {
          weekGroups[week] = []
        }
        weekGroups[week].push(record)
      })
      
      // 转换为周度数据格式
      return Object.keys(weekGroups).sort((a, b) => parseInt(a) - parseInt(b)).map((week, index) => {
        const weekRecords = weekGroups[week]
        const latestRecord = weekRecords[weekRecords.length - 1] // 取该周最新记录
        
        // 计算累计投资金额（从历史记录中累加）
        const weekNumber = parseInt(week)
        let totalInvested = 0
        
        // 计算到当前周为止的累计投资
        const sortedRecords = [...this.history].sort((a, b) => new Date(a.date) - new Date(b.date))
        for (let i = 0; i <= index; i++) {
          if (sortedRecords[i]) {
            totalInvested += sortedRecords[i].weeklyInvestment
          }
        }
        
        // 计算收益率
        const profitRate = totalInvested > 0 ? ((latestRecord.totalMarketValue - totalInvested) / totalInvested) * 100 : 0
        
        return {
          week: weekNumber,
          date: latestRecord.date,
          weeklyInvestment: latestRecord.weeklyInvestment,
          totalMarketValue: latestRecord.totalMarketValue,
          totalInvested: totalInvested,
          profitRate: profitRate,
          data: weekRecords // 保存该周所有记录用于详情显示
        }
      })
    }
  },
  watch: {
    sortedHistory: {
      handler() {
        this.$nextTick(() => {
          if (this.chart) {
            this.updateChart()
          } else {
            this.initChart()
          }
        })
      },
      deep: true
    }
  },
  mounted() {
    this.$nextTick(() => {
      // 延迟初始化图表，确保DOM完全渲染
      setTimeout(() => {
        this.initChart()
      }, 300)
    })
    
    // 监听窗口大小变化
    window.addEventListener('resize', this.handleResize)
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.handleResize)
    if (this.chart) {
      this.chart.dispose()
    }
  },
  methods: {
    calculateProfitRate(totalMarketValue, weeklyInvestment) {
      if (weeklyInvestment <= 0) return 0
      const totalInvested = totalMarketValue - weeklyInvestment
      return totalInvested > 0 ? ((totalMarketValue - totalInvested) / totalInvested) * 100 : 0
    },
    formatDate(dateString) {
      return new Date(dateString).toLocaleDateString('zh-CN')
    },
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
        '追加投资': 'warning',
        '获利了结': 'success',
        '目标达成': 'info',
        '待投资': 'info'
      }
      return types[status] || 'info'
    },
    getInvestmentStatusType(amount) {
      if (amount < 0) return 'success' // 获利了结
      if (amount > 0) return 'warning' // 追加投资
      return 'info' // 目标达成
    },
    getInvestmentStatus(amount) {
      if (amount < 0) return '获利了结'
      if (amount > 0) return '追加投资'
      return '目标达成'
    },
    viewDetails(record) {
      this.selectedRecord = record
      this.detailDialogVisible = true
    },
    deleteRecord(record) {
      this.$confirm('确定要删除这条记录吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$emit('delete-record', record)
        this.$message.success('记录已删除')
      })
    },
    exportData() {
      const data = this.sortedHistory.map(record => ({
        周数: record.week,
        日期: this.formatDate(record.date),
        本次投资: this.formatMoney(record.weeklyInvestment),
        累计投资: this.formatMoney(record.totalInvested),
        总市值: this.formatMoney(record.totalMarketValue),
        收益率: this.formatPercentage(record.profitRate)
      }))
      
      const csvContent = this.convertToCSV(data)
      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      link.download = `定投历史记录_${new Date().toISOString().split('T')[0]}.csv`
      link.click()
    },
    convertToCSV(data) {
      const headers = Object.keys(data[0])
      const csvRows = [headers.join(',')]
      
      for (const row of data) {
        const values = headers.map(header => `"${row[header]}"`)
        csvRows.push(values.join(','))
      }
      
      return csvRows.join('\n')
    },
    clearHistory() {
      this.$confirm('确定要清空所有历史记录吗？此操作不可恢复。', '警告', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$emit('clear-history')
        this.$message.success('历史记录已清空')
      })
    },
    initChart() {
      if (this.$refs.chartContainer) {
        // 强制设置容器宽度和高度
        this.$refs.chartContainer.style.width = '100%'
        this.$refs.chartContainer.style.minWidth = '100%'
        this.$refs.chartContainer.style.height = '400px'
        this.$refs.chartContainer.style.boxSizing = 'border-box'
        
        // 确保容器有尺寸后再初始化图表
        if (this.$refs.chartContainer.clientWidth > 0 && this.$refs.chartContainer.clientHeight > 0) {
          this.chart = echarts.init(this.$refs.chartContainer)
          this.updateChart()
        } else {
          // 如果容器还没有尺寸，延迟重试
          setTimeout(() => {
            this.initChart()
          }, 100)
        }
      } else {
        // 如果容器还没有尺寸，延迟重试
        setTimeout(() => {
          this.initChart()
        }, 100)
      }
    },
    updateChart() {
      if (!this.chart) {
        return
      }
      
      if (this.sortedHistory.length === 0) {
        return
      }
      
      // 限制数据点数量以提高性能
      const maxDataPoints = 50
      const data = this.sortedHistory.slice(-maxDataPoints)
      
      const weeks = data.map(item => item.week)
      const weeklyInvestments = data.map(item => item.weeklyInvestment || 0)
      const totalInvested = data.map(item => item.totalInvested || 0)
      const marketValues = data.map(item => item.totalMarketValue || 0)
      const profitRates = data.map(item => item.profitRate || 0)
      
      const option = {
        title: {
          text: '定投趋势分析',
          left: 'center'
        },
        tooltip: {
          trigger: 'axis',
          formatter: function(params) {
            let result = `第${params[0].axisValue}周<br/>`
            params.forEach(param => {
              if (param.seriesName === '收益率') {
                result += `${param.seriesName}: ${param.value.toFixed(2)}%<br/>`
              } else {
                result += `${param.seriesName}: ¥${param.value.toLocaleString()}<br/>`
              }
            })
            return result
          }
        },
        legend: {
          data: ['本次投资', '累计投资', '总市值', '收益率'],
          top: 30
        },
        xAxis: {
          type: 'category',
          data: weeks,
          name: '周数'
        },
        yAxis: [
          {
            type: 'value',
            name: '金额 (元)',
            position: 'left'
          },
          {
            type: 'value',
            name: '收益率 (%)',
            position: 'right'
          }
        ],
        series: [
          {
            name: '本次投资',
            type: 'line',
            data: weeklyInvestments,
            yAxisIndex: 0,
            itemStyle: { color: '#409EFF' },
            smooth: true
          },
          {
            name: '累计投资',
            type: 'line',
            data: totalInvested,
            yAxisIndex: 0,
            itemStyle: { color: '#909399' },
            smooth: true
          },
          {
            name: '总市值',
            type: 'line',
            data: marketValues,
            yAxisIndex: 0,
            itemStyle: { color: '#67C23A' },
            smooth: true
          },
          {
            name: '收益率',
            type: 'line',
            data: profitRates,
            yAxisIndex: 1,
            itemStyle: { color: '#E6A23C' },
            smooth: true
          }
        ],
        animation: false // 关闭动画以提高性能
      }
      
      this.chart.setOption(option)
    },
    handleResize() {
      if (this.chart) {
        this.chart.resize()
      }
    }
  }
}
</script>

<style scoped>
.investment-history {
  padding: 20px;
  width: 100%;
  min-width: 100%;
  box-sizing: border-box;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-actions {
  display: flex;
  gap: 10px;
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

.investment {
  color: #E6A23C;
  font-weight: bold;
}

.summary-chart {
  margin-top: 30px;
  width: 100%;
  min-width: 100%;
  box-sizing: border-box;
}

.summary-chart h3 {
  margin-bottom: 20px;
  text-align: center;
}

.summary-chart div {
  width: 100% !important;
  min-width: 100% !important;
  max-width: none !important;
  min-height: 400px;
  box-sizing: border-box !important;
}
</style> 