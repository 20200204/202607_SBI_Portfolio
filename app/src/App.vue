<script setup lang="ts">
import { ref, computed, nextTick, watch, onMounted } from 'vue'
import {
  Chart,
  DoughnutController,
  ArcElement,
  Tooltip,
  Legend,
  type ChartConfiguration,
} from 'chart.js'

Chart.register(DoughnutController, ArcElement, Tooltip, Legend)

// ── Types ──────────────────────────────────────────────────────────────
interface AssetItem {
  name: string
  category: string
  evaluationAmount: number
  id?: string
  isManual?: boolean
}

type Cat1Option = '円建' | 'ドル建'
type Cat2Option = '日本株式' | '海外株式' | '貴金属'
type ChartMode = 'asset' | 'cat1' | 'cat2'
type PageMode = 'view' | 'edit'

interface CategoryEntry {
  cat1: Cat1Option | ''
  cat2: Cat2Option | ''
}

// ── Constants ──────────────────────────────────────────────────────────
const CAT1_OPTIONS: Cat1Option[] = ['円建', 'ドル建']
const CAT2_OPTIONS: Cat2Option[] = ['日本株式', '海外株式', '貴金属']
const UNSET_LABEL = '未設定'
const STORAGE_KEY = 'portfolio_categories_v1'

const CAT1_COLORS: Record<string, string> = {
  '円建': '#4e79a7',
  'ドル建': '#f28e2b',
  [UNSET_LABEL]: '#d1d5db',
}

const CAT2_COLORS: Record<string, string> = {
  '日本株式': '#e15759',
  '海外株式': '#4e79a7',
  '貴金属': '#edc948',
  [UNSET_LABEL]: '#d1d5db',
}

// ── State ──────────────────────────────────────────────────────────────
const assets = ref<AssetItem[]>([])
const errorMessage = ref('')
const fileName = ref('')
const canvasRef = ref<HTMLCanvasElement | null>(null)
const chartMode = ref<ChartMode>('asset')
const pageMode = ref<PageMode>('view')
const categoryMap = ref<Record<string, CategoryEntry>>({})
let chartInstance: Chart | null = null
const manualAssets = ref<AssetItem[]>([])
const showAddForm = ref(false)
const newName = ref('')
const newCategory = ref('')
const newAmount = ref('')
const addFormError = ref('')

// ── Computed ───────────────────────────────────────────────────────────
const allAssets = computed((): AssetItem[] => [...assets.value, ...manualAssets.value])

const totalAmount = computed(() =>
  allAssets.value.reduce((sum, a) => sum + a.evaluationAmount, 0),
)

// ── Category helpers ───────────────────────────────────────────────────
function assetKey(a: AssetItem): string {
  return `${a.name}__${a.category}`
}

function ensureCatEntry(a: AssetItem) {
  const key = assetKey(a)
  if (!categoryMap.value[key]) {
    categoryMap.value[key] = { cat1: '', cat2: '' }
  }
}

function saveCategories() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(categoryMap.value))
  } catch { /* ignore storage errors */ }
}

function loadCategories() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) categoryMap.value = JSON.parse(raw) as Record<string, CategoryEntry>
  } catch { /* ignore */ }
}

loadCategories()

// ── Manual assets ──────────────────────────────────────────────────────
const MANUAL_KEY = 'portfolio_manual_assets_v1'

function saveManualAssets() {
  try {
    localStorage.setItem(MANUAL_KEY, JSON.stringify(manualAssets.value))
  } catch { /* ignore */ }
}

function loadManualAssets() {
  try {
    const raw = localStorage.getItem(MANUAL_KEY)
    if (raw) {
      const loaded = JSON.parse(raw) as AssetItem[]
      manualAssets.value = loaded
      for (const a of loaded) ensureCatEntry(a)
    }
  } catch { /* ignore */ }
}

loadManualAssets()

function addAsset() {
  const name = newName.value.trim()
  const category = newCategory.value.trim() || 'その他'
  const amount = parseFloat(newAmount.value)

  if (!name) { addFormError.value = '銘柄名を入力してください'; return }
  if (isNaN(amount) || amount <= 0) { addFormError.value = '評価額に正の数値を入力してください'; return }

  const item: AssetItem = {
    id: `m_${Date.now()}_${Math.random().toString(36).slice(2, 7)}`,
    name,
    category,
    evaluationAmount: amount,
    isManual: true,
  }
  manualAssets.value = [...manualAssets.value, item]
  ensureCatEntry(item)
  saveManualAssets()
  newName.value = ''
  newCategory.value = ''
  newAmount.value = ''
  addFormError.value = ''
  showAddForm.value = false
  nextTick(() => renderChart())
}

