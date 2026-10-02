<script setup>
import { computed } from 'vue'
import { useEventStore } from '@/store/event'
const store = useEventStore()

const bestAttack = computed(() => {
  const rows = []
  store.sports.forEach(s => {
    (store.standings[s.id] || []).forEach(r => { if (r.play > 0) rows.push({ t: r.tname, sport: s.name, gf: r.gf - r.ga, unit: r.unit, color: r.color }) })
  })
  return rows.sort((a, b) => b.gf - a.gf).slice(0, 6)
})
const maxGD = computed(() => Math.max(1, ...bestAttack.value.map(r => r.gf)))

const unitPoints = computed(() => {
  const m = {}
  const u = store.units
  u.forEach(x => m[x.id] = { name: x.name, color: x.color, pts: 0, gold: 0 })
  store.teams.forEach(t => {
    const st = (store.standings[t.sport_id] || []).find(r => r.team_id === t.id)
    if (st && m[t.unit_id]) m[t.unit_id].pts += st.points
  })
  store.medals.forEach(md => { if (m[md.unit_id]) m[md.unit_id].gold = md.gold })
  return Object.values(m).sort((a, b) => b.pts - a.pts)
})
const maxPts = computed(() => Math.max(1, ...unitPoints.value.map(x => x.pts)))

// —— 裁判排班联动报表 ——
const refereeRows = computed(() => store.workload)
const maxDone = computed(() => Math.max(1, ...refereeRows.value.map(r => (r.done || 0) + (r.upcoming || 0))))
// 整场执法组覆盖率：按岗位（主裁1·助理2·记录台1 = 4 岗/场）统计
const coverage = computed(() => {
  if (store.conflicts?.coverage) return store.conflicts.coverage
  const pend = store.matches.filter(m => m.status === 'scheduled' && m.team_a && m.team_b && store.sports.find(s => s.id === m.sport_id)?.format !== 'track')
  return { matches: pend.length, crew_ready: 0, slots_total: pend.length * 4, slots_filled: 0, pct: 0 }
})
const ROLE_LABEL = { chief: '主裁', assistant: '助理', recorder: '记录台' }
const recentLogs = computed(() => store.assignmentLogs.slice(0, 6)
  .map(l => ({ ...l, name: { assign: '排班', force_assign: '强制排班', auto_assign: '自动排班', release: '解除', reassign: '临时调班', swap: '调班对调', match_change: '赛程变更', schedule_added: '赛程新增', schedule_rebuild: '赛程重排', match_finish: '完赛归档', void_release: '取消解除' }[l.action] || l.action })))
</script>

