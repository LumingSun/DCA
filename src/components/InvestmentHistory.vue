<template>
  <div class="investment-history">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>定投历史记录</span>
          <div class="header-actions">
            <el-select v-model="selectedProduct" placeholder="选择产品" style="width: 200px; margin-right: 10px;" clearable>
              <el-option label="所有产品" value="all" />
              <el-option 
                v-for="product in availableProducts" 
                :key="product.id" 
                :label="product.name" 
                :value="product.id"
              />
            </el-select>
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
        <!-- 汇总记录表格 -->
        <el-table :data="filteredHistory" style="width: 100%">
          <el-table-column prop="week" label="期数" width="80" />
          
          <el-table-column prop="date" label="日期" width="180">
            <template #default="scope">
              {{ formatDate(scope.row.date) }}
            </template>
          </el-table-column>
          
          <el-table-column prop="totalWeeklyInvestment" label="本期总定投" width="120">
            <template #default="scope">
              {{ formatMoney(scope.row.totalWeeklyInvestment) }}
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
          
          <el-table-column prop="totalProfitRate" label="总收益率" width="100">
            <template #default="scope">
              <span :class="{ 'profit': scope.row.totalProfitRate > 0, 'loss': scope.row.totalProfitRate < 0 }">
                {{ formatPercentage(scope.row.totalProfitRate) }}
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
        
        <!-- 产品历史记录 -->
        <div v-if="selectedProduct !== 'all' && selectedProduct" class="product-history">
          <el-divider content-position="left">
            {{ getProductName(selectedProduct) }} - 投资历史
          </el-divider>
          
          <el-table :data="productHistory" style="width: 100%">
            <el-table-column prop="week" label="期数" width="80" />
            
            <el-table-column prop="date" label="日期" width="180">
              <template #default="scope">
                {{ formatDate(scope.row.date) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="currentMarketValue" label="当前市值" width="120">
              <template #default="scope">
                {{ formatMoney(scope.row.currentMarketValue) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="targetValue" label="目标市值" width="120">
              <template #default="scope">
                {{ formatMoney(calculateProductTargetValue(selectedProduct, scope.row.week)) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="weeklyInvestment" label="本期定投" width="120">
              <template #default="scope">
                <span :class="{ 'profit': scope.row.weeklyInvestment < 0, 'investment': scope.row.weeklyInvestment > 0 }">
                  {{ formatMoney(scope.row.weeklyInvestment) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="totalInvested" label="累计投资" width="120">
              <template #default="scope">
                {{ formatMoney(calculateProductTotalInvested(selectedProduct, scope.row.week)) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="profitRate" label="收益率" width="100">
              <template #default="scope">
                <span :class="{ 'profit': calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week) > 0, 'loss': calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week) < 0 }">
                  {{ formatPercentage(calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week)) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="status" label="状态" width="100">
              <template #default="scope">
                <el-tag :type="getInvestmentStatusType(scope.row.weeklyInvestment)">
                  {{ scope.row.status }}
                </el-tag>
              </template>
            </el-table-column>
          </el-table>
        </div>
        
        <div class="summary-chart">
          <h3>投资趋势图</h3>
          <div ref="chartContainer" style="height: 400px; width: 100%; min-width: 100%; max-width: none; box-sizing: border-box;"></div>
        </div>
      </div>
    </el-card>
    
    <!-- 详情对话框 -->
    <el-dialog v-model="detailDialogVisible" title="期度详情" width="90%">
      <div v-if="selectedRecord">
        <h4>第{{ selectedRecord.week }}期定投详情</h4>
        
        <!-- 汇总信息 -->
        <el-descriptions :column="3" border style="margin-bottom: 20px;">
          <el-descriptions-item label="记录时间">
            {{ formatDate(selectedRecord.date) }}
          </el-descriptions-item>
          <el-descriptions-item label="本期总定投">
            {{ formatMoney(selectedRecord.totalWeeklyInvestment) }}
          </el-descriptions-item>
          <el-descriptions-item label="总市值">
            {{ formatMoney(selectedRecord.totalMarketValue) }}
          </el-descriptions-item>
          <el-descriptions-item label="总目标市值">
            {{ formatMoney(selectedRecord.totalTargetValue) }}
          </el-descriptions-item>
          <el-descriptions-item label="总收益率">
            <span :class="{ 'profit': selectedRecord.totalProfitRate > 0, 'loss': selectedRecord.totalProfitRate < 0 }">
              {{ formatPercentage(selectedRecord.totalProfitRate) }}
            </span>
          </el-descriptions-item>
          <el-descriptions-item label="投资状态">
            <el-tag :type="getInvestmentStatusType(selectedRecord.totalWeeklyInvestment)">
              {{ getInvestmentStatus(selectedRecord.totalWeeklyInvestment) }}
            </el-tag>
          </el-descriptions-item>
        </el-descriptions>
        
        <!-- 产品详情表格 -->
        <div v-if="selectedRecord.productRecords && selectedRecord.productRecords.length > 0">
          <h5>产品详情</h5>
          <el-table :data="selectedRecord.productRecords" style="width: 100%">
            <el-table-column prop="productName" label="产品名称" width="150">
              <template #default="scope">
                <span style="font-weight: bold;">{{ scope.row.productName }}</span>
              </template>
            </el-table-column>
            
            <el-table-column prop="currentMarketValue" label="当前市值" width="120">
              <template #default="scope">
                {{ formatMoney(scope.row.currentMarketValue) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="targetValue" label="目标市值" width="120">
              <template #default="scope">
                {{ formatMoney(calculateProductTargetValue(selectedProduct, scope.row.week)) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="weeklyInvestment" label="本期定投" width="120">
              <template #default="scope">
                <span :class="{ 'profit': scope.row.weeklyInvestment < 0, 'investment': scope.row.weeklyInvestment > 0 }">
                  {{ formatMoney(scope.row.weeklyInvestment) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="totalInvested" label="累计投资" width="120">
              <template #default="scope">
                {{ formatMoney(calculateProductTotalInvested(selectedProduct, scope.row.week)) }}
              </template>
            </el-table-column>
            
            <el-table-column prop="profitRate" label="收益率" width="100">
              <template #default="scope">
                <span :class="{ 'profit': calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week) > 0, 'loss': calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week) < 0 }">
                  {{ formatPercentage(calculateProductProfitRate(selectedProduct, scope.row.currentMarketValue, scope.row.week)) }}
                </span>
              </template>
            </el-table-column>
            
            <el-table-column prop="status" label="状态" width="100">
              <template #default="scope">
                <el-tag :type="getInvestmentStatusType(scope.row.weeklyInvestment)">
                  {{ scope.row.status }}
                </el-tag>
              </template>
            </el-table-column>
          </el-table>
        </div>
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
    },
    products: {
      type: Array,
      required: true
    }
  },
  data() {
    return {
      detailDialogVisible: false,
      selectedRecord: null,
      chart: null,
      selectedProduct: 'all'
    }
  },
  computed: {
    availableProducts() {
      // 从历史记录中提取所有出现过的产品
      const productIds = new Set()
      this.history.forEach(record => {
        if (record.productRecords) {
          record.productRecords.forEach(productRecord => {
            productIds.add(productRecord.productId)
          })
        }
      })
      
      return this.products.filter(product => productIds.has(product.id))
    },
    filteredHistory() {
      if (this.selectedProduct === 'all' || !this.selectedProduct) {
        return this.sortedHistory
      }
      
      // 如果选择了特定产品，只显示包含该产品记录的历史
      return this.sortedHistory.filter(record => {
        return record.productRecords && record.productRecords.some(
          productRecord => productRecord.productId === this.selectedProduct
        )
      })
    },
    productHistory() {
      if (this.selectedProduct === 'all' || !this.selectedProduct) {
        return []
      }
      
      // 提取选中产品的历史记录
      const productRecords = []
      this.history.forEach(record => {
        if (record.productRecords) {
          const productRecord = record.productRecords.find(
            pr => pr.productId === this.selectedProduct
          )
          if (productRecord) {
            productRecords.push({
              ...productRecord,
              date: record.date,
              week: record.week
            })
          }
        }
      })
      
      // 按期数排序
      return productRecords.sort((a, b) => a.week - b.week)
    },
    sortedHistory() {
      // 按期数分组记录
      const weekGroups = {}
      this.history.forEach(record => {
        // 使用记录中的期数，如果没有则按时间顺序推断
        let week = record.week || 1
        if (!week && this.history.length > 1) {
          // 如果有多个记录，按时间顺序分配期数
          const sortedRecords = [...this.history].sort((a, b) => new Date(a.date) - new Date(b.date))
          const recordIndex = sortedRecords.findIndex(r => r.date === record.date)
          week = recordIndex + 1
        }
        
        if (!weekGroups[week]) {
          weekGroups[week] = []
        }
        weekGroups[week].push(record)
      })
      
              // 转换为期度数据格式
      return Object.keys(weekGroups).sort((a, b) => parseInt(a) - parseInt(b)).map((week, index) => {
        const weekRecords = weekGroups[week]
        const latestRecord = weekRecords[weekRecords.length - 1] // 取该期最新记录
        const weekNumber = parseInt(week)
        
        // 计算累计投资金额：实时计算所有产品的累计投资
        let totalInvested = 0
        if (latestRecord.productRecords && latestRecord.productRecords.length > 0) {
          totalInvested = latestRecord.productRecords.reduce((sum, productRecord) => {
            return sum + this.calculateProductTotalInvested(productRecord.productId, latestRecord.week)
          }, 0)
        } else {
          // 如果没有产品记录，使用历史记录中的累计投资
          const sortedRecords = [...this.history].sort((a, b) => new Date(a.date) - new Date(b.date))
          for (let i = 0; i <= index; i++) {
            if (sortedRecords[i]) {
              totalInvested += sortedRecords[i].totalWeeklyInvestment || 0
            }
          }
        }
        
        // 实时计算总目标市值
        let totalTargetValue = 0
        if (latestRecord.productRecords && latestRecord.productRecords.length > 0) {
          totalTargetValue = latestRecord.productRecords.reduce((sum, productRecord) => {
            return sum + this.calculateProductTargetValue(productRecord.productId, latestRecord.week)
          }, 0)
        }
        
        // 实时计算总收益率 - 确保只计算一条总收益率线
        let totalProfitRate = 0
        if (totalInvested > 0 && latestRecord.totalMarketValue > 0) {
          totalProfitRate = ((latestRecord.totalMarketValue - totalInvested) / totalInvested) * 100
        }
        
        return {
          week: weekNumber,
          date: latestRecord.date,
          totalWeeklyInvestment: latestRecord.totalWeeklyInvestment || 0,
          totalMarketValue: latestRecord.totalMarketValue || 0,
          totalTargetValue: totalTargetValue, // 添加总目标市值字段
          totalProfitRate: totalProfitRate, // 添加总收益率字段
          totalInvested: totalInvested, // 添加累计投资字段
          productRecords: latestRecord.productRecords || [],
          data: weekRecords // 保存该期所有记录用于详情显示
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
    },
    selectedProduct() {
      this.$nextTick(() => {
        if (this.chart) {
          this.updateChart()
        }
      })
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
    getProductName(productId) {
      const product = this.products.find(p => p.id === productId)
      return product ? product.name : '未知产品'
    },
    calculateProductTotalInvested(productId, currentWeek) {
      // 实时计算产品的累计投资
      let totalInvested = 0
      
      // 从历史记录中累加该产品当前期及之前所有期的定投金额
      this.history.forEach(record => {
        if (record.productRecords) {
          const productRecord = record.productRecords.find(pr => pr.productId === productId)
          if (productRecord && record.week <= currentWeek) {
            totalInvested += productRecord.weeklyInvestment || 0
          }
        }
      })
      
      return totalInvested
    },
    calculateProductTargetValue(productId, week) {
      // 实时计算产品的目标市值
      const product = this.products.find(p => p.id === productId)
      if (!product) return 0
      
      // 找到该产品在历史记录中首次出现的期数
      let firstWeek = null
      for (let i = 0; i < this.history.length; i++) {
        const record = this.history[i]
        if (record.productRecords) {
          const productRecord = record.productRecords.find(pr => pr.productId === productId)
          if (productRecord) {
            firstWeek = record.week
            break
          }
        }
      }
      
      if (firstWeek === null) {
        // 如果产品从未在历史记录中出现过，说明是新产品，从当前期开始算作第1期
        const relativeWeek = 1
        return product.weeklyAmount * product.targetMultiplier * relativeWeek
      } else {
        // 如果产品在历史记录中出现过，计算相对期数
        const relativeWeek = week - firstWeek + 1
        return product.weeklyAmount * product.targetMultiplier * relativeWeek
      }
    },
    calculateProductProfitRate(productId, currentMarketValue, week) {
      // 实时计算产品的收益率
      const totalInvested = this.calculateProductTotalInvested(productId, week)
      if (totalInvested === 0) return 0
      
      return ((currentMarketValue - totalInvested) / totalInvested) * 100
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
      let data = []
      
      if (this.selectedProduct === 'all' || !this.selectedProduct) {
        // 导出汇总数据
        data = this.filteredHistory.map(record => ({
          期数: record.week,
          日期: this.formatDate(record.date),
          本期总定投: this.formatMoney(record.totalWeeklyInvestment),
          总市值: this.formatMoney(record.totalMarketValue),
          总目标市值: this.formatMoney(record.totalTargetValue),
          总收益率: this.formatPercentage(record.totalProfitRate)
        }))
      } else {
        // 导出产品数据
        data = this.productHistory.map(record => ({
          期数: record.week,
          日期: this.formatDate(record.date),
          产品名称: record.productName,
          当前市值: this.formatMoney(record.currentMarketValue),
          目标市值: this.formatMoney(record.targetValue),
          本期定投: this.formatMoney(record.weeklyInvestment),
          累计投资: this.formatMoney(record.totalInvested),
          收益率: this.formatPercentage(record.profitRate),
          状态: record.status
        }))
      }
      
      const csvContent = this.convertToCSV(data)
      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      const fileName = this.selectedProduct === 'all' ? 
        `定投历史记录_${new Date().toISOString().split('T')[0]}.csv` :
        `${this.getProductName(this.selectedProduct)}_投资历史_${new Date().toISOString().split('T')[0]}.csv`
      link.download = fileName
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
      
      let data = []
      let title = '定投趋势分析'
      
      if (this.selectedProduct === 'all' || !this.selectedProduct) {
        // 显示汇总数据
        if (this.filteredHistory.length === 0) {
          return
        }
        
        const maxDataPoints = 50
        data = this.filteredHistory.slice(-maxDataPoints)
        
        const weeks = data.map(item => item.week)
        const totalWeeklyInvestments = data.map(item => item.totalWeeklyInvestment || 0)
        const totalInvested = data.map(item => item.totalInvested || 0)
        const totalTargetValues = data.map(item => item.totalTargetValue || 0)
        const totalProfitRates = data.map(item => item.totalProfitRate || 0)
        
        // 确保只显示一条总收益率线
        const option = {
          title: {
            text: title,
            left: 'center'
          },
          tooltip: {
            trigger: 'axis',
            formatter: function(params) {
              let result = `第${params[0].axisValue}期<br/>`
              params.forEach(param => {
                if (param.seriesName === '总收益率') {
                  result += `${param.seriesName}: ${param.value.toFixed(2)}%<br/>`
                } else {
                  result += `${param.seriesName}: ¥${param.value.toLocaleString()}<br/>`
                }
              })
              return result
            }
          },
          legend: {
            data: ['本期总定投', '累计投资', '总目标市值', '总收益率'],
            top: 30,
            selectedMode: true
          },
          xAxis: {
            type: 'category',
            data: weeks,
            name: '期数'
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
              name: '本期总定投',
              type: 'line',
              data: totalWeeklyInvestments,
              yAxisIndex: 0,
              itemStyle: { color: '#409EFF' },
              smooth: true
            },
            {
              name: '累计投资',
              type: 'line',
              data: totalInvested,
              yAxisIndex: 0,
              itemStyle: { color: '#67C23A' },
              smooth: true
            },
            {
              name: '总目标市值',
              type: 'line',
              data: totalTargetValues,
              yAxisIndex: 0,
              itemStyle: { color: '#909399' },
              smooth: true
            },
            {
              name: '总收益率',
              type: 'line',
              data: totalProfitRates,
              yAxisIndex: 1,
              itemStyle: { color: '#E6A23C' },
              smooth: true
            }
          ],
          animation: false
        }
        
        // 清除之前的图表配置，确保只显示当前配置的系列
        this.chart.clear()
        this.chart.setOption(option, true) // 第二个参数 true 表示不合并配置，完全替换
      } else {
        // 显示产品数据
        if (this.productHistory.length === 0) {
          return
        }
        
        const maxDataPoints = 50
        data = this.productHistory.slice(-maxDataPoints)
        
        const weeks = data.map(item => item.week)
        const currentMarketValues = data.map(item => item.currentMarketValue || 0)
        const targetValues = data.map(item => this.calculateProductTargetValue(this.selectedProduct, item.week))
        const weeklyInvestments = data.map(item => item.weeklyInvestment || 0)
        const totalInvested = data.map(item => this.calculateProductTotalInvested(this.selectedProduct, item.week))
        const profitRates = data.map(item => this.calculateProductProfitRate(this.selectedProduct, item.currentMarketValue || 0, item.week))
        
        title = `${this.getProductName(this.selectedProduct)} - 投资趋势`
        
        const option = {
          title: {
            text: title,
            left: 'center'
          },
          tooltip: {
            trigger: 'axis',
            formatter: function(params) {
              let result = `第${params[0].axisValue}期<br/>`
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
            data: ['当前市值', '目标市值', '本期定投', '累计投资', '收益率'],
            top: 30,
            selectedMode: true
          },
          xAxis: {
            type: 'category',
            data: weeks,
            name: '期数'
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
              name: '当前市值',
              type: 'line',
              data: currentMarketValues,
              yAxisIndex: 0,
              itemStyle: { color: '#67C23A' },
              smooth: true
            },
            {
              name: '目标市值',
              type: 'line',
              data: targetValues,
              yAxisIndex: 0,
              itemStyle: { color: '#909399' },
              smooth: true
            },
            {
              name: '本期定投',
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
              itemStyle: { color: '#F56C6C' },
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
          animation: false
        }
        
        // 清除之前的图表配置，确保只显示当前配置的系列
        this.chart.clear()
        this.chart.setOption(option, true) // 第二个参数 true 表示不合并配置，完全替换
      }
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
  align-items: center;
}

.empty-state {
  text-align: center;
  padding: 40px;
}

.product-history {
  margin-top: 20px;
}

.profit {
  color: #F56C6C;
  font-weight: bold;
}

.loss {
  color: #67C23A;
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