function removeManualAsset(id: string) {
  manualAssets.value = manualAssets.value.filter((a) => a.id !== id)
  saveManualAssets()
  nextTick(() => renderChart())
}

const SECTION_MAP: [string, string][] = [
  ['株式（現物/特定預り）', '株式（特定預り）'],
  ['投資信託（金額/特定預り）', '投資信託（特定預り）'],
  ['投資信託（金額/NISA預り（成長投資枠））', 'NISA（成長投資枠）'],
  ['投資信託（金額/NISA預り（つみたて投資枠））', 'NISA（つみたて投資枠）'],
  ['投資信託（金額/旧NISA預り）', '旧NISA預り'],
]

function parseCSVLine(line: string): string[] {
  const fields: string[] = []
  let current = ''
  let inQuote = false
  for (const ch of line) {
    if (ch === '"') {
      inQuote = !inQuote
    } else if (ch === ',' && !inQuote) {
      fields.push(current)
      current = ''
    } else {
      current += ch
    }
  }
  if (current) fields.push(current)
  return fields
}

function parsePortfolioCSV(text: string): AssetItem[] {
  const lines = text.split(/\r?\n/).map((l) => l.trim()).filter(Boolean)
  const result: AssetItem[] = []
  let currentCategory = ''

  for (const line of lines) {
    for (const [key, label] of SECTION_MAP) {
      if (line.includes(key)) {
        currentCategory = label
        break
      }
    }
    const fields = parseCSVLine(line)
    if (fields.length >= 10 && fields[1] === '----/--/--') {
      const name = fields[0]
      const evalAmount = parseFloat(fields[9])
      if (name && !isNaN(evalAmount)) {
        result.push({ name, category: currentCategory, evaluationAmount: evalAmount })
      }
    }
  }
  return result
}

function readFileAsText(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const tryRead = (encoding: string) => {
      const reader = new FileReader()
      reader.onload = (e) => {
        const text = e.target?.result as string
        if (encoding === 'utf-8' && text.includes('\uFFFD')) {
          tryRead('shift_jis')
        } else {
          resolve(text)
        }
      }
      reader.onerror = () => reject(new Error('ファイルの読み込みに失敗しました'))
      reader.readAsText(file, encoding)
    }
    tryRead('utf-8')
  })
}

const PALETTE = [
  '#4e79a7', '#f28e2b', '#e15759', '#76b7b2', '#59a14f',
  '#edc948', '#b07aa1', '#ff9da7', '#9c755f', '#bab0ac',
  '#86bcb6', '#f1ce63', '#d37295', '#a0cbe8', '#ffbe7d',
  '#8cd17d', '#b6992d',
]

// ── Chart data builder ─────────────────────────────────────────────────
interface ChartItem {
  label: string
  fullLabel: string
  value: number
  color: string
}

function buildChartItems(): ChartItem[] {
  const truncate = (s: string, max = 16) => (s.length > max ? s.slice(0, max) + '…' : s)

  if (chartMode.value === 'asset') {
    return allAssets.value.map((a, i) => ({
      label: truncate(a.name),
      fullLabel: `${a.name}（${a.category}）`,
      value: a.evaluationAmount,
      color: PALETTE[i % PALETTE.length],
    }))
  }

  const grouped: Record<string, number> = {}
  for (const a of allAssets.value) {
    const entry = categoryMap.value[assetKey(a)]
    const key =
      chartMode.value === 'cat1'
        ? (entry?.cat1 || UNSET_LABEL)
        : (entry?.cat2 || UNSET_LABEL)
    grouped[key] = (grouped[key] || 0) + a.evaluationAmount
  }

  const colorMap = chartMode.value === 'cat1' ? CAT1_COLORS : CAT2_COLORS
  return Object.entries(grouped)
    .sort((a, b) => b[1] - a[1])
    .map(([label, value]) => ({
      label,
      fullLabel: label,
      value,
      color: colorMap[label] ?? '#9ca3af',
    }))
}