<template>
  <div v-if="store.loaded">
    <div class="page-h">
      <div><h2>📊 报表中心</h2><div class="sub">赛事数据洞察：净胜球排行 · 综合积分 · 项目概览</div></div>
    </div>

    <div class="grid g2">
      <!-- 净胜球排行 -->
      <div class="card">
        <div class="caption">🔥 最佳攻击线（净胜球）</div>
        <div class="pad" style="display:flex;flex-direction:column;gap:14px">
          <div v-for="(r, ri) in bestAttack" :key="ri + '-' + r.t">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge"><span class="dot" :style="{ background: r.color }"></span>{{ r.t }} <span class="tag gray" style="margin-left:4px">{{ r.sport }}</span></span>
              <b class="mono">{{ r.gf > 0 ? '+' : '' }}{{ r.gf }}</b>
            </div>
            <div class="hbar"><i :style="{ width: (r.gf / maxGD) * 100 + '%', background: r.gf > 0 ? 'var(--accent)' : '#e5484d' }"></i></div>
          </div>
        </div>
      </div>

      <!-- 单位综合积分 -->
      <div class="card">
        <div class="caption">🏅 单位综合积分（球类积分之和）</div>
        <div class="pad" style="display:flex;flex-direction:column;gap:14px">
          <div v-for="u in unitPoints" :key="u.name">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge"><span class="dot" :style="{ background: u.color }"></span>{{ u.name }}</span>
              <span><span class="tag y">🥇{{ u.gold }}</span> <b class="mono" style="font-size:16px;color:var(--accent)">{{ u.pts }}</b></span>
            </div>
            <div class="hbar"><i :style="{ width: (u.pts / maxPts) * 100 + '%' }"></i></div>
          </div>
        </div>
      </div>
    </div>

    <!-- 裁判执法工作量与排班覆盖 -->
    <div class="grid g2 mt">
      <div class="card">
        <div class="caption">🧑‍⚖️ 裁判执法工作量 <span class="hint">已完赛 / 待赛（按主裁·助理·记录台拆分，含历史场次）</span></div>
        <div class="pad" style="display:flex;flex-direction:column;gap:13px">
          <div v-for="r in refereeRows" :key="r.id">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge">{{ r.name }} <span class="tag gray" style="margin-left:4px">{{ r.sport || '综合执法' }} · {{ r.level }}</span></span>
              <b class="mono"><span style="color:var(--accent2)">{{ r.done || 0 }}</span> / <span style="color:var(--accent3)">{{ r.upcoming || 0 }}</span>
                <span class="hint">主{{ (r.done_chief||0)+(r.up_chief||0) }} · 助{{ (r.done_assistant||0)+(r.up_assistant||0) }} · 记{{ (r.done_recorder||0)+(r.up_recorder||0) }}</span>
              </b>
            </div>
            <div class="hbar-duo">
              <i class="d" :style="{ width: ((r.done || 0) / maxDone) * 100 + '%' }"></i><i class="u" :style="{ width: ((r.upcoming || 0) / maxDone) * 100 + '%' }"></i>
            </div>
          </div>
        </div>
      </div>
      <div class="card">
        <div class="caption">📡 整场执法组覆盖率与最新变更</div>
        <div class="pad">
          <div class="row spread" style="margin-bottom:8px">
            <span class="badge">执法岗位覆盖率（主裁1·助理2·记录台1）</span>
            <b class="mono" style="font-size:16px;color:var(--accent)">{{ coverage.slots_filled }}/{{ coverage.slots_total }}（{{ coverage.pct }}%）</b>
          </div>
          <div class="hbar"><i :style="{ width: coverage.pct + '%', background: coverage.pct === 100 ? 'var(--accent2)' : 'var(--accent)' }"></i></div>
          <div class="row spread mt16" style="margin-bottom:8px">
            <span class="badge">执法组满编场次</span>
            <b class="mono" :style="{ color: coverage.crew_ready === coverage.matches ? 'var(--accent2)' : 'var(--accent)' }">{{ coverage.crew_ready }}/{{ coverage.matches }} 场</b>
          </div>
          <div class="row spread" style="margin-bottom:8px"><span class="badge">⛔ 未决冲突</span>
            <b :style="{ color: (store.conflicts?.referee_conflicts.length || 0) + (store.conflicts?.venue_conflicts.length || 0) ? '#e5484d' : 'var(--accent2)' }">
              {{ (store.conflicts?.referee_conflicts.length || 0) + (store.conflicts?.venue_conflicts.length || 0) }} 起
            </b>
          </div>
          <div v-if="store.conflicts?.incomplete_matches?.length" class="row" style="margin-bottom:8px;flex-direction:column;align-items:stretch;gap:4px">
            <span class="hint">🟠 未满编场次：</span>
            <span v-for="x in store.conflicts.incomplete_matches.slice(0, 5)" :key="x.match_id" class="tag o" style="font-size:11px">{{ x.title }} — 缺 {{ Object.entries(x.missing_roles).map(([r,n]) => (ROLE_LABEL[r]||r) + '×' + n).join('、') }}</span>
          </div>
          <table style="margin-top:6px">
            <thead><tr><th>操作</th><th>详情</th><th>操作人</th></tr></thead>
            <tbody>
              <tr v-for="l in recentLogs" :key="l.id">
                <td style="white-space:nowrap">{{ l.name }}</td>
                <td class="ph" style="max-width:280px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ l.detail }}</td>
                <td>{{ l.operator }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 项目明细 -->
    <div class="card mt">
      <div class="caption">🗂️ 赛事项目明细与规则</div>
      <div class="pad">
        <table>
          <thead><tr><th>项目</th><th>类别</th><th>赛制</th><th>场地</th><th>已完成/总场次</th><th>冠军归属</th></tr></thead>
          <tbody>
            <tr v-for="s in store.sports" :key="s.id">
              <td><b>{{ s.name }}</b></td>
              <td><span class="tag b">{{ s.category }}</span></td>
              <td>{{ s.format === 'roundrobin' ? '单循环积分制' : s.format === 'group_knockout' ? '小组赛 + 淘汰赛' : s.format === 'knockout' ? '单败淘汰' : '计时成绩' }}</td>
              <td>{{ s.venue }}</td>
              <td class="mono">{{ store.overview?.sportDone?.find(x=>x.id===s.id)?.done || 0 }}/{{ store.overview?.sportDone?.find(x=>x.id===s.id)?.total || 0 }}</td>
              <td class="ph">{{ s.finished ? '已产生' : '待结算' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.hbar-duo { position: relative; height: 9px; border-radius: 20px; background: var(--bg2); overflow: hidden; display: flex; }
.hbar-duo i { display: block; height: 100%; border-radius: 20px; }
.hbar-duo i.d { background: var(--accent2); }
.hbar-duo i.u { background: var(--accent3); margin-left: 2px; }
</style>