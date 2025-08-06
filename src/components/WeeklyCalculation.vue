<template>
  <div class="weekly-calculation">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>第{{ currentWeek }}周定投计算</span>
          <div class="week-controls">
            <el-button @click="previousWeek" :disabled="currentWeek <= 1">
              <el-icon><ArrowLeft /></el-icon>
              上一周
            </el-button>
            <el-button type="primary" @click="nextWeek">
              下一周
              <el-icon><ArrowRight /></el-icon>
            </el-button>
          </div>
        </div>
      </template>
      
      <!-- 第一周提示 -->
      <el-alert 
        v-if="currentWeek === 1" 
        title="第一周提示" 
        type="info" 
        description="第一周时，如果还没有开始投资，当前总市值可以填写0"
        :closable="false"
        style="margin-bottom: 16px"
      />

      <!-- 产品计算区域 -->
      <div class="products-calculation">
        <div 
          v-for="product in products" 
          :key="product.id" 
          class="product-calculation-card"
        >
          <el-card :style="{ borderLeft: `4px solid ${product.color}` }">
            <template #header>
              <div class="product-header">
                <span class="product-name">{{ product.name }}</span>
                <el-tag :color="product.color" text-color="white">
                  周定投 {{ formatMoney(product.weeklyAmount) }}
                </el-tag>
              </div>
            </template>
            
            <el-row :gutter="20">
              <el-col :span="8">
                <div class="input-group">
                  <label class="input-label">当前市值：</label>
                  <el-input-number 
                    v-model="productData[product.id].currentMarketValue" 
                    :min="0" 
                    :precision="2"
                    :placeholder="currentWeek === 1 ? '第一周可填写0' : '请输入当前市值'"
                    style="width: 100%"
                    @change="calculateProductInvestment(product.id)"
                  />
                </div>
              </el-col>
              <el-col :span="8">
                <div class="input-group">
                  <label class="input-label">目标市值：</label>
                  <div class="target-value-display">
                    <span class="amount">{{ formatMoney(getProductTargetValue(product)) }}</span>
                  </div>
                </div>
              </el-col>
              <el-col :span="8">
                <div class="input-group">
                  <label class="input-label">本周定投：</label>
                  <div class="investment-amount-display">
                    <span class="amount">{{ formatMoney(productData[product.id].weeklyInvestment) }}</span>
                    <el-tag :type="getInvestmentType(productData[product.id].weeklyInvestment)" class="investment-tag">
                      {{ getInvestmentStatus(productData[product.id].weeklyInvestment) }}
                    </el-tag>
                  </div>
                </div>
              </el-col>
            </el-row>
            
            <!-- 产品计算详情 -->
            <div class="product-details" v-if="productData[product.id].currentMarketValue >= 0">
              <el-divider content-position="left">计算详情</el-divider>
              
              <el-descriptions :column="2" border size="small">
                <el-descriptions-item label="目标倍数">
                  {{ product.targetMultiplier }}x
                </el-descriptions-item>
                <el-descriptions-item label="市值差额">
                  <span :class="{ 'profit': getProductMarketValueDifference(product.id) > 0, 'loss': getProductMarketValueDifference(product.id) < 0 }">
                    {{ formatMoney(getProductMarketValueDifference(product.id)) }}
                  </span>
                </el-descriptions-item>
                <el-descriptions-item label="累计投资">
                  {{ formatMoney(getProductTotalInvested(product.id)) }}
                </el-descriptions-item>
                <el-descriptions-item label="收益率">
                  <span :class="{ 'profit': getProductProfitRate(product.id) > 0, 'loss': getProductProfitRate(product.id) < 0 }">
                    {{ formatPercentage(getProductProfitRate(product.id)) }}
                  </span>
                </el-descriptions-item>
              </el-descriptions>
            </div>
          </el-card>
        </div>
      </div>

      <!-- 汇总信息 -->
      <div class="summary-section">
        <el-divider content-position="left">汇总信息</el-divider>
        
        <el-row :gutter="20">
          <el-col :span="6">
            <div class="summary-item">
              <label class="summary-label">总目标市值：</label>
              <div class="summary-value">{{ formatMoney(totalTargetValue) }}</div>
            </div>
          </el-col>
          <el-col :span="6">
            <div class="summary-item">
              <label class="summary-label">总当前市值：</label>
              <div class="summary-value">{{ formatMoney(totalCurrentValue) }}</div>
            </div>
          </el-col>
          <el-col :span="6">
            <div class="summary-item">
              <label class="summary-label">本周总定投：</label>
              <div class="summary-value" :class="{ 'profit': totalWeeklyInvestment < 0, 'investment': totalWeeklyInvestment > 0 }">
                {{ formatMoney(totalWeeklyInvestment) }}
              </div>
            </div>
          </el-col>
          <el-col :span="6">
            <div class="summary-item">
              <label class="summary-label">总收益率：</label>
              <div class="summary-value" :class="{ 'profit': totalProfitRate > 0, 'loss': totalProfitRate < 0 }">
                {{ formatPercentage(totalProfitRate) }}
              </div>
            </div>
          </el-col>
        </el-row>
      </div>
      
      <div class="actions">
        <el-button type="success" @click="saveCurrentRecord" :disabled="!canSave">
          保存当前记录
        </el-button>
        <el-button @click="resetCalculation">
          重置计算
        </el-button>
      </div>
    </el-card>
  </div>
