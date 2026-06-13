<template>
  <div class="calendar-page">
    <div class="page-header">
      <div class="container">
        <h1>📅 借用日历</h1>
        <p>一目了然，哪天取琴、练琴、还琴</p>
      </div>
    </div>

    <div class="container">
      <div class="calendar-layout">
        <div class="calendar-main">
          <div class="card calendar-card">
            <div class="calendar-nav">
              <el-button :icon="ArrowLeft" @click="prevMonth" circle />
              <h2 class="month-title">{{ currentYear }}年{{ currentMonth }}月</h2>
              <el-button :icon="ArrowRight" @click="nextMonth" circle />
              <el-button size="small" @click="goToday" class="today-btn">今天</el-button>
            </div>

            <div class="calendar-weekdays">
              <div v-for="d in weekdays" :key="d" class="weekday">{{ d }}</div>
            </div>

            <div class="calendar-grid">
              <div
                v-for="(day, idx) in calendarDays"
                :key="idx"
                class="calendar-cell"
                :class="{
                  'other-month': !day.currentMonth,
                  'is-today': day.isToday,
                  'has-events': day.events.length > 0,
                  'is-selected': selectedDate === day.dateStr
                }"
                @click="selectDate(day)"
              >
                <span class="day-number">{{ day.day }}</span>
                <div class="day-events" v-if="day.events.length">
                  <div
                    v-for="(ev, eIdx) in day.events.slice(0, 3)"
                    :key="eIdx"
                    class="day-event-dot"
                    :class="'type-' + ev.type"
                    :title="ev.title"
                  >{{ ev.shortLabel }}</div>
                  <div v-if="day.events.length > 3" class="day-more">+{{ day.events.length - 3 }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="calendar-sidebar">
          <div class="legend card mb-20">
            <h3>图例</h3>
            <div class="legend-item"><span class="legend-dot type-pickup"></span>取琴</div>
            <div class="legend-item"><span class="legend-dot type-return"></span>还琴</div>
            <div class="legend-item"><span class="legend-dot type-trial"></span>试奏预约</div>
            <div class="legend-item"><span class="legend-dot type-practice"></span>练习邀约</div>
          </div>

          <div class="card day-detail" v-if="selectedEvents.length">
            <h3>{{ selectedLabel }} 的事项</h3>
            <div class="detail-list">
              <div v-for="ev in selectedEvents" :key="ev.id + ev.type" class="detail-item" :class="'type-' + ev.type">
                <div class="detail-badge">{{ ev.typeLabel }}</div>
                <div class="detail-body">
                  <div class="detail-title">{{ ev.title }}</div>
                  <div class="detail-meta">{{ ev.meta }}</div>
                  <div class="detail-status">
                    <span class="badge" :class="statusClass(ev.raw?.status)">{{ statusText(ev.raw?.status) }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="card day-detail" v-else>
            <div class="empty-state small">
              <el-icon><Calendar /></el-icon>
              <p>选择日期查看事项</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '../stores/user'
import { borrowApi, invitationApi } from '../api'
import { ArrowLeft, ArrowRight } from '@element-plus/icons-vue'
import { Calendar } from '@element-plus/icons-vue'

const userStore = useUserStore()

const today = new Date()
const currentYear = ref(today.getFullYear())
const currentMonth = ref(today.getMonth() + 1)
const selectedDate = ref('')
const borrows = ref([])
const invitations = ref([])

const weekdays = ['一', '二', '三', '四', '五', '六', '日']

const statusMap = {
  pending: { text: '待处理', class: 'badge-warning' },
  confirmed: { text: '已确认', class: 'badge-primary' },
  borrowing: { text: '借用中', class: 'badge-primary' },
  returned: { text: '已归还', class: 'badge-success' },
  rejected: { text: '已拒绝', class: 'badge-danger' },
  accepted: { text: '已接受', class: 'badge-success' },
  completed: { text: '已完成', class: 'badge-success' }
}
const statusText = (s) => statusMap[s]?.text || s || ''
const statusClass = (s) => statusMap[s]?.class || 'badge-primary'

const pad = (n) => String(n).padStart(2, '0')
const fmtDate = (y, m, d) => `${y}-${pad(m)}-${pad(d)}`

function parseDateStr(str) {
  if (!str) return null
  const m = str.match(/^(\d{4})[\/\-](\d{1,2})[\/\-](\d{1,2})/)
  if (m) return fmtDate(+m[1], +m[2], +m[3])
  return str.slice(0, 10)
}

function parseMeetTimeDate(str) {
  if (!str) return null
  const m = str.match(/^(\d{4})[\/\-](\d{1,2})[\/\-](\d{1,2})/)
  if (m) return fmtDate(+m[1], +m[2], +m[3])
  return str.slice(0, 10)
}

const calendarDays = computed(() => {
  const y = currentYear.value
  const m = currentMonth.value
  const firstDay = new Date(y, m - 1, 1)
  let startWeekday = firstDay.getDay()
  if (startWeekday === 0) startWeekday = 7
  const daysInMonth = new Date(y, m, 0).getDate()
  const prevMonthDays = new Date(y, m - 1, 0).getDate()

  const todayStr = fmtDate(today.getFullYear(), today.getMonth() + 1, today.getDate())
  const days = []

  for (let i = startWeekday - 1; i > 0; i--) {
    const d = prevMonthDays - i + 1
    const pm = m === 1 ? 12 : m - 1
    const py = m === 1 ? y - 1 : y
    const dateStr = fmtDate(py, pm, d)
    days.push({ day: d, dateStr, currentMonth: false, isToday: dateStr === todayStr, events: getEventsForDate(dateStr) })
  }

  for (let d = 1; d <= daysInMonth; d++) {
    const dateStr = fmtDate(y, m, d)
    days.push({ day: d, dateStr, currentMonth: true, isToday: dateStr === todayStr, events: getEventsForDate(dateStr) })
  }

  const remaining = 42 - days.length
  for (let d = 1; d <= remaining; d++) {
    const nm = m === 12 ? 1 : m + 1
    const ny = m === 12 ? y + 1 : y
    const dateStr = fmtDate(ny, nm, d)
    days.push({ day: d, dateStr, currentMonth: false, isToday: dateStr === todayStr, events: getEventsForDate(dateStr) })
  }

  return days
})

function getEventsForDate(dateStr) {
  const events = []
  const userId = userStore.userId

  for (const b of borrows.value) {
    const startDate = parseDateStr(b.startDate)
    const endDate = parseDateStr(b.endDate)
    const isBorrower = b.borrowerId === userId
    const isOwner = b.ownerId === userId
    if (!isBorrower && !isOwner) continue
    const instName = b.instrument?.name || '乐器'

    if (startDate === dateStr) {
      if (isBorrower) {
        events.push({
          id: b.id, type: 'pickup', typeLabel: '取琴',
          shortLabel: '取', title: `取琴：${instName}`,
          meta: `向 ${b.owner?.username || '主人'} 借用`,
          raw: b
        })
      }
      if (isOwner) {
        events.push({
          id: b.id, type: 'trial', typeLabel: '试奏预约',
          shortLabel: '试', title: `试奏预约：${instName}`,
          meta: `${b.borrower?.username || '借用人'} 来取琴`,
          raw: b
        })
      }
    }

    if (endDate === dateStr && isBorrower) {
      events.push({
        id: b.id, type: 'return', typeLabel: '还琴',
        shortLabel: '还', title: `还琴：${instName}`,
        meta: `归还给 ${b.owner?.username || '主人'}`,
        raw: b
      })
    }
    if (endDate === dateStr && isOwner) {
      events.push({
        id: b.id, type: 'trial', typeLabel: '收琴',
        shortLabel: '收', title: `收琴：${instName}`,
        meta: `${b.borrower?.username || '借用人'} 归还`,
        raw: b
      })
    }
  }

  for (const inv of invitations.value) {
    const invDate = parseMeetTimeDate(inv.meetTime)
    if (invDate === dateStr) {
      const isInviter = inv.inviterId === userId
      events.push({
        id: inv.id, type: 'practice', typeLabel: '练习邀约',
        shortLabel: '练', title: `练琴：${inv.instrument || ''} - ${inv.piece || ''}`,
        meta: isInviter
          ? `与 ${inv.invitee?.username || '搭子'} 练琴`
          : `${inv.inviter?.username || '搭子'} 邀你练琴`,
        raw: inv
      })
    }
  }

  return events
}

const selectedEvents = computed(() => {
  if (!selectedDate.value) return []
  return getEventsForDate(selectedDate.value)
})

const selectedLabel = computed(() => {
  if (!selectedDate.value) return ''
  const parts = selectedDate.value.split('-')
  return `${+parts[1]}月${+parts[2]}日`
})

function selectDate(day) {
  selectedDate.value = day.dateStr
}

function prevMonth() {
  if (currentMonth.value === 1) {
    currentMonth.value = 12
    currentYear.value--
  } else {
    currentMonth.value--
  }
  selectedDate.value = ''
}

function nextMonth() {
  if (currentMonth.value === 12) {
    currentMonth.value = 1
    currentYear.value++
  } else {
    currentMonth.value++
  }
  selectedDate.value = ''
}

function goToday() {
  currentYear.value = today.getFullYear()
  currentMonth.value = today.getMonth() + 1
  selectedDate.value = fmtDate(today.getFullYear(), today.getMonth() + 1, today.getDate())
}

onMounted(async () => {
  if (!userStore.isLoggedIn) return
  try {
    const [asOwner, asBorrower] = await Promise.all([
      borrowApi.list({ ownerId: userStore.userId, currentUserId: userStore.userId }),
      borrowApi.list({ borrowerId: userStore.userId, currentUserId: userStore.userId })
    ])
    const map = new Map()
    asOwner.forEach(b => map.set(b.id, b))
    asBorrower.forEach(b => { if (!map.has(b.id)) map.set(b.id, b) })
    borrows.value = Array.from(map.values())
  } catch (e) {
    console.error(e)
  }
  try {
    invitations.value = await invitationApi.listByUser(userStore.userId)
  } catch (e) {
    console.error(e)
  }
  goToday()
})
</script>

<style scoped>
.calendar-layout {
  display: grid;
  grid-template-columns: 1fr 320px;
  gap: 24px;
  margin-top: 20px;
}

.calendar-card {
  padding: 24px;
}

.calendar-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  margin-bottom: 20px;
}

.month-title {
  font-size: 20px;
  font-weight: 700;
  min-width: 160px;
  text-align: center;
}

.today-btn {
  margin-left: 8px;
}

.calendar-weekdays {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  text-align: center;
  margin-bottom: 8px;
}

.weekday {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-secondary);
  padding: 8px 0;
}