function renderChart() {
  if (!canvasRef.value || allAssets.value.length === 0) return
  if (chartInstance) {
    chartInstance.destroy()
    chartInstance = null
  }

  const items = buildChartItems()
  const total = totalAmount.value

  const config: ChartConfiguration<'doughnut'> = {
    type: 'doughnut',
    data: {
      labels: items.map((i) => i.label),
      datasets: [
        {
          data: items.map((i) => i.value),
          backgroundColor: items.map((i) => i.color),
          borderColor: '#ffffff',
          borderWidth: 2,
          hoverOffset: 10,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: true,
      plugins: {
        legend: {
          position: 'bottom',
          labels: {
            font: { size: 11 },
            padding: 10,
            boxWidth: 14,
            color: '#374151',
          },
        },
        tooltip: {
          callbacks: {
            title: (ctxItems) => items[ctxItems[0].dataIndex].fullLabel,
            label: (ctx) => {
              const val = ctx.parsed as number
              const pct = ((val / total) * 100).toFixed(1)
              return `  ¥${val.toLocaleString()}  (${pct}%)`
            },
          },
        },
      },
    },
  }

  chartInstance = new Chart(canvasRef.value, config)
}

// ── Watchers ───────────────────────────────────────────────────────────
watch(chartMode, async () => {
  if (pageMode.value === 'view') {
    await nextTick()
    renderChart()
  }
})

// ── Page mode ──────────────────────────────────────────────────────────
function enterEditMode() {
  showAddForm.value = false
  for (const a of allAssets.value) ensureCatEntry(a)
  pageMode.value = 'edit'
}

async function exitEditMode() {
  saveCategories()
  pageMode.value = 'view'
  await nextTick()
  renderChart()
}

async function onFileChange(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  errorMessage.value = ''
  fileName.value = file.name

  try {
    const text = await readFileAsText(file)
    const parsed = parsePortfolioCSV(text)
    if (parsed.length === 0) {
      errorMessage.value =
        'データが見つかりませんでした。CSVフォーマットを確認してください。'
      assets.value = []
    } else {
      assets.value = parsed
      for (const a of [...parsed, ...manualAssets.value]) ensureCatEntry(a)
      pageMode.value = 'view'
      await nextTick()
      renderChart()
    }
  } catch (err) {
    errorMessage.value = err instanceof Error ? err.message : '不明なエラーが発生しました'
  }

  input.value = ''
}

onMounted(async () => {
  if (allAssets.value.length > 0) {
    await nextTick()
    renderChart()
  }
})
</script>

<template>
  <div class="app">
    <header class="header">
      <h1>資産ポートフォリオ</h1>
      <p class="subtitle">SBI証券ポートフォリオCSVを読み込んで資産構成を可視化</p>
    </header>

    <main class="main">
      <div class="upload-card">
        <label class="upload-btn">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="20"
            height="20"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
            <polyline points="17 8 12 3 7 8" />
            <line x1="12" y1="3" x2="12" y2="15" />
          </svg>
          <span>CSVファイルをアップロード</span>
          <input type="file" accept=".csv" @change="onFileChange" />
        </label>
        <p v-if="fileName" class="file-name">{{ fileName }}</p>
        <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
      </div>

      <template v-if="allAssets.length > 0">
        <div class="summary-row">
          <div class="summary-card">
            <div class="summary-label">総評価額</div>
            <div class="summary-value">¥{{ totalAmount.toLocaleString() }}</div>
          </div>
          <div class="summary-card">
            <div class="summary-label">銘柄数</div>
            <div class="summary-value">{{ allAssets.length }} 銘柄</div>
          </div>
        </div>

        <!-- ── Toolbar ── -->
        <div class="toolbar">
          <div class="tabs" role="tablist">
            <button
              role="tab"
              :class="['tab', chartMode === 'asset' ? 'active' : '']"
              @click="chartMode = 'asset'"
            >銘柄毎</button>
            <button
              role="tab"
              :class="['tab', chartMode === 'cat1' ? 'active' : '']"
              @click="chartMode = 'cat1'"
            >カテゴリ１別</button>
            <button
              role="tab"
              :class="['tab', chartMode === 'cat2' ? 'active' : '']"
              @click="chartMode = 'cat2'"
            >カテゴリ２別</button>
          </div>
          <div class="toolbar-actions">
            <template v-if="pageMode === 'view'">
              <button class="action-btn" @click="showAddForm = !showAddForm">
                <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
                銘柄を追加
              </button>
              <button class="action-btn" @click="enterEditMode">
                <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>
                カテゴリ編集
              </button>
            </template>
            <button v-else class="action-btn save" @click="exitEditMode">
              <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"/></svg>
              保存して戻る
            </button>
          </div>
        </div>

        <!-- ── Add form ── -->
        <div v-if="showAddForm && pageMode === 'view'" class="add-form-card">
          <h2>銘柄を追加</h2>
          <div class="add-form-grid">
            <div class="form-row">
              <label class="form-label">銘柄名 <span class="required">*</span></label>
              <input v-model="newName" type="text" placeholder="例: 現金、不動産、金" class="form-input" @keyup.enter="addAsset" />
            </div>
            <div class="form-row">
              <label class="form-label">預り区分</label>
              <input v-model="newCategory" type="text" placeholder="例: 特定預り、NISA、その他" class="form-input" />
            </div>
            <div class="form-row">
              <label class="form-label">評価額（円） <span class="required">*</span></label>
              <input v-model="newAmount" type="number" min="0" step="1" placeholder="例: 500000" class="form-input" @keyup.enter="addAsset" />
            </div>
          </div>
          <p v-if="addFormError" class="error form-error">{{ addFormError }}</p>
          <div class="add-form-actions">
            <button class="action-btn" @click="showAddForm = false; addFormError = ''">キャンセル</button>
            <button class="action-btn save" @click="addAsset">追加</button>
          </div>
        </div>

        <!-- ── View mode ── -->
        <template v-if="pageMode === 'view'">
          <div class="chart-card">
            <h2>資産構成（評価額）</h2>
            <div class="chart-container">
              <canvas ref="canvasRef"></canvas>
            </div>
          </div>

          <div class="table-card">
            <h2>銘柄一覧</h2>
            <div class="table-wrapper">
              <table>
                <thead>
                  <tr>
                    <th class="col-color"></th>
                    <th>銘柄名</th>
                    <th>預り区分</th>
                    <th>カテゴリ１</th>
                    <th>カテゴリ２</th>
                    <th class="num">評価額</th>
                    <th class="num">割合</th>
                    <th class="col-del"></th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(asset, i) in allAssets" :key="asset.id ?? i">
                    <td class="col-color">
                      <span
                        class="dot"
                        :style="{ backgroundColor: PALETTE[i % PALETTE.length] }"
                      ></span>
                    </td>
                    <td class="name">
                      {{ asset.name }}
                      <span v-if="asset.isManual" class="manual-badge">手動</span>
                    </td>
                    <td><span class="badge">{{ asset.category }}</span></td>
                    <td>
                      <span
                        v-if="categoryMap[assetKey(asset)]?.cat1"
                        class="cat-tag"
                        :style="{ background: CAT1_COLORS[categoryMap[assetKey(asset)].cat1] + '22', color: CAT1_COLORS[categoryMap[assetKey(asset)].cat1] }"
                      >{{ categoryMap[assetKey(asset)].cat1 }}</span>
                      <span v-else class="cat-tag unset">未設定</span>
                    </td>
                    <td>
                      <span
                        v-if="categoryMap[assetKey(asset)]?.cat2"
                        class="cat-tag"
                        :style="{ background: CAT2_COLORS[categoryMap[assetKey(asset)].cat2] + '22', color: CAT2_COLORS[categoryMap[assetKey(asset)].cat2] }"
                      >{{ categoryMap[assetKey(asset)].cat2 }}</span>
                      <span v-else class="cat-tag unset">未設定</span>
                    </td>
                    <td class="num">¥{{ asset.evaluationAmount.toLocaleString() }}</td>
                    <td class="num pct">
                      {{ ((asset.evaluationAmount / totalAmount) * 100).toFixed(1) }}%
                    </td>
                    <td class="col-del">
                      <button
                        v-if="asset.isManual"
                        class="del-btn"
                        title="削除"
                        @click="removeManualAsset(asset.id!)"
                      >
                        <svg xmlns="http://www.w3.org/2000/svg" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14H6L5 6"/><path d="M10 11v6"/><path d="M14 11v6"/><path d="M9 6V4h6v2"/></svg>
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </template>

        <!-- ── Edit mode ── -->
        <template v-else>
          <div class="edit-card">
            <h2>カテゴリ編集</h2>
            <p class="edit-hint">各銘柄のカテゴリを設定してください。「保存して戻る」で変更が反映されます。</p>
            <div class="table-wrapper">
              <table>
                <thead>
                  <tr>
                    <th>銘柄名</th>
                    <th>預り区分</th>
                    <th>カテゴリ１（円建・ドル建）</th>
                    <th>カテゴリ２（資産クラス）</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(asset, i) in allAssets" :key="asset.id ?? i">
                    <td class="name">
                      {{ asset.name }}
                      <span v-if="asset.isManual" class="manual-badge">手動</span>
                    </td>
                    <td><span class="badge">{{ asset.category }}</span></td>
                    <td>
                      <select v-model="categoryMap[assetKey(asset)].cat1" class="cat-select">
                        <option value="">未設定</option>
                        <option v-for="opt in CAT1_OPTIONS" :key="opt" :value="opt">{{ opt }}</option>
                      </select>
                    </td>
                    <td>
                      <select v-model="categoryMap[assetKey(asset)].cat2" class="cat-select">
                        <option value="">未設定</option>
                        <option v-for="opt in CAT2_OPTIONS" :key="opt" :value="opt">{{ opt }}</option>
                      </select>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div class="edit-actions">
              <button class="action-btn save large" @click="exitEditMode">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"/></svg>
                保存して戻る
              </button>
            </div>
          </div>
        </template>
      </template>
    </main>
  </div>
</template>

<style scoped>
*,
*::before,
*::after {
  box-sizing: border-box;
}

.app {
  min-height: 100vh;
  background: #f3f4f6;
  font-family: 'Hiragino Kaku Gothic ProN', 'Noto Sans JP', sans-serif;
  color: #111827;
}

.header {
  background: #1e3a5f;
  color: #fff;
  padding: 2rem 1.5rem 1.5rem;
  text-align: center;
}

.header h1 {
  margin: 0;
  font-size: 1.75rem;
  font-weight: 700;
  letter-spacing: 0.04em;
}

.subtitle {
  margin: 0.5rem 0 0;
  font-size: 0.875rem;
  opacity: 0.75;
}

.main {
  max-width: 900px;
  margin: 0 auto;
  padding: 1.5rem 1rem 3rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

/* ── Upload card ── */
.upload-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
}

.upload-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.65rem 1.5rem;
  background: #1e3a5f;
  color: #fff;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.upload-btn:hover {
  background: #2a5082;
}

.upload-btn input {
  display: none;
}

.file-name {
  font-size: 0.85rem;
  color: #6b7280;
  margin: 0;
}

.error {
  color: #dc2626;
  font-size: 0.875rem;
  margin: 0;
}

/* ── Summary row ── */
.summary-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.summary-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 1.25rem 1.5rem;
  text-align: center;
}

