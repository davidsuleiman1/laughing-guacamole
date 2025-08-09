// Простой SPA‑роутер и экраны: Дашборд, Канбан, Список, Карточка, Заказчики, Отчёты

const $ = (sel, root=document) => root.querySelector(sel);
const $$ = (sel, root=document) => Array.from(root.querySelectorAll(sel));
const format = {
  money: v => new Intl.NumberFormat('ru-RU').format(v) + " ₽",
  date: ts => new Date(ts).toLocaleString('ru-RU')
};

const Router = {
  routes: {},
  init(){
    window.addEventListener('hashchange', ()=>this.resolve());
    if (!location.hash) location.hash = '#/dashboard';
    this.resolve();
  },
  on(path, handler){ this.routes[path]=handler; },
  resolve(){
    const hash = location.hash || '#/dashboard';
    const [_, path, id] = hash.split('/');
    const full = `/${path}` + (id ? '/:id' : '');
    const state = DB.getState();
    (this.routes[full] || this.routes['/404'])({ state, params:{ id } });
  }
};

// Навигация подсветка
const highlightNav = () => {
  const path = location.hash.split('?')[0];
  $$('.nav-link').forEach(a => {
    a.classList.toggle('active', a.getAttribute('href') === path);
  });
};

// РЕНДЕРЫ

function renderDashboard({state}){
  const totalCandidates = state.candidates.length;
  const totalClients = state.clients.length;
  const openVacancies = state.vacancies.length;
  const byStage = DB.stages.map(s => ({stage:s, count: state.candidates.filter(c=>c.stage===s).length}));
  const offerCount = state.candidates.filter(c=>c.stage==='Оффер').length;
  const hiredCount = state.candidates.filter(c=>c.stage==='Выход').length;
  const conv = totalCandidates ? Math.round((hiredCount/totalCandidates)*100) : 0;

  $('#app').innerHTML = `
    <h2 class="section-title">Дашборд</h2>
    <div class="row cols-4">
      <div class="card kpi"><div><h3>Кандидаты</h3><div class="value">${totalCandidates}</div></div></div>
      <div class="card kpi"><div><h3>Заказчики</h3><div class="value">${totalClients}</div></div></div>
      <div class="card kpi"><div><h3>Вакансии</h3><div class="value">${openVacancies}</div></div></div>
      <div class="card kpi"><div><h3>Конверсия</h3><div class="value">${conv}%</div></div></div>
    </div>

    <div class="row cols-2" style="margin-top:16px">
      <div class="card">
        <h3 class="section-title">Воронка</h3>
        <div class="row cols-1">
          ${byStage.map(x=>`
            <div>
              <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:6px">
                <strong>${x.stage}</strong><span class="badge">${x.count}</span>
              </div>
              <div class="progress"><span style="width:${totalCandidates? (x.count/totalCandidates*100).toFixed(0):0}%"></span></div>
            </div>
          `).join('')}
        </div>
      </div>
      <div class="card">
        <h3 class="section-title">На контроле</h3>
        <ul>
          <li>Офферы: <strong>${offerCount}</strong></li>
          <li>Выходы (приняты): <strong>${hiredCount}</strong></li>
          <li>Новых сегодня: <strong>${state.candidates.filter(c=>c.createdAt && new Date(c.createdAt).toDateString()===new Date().toDateString()).length}</strong></li>
        </ul>
        <div style="margin-top:12px;display:flex;gap:8px;flex-wrap:wrap">
          <a class="btn" href="#/kanban">Канбан</a>
          <a class="btn secondary" href="#/candidates">Список</a>
          <a class="btn ghost" href="#/reports">Отчёты</a>
        </div>
      </div>
    </div>
  `;
  highlightNav();
}

