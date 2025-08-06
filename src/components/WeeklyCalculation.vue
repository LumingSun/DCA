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
      
      <!-- 总市值输入区域 -->
      <div class="total-market-value-section">
        <el-alert 
          v-if="currentWeek === 1" 
          title="第一周提示" 
          type="info" 
          description="第一周时，如果还没有开始投资，当前总市值可以填写0"
          :closable="false"
          style="margin-bottom: 16px"
        />
        <el-row :gutter="20">
          <el-col :span="12">
            <div class="input-group">
              <label class="input-label">当前总市值：</label>
              <el-input-number 
                v-model="totalMarketValue" 
                :min="0" 
                :precision="2"
                :placeholder="currentWeek === 1 ? '第一周可填写0' : '请输入当前总市值'"
                style="width: 100%"
                @change="calculateWeeklyInvestment"
              />
            </div>
          </el-col>
          <el-col :span="12">
            <div class="input-group">
              <label class="input-label">本周应定投金额：</label>
              <div class="investment-amount-display">
                <span class="amount">{{ formatMoney(weeklyInvestmentAmount) }}</span>
                <el-tag :type="getInvestmentType(weeklyInvestmentAmount)" class="investment-tag">
                  {{ getInvestmentStatus(weeklyInvestmentAmount) }}
                </el-tag>
              </div>
            </div>
          </el-col>
        </el-row>
        
        <!-- 累计投资信息 -->
        <el-row :gutter="20" style="margin-top: 16px;">
          <el-col :span="12">
            <div class="input-group">
              <label class="input-label">累计投资金额：</label>
              <div class="total-invested-display">
                <span class="amount">{{ formatMoney(totalInvestedAmount) }}</span>
              </div>
            </div>
          </el-col>
          <el-col :span="12">
            <div class="input-group">
              <label class="input-label">投资收益率：</label>
              <div class="profit-rate-display">
                <span class="amount" :class="{ 'profit': profitRate > 0, 'loss': profitRate < 0 }">
                  {{ formatPercentage(profitRate) }}
                </span>
              </div>
            </div>
          </el-col>
        </el-row>
      </div>

      <!-- 计算详情 -->
      <div class="calculation-details" v-if="totalMarketValue >= 0">
        <el-divider content-position="left">计算详情</el-divider>
        
        <el-descriptions :column="2" border>
          <el-descriptions-item label="当前周数">
            第{{ currentWeek }}周
          </el-descriptions-item>
          <el-descriptions-item label="计划周定投金额">
            {{ formatMoney(plannedWeeklyAmount) }}
          </el-descriptions-item>
          <el-descriptions-item label="目标市值">
            {{ formatMoney(targetMarketValue) }}
          </el-descriptions-item>
          <el-descriptions-item label="当前市值">
            {{ formatMoney(totalMarketValue) }}
          </el-descriptions-item>
          <el-descriptions-item label="市值差额">
            <span :class="{ 'profit': marketValueDifference > 0, 'loss': marketValueDifference < 0 }">
              {{ formatMoney(marketValueDifference) }}
            </span>
          </el-descriptions-item>
          <el-descriptions-item label="本周定投金额">
            <span :class="{ 'profit': weeklyInvestmentAmount < 0, 'investment': weeklyInvestmentAmount > 0 }">
              {{ formatMoney(weeklyInvestmentAmount) }}
            </span>
          </el-descriptions-item>
        </el-descriptions>
      </div>


      
      <div class="actions">
                 <el-button type="success" @click="saveCurrentRecord" :disabled="totalMarketValue < 0">
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
      totalMarketValue: 0,
      weeklyInvestmentAmount: 0
    }
  },
  computed: {
    plannedWeeklyAmount() {
      return this.products.reduce((sum, product) => sum + product.weeklyAmount, 0)
    },
    targetMarketValue() {
      // 目标市值 = 周定投金额 × 目标倍数 × 当前周数
      return this.products.reduce((sum, product) => {
        return sum + (product.weeklyAmount * product.targetMultiplier * this.currentWeek)
      }, 0)
    },
    marketValueDifference() {
      return this.targetMarketValue - this.totalMarketValue
    },
    totalInvestedAmount() {
      // 累计投资金额 = 当前周数 × 计划周定投金额
      return this.currentWeek * this.plannedWeeklyAmount
    },
    profitRate() {
      if (this.totalInvestedAmount === 0) return 0
      return ((this.totalMarketValue - this.totalInvestedAmount) / this.totalInvestedAmount) * 100
    }
  },
  watch: {
    currentWeek: {
      handler() {
        this.initializeWeekData()
      },
      immediate: true
    }
  },
  methods: {
    calculateWeeklyInvestment() {
      // 第一周时，如果当前市值为0，则按计划金额定投
      if (this.currentWeek === 1 && this.totalMarketValue <= 0) {
        this.weeklyInvestmentAmount = this.plannedWeeklyAmount
        return
      }
      
      // 其他情况：计算本周应定投金额
      // 如果当前市值低于目标市值，需要追加投资
      // 如果当前市值高于目标市值，可以获利了结
      this.weeklyInvestmentAmount = this.marketValueDifference
    },
    formatMoney(amount) {
      return new Intl.NumberFormat('zh-CN', {
        style: 'currency',
        currency: 'CNY'
      }).format(amount)
    },
    formatDate(dateString) {
      return new Date(dateString).toLocaleString('zh-CN')
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
      // 计算保存后的总市值
      const newTotalMarketValue = this.totalMarketValue + this.weeklyInvestmentAmount
      
      const record = {
        date: new Date().toISOString(),
        totalMarketValue: newTotalMarketValue,
        weeklyInvestment: this.weeklyInvestmentAmount,
        status: this.getInvestmentStatus(this.weeklyInvestmentAmount)
      }
      
      // 只发送记录到父组件，不保存到本地
      this.$emit('save-week-data', record)
      
      // 更新当前总市值为保存后的值
      this.totalMarketValue = newTotalMarketValue
      
      if (this.weeklyInvestmentAmount > 0) {
        this.$message.success(`记录已保存，总市值已更新为 ${this.formatMoney(this.totalMarketValue)}`)
      } else {
        this.$message.success('记录已保存')
      }
    },

    initializeWeekData() {
      // 重置当前总市值为0，让用户重新输入
      this.totalMarketValue = 0
      
      // 重新计算本周定投金额
      this.calculateWeeklyInvestment()
    },
    resetCalculation() {
      this.totalMarketValue = 0
      // 第一周时默认显示计划定投金额，其他周时显示0
      if (this.currentWeek === 1) {
        this.weeklyInvestmentAmount = this.plannedWeeklyAmount
      } else {
        this.weeklyInvestmentAmount = 0
      }
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

.total-market-value-section {
  margin-bottom: 20px;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.input-label {
  font-weight: bold;
  color: #303133;
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
  font-size: 18px;
  font-weight: bold;
  color: #303133;
}

.investment-tag {
  margin-left: auto;
}

.total-invested-display,
.profit-rate-display {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
}

.calculation-details {
  margin: 20px 0;
}

.history-section {
  margin: 20px 0;
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
</style> 