.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 2px;
}

.calendar-cell {
  min-height: 90px;
  padding: 6px;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.15s;
  background: white;
}

.calendar-cell:hover {
  background: #f5f3ff;
}

.calendar-cell.other-month {
  background: #fafafa;
}

.calendar-cell.other-month .day-number {
  color: #ccc;
}

.calendar-cell.is-today {
  border-color: var(--primary-color);
  background: #eef2ff;
}

.calendar-cell.is-today .day-number {
  background: var(--primary-color);
  color: white;
  border-radius: 50%;
  width: 26px;
  height: 26px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.calendar-cell.is-selected {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.2);
}

.day-number {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 4px;
}

.day-events {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.day-event-dot {
  font-size: 11px;
  padding: 1px 5px;
  border-radius: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-weight: 500;
  line-height: 1.5;
}

.day-event-dot.type-pickup {
  background: #dbeafe;
  color: #1d4ed8;
}

.day-event-dot.type-return {
  background: #fef3c7;
  color: #b45309;
}

.day-event-dot.type-trial {
  background: #ede9fe;
  color: #7c3aed;
}

.day-event-dot.type-practice {
  background: #d1fae5;
  color: #047857;
}

.day-more {
  font-size: 10px;
  color: var(--text-secondary);
  text-align: center;
}

.legend h3 {
  font-size: 15px;
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  margin-bottom: 8px;
  color: var(--text-primary);
}

.legend-dot {
  width: 16px;
  height: 16px;
  border-radius: 4px;
  flex-shrink: 0;
}

.legend-dot.type-pickup {
  background: #dbeafe;
  border: 2px solid #1d4ed8;
}

.legend-dot.type-return {
  background: #fef3c7;
  border: 2px solid #b45309;
}

.legend-dot.type-trial {
  background: #ede9fe;
  border: 2px solid #7c3aed;
}

.legend-dot.type-practice {
  background: #d1fae5;
  border: 2px solid #047857;
}

.day-detail h3 {
  font-size: 15px;
  margin-bottom: 14px;
}

.detail-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-item {
  display: flex;
  gap: 10px;
  padding: 10px;
  border-radius: 8px;
  background: var(--bg-light);
}

.detail-badge {
  font-size: 11px;
  font-weight: 600;
  padding: 4px 8px;
  border-radius: 6px;
  white-space: nowrap;
  height: fit-content;
}

.detail-item.type-pickup .detail-badge {
  background: #dbeafe;
  color: #1d4ed8;
}

.detail-item.type-return .detail-badge {
  background: #fef3c7;
  color: #b45309;
}

.detail-item.type-trial .detail-badge {
  background: #ede9fe;
  color: #7c3aed;
}

.detail-item.type-practice .detail-badge {
  background: #d1fae5;
  color: #047857;
}

.detail-body {
  flex: 1;
  min-width: 0;
}

.detail-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 4px;
}

.detail-meta {
  font-size: 12px;
  color: var(--text-secondary);
  margin-bottom: 6px;
}

.detail-status .badge {
  font-size: 11px;
}

.empty-state.small {
  padding: 30px 10px;
}

.empty-state.small .el-icon {
  font-size: 40px;
}

@media (max-width: 900px) {
  .calendar-layout {
    grid-template-columns: 1fr;
  }

  .calendar-cell {
    min-height: 60px;
  }

  .day-event-dot {
    font-size: 10px;
    padding: 0 3px;
  }
}
</style>