function renderKanban({state}){
  // Группировка по стадиям
  const grouped = Object.fromEntries(DB.stages.map(s=>[s, []]));
  state.candidates.forEach(c => grouped[c.stage]?.push(c));

  $('#app').innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h2 class="section-title">Канбан по вакансиям</h2>
      <div class="toolbar">
        <select id="vacFilter" class="select">
          <option value="">Все вакансии</option>
          ${state.vacancies.map(v=>`<option value="${v.id}">${v.title}</option>`).join('')}
        </select>
        <button class="btn" id="resetData">Сброс демо‑данных</button>
      </div>
    </div>
    <div class="kanban">
      ${DB.stages.map(stage=>`
        <div class="column" data-stage="${stage}">
          <div class="column-header">
            <strong>${stage}</strong>
            <span class="badge">${grouped[stage].length}</span>
          </div>
          <div class="droppable" data-stage="${stage}">
            ${grouped[stage].map(c=>CardItem(c, state)).join('')}
          </div>
        </div>
      `).join('')}
    </div>
  `;

  // DnD
  let draggingId = null;
  $$('.card-item').forEach(el=>{
    el.addEventListener('dragstart', e=>{ draggingId = el.dataset.id; el.classList.add('dragging'); });
    el.addEventListener('dragend', e=>{ draggingId = null; el.classList.remove('dragging'); });
  });
  $$('.droppable').forEach(d=>{
    d.addEventListener('dragover', e=>e.preventDefault());
    d.addEventListener('drop', e=>{
      e.preventDefault();
      if(!draggingId) return;
      const newStage = d.dataset.stage;
      const st = DB.getState();
      const idx = st.candidates.findIndex(x=>x.id===draggingId);
      if (idx>-1) {
        st.candidates[idx].stage = newStage;
        DB.saveState(st);
        renderKanban({state: st});
      }
    });
  });

  // Фильтр по вакансии
  $('#vacFilter').addEventListener('change', (e)=>{
    const vId = e.target.value;
    const st = DB.getState();
    const filtered = vId ? {...st, candidates: st.candidates.filter(c=>c.vacancyId===vId)} : st;
    renderKanban({state: filtered});
    $('#vacFilter').value = vId;
  });

  $('#resetData').addEventListener('click', ()=> DB.reset());

  highlightNav();
}

function CardItem(c, state){
  const vac = state.vacancies.find(v=>v.id===c.vacancyId);
  return `
    <div class="card-item" draggable="true" data-id="${c.id}">
      <div style="display:flex;justify-content:space-between;gap:8px">
        <div>
          <div style="font-weight:700"><a class="link" href="#/candidate/${c.id}">${c.name}</a></div>
          <div class="muted" style="font-size:.9rem;color:#475569">${c.position}${vac? ' • '+vac.title:''}</div>
        </div>
        <div><span class="badge">${c.city||''}</span></div>
      </div>
      <div style="margin-top:8px">${(c.tags||[]).map(t=>`<span class="tag">${t}</span>`).join('')}</div>
    </div>
  `;
}

function renderCandidates({state}){
  $('#app').innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h2 class="section-title">Список кандидатов</h2>
      <div class="toolbar">
        <input id="search" class="input" placeholder="Поиск по имени, навыкам, email" />
        <select id="stage" class="select">
          <option value="">Все стадии</option>
          ${DB.stages.map(s=>`<option>${s}</option>`).join('')}
        </select>
        <a class="btn" href="#/kanban">Канбан</a>
      </div>
    </div>

    <div class="card">
      <table class="table">
        <thead>
          <tr><th>Имя</th><th>Позиция</th><th>Стадия</th><th>Город</th><th>Ожидания</th><th>Вакансия</th></tr>
        </thead>
        <tbody id="rows">
          ${state.candidates.map(c=>RowCandidate(c,state)).join('')}
        </tbody>
      </table>
      ${state.candidates.length? '' : `<div class="empty">Нет кандидатов</div>`}
    </div>
  `;

  const rerender = ()=>{
    const q = ($('#search').value || '').toLowerCase();
    const stg = $('#stage').value || '';
    const st = DB.getState();
    const filtered = st.candidates.filter(c=>{
      const line = [c.name,c.position,c.email,(c.tags||[]).join(' ')].join(' ').toLowerCase();
      const okQ = !q || line.includes(q);
      const okS = !stg || c.stage===stg;
      return okQ && okS;
    });
    $('#rows').innerHTML = filtered.map(c=>RowCandidate(c, st)).join('');
  };

  $('#search').addEventListener('input', rerender);
  $('#stage').addEventListener('change', rerender);

  highlightNav();
}