.summary-label {
  font-size: 0.8rem;
  color: #6b7280;
  margin-bottom: 0.4rem;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.6rem;
  font-weight: 700;
  color: #1e3a5f;
  letter-spacing: -0.02em;
}

/* ── Chart card ── */
.chart-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 1.5rem;
}

.chart-card h2,
.table-card h2 {
  margin: 0 0 1.25rem;
  font-size: 1rem;
  font-weight: 700;
  color: #374151;
  border-left: 4px solid #1e3a5f;
  padding-left: 0.6rem;
}

.chart-container {
  max-width: 620px;
  margin: 0 auto;
}

/* ── Table card ── */
.table-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 1.5rem;
}

.table-wrapper {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

thead th {
  background: #f9fafb;
  color: #6b7280;
  font-weight: 600;
  padding: 0.6rem 0.75rem;
  text-align: left;
  border-bottom: 1px solid #e5e7eb;
  white-space: nowrap;
}

tbody tr {
  border-bottom: 1px solid #f3f4f6;
  transition: background 0.1s;
}

tbody tr:last-child {
  border-bottom: none;
}

tbody tr:hover {
  background: #f9fafb;
}

tbody td {
  padding: 0.6rem 0.75rem;
  vertical-align: middle;
}

.col-color {
  width: 28px;
  padding-right: 0;
}

.dot {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
}

.name {
  max-width: 260px;
  word-break: break-all;
}

.badge {
  display: inline-block;
  background: #eff6ff;
  color: #1e40af;
  border-radius: 4px;
  padding: 0.15rem 0.45rem;
  font-size: 0.75rem;
  white-space: nowrap;
}

.num {
  text-align: right;
  white-space: nowrap;
  font-variant-numeric: tabular-nums;
}

.pct {
  color: #1e3a5f;
  font-weight: 600;
}

/* ── Toolbar ── */
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.tabs {
  display: flex;
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 4px;
  gap: 2px;
}

.tab {
  border: none;
  background: transparent;
  color: #6b7280;
  font-size: 0.875rem;
  font-weight: 500;
  padding: 0.45rem 1rem;
  border-radius: 7px;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
  font-family: inherit;
}

.tab:hover {
  background: #f3f4f6;
  color: #374151;
}

.tab.active {
  background: #1e3a5f;
  color: #fff;
  font-weight: 600;
}

.action-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.5rem 1.1rem;
  background: #fff;
  color: #374151;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
  font-family: inherit;
  white-space: nowrap;
}