</template>

<script>
import { ArrowLeft, ArrowRight } from '@element-plus/icons-vue'

export default {
  name: 'WeeklyCalculation',
  components: {
    ArrowLeft,
    ArrowRight
  },
  props: {
    products: {
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
      productData: {}
    }
  },
  computed: {
    totalTargetValue() {
      return this.products.reduce((sum, product) => {
        return sum + this.getProductTargetValue(product)
      }, 0)
    },
    totalCurrentValue() {
      return this.products.reduce((sum, product) => {
        return sum + (this.productData[product.id]?.currentMarketValue || 0)
      }, 0)
    },
    totalWeeklyInvestment() {
      return this.products.reduce((sum, product) => {
        return sum + (this.productData[product.id]?.weeklyInvestment || 0)
      }, 0)
    },
    totalInvestedAmount() {
      // 总累计投资应该是本周之前的累计投资，不包含本周的定投金额
      return this.products.reduce((sum, product) => {
        return sum + ((this.currentWeek - 1) * product.weeklyAmount)
      }, 0)
    },
    totalProfitRate() {
      if (this.totalInvestedAmount === 0) return 0
      
      return ((this.totalCurrentValue - this.totalInvestedAmount) / this.totalInvestedAmount) * 100
    },
    canSave() {
      return this.products.every(product => 
        this.productData[product.id]?.currentMarketValue >= 0
      )
    }
  },
  watch: {
    currentWeek: {
      handler() {
        this.initializeWeekData()
      },
      immediate: true
    },
    products: {
      handler() {
        this.initializeProductData()
      },
      immediate: true
    }
  },
  methods: {
    initializeProductData() {
      this.products.forEach(product => {
        if (!this.productData[product.id]) {
          this.productData[product.id] = {
            currentMarketValue: 0,
            weeklyInvestment: 0
          }
        }
      })
    },
    getProductTargetValue(product) {
      return product.weeklyAmount * product.targetMultiplier * this.currentWeek
    },
    getProductMarketValueDifference(productId) {
      const product = this.products.find(p => p.id === productId)
      if (!product) return 0
      
      const targetValue = this.getProductTargetValue(product)
      const currentValue = this.productData[productId]?.currentMarketValue || 0
      return targetValue - currentValue
    },
    getProductTotalInvested(productId) {
      // 累计投资应该是本周之前的累计投资，不包含本周的定投金额
      return (this.currentWeek - 1) * (this.products.find(p => p.id === productId)?.weeklyAmount || 0)
    },
    getProductProfitRate(productId) {
      const product = this.products.find(p => p.id === productId)
      if (!product) return 0
      
      const totalInvested = this.getProductTotalInvested(productId)
      const currentValue = this.productData[productId]?.currentMarketValue || 0
      
      // 如果累计投资为0（第一周），返回0%
      if (totalInvested === 0) return 0
      
      return ((currentValue - totalInvested) / totalInvested) * 100
    },
    calculateProductInvestment(productId) {
      const product = this.products.find(p => p.id === productId)
      if (!product) return
      
      const currentValue = this.productData[productId]?.currentMarketValue || 0
      
      // 第一周时，如果当前市值为0，则按计划金额定投
      if (this.currentWeek === 1 && currentValue <= 0) {
        this.productData[productId].weeklyInvestment = product.weeklyAmount
        return
      }
      
      // 其他情况：计算本周应定投金额
      const targetValue = this.getProductTargetValue(product)
      this.productData[productId].weeklyInvestment = targetValue - currentValue
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
    getInvestmentType(amount) {
      if (amount < 0) return 'success' // 获利了结
      if (amount > 0) return 'warning' // 追加投资
      return 'info' // 目标达成
    },
    getInvestmentStatus(amount) {
      if (amount < 0) return '获利了结'
      if (amount > 0) return '追加投资'
      return '目标达成'
    },
    previousWeek() {
      if (this.currentWeek > 1) {
        this.$emit('update:currentWeek', this.currentWeek - 1)
      }
    },
    nextWeek() {
      this.$emit('update:currentWeek', this.currentWeek + 1)
    },
    saveCurrentRecord() {
      // 为每个产品创建记录
      const productRecords = this.products.map(product => {
        const newMarketValue = (this.productData[product.id]?.currentMarketValue || 0) + 
                              (this.productData[product.id]?.weeklyInvestment || 0)
        
        return {
          productId: product.id,
          productName: product.name,
          date: new Date().toISOString(),
          week: this.currentWeek,
          currentMarketValue: newMarketValue,
          weeklyInvestment: this.productData[product.id]?.weeklyInvestment || 0,
          status: this.getInvestmentStatus(this.productData[product.id]?.weeklyInvestment || 0),
          targetValue: this.getProductTargetValue(product),
          totalInvested: this.getProductTotalInvested(product.id),
          profitRate: this.getProductProfitRate(product.id)
        }
      })
      
      // 创建汇总记录
      const summaryRecord = {
        date: new Date().toISOString(),
        week: this.currentWeek,
        totalMarketValue: this.totalCurrentValue + this.totalWeeklyInvestment,
        totalWeeklyInvestment: this.totalWeeklyInvestment,
        totalTargetValue: this.totalTargetValue,
        totalProfitRate: this.totalProfitRate,
        productRecords: productRecords
      }
      
      // 发送记录到父组件
      this.$emit('save-week-data', summaryRecord)
      
      // 更新当前市值为保存后的值
      this.products.forEach(product => {
        if (this.productData[product.id]) {
          this.productData[product.id].currentMarketValue += this.productData[product.id].weeklyInvestment
        }
      })
      
      this.$message.success('所有产品记录已保存')
    },
    initializeWeekData() {
      // 重置所有产品的当前市值为0
      this.products.forEach(product => {
        if (!this.productData[product.id]) {
          this.productData[product.id] = {
            currentMarketValue: 0,
            weeklyInvestment: 0
          }
        } else {
          this.productData[product.id].currentMarketValue = 0
        }
      })
      
      // 重新计算所有产品的本周定投金额
      this.products.forEach(product => {
        this.calculateProductInvestment(product.id)
      })
    },
    resetCalculation() {
      this.products.forEach(product => {
        if (this.productData[product.id]) {
          this.productData[product.id].currentMarketValue = 0
          // 第一周时默认显示计划定投金额，其他周时显示0
          if (this.currentWeek === 1) {
            this.productData[product.id].weeklyInvestment = product.weeklyAmount
          } else {
            this.productData[product.id].weeklyInvestment = 0
          }
        }
      })
    }
  }
}
</script>

<style scoped>
.weekly-calculation {
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.week-controls {
  display: flex;
  gap: 10px;
}

.products-calculation {
  margin-bottom: 20px;
}

.product-calculation-card {
  margin-bottom: 16px;
}

.product-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.product-name {
  font-weight: bold;
  font-size: 16px;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.input-label {
  font-weight: bold;
  color: #303133;
  font-size: 14px;
}

.target-value-display {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
}

.investment-amount-display {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
}

.amount {
  font-size: 16px;
  font-weight: bold;
  color: #303133;
}

.investment-tag {
  margin-left: auto;
}

.product-details {
  margin-top: 16px;
}

.summary-section {
  margin: 20px 0;
  padding: 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.summary-label {
  font-weight: bold;
  color: #495057;
  font-size: 14px;
}

.summary-value {
  font-size: 18px;
  font-weight: bold;
  color: #303133;
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

.actions {
  margin-top: 20px;
  text-align: center;
}

/* 确保表格中的输入框正确显示 */
.el-table .el-input-number {
  width: 100%;
}

.el-table .el-input {
  width: 100%;
}

/* 确保表格单元格有足够的内边距 */
.el-table .cell {
  padding: 8px 12px;
}
</style> 