function RowCandidate(c, state){
  const vac = state.vacancies.find(v=>v.id===c.vacancyId);
  return `
    <tr>
      <td><a class="link" href="#/candidate/${c.id}">${c.name}</a></td>
      <td>${c.position || ''}</td>
      <td>${c.stage}</td>
      <td>${c.city || ''}</td>
      <td>${c.salary ? format.money(c.salary) : '—'}</td>
      <td>${vac ? vac.title : '—'}</td>
    </tr>
  `;
}

function renderCandidateCard({state, params}){
  const c = state.candidates.find(x=>x.id===params.id);
  if(!c){
    $('#app').innerHTML = `<div class="card empty">Кандидат не найден</div>`;
    return;
  }
  const vac = state.vacancies.find(v=>v.id===c.vacancyId);

  $('#app').innerHTML = `
    <div class="toolbar" style="justify-content:space-between">
      <a class="btn secondary" href="#/candidates">Назад</a>
      <div style="display:flex;gap:8px">
        <a class="btn" href="https://t.me/${(c.tg||'').replace('@','')}" target="_blank">Написать в TG</a>
        <a class="btn" href="https://wa.me/${(c.wa||'').replace(/\D/g,'')}" target="_blank">Написать в WA</a>
      </div>
    </div>

    <div class="row cols-2">
      <div class="card">
        <h2 class="section-title">Карточка кандидата</h2>
        <div class="row cols-2">
          <div class="field"><div class="label">Имя</div><input class="input" id="fName" value="${c.name||''}"></div>
          <div class="field"><div class="label">Позиция</div><input class="input" id="fPos" value="${c.position||''}"></div>
          <div class="field"><div class="label">Email</div><input class="input" id="fEmail" value="${c.email||''}"></div>
          <div class="field"><div class="label">Телефон</div><input class="input" id="fPhone" value="${c.phone||''}"></div>
          <div class="field"><div class="label">Город</div><input class="input" id="fCity" value="${c.city||''}"></div>
          <div class="field"><div class="label">Ожидания по ЗП</div><input class="input" id="fSalary" type="number" value="${c.salary||''}"></div>
          <div class="field"><div class="label">Вакансия</div>
            <select class="select" id="fVac">
              ${state.vacancies.map(v=>`<option value="${v.id}" ${v.id===c.vacancyId?'selected':''}>${v.title}</option>`).join('')}
            </select>
          </div>
          <div class="field"><div class="label">Стадия</div>
            <select class="select" id="fStage">
              ${DB.stages.map(s=>`<option ${s===c.stage?'selected':''}>${s}</option>`).join('')}
            </select>
          </div>
          <div class="field" style="grid-column:1/-1">
            <div class="label">Теги (через запятую)</div>
            <input class="input" id="fTags" value="${(c.tags||[]).join(', ')}">
          </div>
          <div class="field" style="grid-column:1/-1">
            <div class="label">Заметки</div>
            <textarea class="textarea" id="fNotes" rows="4">${c.notes||''}</textarea>
          </div>
        </div>
        <div style="margin-top:12px;display:flex;gap:8px">
          <button class="btn success" id="saveCandidate">Сохранить</button>
          <button class="btn danger" id="deleteCandidate">Удалить</button>
        </div>
      </div>

      <div class="card">
        <h3 class="section-title">Переписка</h3>
        <div id="msgList" style="display:flex;flex-direction:column;gap:8px;max-height:320px;overflow:auto;margin-bottom:12px">
          ${state.messages.filter(m=>m.candidateId===c.id).map(m=>`
            <div class="card" style="padding:10px">
              <div style="font-size:.85rem;color:var(--muted)">${m.channel} • ${m.from} • ${format.date(m.ts)}</div>
              <div>${m.text}</div>
            </div>
          `).join('')}
        </div>
        <div class="field">
          <div class="label">Новое сообщение</div>
          <textarea class="textarea" id="newMsg" rows="3" placeholder="Напишите сообщение кандидату..."></textarea>
        </div>
        <div style="display:flex;gap:8px;margin-top:8px">
          <select class="select" id="msgChannel" style="max-width:200px">
            <option>Telegram</option><option>WhatsApp</option><option>Email</option>
          </select>
          <button class="btn" id="sendMsg">Отправить</button>
        </div>
      </div>
    </div>
  `;

  $('#saveCandidate').addEventListener('click', ()=>{
    const st = DB.getState();
    const i = st.candidates.findIndex(x=>x.id===c.id);
    const upd = {
      ...c,
      name: $('#fName').value.trim(),
      position: $('#fPos').value.trim(),
      email: $('#fEmail').value.trim(),
      phone: $('#fPhone').value.trim(),
      city: $('#fCity').value.trim(),
      salary: parseInt($('#fSalary').value,10)||null,
      vacancyId: $('#fVac').value,
      stage: $('#fStage').value,
      tags: $('#fTags').value.split(',').map(x=>x.trim()).filter(Boolean),
      notes: $('#fNotes').value
    };
    st.candidates[i] = upd;
    DB.saveState(st);
    renderCandidateCard({state: st, params:{id:c.id}});
  });

  $('#deleteCandidate').addEventListener('click', ()=>{
    if(!confirm('Удалить кандидата?')) return;
    const st = DB.getState();
    st.candidates = st.candidates.filter(x=>x.id!==c.id);
    st.messages = st.messages.filter(m=>m.candidateId!==c.id);
    DB.saveState(st);
    location.hash = '#/candidates';
  });

  $('#sendMsg').addEventListener('click', ()=>{
    const text = $('#newMsg').value.trim();
    if(!text) return;
    const ch = $('#msgChannel').value;
    const st = DB.getState();
    st.messages.push({ id:'m'+(Date.now()), candidateId:c.id, channel:ch, from:'Recruiter', text, ts:Date.now() });
    DB.saveState(st);
    renderCandidateCard({state: st, params:{id:c.id}});
  });
}

