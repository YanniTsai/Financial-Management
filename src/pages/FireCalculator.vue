<template>
    <div class="q-pa-md bg-grey-10 text-grey-1">
        <h3>FIRE 🔥🔥🔥</h3>
        <div class="grid q-mb-md">
            <div class="input-field flex column q-pa-md" v-for="column in inputColumns" :key="column.field">
                <span class="q-mb-sm">{{ column.label }}</span>
                <input type="number" v-model.number="inputs[column.field]" min="0" />
            </div>
        </div>

        <div class="cards">
            <div class="card">
                <h3>今天要的 FIRE 數字</h3>
                <p class="num">{{ fmtCurrency(fiNow) }}</p>
                <p class="hint">= 年支出 ÷ (FIRE 百分比)</p>
            </div>

            <div class="card">
                <h3>{{ yearsToFire !== null ? yearsToFire + ' 年後（' + fireYear + ' 年）FIRE 數字' : 'FIRE 目標金額' }}</h3>
                <p class="num">
                    {{ fiAtFireYear !== null ? fmtCurrency(fiAtFireYear) : '無法達成' }}
                </p>
                <p class="hint">年支出 × (1+通膨)^年數 ÷ 安全提領率</p>
            </div>

            <div class="card">
                <h3>預計達成 FIRE 年份</h3>
                <p class="num">
                    <template v-if="yearsToFire !== null">
                    {{ fireYear }} 年（{{ yearsToFire }} 年後）
                    </template>
                    <template v-else>
                    無法達成
                    </template>
                </p>
                <p class="hint">資產首次超過當年 FIRE 目標的年份</p>
            </div>
        </div>

        <details open>
            <summary>{{ yearsToFire }} 年資產走勢（年末）</summary>
            <table>
            <thead>
                <tr>
                <th>年份</th>
                <th>年度投入</th>
                <th>年末資產</th>
                <th>當年 FIRE 目標（含通膨）</th>
                <th>差距</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="row in projection" :key="row.year">
                <td>{{ row.year }}</td>
                <td>{{ fmtCurrency(row.contribution) }}</td>
                <td>{{ fmtCurrency(row.endBalance) }}</td>
                <td>{{ fmtCurrency(row.fiTargetThisYear) }}</td>
                <td :class="{'ok': row.gap <= 0, 'warn': row.gap > 0}">
                    {{ fmtCurrency(row.gap) }}
                </td>
                </tr>
            </tbody>
            </table>
        </details>
    </div>
</template>

<script lang="ts" setup>
import { computed,  ref} from 'vue'

type InputField = keyof typeof inputs.value

// ======== 用戶輸入資料 ========
const inputs = ref({
  currentAssets: 300000,
  annualIncome: 500000,
  annualExpense: 400000,
  dividendReturnRatePct: 7,  // 高股息報酬率
  growthReturnRatePct: 10,   // 成長股報酬率
  growthAllocationPct: 60,   // 成長股比例 %
  inflationRatePct: 2,       // 通膨率
  fireRatePct: 4,            // FIRE 提領率 %
})

// watch(() => inputs.value, (newValue) => {
//     setTimeout(() => localStorage.setItem("fireCalculatorInputs", JSON.stringify(newValue)), 5000)
//   },
//   { deep: true }
// )

const inputColumns: {field: InputField, label: string}[] = [
    {field: 'currentAssets', label: '現有資產'},
    {field: 'annualIncome', label: '年收入'},
    {field: 'annualExpense', label: '年支出'},
    {field: 'dividendReturnRatePct', label: '高股息報酬率'},
    {field: 'growthReturnRatePct', label: '成長股報酬率'},
    {field: 'growthAllocationPct', label: '成長股比例 %'},
    {field: 'inflationRatePct', label: '通膨率'},
    {field: 'fireRatePct', label: 'FIRE 提領率 %'},
]

// ======== Helper: 將百分比轉為小數 ========
const toRate = (pct: number) => (pct || 0) / 100
// 年投入金額（年收入 - 年支出）
const annualContribution = computed(() => 
  Math.max(0, inputs.value.annualIncome - inputs.value.annualExpense)
)

