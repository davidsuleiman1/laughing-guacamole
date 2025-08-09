// Исходные демо-данные + работа с localStorage

const DEFAULT_STAGES = ["Новый","Скрин","Интервью","Оффер","Выход"];

const seed = {
  vacancies: [
    { id: "v1", title: "Frontend разработчик", stages: DEFAULT_STAGES, city: "Москва", priority: "Высокий" },
    { id: "v2", title: "HR Generalist", stages: DEFAULT_STAGES, city: "Санкт‑Петербург", priority: "Средний" }
  ],
  candidates: [
    { id:"c1", name:"Анна Крылова", position:"Frontend DEV", phone:"+7 999 111‑22‑33", email:"anna@ex.com",
      tg:"@anna_dev", wa:"+79991112233", city:"Москва", salary:180000, tags:["React","TS"], vacancyId:"v1", stage:"Скрин", notes:"Сильная по React, есть pet‑проекты." },
    { id:"c2", name:"Игорь Соколов", position:"HR Generalist", phone:"+7 916 555‑66‑77", email:"igor@ex.com",
      tg:"@igor_hr", wa:"+79165556677", city:"СПб", salary:120000, tags:["HR","Sourcing"], vacancyId:"v2", stage:"Новый", notes:"Готов к переезду." },
    { id:"c3", name:"Мария Лебедева", position:"Frontend DEV", phone:"+7 903 444‑55‑66", email:"mleb@ex.com",
      tg:"@mleb", wa:"+79034445566", city:"Москва", salary:210000, tags:["Next.js"], vacancyId:"v1", stage:"Интервью", notes:"Ожидание результата теста." }
  ],
  clients: [
    { id:"cl1", company:"ООО ТехСфера", contact:"Павел Д.", email:"pd@tech.ru", phone:"+7 495 000‑00‑00", comment:"Фокус на фронт" }
  ],
  messages: [
    { id:"m1", candidateId:"c1", channel:"Telegram", from:"Recruiter", text:"Привет! Удобно соз созвониться завтра?", ts: Date.now()-86400000 },
    { id:"m2", candidateId:"c1", channel:"Email", from:"Candidate", text:"Да, после 15:00.", ts: Date.now()-80000000 }
  ]
};

const loadState = () => {
  const raw = localStorage.getItem("hireflow_state");
  if (!raw) return JSON.parse(JSON.stringify(seed));
  try { return JSON.parse(raw); } catch { return JSON.parse(JSON.stringify(seed)); }
};

const saveState = (s) => localStorage.setItem("hireflow_state", JSON.stringify(s));

window.DB = {
  getState: loadState,
  saveState,
  reset: () => { localStorage.removeItem("hireflow_state"); location.reload(); },
  stages: DEFAULT_STAGES
};
