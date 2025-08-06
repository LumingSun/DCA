<template>
  <div class="investment-plan">
    <el-card>
      <template #header>
        <div class="card-header">
          <span>定投产品配置</span>
          <el-button type="primary" @click="addProduct">
            <el-icon><Plus /></el-icon>
            添加产品
          </el-button>
        </div>
      </template>
      
      <el-table :data="localProducts" style="width: 100%">
        <el-table-column prop="name" label="产品名称" width="160">
          <template #default="{ row }">
            <el-input v-model="row.name" placeholder="产品名称" style="width: 100%" />
          </template>
        </el-table-column>
        
        <el-table-column prop="weeklyAmount" label="周定投金额" width="180">
          <template #default="{ row }">
            <el-input-number 
              v-model="row.weeklyAmount" 
              :min="0" 
              :precision="2"
              placeholder="定投金额"
              style="width: 100%"
            />
          </template>
        </el-table-column>
        
        <el-table-column prop="targetMultiplier" label="目标倍数" width="180">
          <template #default="{ row }">
            <el-input-number 
              v-model="row.targetMultiplier" 
              :min="1" 
              :precision="2"
              placeholder="目标倍数"
              style="width: 100%"
            />
          </template>
        </el-table-column>
        
        <el-table-column prop="color" label="颜色" width="100">
          <template #default="{ row }">
            <el-color-picker v-model="row.color" />
          </template>
        </el-table-column>
        
        <el-table-column label="操作" width="120">
          <template #default="{ $index }">
            <el-button 
              type="danger" 
              size="small" 
              @click="removeProduct($index)"
            >
              删除
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      
      <div class="summary">
        <el-alert
          title="定投策略说明"
          type="info"
          :closable="false"
          show-icon
        >
          <p><strong>市值恒定定投法原理：</strong></p>
          <p>1. 第一周按预设金额定投</p>
          <p>2. 第二周开始，计算目标市值 = 周定投金额 × 目标倍数 × 当前周数</p>
          <p>3. 追加投资金额 = 目标市值 - 当前累计市值</p>
          <p>4. 如果当前市值超过目标市值，则卖出超出部分获利了结</p>
        </el-alert>
      </div>
    </el-card>
  </div>
</template>

<script>
import { Plus } from '@element-plus/icons-vue'

export default {
  name: 'InvestmentPlan',
  components: {
    Plus
  },
  props: {
    products: {
      type: Array,
      required: true
    }
  },
  data() {
    return {
      localProducts: []
    }
  },
  watch: {
    products: {
      handler(newProducts) {
        // 避免循环更新
        if (JSON.stringify(this.localProducts) !== JSON.stringify(newProducts)) {
          this.localProducts = JSON.parse(JSON.stringify(newProducts))
        }
      },
      immediate: true,
      deep: true
    },
    localProducts: {
      handler(newProducts) {
        // 避免循环更新
        if (JSON.stringify(this.products) !== JSON.stringify(newProducts)) {
          this.$emit('update:products', newProducts)
        }
      },
      deep: true
    }
  },
  methods: {
    addProduct() {
      const newProduct = {
        id: Date.now(),
        name: '新产品',
        weeklyAmount: 1000,
        targetMultiplier: 2,
        color: '#409EFF'
      }
      this.localProducts.push(newProduct)
    },
    removeProduct(index) {
      this.localProducts.splice(index, 1)
    }
  }
}
</script>

<style scoped>
.investment-plan {
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.summary {
  margin-top: 20px;
}

.summary p {
  margin: 5px 0;
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