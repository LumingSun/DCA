<template>
  <div id="app">
    <el-container>
      <el-header>
        <h1>市值恒定定投法 - 统计计算程序</h1>
      </el-header>
      
      <el-main>
        <el-tabs v-model="activeTab" type="border-card">
          <el-tab-pane label="定投计划" name="plan">
            <InvestmentPlan 
              :products="products" 
              @update:products="updateProducts"
            />
          </el-tab-pane>
          
          <el-tab-pane label="期度计算" name="weekly">
            <WeeklyCalculation 
              :products="products" 
              :currentWeek="currentWeek"
              :history="investmentHistory"
              @update:currentWeek="updateCurrentWeek"
              @save-week-data="saveWeekData"
            />
          </el-tab-pane>
          
          <el-tab-pane label="历史记录" name="history">
            <InvestmentHistory 
              :history="investmentHistory"
              :currentWeek="currentWeek"
              :products="products"
              @delete-record="deleteRecord"
              @clear-history="clearHistory"
            />
          </el-tab-pane>
          
          <el-tab-pane label="统计分析" name="analysis">
            <InvestmentAnalysis 
              :products="products"
              :history="investmentHistory"
            />
          </el-tab-pane>
        </el-tabs>
      </el-main>
    </el-container>
  </div>
</template>

<script>
import InvestmentPlan from './components/InvestmentPlan.vue'
import WeeklyCalculation from './components/WeeklyCalculation.vue'
import InvestmentHistory from './components/InvestmentHistory.vue'
import InvestmentAnalysis from './components/InvestmentAnalysis.vue'

export default {
  name: 'App',
  components: {
    InvestmentPlan,
    WeeklyCalculation,
    InvestmentHistory,
    InvestmentAnalysis
  },
  data() {
    return {
      activeTab: 'plan',
      currentWeek: 1,
      products: [
        {
          id: 1,
          name: '深成指B',
          weeklyAmount: 1200,
          targetMultiplier: 2, // 目标市值倍数
          color: '#67C23A'
        },
        {
          id: 2,
          name: '医药ETF',
          weeklyAmount: 1200,
          targetMultiplier: 2,
          color: '#409EFF'
        },
        {
          id: 3,
          name: '恒生ETF',
          weeklyAmount: 1200,
          targetMultiplier: 2,
          color: '#E6A23C'
        },
        {
          id: 4,
          name: 'H股ETF',
          weeklyAmount: 1200,
          targetMultiplier: 2,
          color: '#F56C6C'
        },
        {
          id: 5,
          name: '标普ETF',
          weeklyAmount: 400,
          targetMultiplier: 2,
          color: '#909399'
        },
        {
          id: 6,
          name: '华宝油气',
          weeklyAmount: 800,
          targetMultiplier: 2,
          color: '#9C27B0'
        }
      ],
      investmentHistory: []
    }
  },
  methods: {
    updateProducts(newProducts) {
      this.products = newProducts
      this.saveToLocalStorage()
    },
    updateCurrentWeek(week) {
      this.currentWeek = week
      this.saveToLocalStorage()
    },
    saveWeekData(record) {
      // 新的数据格式：包含产品记录的汇总记录
      this.investmentHistory.push(record)
      this.saveToLocalStorage()
    },
    deleteRecord(record) {
      const index = this.investmentHistory.findIndex(item => 
        item.week === record.week && item.date === record.date
      )
      if (index !== -1) {
        this.investmentHistory.splice(index, 1)
        this.saveToLocalStorage()
      }
    },
    clearHistory() {
      this.investmentHistory = []
      this.saveToLocalStorage()
    },
    saveToLocalStorage() {
      const data = {
        products: this.products,
        currentWeek: this.currentWeek,
        history: this.investmentHistory
      }
      localStorage.setItem('investmentData', JSON.stringify(data))
    },
    loadFromLocalStorage() {
      const data = localStorage.getItem('investmentData')
      if (data) {
        const parsed = JSON.parse(data)
        this.products = parsed.products || this.products
        this.currentWeek = parsed.currentWeek || 1
        this.investmentHistory = parsed.history || []
      } else {
      }
    }
  },
  mounted() {
    this.loadFromLocalStorage()
  },
  beforeUnmount() {
    // 清理定时器
    if (this.calculationTimer) {
      clearTimeout(this.calculationTimer)
    }
  }
}
</script>

<style>
#app {
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

.el-header {
  background-color: #409EFF;
  color: white;
  text-align: center;
  line-height: 80px;
  font-size: 22px;
  font-weight: bold;
  padding: 0 20px;
  height: 80px !important;
  display: flex;
  align-items: center;
  justify-content: center;
}

.el-main {
  padding: 20px;
  background-color: #f5f7fa;
  min-height: calc(100vh - 80px);
}

.el-tabs {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}
</style> 