function renderClients({state}){
  $('#app').innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h2 class="section-title">Заказчики</h2>
      <div class="toolbar">
        <button class="btn" id="addClient">Добавить заказчика</button>
      </div>
    </div>

    <div class="row cols-2">
      <div class="card">
        <h3 class="section-title">Добавление</h3>
        <div class="row cols-2">
          <div class="field"><div class="label">Компания</div><input class="input" id="clCompany" placeholder="ООО Пример"></div>
          <div class="field"><div class="label">Контактное лицо</div><input class="input" id="clContact" placeholder="Иван И."></div>
          <div class="field"><div class="label">Email</div><input class="input" id="clEmail" placeholder="contact@company.ru"></div>
          <div class="field"><div class="label">Телефон</div><input class="input" id="clPhone" placeholder="+7 ..."></div>
          <div class="field" style="grid-column:1/-1"><div class="label">Комментарий</div><textarea class="textarea" id="clComment" rows="3"></textarea></div>
        </div>
        <div style="margin-top:12px">
          <button class="btn success" id="saveClient">Сохранить заказчика</button>
        </div>
      </div>

      <div class="card">
        <h3 class="section-title">Список заказчиков</h3>
        <table class="table">
          <thead><tr><th>Компания</th><th>Контакт</th><th>Email</th><th>Телефон</th><th></th></tr></thead>
          <tbody id="clRows">
            ${state.clients.map(ClientRow).join('')}
          </tbody>
        </table>
        ${state.clients.length? '' : `<div class="empty">Пока нет заказчиков</div>`}
      </div>
    </div>
  `;

  $('#saveClient').addEventListener('click', ()=>{
    const st = DB.getState();
    const client = {
      id: 'cl'+Date.now(),
      company: $('#clCompany').value.trim(),
      contact: $('#clContact').value.trim(),
      email: $('#clEmail').value.trim(),
      phone: $('#clPhone').value.trim(),
      comment: $('#clComment').value.trim()
    };
    if(!client.company){ alert('Укажи название компании'); return; }
    st.clients.push(client);
    DB.saveState(st);
    renderClients({state: st});
  });

  $$('#clRows .btn.danger').forEach(btn=>{
    btn.addEventListener('click', (e)=>{
      const id = e.target.dataset.id;
      const st = DB.getState();
      st.clients = st.clients.filter(c=>c.id!==id);
      DB.saveState(st);
      renderClients({state: st});
    });
  });

  highlightNav();
}

function ClientRow(c){
  return `
    <tr>
      <td>${c.company}</td>
      <td>${c.contact||'—'}</td>
      <td>${c.email||'—'}</td>
      <td>${c.phone||'—'}</td>
      <td style="text-align:right"><button class="btn danger" data-id="${c.id}">Удалить</button></td>
    </tr>
  `;
}

function renderReports({state}){
  const byVac = state.vacancies.map(v=>{
    const list = state.candidates.filter(c=>c.vacancyId===v.id);
    const stages = Object.fromEntries(DB.stages.map(s=>[s, list.filter(c=>c.stage===s).length]));
    const hired = stages["Выход"]||0;
    const conv = list.length ? Math.round(hired/list.length*100) : 0;
    return { vac:v, total:list.length, stages, conv };
  });

  $('#app').innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h2 class="section-title">Отчёты</h2>
      <div class="toolbar">
        <button class="btn" id="exportCsv">Экспорт CSV</button>
      </div>
    </div>

    <div class="card">
      <table class="table">
        <thead>
          <tr>
            <th>Вакансия</th>
            ${DB.stages.map(s=>`<th>${s}</th>`).join('')}
            <th>Итого</th>
            <th>Конверсия</th>
          </tr>
        </thead>
        <tbody id="repRows">
          ${byVac.map(x=>`
            <tr>
              <td>${x.vac.title}</td>
              ${DB.stages.map(s=>`<td>${x.stages[s]||0}</td>`).join('')}
              <td>${x.total}</td>
              <td>${x.conv}%</td>
            </tr>
          `).join('')}
        </tbody>
      </table>
      ${byVac.length? '' : `<div class="empty">Нет данных для отчёта</div>`}
    </div>
  `;

  $('#exportCsv').addEventListener('click', ()=>{
    const headers = ['Вакансия', ...DB.stages, 'Итого','Конверсия'];
    const rows = byVac.map(x=>[
      x.vac.title, ...DB.stages.map(s=>x.stages[s]||0), x.total, `${x.conv}%`
    ]);
    const csv = [headers, ...rows].map(r=>r.map(val=>`"${String(val).replace(/"/g,'""')}"`).join(',')).join('\n');
    const blob = new Blob([csv], {type:'text/csv;charset=utf-8;'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'hireflow-report.csv'; a.click();
    URL.revokeObjectURL(url);
  });

  highlightNav();
}

// Роуты
Router.on('/dashboard', renderDashboard);
Router.on('/kanban', renderKanban);
Router.on('/candidates', renderCandidates);
Router.on('/candidate/:id', renderCandidateCard);
Router.on('/clients', renderClients);
Router.on('/reports', renderReports);
Router.on('/404', () => { $('#app').innerHTML = `<div class="card empty">Страница не найдена</div>`; });

// Старт
Router.init();