// 總投資報酬率（加權平均）
const totalReturnRate = computed(() => {
  const dividendRate = toRate(inputs.value.dividendReturnRatePct)
  const growthRate = toRate(inputs.value.growthReturnRatePct)
  const growthAlloc = toRate(inputs.value.growthAllocationPct)
  const dividendAlloc = 1 - growthAlloc
  console.log(dividendRate * dividendAlloc + growthRate * growthAlloc)
  return dividendRate * dividendAlloc + growthRate * growthAlloc
})

// 今天的 FIRE 數字
const fiNow = computed(() => {
  const swr = Math.max(toRate(inputs.value.fireRatePct), 0.000001)
  return inputs.value.annualExpense / swr
})

// 資產模擬
const projection = computed(() => {
  const r = totalReturnRate.value //投資報酬率
  const infl = toRate(inputs.value.inflationRatePct) // 通膨率
  const swr = Math.max(toRate(inputs.value.fireRatePct), 0.000001) // 安全提領率
  let balance = inputs.value.currentAssets || 0
  const rows = []
  const maxYears = 100
  for (let y = 1; y <= maxYears; y++) {
    balance = balance * (1 + r) + annualContribution.value
    const futureExpense = inputs.value.annualExpense * Math.pow(1 + infl, y) // 年支出（含通膨）
    const fiTargetThisYear = futureExpense / swr // 當年 FIRE 金額
    const gap = fiTargetThisYear - balance // FIRE 差距金額
    rows.push({
      year: y,
      contribution: annualContribution.value,
      endBalance: balance,
      fiTargetThisYear,
      gap,
    })
    if (gap <= 0) break
  }
  return rows
})

// 達成 FIRE 所需年數
const yearsToFire = computed(() => {
  const result = projection.value.find(p => p.gap <= 0)
  return result ? result.year : null
})

// 達成年份
const fireYear = computed(() => 
  yearsToFire.value !== null ? new Date().getFullYear() + yearsToFire.value : null
)

// 達成 FIRE 時的金額（含通膨）
const fiAtFireYear = computed(() => {
  if (yearsToFire.value === null) return null
  const lastData = projection.value[projection.value.length - 1]
  return lastData ? lastData.fiTargetThisYear : null
})

// 格式化貨幣
const fmtCurrency = (n: number) =>
  new Intl.NumberFormat("zh-TW", {
    style: 'currency',
    currency: 'TWD',
    maximumFractionDigits: 0,
  }).format(isFinite(n) ? n : 0)
</script>

<style lang="scss" scoped>
.grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;

  .input-field{
    border-radius: 10px;
    border: 1px solid #BDBDBD;
  }
}

input[type="number"] {
  width: 100%; 
  padding: 10px 12px; 
  border-radius: 8px; 
  border: 1px solid #475569;
  background: #0b1020; 
  color: #e5e7eb; 
  outline: none;

  &:focus{
    border-color: #60a5fa; 
  }
}

.cards {
  display: grid; 
  grid-template-columns: repeat(3, 1fr); 
  gap: 12px; 
  margin: 16px 0 8px;
}

.card {
  padding: 12px;
  border-radius: 10px;
  border: 1px solid #BDBDBD;
}

.card h3 { 
    margin: 0 0 6px; 
    font-size: 16px; 
    color: #cbd5e1; 
}

.card .num { 
    margin: 6px 0 0; 
    font-size: 22px; 
    font-weight: 700; 
}

.card .hint { 
    margin: 6px 0 0; 
    font-size: 12px; 
    color: var(--muted); 
}

details { 
    margin-top: 12px; 
}

summary { 
    cursor: pointer; 
    padding: 8px 0; 
    font-weight: 600; 
}

table { 
    width: 100%; 
    border-collapse: collapse; 
    overflow: hidden; 
    border-radius: 12px; 
}

thead th {
  text-align: left; 
  padding: 10px; 
  background: #0b1222; 
  border-bottom: 1px solid var(--ring); 
  font-size: 13px; 
  color: #cbd5e1;
}

tbody td { padding: 10px; border-bottom: 1px solid #1f2937; font-size: 14px; }
.ok { 
    color: #16a34a; 
    font-weight: 700; 
}

.warn { 
    color: #ef4444; 
    font-weight: 700; 
}

.note { 
    color: #94a3b8; 
    font-size: 12px; 
    margin-top: 12px; 
    line-height: 1.6; 
}

@media (max-width: 920px) {
  .grid { grid-template-columns: 1fr 1fr; }
  .cards { grid-template-columns: 1fr; }
}

</style>