.action-btn:hover {
  background: #f9fafb;
  border-color: #9ca3af;
}

.action-btn.save {
  background: #1e3a5f;
  color: #fff;
  border-color: #1e3a5f;
}

.action-btn.save:hover {
  background: #2a5082;
  border-color: #2a5082;
}

.action-btn.large {
  padding: 0.65rem 1.5rem;
  font-size: 0.95rem;
}

/* ── Edit card ── */
.edit-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  padding: 1.5rem;
}

.edit-card h2 {
  margin: 0 0 0.5rem;
  font-size: 1rem;
  font-weight: 700;
  color: #374151;
  border-left: 4px solid #1e3a5f;
  padding-left: 0.6rem;
}

.edit-hint {
  margin: 0 0 1.25rem;
  font-size: 0.8rem;
  color: #9ca3af;
}

.cat-select {
  width: 100%;
  min-width: 110px;
  padding: 0.3rem 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.85rem;
  color: #374151;
  background: #f9fafb;
  cursor: pointer;
  font-family: inherit;
  transition: border-color 0.15s;
}

.cat-select:focus {
  outline: none;
  border-color: #1e3a5f;
}

.edit-actions {
  display: flex;
  justify-content: flex-end;
  padding-top: 1rem;
  border-top: 1px solid #f3f4f6;
  margin-top: 1rem;
}

