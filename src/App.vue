<template>
  <div class="page">
    <div class="container">
      <header class="header">
        <div>
          <h1>Scholar 论文导航器</h1>
          <p class="desc">
            上传 CSV 后显示：序号、论文名、第一作者、第一作者组织、第一作者国家；支持上一篇 / 下一篇、序号跳转、打开 Scholar、自动保存进度。
          </p>
        </div>
      </header>

      <section class="card upload-card">
        <div class="section-title">上传 CSV</div>
        <div class="upload-row">
          <input type="file" accept=".csv" @change="handleFileChange" />
          <button class="btn btn-secondary" @click="restoreProgress" :disabled="!rows.length">
            恢复上次进度
          </button>
          <button class="btn btn-danger" @click="clearProgress" :disabled="!rows.length">
            清除进度
          </button>
        </div>
        <div class="tips">
          需要列名：序号、论文名、第一作者、第一作者组织、第一作者国家
        </div>
        <div v-if="fileName" class="meta">当前文件：{{ fileName }}</div>
        <div v-if="error" class="error">{{ error }}</div>
      </section>

      <section v-if="rows.length" class="card info-card">
        <div class="top-bar">
          <div class="badge-group">
            <span class="badge">当前第 {{ currentIndex + 1 }} / {{ rows.length }} 条</span>
            <span class="badge badge-outline">序号：{{ currentPaper['序号'] || '-' }}</span>
          </div>
          <div class="status">已自动保存进度</div>
        </div>

        <div class="paper-grid">
          <div class="field full">
            <div class="label">论文名</div>
            <div class="value title">{{ currentPaper['论文名'] || '-' }}</div>
          </div>

          <div class="field">
            <div class="label">序号</div>
            <div class="value">{{ currentPaper['序号'] || '-' }}</div>
          </div>

          <div class="field">
            <div class="label">第一作者</div>
            <div class="value">{{ currentPaper['第一作者'] || '-' }}</div>
          </div>

          <div class="field full">
            <div class="label">第一作者组织</div>
            <div class="value">{{ currentPaper['第一作者组织'] || '-' }}</div>
          </div>

          <div class="field">
            <div class="label">第一作者国家</div>
            <div class="value">{{ currentPaper['第一作者国家'] || '-' }}</div>
          </div>
        </div>
      </section>

      <section v-if="rows.length" class="card action-card">
        <div class="section-title">操作</div>

        <div class="action-row action-grid">
          <button class="btn" @click="prevPaper" :disabled="currentIndex === 0">上一篇</button>
          <button class="btn" @click="nextPaper" :disabled="currentIndex === rows.length - 1">下一篇</button>
          <button class="btn btn-primary" @click="openScholar">打开 Scholar</button>
          <button class="btn btn-secondary" @click="copyTitle">复制论文名</button>
        </div>

        <div class="jump-row">
          <input
            v-model.trim="jumpValue"
            class="jump-input"
            type="text"
            inputmode="numeric"
            placeholder="输入序号或第几条"
            @keyup.enter="jumpToSequence"
          />
          <button class="btn btn-primary" @click="jumpToSequence">序号跳转</button>
        </div>

        <div class="tips">
          跳转逻辑：优先按 CSV 中的“序号”查找；如果没找到，则按“第几条记录”跳转。
        </div>
      </section>

      <section v-if="rows.length" class="card table-card">
        <div class="section-title">前 10 条预览</div>
        <div class="table-wrapper">
          <table>
            <thead>
              <tr>
                <th>序号</th>
                <th>论文名</th>
                <th>第一作者</th>
                <th>第一作者组织</th>
                <th>第一作者国家</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, index) in previewRows"
                :key="index"
                :class="{ active: index === currentIndex }"
                @click="goToIndex(index)"
              >
                <td>{{ item['序号'] }}</td>
                <td>{{ item['论文名'] }}</td>
                <td>{{ item['第一作者'] }}</td>
                <td>{{ item['第一作者组织'] }}</td>
                <td>{{ item['第一作者国家'] }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <section v-if="!rows.length" class="card empty-card">
        <div class="empty">请先上传 CSV 文件</div>
      </section>
    </div>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue'
import Papa from 'papaparse'

const STORAGE_KEY_DATA = 'scholar_csv_rows_v1'
const STORAGE_KEY_INDEX = 'scholar_csv_current_index_v1'
const STORAGE_KEY_FILENAME = 'scholar_csv_file_name_v1'

const REQUIRED_COLUMNS = ['序号', '论文名', '第一作者', '第一作者组织', '第一作者国家']

const rows = ref([])
const currentIndex = ref(0)
const jumpValue = ref('')
const fileName = ref('')
const error = ref('')

const currentPaper = computed(() => rows.value[currentIndex.value] || {})
const previewRows = computed(() => rows.value.slice(0, 10))

const normalizeValue = (value) => {
  if (value === null || value === undefined) return ''
  return String(value).trim()
}

const buildScholarUrl = (title) => {
  return `https://scholar.google.com/scholar?q=${encodeURIComponent(title || '')}`
}

const saveProgress = () => {
  localStorage.setItem(STORAGE_KEY_DATA, JSON.stringify(rows.value))
  localStorage.setItem(STORAGE_KEY_INDEX, String(currentIndex.value))
  localStorage.setItem(STORAGE_KEY_FILENAME, fileName.value)
}

const restoreProgress = () => {
  try {
    const cachedRows = localStorage.getItem(STORAGE_KEY_DATA)
    const cachedIndex = localStorage.getItem(STORAGE_KEY_INDEX)
    const cachedFileName = localStorage.getItem(STORAGE_KEY_FILENAME)

    if (!cachedRows) {
      error.value = '未找到已保存的进度'
      return
    }

    const parsedRows = JSON.parse(cachedRows)
    if (!Array.isArray(parsedRows) || !parsedRows.length) {
      error.value = '本地进度为空或损坏'
      return
    }

    rows.value = parsedRows
    fileName.value = cachedFileName || '上次上传的文件'

    const indexNum = Number(cachedIndex)
    if (Number.isInteger(indexNum) && indexNum >= 0 && indexNum < parsedRows.length) {
      currentIndex.value = indexNum
    } else {
      currentIndex.value = 0
    }

    error.value = ''
  } catch (e) {
    error.value = '恢复进度失败'
  }
}

const clearProgress = () => {
  localStorage.removeItem(STORAGE_KEY_DATA)
  localStorage.removeItem(STORAGE_KEY_INDEX)
  localStorage.removeItem(STORAGE_KEY_FILENAME)
  error.value = '已清除本地进度'
}

const handleFileChange = (event) => {
  const file = event.target.files?.[0]
  if (!file) return

  error.value = ''
  fileName.value = file.name

  Papa.parse(file, {
    header: true,
    skipEmptyLines: true,
    complete: (results) => {
      const data = results.data || []
      const firstRow = data[0] || {}

      const missingColumns = REQUIRED_COLUMNS.filter((column) => !(column in firstRow))
      if (missingColumns.length) {
        rows.value = []
        currentIndex.value = 0
        error.value = `CSV 缺少列：${missingColumns.join('、')}`
        return
      }

      const cleanedRows = data
        .map((item) => ({
          序号: normalizeValue(item['序号']),
          论文名: normalizeValue(item['论文名']),
          第一作者: normalizeValue(item['第一作者']),
          第一作者组织: normalizeValue(item['第一作者组织']),
          第一作者国家: normalizeValue(item['第一作者国家'])
        }))
        .filter((item) => item['论文名'] !== '')

      if (!cleanedRows.length) {
        rows.value = []
        currentIndex.value = 0
        error.value = 'CSV 中没有可显示的数据'
        return
      }

      rows.value = cleanedRows
      currentIndex.value = 0
      jumpValue.value = ''
      error.value = ''
      saveProgress()
    },
    error: () => {
      rows.value = []
      currentIndex.value = 0
      error.value = 'CSV 解析失败'
    }
  })
}

const prevPaper = () => {
  if (currentIndex.value > 0) {
    currentIndex.value -= 1
  }
}

const nextPaper = () => {
  if (currentIndex.value < rows.value.length - 1) {
    currentIndex.value += 1
  }
}

const goToIndex = (index) => {
  currentIndex.value = index
}

const jumpToSequence = () => {
  const text = jumpValue.value.trim()
  if (!text) {
    error.value = '请输入序号或第几条'
    return
  }

  const num = Number(text)
  if (!Number.isInteger(num)) {
    error.value = '请输入整数'
    return
  }

  const exactIndex = rows.value.findIndex((item) => Number(item['序号']) === num || item['序号'] === text)
  if (exactIndex !== -1) {
    currentIndex.value = exactIndex
    error.value = ''
    return
  }

  if (num >= 1 && num <= rows.value.length) {
    currentIndex.value = num - 1
    error.value = ''
    return
  }

  error.value = `未找到序号 = ${num} 的记录，且它也不在 1 ~ ${rows.value.length} 的范围内`
}

const openScholar = () => {
  const title = currentPaper.value['论文名'] || ''
  if (!title) {
    error.value = '当前论文名为空，无法打开 Scholar'
    return
  }
  window.open(buildScholarUrl(title), '_blank', 'noopener,noreferrer')
}

const copyTitle = async () => {
  const title = currentPaper.value['论文名'] || ''
  if (!title) {
    error.value = '当前论文名为空，无法复制'
    return
  }

  try {
    await navigator.clipboard.writeText(title)
    error.value = '论文名已复制'
  } catch (e) {
    error.value = '复制失败，请手动复制'
  }
}

watch([rows, currentIndex, fileName], () => {
  if (rows.value.length) {
    saveProgress()
  }
}, { deep: true })

restoreProgress()
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.page {
  min-height: 100vh;
  padding: 24px 12px 40px;
  background: linear-gradient(180deg, #f8fafc 0%, #eef2ff 100%);
}

.container {
  max-width: 1100px;
  margin: 0 auto;
}

.header {
  margin-bottom: 18px;
}

.header h1 {
  margin: 0 0 8px;
  font-size: 30px;
  color: #111827;
}

.desc {
  margin: 0;
  color: #6b7280;
  line-height: 1.7;
}

.card {
  background: #ffffff;
  border-radius: 18px;
  padding: 18px;
  margin-bottom: 16px;
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);
}

.section-title {
  font-size: 18px;
  font-weight: 700;
  color: #1f2937;
  margin-bottom: 14px;
}

.upload-row,
.action-row,
.jump-row {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-items: center;
}

.action-grid {
  margin-bottom: 14px;
}

.meta,
.tips,
.status {
  margin-top: 10px;
  font-size: 14px;
  color: #6b7280;
}

.error {
  margin-top: 12px;
  padding: 12px 14px;
  border-radius: 12px;
  background: #fef2f2;
  color: #b91c1c;
  font-size: 14px;
}

.top-bar {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: center;
  flex-wrap: wrap;
  margin-bottom: 16px;
}

.badge-group {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.badge {
  display: inline-flex;
  align-items: center;
  padding: 8px 12px;
  border-radius: 999px;
  background: #e0e7ff;
  color: #3730a3;
  font-size: 14px;
  font-weight: 600;
}

.badge-outline {
  background: #ffffff;
  border: 1px solid #c7d2fe;
}

.paper-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
}

.field {
  padding: 14px;
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  background: #fafafa;
}

.field.full {
  grid-column: 1 / -1;
}

.label {
  font-size: 13px;
  color: #6b7280;
  margin-bottom: 8px;
}

.value {
  color: #111827;
  line-height: 1.7;
  word-break: break-word;
}

.value.title {
  font-size: 18px;
  font-weight: 700;
}

.btn {
  border: none;
  border-radius: 12px;
  padding: 11px 18px;
  font-size: 15px;
  cursor: pointer;
  background: #e5e7eb;
  color: #111827;
  transition: 0.2s ease;
}

.btn:hover:not(:disabled) {
  transform: translateY(-1px);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: #2563eb;
  color: #ffffff;
}

.btn-secondary {
  background: #eef2ff;
  color: #3730a3;
}

.btn-danger {
  background: #fee2e2;
  color: #b91c1c;
}

.jump-input {
  flex: 1;
  min-width: 220px;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  padding: 11px 14px;
  font-size: 15px;
  outline: none;
}

.jump-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.12);
}

.table-wrapper {
  overflow-x: auto;
}

table {
  width: 100%;
  min-width: 900px;
  border-collapse: collapse;
}

thead {
  background: #eef2ff;
}

th,
td {
  text-align: left;
  padding: 14px 12px;
  border-bottom: 1px solid #e5e7eb;
  vertical-align: top;
}

th {
  color: #374151;
}

td {
  color: #4b5563;
  line-height: 1.7;
}

tbody tr {
  cursor: pointer;
}

tbody tr:hover {
  background: #f8fafc;
}

tbody tr.active {
  background: #eff6ff;
}

.empty {
  text-align: center;
  color: #9ca3af;
  padding: 36px 0;
}

@media (max-width: 768px) {
  .page {
    padding: 16px 10px 30px;
  }

  .header h1 {
    font-size: 24px;
  }

  .paper-grid {
    grid-template-columns: 1fr;
  }

  .jump-input {
    min-width: 100%;
  }

  .btn {
    width: 100%;
  }

  .action-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .action-grid .btn {
    width: 100%;
  }
}
</style>
