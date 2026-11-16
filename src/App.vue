<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
type Category = { id: number; name: string; groupingId: number }; type Source = { id: number; name: string; description?: string | null }; type PaymentType = { id: number; description: string }; type Expense = { id: number; date: string; sourceId: number; categoryId: number; paymentTypeId: number; purchaseAmount: number; discountAmount: number; total: number; category?: Category; source?: Source }; type Income = { id: number; date: string; description: string; amount: number }; type Row = { id: number; date: string; name: string; detail: string; kind: 'Receita' | 'Despesa'; amount: number; category: string }
const api = (import.meta.env.VITE_API_URL || '').replace(/\/$/, ''); const categories = ref<Category[]>([]), sources = ref<Source[]>([]), payments = ref<PaymentType[]>([]), expenses = ref<Expense[]>([]), incomes = ref<Income[]>([]); const loading = ref(true), error = ref(''), notice = ref(''), modal = ref(false), kind = ref<'expense' | 'income'>('expense'), saving = ref(false)
const today = new Date().toISOString().slice(0, 10); const filters = reactive({ from: new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString().slice(0, 10), to: today, category: '' }); const form = reactive({ date: today, source: '', category: '', payment: '', amount: '', discount: '', description: '' })
async function get<T>(path: string, options: RequestInit = {}): Promise<T> { const r = await fetch(`${api}/api/${path}`, { ...options, headers: { ...(options.body ? { 'Content-Type': 'application/json' } : {}), ...options.headers } }); if (!r.ok) throw new Error((await r.text()) || `Erro ${r.status}`); return r.status === 204 ? undefined as T : r.json() as Promise<T> }
async function load() { loading.value = true; error.value = ''; try { const query = new URLSearchParams(); if (filters.from) query.set('startDate', filters.from); if (filters.to) query.set('endDate', filters.to); const range = query.toString(); const suffix = range ? '?' + range : ''; const [c, s, p, e, i] = await Promise.all([get<Category[]>('categories'), get<Source[]>('sources'), get<PaymentType[]>('paymenttypes'), get<Expense[]>(`expenses${suffix}`), get<Income[]>(`incomes${suffix}`)]); categories.value = c; sources.value = s; payments.value = p; expenses.value = e; incomes.value = i } catch (e) { error.value = e instanceof Error ? e.message : 'Falha ao carregar dados.' } finally { loading.value = false } }
const filtered = computed(() => expenses.value.filter(x => !filters.category || x.categoryId === Number(filters.category))); const incomeTotal = computed(() => incomes.value.reduce((s, x) => s + Number(x.amount), 0)); const expenseTotal = computed(() => filtered.value.reduce((s, x) => s + Number(x.total), 0)); const balance = computed(() => incomeTotal.value - expenseTotal.value); const fmt = (n: number) => new Intl.NumberFormat('pt-PT', { style: 'currency', currency: 'EUR' }).format(n); const catName = (x: Expense) => x.category?.name || categories.value.find(c => c.id === x.categoryId)?.name || 'Sem categoria'; const srcName = (x: Expense) => x.source?.name || sources.value.find(s => s.id === x.sourceId)?.name || '—'; const dateFmt = (s: string) => new Intl.DateTimeFormat('pt-PT', { day: '2-digit', month: 'short' }).format(new Date(`${s}T12:00:00`))
const rows = computed<Row[]>(() => [...filtered.value.map(x => ({ id: x.id, date: x.date, name: catName(x), detail: srcName(x), kind: 'Despesa' as const, amount: -Number(x.total), category: catName(x) })), ...incomes.value.map(x => ({ id: x.id, date: x.date, name: x.description, detail: 'Receita', kind: 'Receita' as const, amount: Number(x.amount), category: 'Receita' }))].sort((a, b) => b.date.localeCompare(a.date)))
const cats = computed(() => { const sums = new Map<string, number>(); filtered.value.forEach(x => sums.set(catName(x), (sums.get(catName(x)) || 0) + Number(x.total))); const colors = ['#1c6b55', '#e7aa71', '#9183c6', '#df7d65', '#7eaaa0']; return [...sums].sort((a, b) => b[1] - a[1]).slice(0, 5).map(([name, value], i) => ({ name, value, color: colors[i % colors.length] })) })
const months = computed(() => { const end = new Date(`${(filters.to || today)}T12:00:00`), start = new Date(`${filters.from || today}T12:00:00`), count = Math.max(1, Math.min(12, (end.getFullYear() - start.getFullYear()) * 12 + end.getMonth() - start.getMonth() + 1)); return Array.from({ length: count }, (_, i) => { const d = new Date(end.getFullYear(), end.getMonth() - count + 1 + i, 1), key = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}`; return { label: new Intl.DateTimeFormat('pt-PT', { month: 'short' }).format(d), income: incomes.value.filter(x => x.date.startsWith(key)).reduce((s, x) => s + Number(x.amount), 0), expense: filtered.value.filter(x => x.date.startsWith(key)).reduce((s, x) => s + Number(x.total), 0) } }) }); const monthMax = computed(() => Math.max(1, ...months.value.flatMap(m => [m.income, m.expense])))
function create(k: 'expense' | 'income') { kind.value = k; Object.assign(form, { date: today, source: '', category: '', payment: '', amount: '', discount: '', description: '' }); notice.value = ''; modal.value = true }
async function save() { saving.value = true; notice.value = ''; try { if (kind.value === 'expense') { const amount = Number(form.amount), discount = Number(form.discount || 0); await get('expenses', { method: 'POST', headers: { 'Idempotency-Key': crypto.randomUUID() }, body: JSON.stringify({ id: 0, date: form.date, sourceId: Number(form.source), categoryId: Number(form.category), paymentTypeId: Number(form.payment), purchaseAmount: amount, discountAmount: discount, total: amount - discount }) }) } else await get('incomes', { method: 'POST', headers: { 'Idempotency-Key': crypto.randomUUID() }, body: JSON.stringify({ id: 0, date: form.date, description: form.description, amount: Number(form.amount) }) }); modal.value = false; notice.value = 'Movimento criado com sucesso.'; await load() } catch (e) { notice.value = e instanceof Error ? e.message : 'Não foi possível guardar.' } finally { saving.value = false } }
onMounted(load)
</script>
<template>
    <div class="shell">
        <aside class="sidebar"><a class="brand"><span class="logo">⌂</span>home<span
                    class="muted">ledger</span></a><small class="section-label">ESPAÇO PESSOAL</small>
            <nav><a class="selected">▦ &nbsp; Visão geral</a><a href="#movimentos">↕ &nbsp; Movimentos</a><a
                    href="#relatorios">◴ &nbsp; Relatórios</a></nav>
            <div class="side-bottom"><span class="online" :class="{ offline: error }"></span>{{ error ? 'API indisponível':loading?'A ligar à API':'API ligada'}}<div class="profile"><b>HL</b><span><strong>Home
                            Ledger</strong><small>Gestão doméstica</small></span></div>
            </div>
        </aside>
        <main>
            <header>Finanças <i>/</i> <b>Visão geral</b><button @click="load" title="Atualizar">↻</button></header>
            <section class="intro">
                <div><small>O TEU DINHEIRO, COM CLAREZA</small>
                    <h1>Visão geral</h1>
                    <p>Acompanha as tuas finanças num só lugar.</p>
                </div>
                <div class="actions"><button class="outline" @click="create('income')">＋ Nova receita</button><button
                        class="primary" @click="create('expense')">＋ Nova despesa</button></div>
            </section>
            <section class="filters"><b>☷ &nbsp; Período e filtros</b><label>De <input v-model="filters.from"
                        type="date" @change="load"></label><label>Até <input v-model="filters.to" type="date"
                        @change="load"></label><label>Categoria <select v-model="filters.category" @change="load">
                        <option value="">Todas as categorias</option>
                        <option v-for="c in categories" :key="c.id" :value="String(c.id)">{{ c.name }}</option>
                    </select></label><button class="clear"
                    @click="filters.from = ''; filters.to = ''; filters.category = ''; load()">Limpar filtros</button></section>
            <div v-if="error" class="error"><b>Não foi possível carregar os dados.</b> {{ error }} <small>Verifica se a
                    API está ativa e define VITE_API_URL.</small><button class="outline" @click="load">Tentar
                    novamente</button></div>
            <div v-if="notice" class="notice">{{ notice }} <button @click="notice = ''">×</button></div>
            <section class="stats">
                <article><span>Saldo do período <i
                            class="badge green">↗</i></span><strong>{{ fmt(balance) }}</strong><small>Receitas menos
                        despesas</small></article>
                <article><span>Receitas <i
                            class="badge mint">↓</i></span><strong>{{ fmt(incomeTotal) }}</strong><small>{{ incomes.length }}
                        movimentos</small></article>
                <article><span>Despesas <i
                            class="badge peach">↑</i></span><strong>{{ fmt(expenseTotal) }}</strong><small>{{ filtered.length }}
                        movimentos</small></article>
                <article><span>Média por despesa <i
                            class="badge purple">◷</i></span><strong>{{ fmt(filtered.length ? expenseTotal / filtered.length : 0) }}</strong><small>Neste
                        período</small></article>
            </section>
            <section class="charts" id="relatorios">
                <article class="panel">
                    <div class="panel-title">
                        <div>
                            <h2>Fluxo financeiro</h2><small>Receitas e despesas ao longo do período</small>
                        </div><span class="legend"><i class="inc"></i> Receitas &nbsp; <i class="exp"></i>
                            Despesas</span>
                    </div>
                    <div class="bar-chart">
                        <div class="axis"><span>{{ fmt(monthMax) }}</span><span>{{ fmt(monthMax / 2) }}</span><span>€0</span>
                        </div>
                        <div class="plot">
                            <div class="grid g1"></div>
                            <div class="grid g2"></div>
                            <div class="grid g3"></div>
                            <div v-for="m in months" :key="m.label" class="month">
                                <div class="bars"><i class="bar inc"
                                        :style="{ height: `${Math.max(2, m.income / monthMax * 100)}%` }"
                                        :title="`Receitas: ${fmt(m.income)}`"></i><i class="bar exp"
                                        :style="{ height: `${Math.max(2, m.expense / monthMax * 100)}%` }"
                                        :title="`Despesas: ${fmt(m.expense)}`"></i></div><small>{{ m.label }}</small>
                            </div>
                        </div>
                    </div>
                </article>
                <article class="panel">
                    <div class="panel-title">
                        <div>
                            <h2>Despesas por categoria</h2><small>Distribuição no período</small>
                        </div><span>•••</span>
                    </div>
                    <div v-if="cats.length" class="cat-content">
                        <div class="donut"
                            :style="{ background: `conic-gradient(${cats.map((c, i) => `${c.color} ${cats.slice(0, i).reduce((s, x) => s + x.value, 0) / expenseTotal * 100}% ${cats.slice(0, i + 1).reduce((s, x) => s + x.value, 0) / expenseTotal * 100}%`).join(',')})` }">
                            <div><b>{{ fmt(expenseTotal) }}</b><small>Total gasto</small></div>
                        </div>
                        <div class="cat-list">
                            <div v-for="c in cats" :key="c.name"><span><i
                                        :style="{ background: c.color }"></i>{{ c.name }}</span><b>{{ fmt(c.value) }}</b></div>
                        </div>
                    </div>
                    <div v-else class="empty">Ainda não há despesas neste período.</div>
                </article>
            </section>
            <section class="panel transactions" id="movimentos">
                <div class="panel-title">
                    <div>
                        <h2>Movimentos recentes</h2><small>As últimas receitas e despesas registadas</small>
                    </div><a href="#movimentos">Ver movimentos →</a>
                </div>
                <div v-if="loading" class="empty">A carregar movimentos…</div>
                <div v-else-if="!rows.length" class="empty">Não existem movimentos para os filtros selecionados.</div>
                <div v-else class="table-wrap">
                    <table>
                        <thead>
                            <tr>
                                <th>DESCRIÇÃO</th>
                                <th>CATEGORIA</th>
                                <th>DATA</th>
                                <th>TIPO</th>
                                <th class="right">VALOR</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="r in rows.slice(0, 8)" :key="`${r.kind}-${r.id}`">
                                <td><b class="row-icon"
                                        :class="r.kind === 'Receita' ? 'income' : 'expense'">{{ r.kind === 'Receita' ? '↓' : '↑' }}</b><span
                                        class="desc"><strong>{{ r.name }}</strong><small>{{ r.detail }}</small></span></td>
                                <td><span class="pill">{{ r.category }}</span></td>
                                <td>{{ dateFmt(r.date) }}</td>
                                <td :class="r.kind === 'Receita' ? 'pos' : 'neg'">{{ r.kind }}</td>
                                <td class="right amount" :class="{ pos: r.amount > 0 }">{{ r.amount > 0 ? '+' : '−' }}
                                    {{ fmt(Math.abs(r.amount)) }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </section>
            <footer>Home Ledger · As tuas finanças, em equilíbrio.</footer>
        </main>
        <div v-if="modal" class="backdrop" @click.self="modal = false">
            <form class="dialog" @submit.prevent="save">
                <div class="dialog-head">
                    <div><small>NOVO MOVIMENTO</small>
                        <h2>{{ kind === 'expense' ? 'Registar despesa' : 'Registar receita' }}</h2>
                    </div><button type="button" @click="modal = false">×</button>
                </div><label>Data<input v-model="form.date" type="date" required></label><template
                    v-if="kind === 'expense'"><label>Categoria<select v-model="form.category" required>
                            <option value="" disabled>Seleciona categoria</option>
                            <option v-for="c in categories" :key="c.id" :value="String(c.id)">{{ c.name }}</option>
                        </select></label><label>Origem<select v-model="form.source" required>
                            <option value="" disabled>Seleciona origem</option>
                            <option v-for="s in sources" :key="s.id" :value="String(s.id)">{{ s.name }}</option>
                        </select></label><label>Tipo de pagamento<select v-model="form.payment" required>
                            <option value="" disabled>Seleciona pagamento</option>
                            <option v-for="p in payments" :key="p.id" :value="String(p.id)">{{ p.description }}</option>
                        </select></label></template><label v-else>Descrição<input v-model="form.description"
                        placeholder="Ex.: Salário" required></label>
                <div class="form-row"><label>{{ kind === 'expense' ? 'Valor da compra' : 'Valor' }}<span class="euro"><input
                                v-model="form.amount" type="number" min="0.01" step="0.01" placeholder="0,00"
                                required>€</span></label><label v-if="kind === 'expense'">Desconto<span
                            class="euro"><input v-model="form.discount" type="number" min="0" step="0.01"
                                placeholder="0,00">€</span></label></div>
                <small v-if="kind === 'expense'" class="total">Total:
                    {{ fmt(Math.max(0, Number(form.amount || 0) - Number(form.discount || 0))) }}</small>
                <div class="dialog-actions"><button type="button" class="outline"
                        @click="modal = false">Cancelar</button><button class="primary" :disabled="saving">{{ saving ? 'A guardar…':'Guardar movimento'}}</button></div>
            </form>
        </div>
    </div>
</template>