/* ── Category tags ── */
.cat-tag {
  display: inline-block;
  border-radius: 4px;
  padding: 0.15rem 0.5rem;
  font-size: 0.75rem;
  font-weight: 600;
  white-space: nowrap;
}

.cat-tag.unset {
  background: #f3f4f6;
  color: #9ca3af;
}

/* ── Toolbar actions ── */
.toolbar-actions {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

/* ── Add form card ── */
.add-form-card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  border: 2px dashed #d1d5db;
  padding: 1.5rem;
}

.add-form-card h2 {
  margin: 0 0 1rem;
  font-size: 1rem;
  font-weight: 700;
  color: #374151;
  border-left: 4px solid #1e3a5f;
  padding-left: 0.6rem;
}

.add-form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 0.75rem 1.25rem;
  margin-bottom: 0.75rem;
}

.form-row {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.form-label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #6b7280;
}

.required {
  color: #dc2626;
}

.form-input {
  padding: 0.45rem 0.65rem;
  border: 1px solid #d1d5db;
  border-radius: 7px;
  font-size: 0.875rem;
  color: #111827;
  font-family: inherit;
  transition: border-color 0.15s;
}

.form-input:focus {
  outline: none;
  border-color: #1e3a5f;
}

.form-error {
  margin: 0 0 0.5rem;
}

.add-form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  padding-top: 0.75rem;
  border-top: 1px solid #f3f4f6;
  margin-top: 0.5rem;
}

/* ── Manual badge & delete button ── */
.manual-badge {
  display: inline-block;
  background: #fef3c7;
  color: #d97706;
  border-radius: 4px;
  padding: 0.1rem 0.35rem;
  font-size: 0.7rem;
  font-weight: 700;
  margin-left: 0.35rem;
  vertical-align: middle;
}

.col-del {
  width: 32px;
  padding-left: 0;
  text-align: center;
}

.del-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border: none;
  background: transparent;
  color: #d1d5db;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}

.del-btn:hover {
  background: #fee2e2;
  color: #dc2626;
}
</style>
