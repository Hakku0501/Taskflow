<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1.0,viewport-fit=cover"/>
  <title>TaskFlow — Team Operations</title>

  <!-- ── PWA META ── -->
  <meta name="apple-mobile-web-app-capable" content="yes"/>
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent"/>
  <meta name="apple-mobile-web-app-title" content="TaskFlow"/>
  <meta name="theme-color" content="#0c0c0c"/>
  <meta name="description" content="TaskFlow — Team Operations Platform"/>
  <meta name="mobile-web-app-capable" content="yes"/>

  <!-- PWA Manifest injected via JS below (works without a server) -->

  <!-- ── FONTS ── -->
  <link rel="preconnect" href="https://fonts.googleapis.com"/>
  <link href="https://fonts.googleapis.com/css2?family=Space+Mono:ital,wght@0,400;0,700;1,400&family=Syne:wght@400;600;700;800&display=swap" rel="stylesheet"/>

  <style>
    /* ── RESET ── */
    *{box-sizing:border-box;margin:0;padding:0}
    html,body{height:100%;overflow:hidden;background:#0c0c0c}
    body{color:#e0e0e0;font-family:'Space Mono',monospace;-webkit-font-smoothing:antialiased;-webkit-tap-highlight-color:transparent}
    input,textarea,select,button{font-family:'Space Mono',monospace;outline:none}
    a{color:inherit;text-decoration:none}
    button{cursor:pointer}

    /* ── SCROLLBAR ── */
    ::-webkit-scrollbar{width:3px;height:3px}
    ::-webkit-scrollbar-track{background:#0c0c0c}
    ::-webkit-scrollbar-thumb{background:#2a2a2a;border-radius:2px}

    /* ── ANIMATIONS ── */
    .fade-in{animation:fadeIn 0.22s ease}
    @keyframes fadeIn{from{opacity:0;transform:translateY(5px)}to{opacity:1;transform:translateY(0)}}
    .slide-in{animation:slideIn 0.2s ease}
    @keyframes slideIn{from{opacity:0;transform:translateX(8px)}to{opacity:1;transform:translateX(0)}}
    .pulse{animation:pulse 2s ease infinite}
    @keyframes pulse{0%,100%{opacity:1}50%{opacity:0.35}}

    /* ── SAFE AREA ── */
    :root{--sat:env(safe-area-inset-top,0px);--sab:env(safe-area-inset-bottom,0px);--sal:env(safe-area-inset-left,0px);--sar:env(safe-area-inset-right,0px)}

    /* ── LAYOUT ── */
    .app-wrap{display:flex;height:100vh;background:#0c0c0c;overflow:hidden}
    /* Desktop sidebar */
    .sidebar{width:210px;background:#0a0a0a;border-right:1px solid #1e1e1e;display:none;flex-direction:column;padding:20px 0;flex-shrink:0}
    /* Mobile bottom nav */
    .bot-nav{position:fixed;bottom:0;left:0;right:0;background:rgba(10,10,10,0.97);border-top:1px solid #1e1e1e;display:flex;padding:10px 0;padding-bottom:calc(10px + var(--sab));z-index:99;backdrop-filter:blur(10px)}
    /* Main area */
    .main-col{flex:1;overflow:hidden;display:flex;flex-direction:column}
    .page-body{flex:1;overflow:hidden}
    /* scroll helper */
    .scroll{overflow:auto;height:100%;-webkit-overflow-scrolling:touch}

    @media(min-width:768px){
      .sidebar{display:flex!important}
      .bot-nav{display:none!important}
      .page-body{overflow:auto}
    }

    /* ── MISC ── */
    .syne{font-family:'Syne',sans-serif}
    .mono{font-family:'Space Mono',monospace}
    .table-wrap{overflow:auto;-webkit-overflow-scrolling:touch}
    input[type="date"]{color-scheme:dark}
    select{color-scheme:dark}
  </style>
</head>
<body>
<div id="root"></div>

<!-- React -->
<script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

<script type="text/babel">
/*
╔══════════════════════════════════════════════════════════════════════════╗
║  TASKFLOW  —  7-Pillar Web App Architecture                              ║
║                                                                          ║
║  PILLAR 1 · Data Access Layer  (DAL)       — unified storage API         ║
║  PILLAR 2 · Domain / Business Logic Layer  — pure task/clock/user ops   ║
║  PILLAR 3 · Repository Layer               — domain-aware read/write     ║
║  PILLAR 4 · State Management Layer         — React context + useReducer  ║
║  PILLAR 5 · UI Component Library           — shared, styled primitives   ║
║  PILLAR 6 · Presentation / Page Layer      — individual feature pages    ║
║  PILLAR 7 · App Shell / Navigation / PWA   — routing, guards, install    ║
╚══════════════════════════════════════════════════════════════════════════╝

  DEMO ACCOUNT (admin):
    Email    → admin@taskflow.app
    Password → Taskflow@2025
  (Do not expose credentials on the login screen in production)
*/

const { useState, useEffect, useCallback, useRef, useReducer, createContext, useContext } = React;

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 1 — DATA ACCESS LAYER
   All reads/writes go through DAL. Swap to any backend by changing only this.
══════════════════════════════════════════════════════════════════════ */
const DAL = {
  get(key, def = null) {
    try { const r = localStorage.getItem(key); return r !== null ? JSON.parse(r) : def; }
    catch { return def; }
  },
  set(key, val) { try { localStorage.setItem(key, JSON.stringify(val)); } catch {} },
  del(key)      { try { localStorage.removeItem(key); } catch {} },
  exportAll() {
    const keys = ['tf_users','tf_tasks','tf_clock','tf_notifs'];
    const out = {};
    keys.forEach(k => { out[k] = DAL.get(k); });
    return out;
  },
  importAll(data) {
    Object.entries(data).forEach(([k,v]) => { if(v !== undefined) DAL.set(k, v); });
  }
};

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 2 — DOMAIN / BUSINESS LOGIC LAYER
   Pure, framework-agnostic functions. No side effects.
══════════════════════════════════════════════════════════════════════ */
const uid   = () => `${Date.now().toString(36)}${Math.random().toString(36).slice(2,6)}`;
const todayStr = () => new Date().toISOString().slice(0,10);

const PRI = {
  high:{ label:'HIGH', color:'#e05555', glow:'rgba(224,85,85,0.2)', dim:'rgba(224,85,85,0.08)' },
  mid: { label:'MID',  color:'#d4922a', glow:'rgba(212,146,42,0.2)', dim:'rgba(212,146,42,0.08)' },
  low: { label:'LOW',  color:'#5592d4', glow:'rgba(85,146,212,0.2)', dim:'rgba(85,146,212,0.08)' },
};
const STATUS_CFG = {
  open:       { label:'OPEN',        color:'#666',    bg:'rgba(102,102,102,0.1)' },
  inprogress: { label:'IN PROGRESS', color:'#5592d4', bg:'rgba(85,146,212,0.1)'  },
  done:       { label:'DONE',        color:'#5db37e', bg:'rgba(93,179,126,0.1)'  },
};
const AVATAR_COLS = ['#c8a96e','#5592d4','#5db37e','#d45592','#a78bfa','#e67e22','#1abc9c','#e74c3c'];
const DAYS  = ['SUN','MON','TUE','WED','THU','FRI','SAT'];
const MONTHS= ['January','February','March','April','May','June','July','August','September','October','November','December'];

const UserOps = {
  initials:(name)=>(name||'?').split(' ').map(w=>w[0]).join('').toUpperCase().slice(0,2),
  color:(u)=>u?.color||'#c8a96e',
  avatarColor:(idx)=>AVATAR_COLS[idx%AVATAR_COLS.length],
};
const TaskOps = {
  create:(data)=>({ id:uid(), status:'open', createdAt:new Date().toISOString(), ...data }),
  update:(tasks,task)=>tasks.map(t=>t.id===task.id?task:t),
  delete:(tasks,id)=>tasks.filter(t=>t.id!==id),
  toggle:(tasks,id)=>tasks.map(t=>t.id===id?{...t,status:t.status==='done'?'open':'done'}:t),
  forUser:(tasks,uid,role)=>role==='admin'?tasks:tasks.filter(t=>t.assignedTo===uid||t.createdBy===uid),
};
const ClockOps = {
  create:(data)=>({
    id:uid(), timestamp:new Date().toISOString(), date:todayStr(),
    time:new Date().toLocaleTimeString('en-AU',{hour:'2-digit',minute:'2-digit',second:'2-digit'}),
    ...data
  }),
  isIn:(records,userId)=>{
    const r=records.filter(x=>x.userId===userId&&x.date===todayStr());
    return r[r.length-1]?.type==='in';
  },
  duration:(records)=>{
    let total=0,lastIn=null;
    records.forEach(r=>{
      if(r.type==='in')lastIn=new Date(r.timestamp);
      else if(r.type==='out'&&lastIn){total+=new Date(r.timestamp)-lastIn;lastIn=null;}
    });
    if(lastIn)total+=Date.now()-lastIn;
    const h=Math.floor(total/3600000),m=Math.floor((total%3600000)/60000);
    return `${h}h ${m}m`;
  },
};
const NotifOps = {
  create:(data)=>({ id:uid(), createdAt:new Date().toISOString(), read:false, ...data }),
  forUser:(notifs,userId)=>notifs.filter(n=>n.toUserId===userId||n.toUserId==='all'),
  unread:(notifs,userId)=>NotifOps.forUser(notifs,userId).filter(n=>!n.read).length,
};

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 3 — REPOSITORY LAYER
   Wraps DAL with domain-specific read/write. Owns persistence contracts.
══════════════════════════════════════════════════════════════════════ */
const SYSTEM_ADMIN = {
  id:'sadmin_001', name:'Admin', email:'admin@taskflow.app',
  password:'Taskflow@2025', role:'admin',
  initials:'AD', color:'#c8a96e', joinedAt:new Date().toISOString(),
};

const Repo = {
  getUsers(){
    const u=DAL.get('tf_users',null);
    if(!u){ DAL.set('tf_users',[SYSTEM_ADMIN]); return [SYSTEM_ADMIN]; }
    return u.find(x=>x.id===SYSTEM_ADMIN.id)?u:[SYSTEM_ADMIN,...u];
  },
  setUsers:(u)=>DAL.set('tf_users',u),
  getTasks:()=>DAL.get('tf_tasks',[]),
  setTasks:(t)=>DAL.set('tf_tasks',t),
  getClock:()=>DAL.get('tf_clock',[]),
  setClock:(c)=>DAL.set('tf_clock',c),
  getNotifs:()=>DAL.get('tf_notifs',[]),
  setNotifs:(n)=>DAL.set('tf_notifs',n),
  getSession:()=>DAL.get('tf_session',null),
  setSession:(u)=>DAL.set('tf_session',u),
  clearSession:()=>DAL.del('tf_session'),
};

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 4 — STATE MANAGEMENT LAYER
   Central React Context + useReducer. Single source of truth.
══════════════════════════════════════════════════════════════════════ */
const Ctx = createContext(null);
const useApp = () => useContext(Ctx);

function reducer(state, action) {
  switch(action.type) {
    case 'BOOT':      return { ...state, ...action.payload, ready:true };
    case 'SET_USER':  Repo.setSession(action.payload); return { ...state, currentUser:action.payload };
    case 'LOGOUT':    Repo.clearSession(); return { ...state, currentUser:null };
    case 'SET_USERS': Repo.setUsers(action.payload); return { ...state, users:action.payload };
    case 'SET_TASKS': Repo.setTasks(action.payload); return { ...state, tasks:action.payload };
    case 'SET_CLOCK': Repo.setClock(action.payload); return { ...state, clockRecords:action.payload };
    case 'SET_NOTIFS':Repo.setNotifs(action.payload); return { ...state, notifs:action.payload };
    default: return state;
  }
}

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 5 — UI COMPONENT LIBRARY
   Reusable, design-system-level primitives.
══════════════════════════════════════════════════════════════════════ */

/* ── Design tokens ── */
const T={
  bg:'#0c0c0c', s1:'#111', s2:'#161616', s3:'#1e1e1e', s4:'#252525',
  b1:'#1e1e1e', b2:'#2a2a2a', b3:'#363636',
  text:'#e0e0e0', sub:'#888', muted:'#555',
  gold:'#c8a96e', goldDim:'rgba(200,169,110,0.12)',
  green:'#5db37e', red:'#e05555', blue:'#5592d4',
};

/* ── Avatar ── */
function Avatar({ user, size=32 }) {
  return (
    <div style={{width:size,height:size,borderRadius:'50%',background:UserOps.color(user),
      display:'flex',alignItems:'center',justifyContent:'center',
      fontSize:size*0.34,fontWeight:700,color:'#0c0c0c',flexShrink:0,
      letterSpacing:'0.03em',fontFamily:"'Syne',sans-serif",userSelect:'none'}}>
      {user?.initials||UserOps.initials(user?.name||'?')}
    </div>
  );
}

/* ── PriBadge ── */
function PriBadge({ level, small }) {
  const p=PRI[level]||PRI.mid;
  return (
    <span style={{background:p.dim,color:p.color,border:`1px solid ${p.color}44`,
      padding:small?'1px 5px':'2px 8px',fontSize:small?8:10,fontWeight:700,
      letterSpacing:'0.1em',borderRadius:3}}>
      {p.label}
    </span>
  );
}

/* ── StatusBadge ── */
function StatusBadge({ status }) {
  const s=STATUS_CFG[status]||STATUS_CFG.open;
  return (
    <span style={{background:s.bg,color:s.color,border:`1px solid ${s.color}33`,
      padding:'1px 7px',fontSize:9,fontWeight:700,letterSpacing:'0.08em',borderRadius:3}}>
      {s.label}
    </span>
  );
}

/* ── Modal ── */
function Modal({ onClose, title, children, width=460 }) {
  return (
    <div style={{position:'fixed',inset:0,background:'rgba(0,0,0,0.88)',display:'flex',
      alignItems:'center',justifyContent:'center',zIndex:300,padding:16,overflowY:'auto'}}
      onClick={e=>e.target===e.currentTarget&&onClose()}>
      <div className="fade-in" style={{width:'100%',maxWidth:width,background:T.s1,
        border:`1px solid ${T.b2}`,borderRadius:10,padding:24,
        maxHeight:'90vh',overflowY:'auto',margin:'auto'}}>
        <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:22}}>
          <span className="syne" style={{fontSize:14,fontWeight:700,color:T.gold}}>{title}</span>
          <button onClick={onClose} style={{background:'none',border:'none',color:T.muted,fontSize:24,lineHeight:1,padding:'0 2px'}}>×</button>
        </div>
        {children}
      </div>
    </div>
  );
}

/* ── FField ── */
const FField=({label,children,style={}})=>(
  <div style={{display:'flex',flexDirection:'column',gap:5,...style}}>
    <div style={{fontSize:9,color:T.muted,letterSpacing:'0.12em',fontWeight:600}}>{label}</div>
    {children}
  </div>
);

/* ── Shared input styles ── */
const INP={width:'100%',padding:'10px 12px',background:T.s2,border:`1px solid ${T.b2}`,
  color:T.text,fontSize:12,borderRadius:4,transition:'border-color 0.18s'};
const BTNG={padding:'10px 18px',background:T.gold,border:'none',color:'#0c0c0c',
  borderRadius:4,fontSize:10,fontWeight:700,letterSpacing:'0.12em',cursor:'pointer',transition:'opacity 0.15s'};
const BTNGH={padding:'9px 16px',background:'transparent',border:`1px solid ${T.b2}`,
  color:T.sub,borderRadius:4,fontSize:10,letterSpacing:'0.08em',cursor:'pointer'};

/* ── EmptyState ── */
const EmptyState=({icon,text,sub})=>(
  <div style={{padding:'52px 24px',textAlign:'center'}}>
    <div style={{fontSize:36,marginBottom:12}}>{icon}</div>
    <div style={{fontSize:12,color:T.muted}}>{text}</div>
    {sub&&<div style={{fontSize:10,color:'#333',marginTop:4}}>{sub}</div>}
  </div>
);

/* ── Divider ── */
const HR=()=><div style={{height:1,background:T.b1,margin:'4px 0'}}/>;

/* ── Inline focus helper ── */
const focusGold=e=>e.target.style.borderColor=T.gold;
const blurLine =e=>e.target.style.borderColor=T.b2;

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 6 — PRESENTATION / PAGE LAYER
══════════════════════════════════════════════════════════════════════ */

/* ──────────────────────── TASK MODAL ──────────────────────────────── */
function TaskModal({ onClose, onSave, editTask, users, currentUser, defaultDate }) {
  const [title,  setTitle]  = useState(editTask?.title||'');
  const [desc,   setDesc]   = useState(editTask?.desc||'');
  const [pri,    setPri]    = useState(editTask?.priority||'mid');
  const [assign, setAssign] = useState(editTask?.assignedTo||currentUser.id);
  const [date,   setDate]   = useState(editTask?.date||defaultDate||todayStr());
  const [status, setStatus] = useState(editTask?.status||'open');

  const save = () => {
    if(!title.trim())return;
    onSave({
      id:editTask?.id||uid(), title:title.trim(), desc:desc.trim(), priority:pri,
      assignedTo:assign, date, status,
      createdBy:editTask?.createdBy||currentUser.id,
      createdAt:editTask?.createdAt||new Date().toISOString(),
    });
    onClose();
  };

  return (
    <Modal onClose={onClose} title={editTask?'EDIT TASK':'NEW TASK'}>
      <div style={{display:'flex',flexDirection:'column',gap:14}}>
        <FField label="TASK TITLE *">
          <input style={INP} value={title} onChange={e=>setTitle(e.target.value)}
            placeholder="What needs to be done?"
            onFocus={focusGold} onBlur={blurLine} onKeyDown={e=>e.key==='Enter'&&save()}/>
        </FField>
        <FField label="DESCRIPTION">
          <textarea style={{...INP,resize:'vertical',minHeight:72,lineHeight:1.6}}
            value={desc} onChange={e=>setDesc(e.target.value)} placeholder="Optional details..."/>
        </FField>
        <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:12}}>
          <FField label="PRIORITY">
            <div style={{display:'flex',gap:5}}>
              {['high','mid','low'].map(p=>(
                <button key={p} onClick={()=>setPri(p)} style={{flex:1,padding:'7px 3px',borderRadius:4,
                  fontSize:9,fontWeight:700,letterSpacing:'0.08em',
                  background:pri===p?PRI[p].dim:'transparent',
                  border:`1px solid ${pri===p?PRI[p].color:T.b2}`,
                  color:pri===p?PRI[p].color:T.muted,transition:'all 0.14s'}}>
                  {PRI[p].label}
                </button>
              ))}
            </div>
          </FField>
          <FField label="DUE DATE">
            <input type="date" style={INP} value={date} onChange={e=>setDate(e.target.value)}/>
          </FField>
        </div>
        <FField label="ASSIGN TO">
          <select style={{...INP,cursor:'pointer'}} value={assign} onChange={e=>setAssign(e.target.value)}>
            {users.map(u=><option key={u.id} value={u.id}>{u.name} ({u.role})</option>)}
          </select>
        </FField>
        {editTask&&(
          <FField label="STATUS">
            <select style={{...INP,cursor:'pointer'}} value={status} onChange={e=>setStatus(e.target.value)}>
              {Object.entries(STATUS_CFG).map(([k,v])=><option key={k} value={k}>{v.label}</option>)}
            </select>
          </FField>
        )}
        <div style={{display:'flex',gap:10,marginTop:6}}>
          <button style={BTNGH} onClick={onClose}>CANCEL</button>
          <button style={{...BTNG,flex:1}} onClick={save}>SAVE TASK</button>
        </div>
      </div>
    </Modal>
  );
}

/* ──────────────────────── PAGE: CALENDAR ──────────────────────────── */
function CalendarPage() {
  const { tasks, users, currentUser, notifs, dispatch } = useApp();
  const now = new Date();
  const [yr,  setYr]  = useState(now.getFullYear());
  const [mo,  setMo]  = useState(now.getMonth());
  const [sel, setSel] = useState(todayStr());
  const [modal, setModal] = useState(false);
  const [edit,  setEdit]  = useState(null);
  const [flt,   setFlt]   = useState('all');

  const getDs=(d)=>`${yr}-${String(mo+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
  const firstDow=new Date(yr,mo,1).getDay();
  const daysInMo=new Date(yr,mo+1,0).getDate();
  const tsToday=todayStr();

  const visForDay=useCallback((ds)=>{
    const base=TaskOps.forUser(tasks,currentUser.id,currentUser.role);
    return base.filter(t=>t.date===ds&&(flt==='all'||t.assignedTo===currentUser.id));
  },[tasks,currentUser,flt]);

  const selTasks=visForDay(sel);
  const prevMo=()=>mo===0?(setMo(11),setYr(y=>y-1)):setMo(m=>m-1);
  const nextMo=()=>mo===11?(setMo(0),setYr(y=>y+1)):setMo(m=>m+1);
  const getUserById=id=>users.find(u=>u.id===id)||users[0];

  const saveTask=(task)=>{
    const isNew=!tasks.find(t=>t.id===task.id);
    const updated=isNew?[...tasks,task]:TaskOps.update(tasks,task);
    dispatch({type:'SET_TASKS',payload:updated});
    if(isNew&&task.assignedTo&&task.assignedTo!==currentUser.id){
      const n=NotifOps.create({toUserId:task.assignedTo,type:'task_assigned',
        title:'New Task Assigned',
        body:`"${task.title}" was assigned to you by ${currentUser.name}`,taskId:task.id});
      dispatch({type:'SET_NOTIFS',payload:[...notifs,n]});
    }
    setSel(task.date);
  };
  const delTask=id=>dispatch({type:'SET_TASKS',payload:TaskOps.delete(tasks,id)});
  const toggleTask=id=>dispatch({type:'SET_TASKS',payload:TaskOps.toggle(tasks,id)});

  return (
    <div style={{display:'flex',height:'100%',overflow:'hidden'}}>
      {/* Calendar grid */}
      <div style={{flex:1,padding:20,overflow:'auto',minWidth:0}}>
        {/* Controls */}
        <div style={{display:'flex',alignItems:'center',justifyContent:'space-between',marginBottom:18,gap:10,flexWrap:'wrap'}}>
          <div style={{display:'flex',alignItems:'center',gap:8}}>
            <button onClick={prevMo} style={{width:30,height:30,background:T.s2,border:`1px solid ${T.b2}`,color:T.sub,borderRadius:4,fontSize:16}}>‹</button>
            <span className="syne" style={{fontSize:17,fontWeight:700,color:T.text,minWidth:170}}>{MONTHS[mo]} {yr}</span>
            <button onClick={nextMo} style={{width:30,height:30,background:T.s2,border:`1px solid ${T.b2}`,color:T.sub,borderRadius:4,fontSize:16}}>›</button>
            <button onClick={()=>{setYr(now.getFullYear());setMo(now.getMonth());setSel(tsToday);}}
              style={{padding:'4px 10px',background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:4,fontSize:9,letterSpacing:'0.1em'}}>TODAY</button>
          </div>
          <div style={{display:'flex',gap:6,flexWrap:'wrap'}}>
            {['all','mine'].map(f=>(
              <button key={f} onClick={()=>setFlt(f)} style={{padding:'5px 12px',borderRadius:4,fontSize:9,letterSpacing:'0.08em',fontWeight:700,
                background:flt===f?T.goldDim:'transparent',border:`1px solid ${flt===f?T.gold:T.b2}`,color:flt===f?T.gold:T.muted}}>
                {f==='all'?'ALL TASKS':'MY TASKS'}
              </button>
            ))}
            <button onClick={()=>{setEdit(null);setModal(true);}} style={{...BTNG,padding:'6px 14px'}}>+ NEW</button>
          </div>
        </div>

        {/* Day headers */}
        <div style={{display:'grid',gridTemplateColumns:'repeat(7,1fr)',gap:2,marginBottom:2}}>
          {DAYS.map(d=><div key={d} style={{padding:'4px 0',fontSize:9,color:'#444',letterSpacing:'0.1em',textAlign:'center'}}>{d}</div>)}
        </div>

        {/* Cells */}
        <div style={{display:'grid',gridTemplateColumns:'repeat(7,1fr)',gap:2}}>
          {Array.from({length:firstDow}).map((_,i)=>(
            <div key={`e${i}`} style={{minHeight:88,background:'#0e0e0e',borderRadius:4}}/>
          ))}
          {Array.from({length:daysInMo}).map((_,i)=>{
            const d=i+1, ds=getDs(d), dt=visForDay(ds);
            const isToday=ds===tsToday, isSel=ds===sel;
            return (
              <div key={d} onClick={()=>setSel(ds)}
                style={{minHeight:88,background:isSel?'#181818':T.s1,
                  border:`1px solid ${isToday?`${T.gold}55`:isSel?T.b3:T.b1}`,
                  borderRadius:4,padding:'6px 7px',cursor:'pointer',transition:'all 0.12s'}}
                onMouseEnter={e=>!isSel&&(e.currentTarget.style.background='#141414')}
                onMouseLeave={e=>!isSel&&(e.currentTarget.style.background=T.s1)}>
                <div style={{fontSize:11,fontWeight:isToday?700:400,color:isToday?T.gold:'#888',marginBottom:4}}>{d}</div>
                <div style={{display:'flex',flexDirection:'column',gap:2}}>
                  {dt.slice(0,3).map(t=>(
                    <div key={t.id} style={{fontSize:9,padding:'2px 5px',borderRadius:2,
                      background:PRI[t.priority].dim,color:PRI[t.priority].color,
                      borderLeft:`2px solid ${PRI[t.priority].color}`,
                      whiteSpace:'nowrap',overflow:'hidden',textOverflow:'ellipsis',
                      opacity:t.status==='done'?0.5:1,textDecoration:t.status==='done'?'line-through':'none'}}>
                      {t.title}
                    </div>
                  ))}
                  {dt.length>3&&<div style={{fontSize:8,color:'#444'}}>+{dt.length-3} more</div>}
                </div>
              </div>
            );
          })}
        </div>
      </div>

      {/* Day panel */}
      <div style={{width:280,borderLeft:`1px solid ${T.b1}`,background:'#0f0f0f',display:'flex',flexDirection:'column',overflow:'hidden',flexShrink:0}}>
        <div style={{padding:'14px 16px',borderBottom:`1px solid ${T.b1}`}}>
          <div className="syne" style={{fontSize:12,fontWeight:700,color:T.text}}>
            {new Date(sel+'T12:00:00').toLocaleDateString('en-US',{weekday:'long',month:'long',day:'numeric'}).toUpperCase()}
          </div>
          <div style={{fontSize:10,color:T.muted,marginTop:2}}>{selTasks.length} task{selTasks.length!==1?'s':''}</div>
        </div>
        <div style={{flex:1,overflow:'auto',padding:'10px 12px'}}>
          {selTasks.length===0?<EmptyState icon="📅" text="Nothing scheduled" sub="Click a date, add a task"/>:
            selTasks.map(t=>{
              const assignee=getUserById(t.assignedTo);
              return (
                <div key={t.id} className="fade-in" style={{background:'#131313',border:`1px solid ${T.b1}`,
                  borderLeft:`3px solid ${PRI[t.priority].color}55`,borderRadius:6,padding:'10px 11px',
                  marginBottom:7,opacity:t.status==='done'?0.6:1}}>
                  <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-start',gap:6,marginBottom:5}}>
                    <div style={{fontSize:12,color:t.status==='done'?T.muted:T.text,
                      textDecoration:t.status==='done'?'line-through':'none',flex:1,lineHeight:1.4}}>{t.title}</div>
                    <PriBadge level={t.priority} small/>
                  </div>
                  {t.desc&&<div style={{fontSize:10,color:T.muted,marginBottom:7,lineHeight:1.5}}>{t.desc}</div>}
                  <div style={{display:'flex',alignItems:'center',justifyContent:'space-between'}}>
                    <div style={{display:'flex',alignItems:'center',gap:5}}>
                      <Avatar user={assignee} size={17}/>
                      <span style={{fontSize:9,color:T.muted}}>{assignee?.name?.split(' ')[0]}</span>
                    </div>
                    <div style={{display:'flex',gap:3}}>
                      <button onClick={()=>toggleTask(t.id)} style={{padding:'2px 6px',fontSize:8,borderRadius:3,cursor:'pointer',
                        background:t.status==='done'?'rgba(93,179,126,0.12)':'transparent',
                        border:`1px solid ${t.status==='done'?'#5db37e44':T.b2}`,
                        color:t.status==='done'?T.green:T.muted}}>
                        {t.status==='done'?'DONE':'OPEN'}
                      </button>
                      <button onClick={()=>{setEdit(t);setModal(true);}} style={{padding:'2px 6px',fontSize:8,background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:3,cursor:'pointer'}}>EDIT</button>
                      <button onClick={()=>delTask(t.id)} style={{padding:'2px 6px',fontSize:8,background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:3,cursor:'pointer'}}
                        onMouseEnter={e=>{e.currentTarget.style.color=T.red;e.currentTarget.style.borderColor=T.red;}}
                        onMouseLeave={e=>{e.currentTarget.style.color=T.muted;e.currentTarget.style.borderColor=T.b2;}}>DEL</button>
                    </div>
                  </div>
                </div>
              );
            })
          }
        </div>
        <div style={{padding:'10px 12px',borderTop:`1px solid ${T.b1}`}}>
          <button onClick={()=>{setEdit(null);setModal(true);}} style={{width:'100%',padding:'9px',background:'transparent',
            border:`1px dashed ${T.b2}`,color:T.muted,borderRadius:4,fontSize:9,letterSpacing:'0.1em',transition:'all 0.14s'}}
            onMouseEnter={e=>{e.currentTarget.style.borderColor=T.gold;e.currentTarget.style.color=T.gold;}}
            onMouseLeave={e=>{e.currentTarget.style.borderColor=T.b2;e.currentTarget.style.color=T.muted;}}>
            + ADD TASK FOR THIS DAY
          </button>
        </div>
      </div>

      {modal&&<TaskModal onClose={()=>{setModal(false);setEdit(null);}} onSave={saveTask} editTask={edit} users={users} currentUser={currentUser} defaultDate={sel}/>}
    </div>
  );
}

/* ──────────────────────── PAGE: TASKS ─────────────────────────────── */
function TasksPage() {
  const { tasks, users, currentUser, notifs, dispatch } = useApp();
  const [modal, setModal] = useState(false);
  const [edit,  setEdit]  = useState(null);
  const [priF,  setPriF]  = useState('all');
  const [staF,  setStaF]  = useState('all');
  const [assF,  setAssF]  = useState('all');
  const [q,     setQ]     = useState('');

  const mine=TaskOps.forUser(tasks,currentUser.id,currentUser.role);
  const filtered=mine.filter(t=>{
    if(priF!=='all'&&t.priority!==priF)return false;
    if(staF!=='all'&&t.status!==staF)return false;
    if(assF!=='all'&&t.assignedTo!==assF)return false;
    if(q&&!t.title.toLowerCase().includes(q.toLowerCase())&&!t.desc?.toLowerCase().includes(q.toLowerCase()))return false;
    return true;
  });
  const grouped={
    open:filtered.filter(t=>t.status==='open'),
    inprogress:filtered.filter(t=>t.status==='inprogress'),
    done:filtered.filter(t=>t.status==='done'),
  };

  const saveTask=(task)=>{
    const isNew=!tasks.find(t=>t.id===task.id);
    const updated=isNew?[...tasks,task]:TaskOps.update(tasks,task);
    dispatch({type:'SET_TASKS',payload:updated});
    if(isNew&&task.assignedTo&&task.assignedTo!==currentUser.id){
      const n=NotifOps.create({toUserId:task.assignedTo,type:'task_assigned',
        title:'New Task Assigned',body:`"${task.title}" assigned by ${currentUser.name}`,taskId:task.id});
      dispatch({type:'SET_NOTIFS',payload:[...notifs,n]});
    }
  };
  const delTask=id=>dispatch({type:'SET_TASKS',payload:TaskOps.delete(tasks,id)});
  const setStatus=(id,status)=>dispatch({type:'SET_TASKS',payload:tasks.map(t=>t.id===id?{...t,status}:t)});
  const getUserById=id=>users.find(u=>u.id===id);

  const chipStyle=(active,color=T.gold)=>({
    padding:'4px 10px',borderRadius:4,fontSize:9,fontWeight:700,letterSpacing:'0.08em',
    background:active?`${color}18`:'transparent',
    border:`1px solid ${active?color:T.b2}`,
    color:active?color:T.muted,cursor:'pointer',transition:'all 0.14s',whiteSpace:'nowrap',
  });

  return (
    <div style={{padding:20,height:'100%',overflow:'auto'}}>
      {/* Header */}
      <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:18,flexWrap:'wrap',gap:10}}>
        <div className="syne" style={{fontSize:18,fontWeight:700,color:T.text}}>TASKS</div>
        <button onClick={()=>{setEdit(null);setModal(true);}} style={{...BTNG,padding:'8px 14px'}}>+ NEW TASK</button>
      </div>

      {/* Search */}
      <input style={{...INP,marginBottom:12,fontSize:11}} value={q} onChange={e=>setQ(e.target.value)}
        placeholder="🔍  Search tasks..." onFocus={focusGold} onBlur={blurLine}/>

      {/* Filters */}
      <div style={{display:'flex',flexDirection:'column',gap:8,marginBottom:18}}>
        <div style={{display:'flex',gap:5,flexWrap:'wrap',alignItems:'center'}}>
          <span style={{fontSize:9,color:'#444',letterSpacing:'0.1em',marginRight:2}}>PRIORITY</span>
          {['all','high','mid','low'].map(p=><button key={p} style={chipStyle(priF===p,p==='all'?T.gold:PRI[p]?.color)} onClick={()=>setPriF(p)}>{p==='all'?'ALL':PRI[p].label}</button>)}
        </div>
        <div style={{display:'flex',gap:5,flexWrap:'wrap',alignItems:'center'}}>
          <span style={{fontSize:9,color:'#444',letterSpacing:'0.1em',marginRight:2}}>STATUS</span>
          {['all','open','inprogress','done'].map(s=><button key={s} style={chipStyle(staF===s,s==='all'?T.gold:STATUS_CFG[s]?.color)} onClick={()=>setStaF(s)}>{s==='all'?'ALL':STATUS_CFG[s].label}</button>)}
        </div>
        {currentUser.role==='admin'&&(
          <div style={{display:'flex',gap:5,flexWrap:'wrap',alignItems:'center'}}>
            <span style={{fontSize:9,color:'#444',letterSpacing:'0.1em',marginRight:2}}>STAFF</span>
            <button style={chipStyle(assF==='all')} onClick={()=>setAssF('all')}>ANYONE</button>
            {users.map(u=><button key={u.id} style={chipStyle(assF===u.id)} onClick={()=>setAssF(u.id)}>{u.name.split(' ')[0].toUpperCase()}</button>)}
          </div>
        )}
      </div>

      {/* Stats */}
      <div style={{display:'grid',gridTemplateColumns:'repeat(3,1fr)',gap:8,marginBottom:20}}>
        {[
          {l:'OPEN',v:mine.filter(t=>t.status==='open').length,c:T.sub},
          {l:'ACTIVE',v:mine.filter(t=>t.status==='inprogress').length,c:T.blue},
          {l:'DONE',v:mine.filter(t=>t.status==='done').length,c:T.green},
        ].map(s=>(
          <div key={s.l} style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:'12px',textAlign:'center'}}>
            <div className="syne" style={{fontSize:22,fontWeight:800,color:s.c}}>{s.v}</div>
            <div style={{fontSize:9,color:'#444',letterSpacing:'0.08em',marginTop:2}}>{s.l}</div>
          </div>
        ))}
      </div>

      {/* Groups */}
      {filtered.length===0?<EmptyState icon="✓" text="No tasks match" sub="Try changing your filters"/>:
        Object.entries(grouped).map(([status,tlist])=>tlist.length>0&&(
          <div key={status} style={{marginBottom:22}}>
            <div style={{display:'flex',alignItems:'center',gap:8,marginBottom:10}}>
              <StatusBadge status={status}/><span style={{fontSize:9,color:'#444'}}>{tlist.length}</span>
            </div>
            {tlist.map(t=>{
              const assignee=getUserById(t.assignedTo);
              const creator=getUserById(t.createdBy);
              const canEdit=currentUser.role==='admin'||t.createdBy===currentUser.id;
              return (
                <div key={t.id} className="fade-in" style={{background:T.s1,border:`1px solid ${T.b1}`,
                  borderLeft:`3px solid ${PRI[t.priority]?.color||'#555'}66`,
                  borderRadius:6,padding:14,marginBottom:6,opacity:t.status==='done'?0.65:1,transition:'opacity 0.2s'}}>
                  <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-start',gap:8,marginBottom:6}}>
                    <div style={{fontSize:13,color:t.status==='done'?T.muted:T.text,
                      textDecoration:t.status==='done'?'line-through':'none',flex:1,fontWeight:500,lineHeight:1.4}}>{t.title}</div>
                    <PriBadge level={t.priority}/>
                  </div>
                  {t.desc&&<div style={{fontSize:11,color:T.muted,marginBottom:8,lineHeight:1.5}}>{t.desc}</div>}
                  <div style={{display:'flex',gap:6,flexWrap:'wrap',marginBottom:8,alignItems:'center'}}>
                    {t.date&&<span style={{fontSize:9,color:'#555',padding:'2px 7px',background:T.s2,borderRadius:3,border:`1px solid ${T.b1}`}}>📅 {t.date}</span>}
                    {assignee&&<div style={{display:'flex',alignItems:'center',gap:4}}><Avatar user={assignee} size={16}/><span style={{fontSize:10,color:T.muted}}>{assignee.name}</span></div>}
                    {creator&&creator.id!==assignee?.id&&<span style={{fontSize:9,color:'#444'}}>by {creator.name}</span>}
                  </div>
                  <div style={{display:'flex',gap:5,alignItems:'center',flexWrap:'wrap'}}>
                    <select value={t.status} onChange={e=>setStatus(t.id,e.target.value)}
                      style={{padding:'3px 8px',background:T.s2,border:`1px solid ${T.b2}`,color:T.sub,fontSize:9,borderRadius:3,cursor:'pointer'}}>
                      {Object.entries(STATUS_CFG).map(([k,v])=><option key={k} value={k}>{v.label}</option>)}
                    </select>
                    {canEdit&&<button onClick={()=>{setEdit(t);setModal(true);}} style={{padding:'3px 8px',fontSize:9,background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:3,cursor:'pointer'}}>EDIT</button>}
                    {canEdit&&<button onClick={()=>delTask(t.id)} style={{padding:'3px 8px',fontSize:9,background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:3,cursor:'pointer'}}
                      onMouseEnter={e=>e.currentTarget.style.color=T.red} onMouseLeave={e=>e.currentTarget.style.color=T.muted}>DELETE</button>}
                  </div>
                </div>
              );
            })}
          </div>
        ))
      }

      {modal&&<TaskModal onClose={()=>{setModal(false);setEdit(null);}} onSave={saveTask} editTask={edit} users={users} currentUser={currentUser}/>}
    </div>
  );
}

/* ──────────────────────── PAGE: CLOCK IN/OUT ──────────────────────── */
function ClockPage() {
  const { clockRecords, currentUser, dispatch } = useApp();
  const [locMsg, setLocMsg]   = useState('');
  const [loading, setLoading] = useState(false);
  const [time, setTime]       = useState(new Date());

  useEffect(()=>{ const t=setInterval(()=>setTime(new Date()),1000); return()=>clearInterval(t); },[]);

  const ts=todayStr();
  const myToday=clockRecords.filter(r=>r.userId===currentUser.id&&r.date===ts);
  const allMy=clockRecords.filter(r=>r.userId===currentUser.id).slice(-50).reverse();
  const isClockedIn=ClockOps.isIn(clockRecords,currentUser.id);
  const lastRec=myToday[myToday.length-1];

  const getGPS=()=>new Promise((res,rej)=>{
    if(!navigator.geolocation)return rej(new Error('Geolocation not supported'));
    navigator.geolocation.getCurrentPosition(
      p=>res({lat:p.coords.latitude,lng:p.coords.longitude,accuracy:Math.round(p.coords.accuracy)}),
      e=>rej(e),{timeout:12000,maximumAge:0}
    );
  });

  const clockAction=async(type)=>{
    setLoading(true); setLocMsg('Acquiring location...');
    try {
      const loc=await getGPS();
      setLocMsg(`📍 ${loc.lat.toFixed(5)}, ${loc.lng.toFixed(5)}`);
      const rec=ClockOps.create({userId:currentUser.id,userName:currentUser.name,type,...loc});
      dispatch({type:'SET_CLOCK',payload:[...clockRecords,rec]});
      setLocMsg(`✓ Clock-${type} at ${loc.lat.toFixed(4)}, ${loc.lng.toFixed(4)} (±${loc.accuracy}m)`);
    } catch(e){
      const rec=ClockOps.create({userId:currentUser.id,userName:currentUser.name,type,lat:null,lng:null,accuracy:null,locationError:e.message||'Unavailable'});
      dispatch({type:'SET_CLOCK',payload:[...clockRecords,rec]});
      setLocMsg('⚠ No GPS — time recorded only');
    }
    setLoading(false);
  };

  const hms=t=>t.toLocaleTimeString('en-AU',{hour:'2-digit',minute:'2-digit',second:'2-digit'});

  return (
    <div style={{padding:28,maxWidth:680,margin:'0 auto',height:'100%',overflow:'auto'}}>
      {/* Big clock */}
      <div style={{textAlign:'center',marginBottom:32}}>
        <div className="syne" style={{fontSize:58,fontWeight:800,letterSpacing:'-0.02em',lineHeight:1,
          color:isClockedIn?T.green:T.text,
          textShadow:isClockedIn?'0 0 48px rgba(93,179,126,0.25)':'none',transition:'all 0.5s'}}>
          {hms(time)}
        </div>
        <div style={{fontSize:10,color:'#444',letterSpacing:'0.14em',marginTop:7}}>
          {time.toLocaleDateString('en-AU',{weekday:'long',day:'numeric',month:'long',year:'numeric'}).toUpperCase()}
        </div>
      </div>

      {/* Status */}
      <div style={{background:T.s1,border:`1px solid ${isClockedIn?`${T.green}33`:T.b2}`,
        borderRadius:10,padding:'28px 24px',textAlign:'center',marginBottom:18,transition:'border-color 0.5s'}}>
        <div style={{display:'inline-flex',alignItems:'center',gap:8,padding:'5px 16px',borderRadius:20,
          background:isClockedIn?'rgba(93,179,126,0.08)':'rgba(224,85,85,0.08)',
          border:`1px solid ${isClockedIn?'#5db37e44':'#e0555544'}`,marginBottom:10}}>
          <div className={isClockedIn?'pulse':''} style={{width:7,height:7,borderRadius:'50%',background:isClockedIn?T.green:T.red}}/>
          <span style={{fontSize:10,fontWeight:700,letterSpacing:'0.12em',color:isClockedIn?T.green:T.red}}>
            {isClockedIn?'CLOCKED IN':'CLOCKED OUT'}
          </span>
        </div>
        {isClockedIn&&lastRec&&(
          <div style={{fontSize:11,color:'#666',marginBottom:18}}>
            Since {lastRec.time} · {ClockOps.duration(myToday)} on shift today
          </div>
        )}
        {!isClockedIn&&<div style={{fontSize:11,color:'#444',marginBottom:18}}>You are currently off shift</div>}

        <div style={{display:'flex',gap:12,justifyContent:'center',flexWrap:'wrap'}}>
          <button onClick={()=>clockAction('in')} disabled={loading||isClockedIn}
            style={{padding:'13px 36px',borderRadius:5,fontSize:11,fontWeight:700,letterSpacing:'0.12em',
              background:isClockedIn?'#1a1a1a':T.green,border:'none',
              color:isClockedIn?'#333':'#0c0c0c',cursor:isClockedIn?'not-allowed':'pointer',transition:'all 0.2s'}}>
            CLOCK IN
          </button>
          <button onClick={()=>clockAction('out')} disabled={loading||!isClockedIn}
            style={{padding:'13px 36px',borderRadius:5,fontSize:11,fontWeight:700,letterSpacing:'0.12em',
              background:!isClockedIn?'#1a1a1a':T.red,border:'none',
              color:!isClockedIn?'#333':'#fff',cursor:!isClockedIn?'not-allowed':'pointer',transition:'all 0.2s'}}>
            CLOCK OUT
          </button>
        </div>
        {locMsg&&<div style={{marginTop:14,fontSize:10,color:'#666',padding:'7px 12px',background:'#0f0f0f',borderRadius:4,border:`1px solid ${T.b1}`}}>{locMsg}</div>}
      </div>

      {/* Today's log */}
      <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20,marginBottom:16}}>
        <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:14}}>TODAY — {ts}</div>
        {myToday.length===0?<EmptyState icon="🕐" text="No records yet today"/>:
          myToday.map(r=>(
            <div key={r.id} style={{display:'flex',alignItems:'center',justifyContent:'space-between',
              padding:'9px 0',borderBottom:`1px solid #1a1a1a`,gap:8,flexWrap:'wrap'}}>
              <div style={{display:'flex',alignItems:'center',gap:10}}>
                <div style={{width:6,height:6,borderRadius:'50%',background:r.type==='in'?T.green:T.red}}/>
                <span style={{fontSize:12,color:T.text}}>{r.time}</span>
                <span style={{fontSize:9,padding:'1px 6px',borderRadius:2,letterSpacing:'0.1em',
                  background:r.type==='in'?'rgba(93,179,126,0.1)':'rgba(224,85,85,0.1)',
                  color:r.type==='in'?T.green:T.red}}>{r.type.toUpperCase()}</span>
              </div>
              <div style={{fontSize:9,color:'#444'}}>
                {r.lat?<a href={`https://maps.google.com/?q=${r.lat},${r.lng}`} target="_blank" rel="noopener" style={{color:T.blue}}>{r.lat.toFixed(4)}, {r.lng.toFixed(4)}</a>:(r.locationError||'No GPS')}
              </div>
            </div>
          ))
        }
        {myToday.length>0&&<div style={{marginTop:10,fontSize:11,color:T.green,textAlign:'right'}}>Total: {ClockOps.duration(myToday)}</div>}
      </div>

      {/* History */}
      <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20}}>
        <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:14}}>RECENT HISTORY</div>
        {allMy.length===0?<EmptyState icon="📋" text="No history yet"/>:
          allMy.map(r=>(
            <div key={r.id} style={{display:'flex',alignItems:'center',justifyContent:'space-between',
              padding:'8px 0',borderBottom:`1px solid #1a1a1a`,gap:8,flexWrap:'wrap'}}>
              <div style={{display:'flex',alignItems:'center',gap:8}}>
                <span style={{fontSize:9,color:'#444',width:58}}>{r.date}</span>
                <span style={{fontSize:11,color:T.text}}>{r.time}</span>
                <span style={{fontSize:9,padding:'1px 5px',borderRadius:2,
                  background:r.type==='in'?'rgba(93,179,126,0.1)':'rgba(224,85,85,0.1)',
                  color:r.type==='in'?T.green:T.red}}>{r.type.toUpperCase()}</span>
              </div>
              {r.lat&&<a href={`https://maps.google.com/?q=${r.lat},${r.lng}`} target="_blank" rel="noopener" style={{fontSize:9,color:T.blue}}>Open map</a>}
            </div>
          ))
        }
      </div>
    </div>
  );
}

/* ──────────────────────── PAGE: NOTIFICATIONS ─────────────────────── */
function NotificationsPage() {
  const { notifs, currentUser, dispatch } = useApp();
  const mine=NotifOps.forUser(notifs,currentUser.id).sort((a,b)=>new Date(b.createdAt)-new Date(a.createdAt));

  const markRead=id=>dispatch({type:'SET_NOTIFS',payload:notifs.map(n=>n.id===id?{...n,read:true}:n)});
  const markAll=()=>dispatch({type:'SET_NOTIFS',payload:notifs.map(n=>(n.toUserId===currentUser.id||n.toUserId==='all')?{...n,read:true}:n)});
  const del=id=>dispatch({type:'SET_NOTIFS',payload:notifs.filter(n=>n.id!==id)});

  const ICONS={task_assigned:'📋',announcement:'📣',clock_reminder:'⏰',user_added:'👤',default:'🔔'};
  const fmt=iso=>new Date(iso).toLocaleString('en-AU',{day:'numeric',month:'short',hour:'2-digit',minute:'2-digit'});

  return (
    <div style={{padding:20,height:'100%',overflow:'auto',maxWidth:680,margin:'0 auto'}}>
      <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:20}}>
        <div className="syne" style={{fontSize:18,fontWeight:700,color:T.text}}>NOTIFICATIONS</div>
        {mine.some(n=>!n.read)&&(
          <button onClick={markAll} style={{fontSize:9,color:T.gold,background:'transparent',
            border:`1px solid ${T.b2}`,borderRadius:4,padding:'5px 12px',cursor:'pointer',letterSpacing:'0.08em'}}>
            MARK ALL READ
          </button>
        )}
      </div>

      {mine.length===0?<EmptyState icon="🔔" text="No notifications" sub="You'll be notified when tasks are assigned"/>:
        mine.map(n=>(
          <div key={n.id} className="fade-in" onClick={()=>markRead(n.id)}
            style={{display:'flex',gap:12,alignItems:'flex-start',
              background:n.read?T.s1:'#121b12',
              border:`1px solid ${n.read?T.b1:'#2a3a2a'}`,
              borderLeft:`3px solid ${n.read?T.b2:T.green}`,
              borderRadius:6,padding:14,marginBottom:8,cursor:'pointer',transition:'all 0.18s'}}>
            <div style={{fontSize:20,flexShrink:0,marginTop:1}}>{ICONS[n.type]||ICONS.default}</div>
            <div style={{flex:1,minWidth:0}}>
              <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-start',gap:8}}>
                <div style={{fontSize:12,fontWeight:n.read?400:700,color:n.read?T.sub:T.text,lineHeight:1.4}}>{n.title}</div>
                {!n.read&&<div style={{width:6,height:6,borderRadius:'50%',background:T.green,flexShrink:0,marginTop:4}}/>}
              </div>
              <div style={{fontSize:11,color:T.muted,marginTop:3,lineHeight:1.5}}>{n.body}</div>
              <div style={{fontSize:9,color:'#444',marginTop:5}}>{fmt(n.createdAt)}</div>
            </div>
            <button onClick={e=>{e.stopPropagation();del(n.id);}}
              style={{background:'none',border:'none',color:'#333',fontSize:18,cursor:'pointer',flexShrink:0,lineHeight:1}}
              onMouseEnter={e=>e.currentTarget.style.color=T.red}
              onMouseLeave={e=>e.currentTarget.style.color='#333'}>×</button>
          </div>
        ))
      }
    </div>
  );
}

/* ──────────────────────── PAGE: PROFILE ───────────────────────────── */
function ProfilePage({ setPage }) {
  const { currentUser, users, tasks, clockRecords, notifs, dispatch } = useApp();
  const [editing,   setEditing]  = useState(false);
  const [name,      setName]     = useState(currentUser.name);
  const [email,     setEmail]    = useState(currentUser.email);
  const [oldPw,     setOldPw]    = useState('');
  const [newPw,     setNewPw]    = useState('');
  const [pwMsg,     setPwMsg]    = useState('');
  const [colorOpen, setColorOpen]= useState(false);
  const [expMsg,    setExpMsg]   = useState('');

  const myTasks=tasks.filter(t=>t.assignedTo===currentUser.id);
  const doneTasks=myTasks.filter(t=>t.status==='done');
  const myClockToday=clockRecords.filter(r=>r.userId===currentUser.id&&r.date===todayStr());
  const isClockedIn=ClockOps.isIn(clockRecords,currentUser.id);

  const saveProfile=()=>{
    if(!name.trim()||!email.trim())return;
    const updated=users.map(u=>u.id===currentUser.id?{...u,name:name.trim(),email:email.trim(),initials:UserOps.initials(name)}:u);
    dispatch({type:'SET_USERS',payload:updated});
    dispatch({type:'SET_USER',payload:{...currentUser,name:name.trim(),email:email.trim(),initials:UserOps.initials(name)}});
    setEditing(false);
  };

  const changePw=()=>{
    const full=users.find(u=>u.id===currentUser.id);
    if(!full||full.password!==oldPw){setPwMsg('Current password incorrect');return;}
    if(newPw.length<6){setPwMsg('New password must be 6+ characters');return;}
    dispatch({type:'SET_USERS',payload:users.map(u=>u.id===currentUser.id?{...u,password:newPw}:u)});
    setPwMsg('Password updated ✓'); setOldPw(''); setNewPw('');
    setTimeout(()=>setPwMsg(''),3000);
  };

  const setColor=(color)=>{
    const updated=users.map(u=>u.id===currentUser.id?{...u,color}:u);
    dispatch({type:'SET_USERS',payload:updated});
    dispatch({type:'SET_USER',payload:{...currentUser,color}});
    setColorOpen(false);
  };

  const exportData=()=>{
    const data=DAL.exportAll();
    const blob=new Blob([JSON.stringify(data,null,2)],{type:'application/json'});
    const a=document.createElement('a');
    a.href=URL.createObjectURL(blob);
    a.download=`taskflow-backup-${todayStr()}.json`;
    a.click();
    setExpMsg('Exported ✓'); setTimeout(()=>setExpMsg(''),3000);
  };

  const importData=()=>{
    const inp=document.createElement('input'); inp.type='file'; inp.accept='.json';
    inp.onchange=e=>{
      const f=e.target.files[0]; if(!f)return;
      const reader=new FileReader();
      reader.onload=ev=>{
        try{DAL.importAll(JSON.parse(ev.target.result));window.location.reload();}
        catch{alert('Invalid backup file');}
      };
      reader.readAsText(f);
    };
    inp.click();
  };

  return (
    <div style={{padding:20,height:'100%',overflow:'auto',maxWidth:580,margin:'0 auto'}}>
      <div className="syne" style={{fontSize:18,fontWeight:700,color:T.text,marginBottom:20}}>PROFILE</div>

      {/* Avatar card */}
      <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:10,padding:22,marginBottom:14,display:'flex',alignItems:'center',gap:18}}>
        <div style={{position:'relative',flexShrink:0}}>
          <Avatar user={currentUser} size={72}/>
          <button onClick={()=>setColorOpen(!colorOpen)}
            style={{position:'absolute',bottom:-3,right:-3,width:22,height:22,borderRadius:'50%',
              background:T.s3,border:`1px solid ${T.b2}`,color:T.sub,fontSize:10,cursor:'pointer',
              display:'flex',alignItems:'center',justifyContent:'center'}}>✏</button>
          {colorOpen&&(
            <div style={{position:'absolute',top:80,left:0,background:T.s2,border:`1px solid ${T.b2}`,
              borderRadius:8,padding:10,display:'flex',flexWrap:'wrap',gap:6,zIndex:50,width:144}}>
              {AVATAR_COLS.map(c=>(
                <button key={c} onClick={()=>setColor(c)}
                  style={{width:28,height:28,borderRadius:'50%',background:c,cursor:'pointer',
                    border:`2px solid ${currentUser.color===c?'#fff':'transparent'}`,transition:'transform 0.14s'}}/>
              ))}
            </div>
          )}
        </div>
        <div style={{flex:1,minWidth:0}}>
          <div className="syne" style={{fontSize:20,fontWeight:700,color:T.text}}>{currentUser.name}</div>
          <div style={{fontSize:11,color:T.sub,marginTop:3,overflow:'hidden',textOverflow:'ellipsis',whiteSpace:'nowrap'}}>{currentUser.email}</div>
          <div style={{marginTop:7}}>
            <span style={{fontSize:9,padding:'2px 8px',borderRadius:3,fontWeight:700,letterSpacing:'0.1em',
              background:currentUser.role==='admin'?T.goldDim:'rgba(85,146,212,0.1)',
              color:currentUser.role==='admin'?T.gold:T.blue,
              border:`1px solid ${currentUser.role==='admin'?`${T.gold}44`:'#5592d444'}`}}>
              {currentUser.role.toUpperCase()}
            </span>
          </div>
        </div>
      </div>

      {/* Stats */}
      <div style={{display:'grid',gridTemplateColumns:'repeat(3,1fr)',gap:8,marginBottom:14}}>
        {[
          {l:'ASSIGNED',v:myTasks.length,c:T.gold},
          {l:'COMPLETED',v:doneTasks.length,c:T.green},
          {l:'TODAY',v:isClockedIn?ClockOps.duration(myClockToday):'–',c:isClockedIn?T.green:T.muted,sm:true},
        ].map(s=>(
          <div key={s.l} style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:'12px 10px',textAlign:'center'}}>
            <div className="syne" style={{fontSize:s.sm?16:22,fontWeight:800,color:s.c}}>{s.v}</div>
            <div style={{fontSize:8,color:'#444',letterSpacing:'0.08em',marginTop:3}}>{s.l}</div>
          </div>
        ))}
      </div>

      {/* Account info */}
      <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20,marginBottom:14}}>
        <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:16}}>
          <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em'}}>ACCOUNT INFO</div>
          <button onClick={()=>setEditing(!editing)} style={{fontSize:9,background:'transparent',borderRadius:4,padding:'4px 10px',cursor:'pointer',letterSpacing:'0.08em',
            color:editing?T.red:T.gold,border:`1px solid ${editing?`${T.red}33`:T.goldDim}`}}>
            {editing?'CANCEL':'EDIT'}
          </button>
        </div>
        {editing?(
          <div style={{display:'flex',flexDirection:'column',gap:12}}>
            <FField label="FULL NAME"><input style={INP} value={name} onChange={e=>setName(e.target.value)} onFocus={focusGold} onBlur={blurLine}/></FField>
            <FField label="EMAIL"><input type="email" style={INP} value={email} onChange={e=>setEmail(e.target.value)} onFocus={focusGold} onBlur={blurLine}/></FField>
            <button onClick={saveProfile} style={{...BTNG,alignSelf:'flex-start'}}>SAVE CHANGES</button>
          </div>
        ):(
          [{l:'FULL NAME',v:currentUser.name},{l:'EMAIL',v:currentUser.email},{l:'ROLE',v:currentUser.role.toUpperCase()}].map(f=>(
            <div key={f.l} style={{display:'flex',justifyContent:'space-between',padding:'8px 0',borderBottom:`1px solid ${T.b1}`}}>
              <span style={{fontSize:9,color:'#444',letterSpacing:'0.08em'}}>{f.l}</span>
              <span style={{fontSize:11,color:T.sub}}>{f.v}</span>
            </div>
          ))
        )}
      </div>

      {/* Change password */}
      <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20,marginBottom:14}}>
        <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:16}}>CHANGE PASSWORD</div>
        <div style={{display:'flex',flexDirection:'column',gap:12}}>
          <FField label="CURRENT PASSWORD"><input type="password" style={INP} value={oldPw} onChange={e=>setOldPw(e.target.value)} onFocus={focusGold} onBlur={blurLine}/></FField>
          <FField label="NEW PASSWORD"><input type="password" style={INP} value={newPw} onChange={e=>setNewPw(e.target.value)} onFocus={focusGold} onBlur={blurLine}/></FField>
          {pwMsg&&<div style={{fontSize:10,color:pwMsg.includes('✓')?T.green:T.red}}>{pwMsg}</div>}
          <button onClick={changePw} style={{...BTNG,alignSelf:'flex-start'}}>UPDATE PASSWORD</button>
        </div>
      </div>

      {/* Data management (admin) */}
      {currentUser.role==='admin'&&(
        <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20,marginBottom:14}}>
          <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:12}}>DATA MANAGEMENT</div>
          <div style={{fontSize:11,color:T.muted,marginBottom:14,lineHeight:1.65}}>
            All data is stored in your browser's local storage. Export regularly as a JSON backup.
            Import a backup to restore or migrate to another machine.
          </div>
          <div style={{display:'flex',gap:10,flexWrap:'wrap'}}>
            <button onClick={exportData} style={{...BTNG,padding:'9px 16px'}}>EXPORT BACKUP</button>
            <button onClick={importData} style={{...BTNGH}}>IMPORT DATA</button>
          </div>
          {expMsg&&<div style={{fontSize:10,color:T.green,marginTop:8}}>{expMsg}</div>}
        </div>
      )}
    </div>
  );
}

/* ──────────────────────── PAGE: ADMIN ─────────────────────────────── */
function AdminPage() {
  const { users, tasks, clockRecords, notifs, currentUser, dispatch } = useApp();
  const [tab, setTab]   = useState('staff');
  const [showAdd, setShowAdd] = useState(false);
  const [dateF, setDateF] = useState(todayStr());
  const [userF, setUserF] = useState('all');
  const [form, setForm]   = useState({name:'',email:'',password:'',role:'staff'});
  const [fErr, setFErr]   = useState('');
  const [ann,  setAnn]    = useState('');
  const [annMsg,setAnnMsg]= useState('');

  const clockedInNow=users.filter(u=>ClockOps.isIn(clockRecords,u.id));
  const openT=tasks.filter(t=>t.status!=='done').length;
  const highT=tasks.filter(t=>t.priority==='high'&&t.status!=='done').length;

  const addStaff=()=>{
    if(!form.name.trim()||!form.email.trim()||!form.password.trim()){setFErr('All fields required');return;}
    if(users.find(u=>u.email===form.email)){setFErr('Email already in use');return;}
    const nu={id:uid(),name:form.name.trim(),email:form.email.trim(),password:form.password,
      role:form.role,initials:UserOps.initials(form.name),
      color:UserOps.avatarColor(users.length),joinedAt:new Date().toISOString()};
    const updated=[...users,nu];
    dispatch({type:'SET_USERS',payload:updated});
    const n=NotifOps.create({toUserId:'all',type:'user_added',title:'New Team Member',
      body:`${nu.name} has joined as ${nu.role}.`});
    dispatch({type:'SET_NOTIFS',payload:[...notifs,n]});
    setForm({name:'',email:'',password:'',role:'staff'});setFErr('');setShowAdd(false);
  };

  const removeStaff=id=>{
    if(id===SYSTEM_ADMIN.id)return;
    if(!confirm('Remove this staff member? Their tasks and clock records remain.'))return;
    dispatch({type:'SET_USERS',payload:users.filter(u=>u.id!==id)});
  };

  const promoteToggle=id=>{
    dispatch({type:'SET_USERS',payload:users.map(u=>u.id===id?{...u,role:u.role==='admin'?'staff':'admin'}:u)});
  };

  const sendAnn=()=>{
    if(!ann.trim())return;
    const n=NotifOps.create({toUserId:'all',type:'announcement',title:'📣 Team Announcement',body:ann.trim(),sentBy:currentUser.name});
    dispatch({type:'SET_NOTIFS',payload:[...notifs,n]});
    setAnn(''); setAnnMsg('Sent to all staff ✓'); setTimeout(()=>setAnnMsg(''),3000);
  };

  const filtClock=clockRecords.filter(r=>(!dateF||r.date===dateF)&&(userF==='all'||r.userId===userF))
    .sort((a,b)=>new Date(b.timestamp)-new Date(a.timestamp));
  const getU=id=>users.find(u=>u.id===id);

  const tabBtn=(id,lbl)=>({
    padding:'7px 16px',fontSize:10,fontWeight:700,letterSpacing:'0.08em',
    background:tab===id?T.goldDim:'transparent',border:`1px solid ${tab===id?T.gold:T.b2}`,
    color:tab===id?T.gold:T.muted,borderRadius:4,cursor:'pointer',transition:'all 0.14s',
  });

  return (
    <div style={{padding:20,height:'100%',overflow:'auto'}}>
      {/* Stats */}
      <div style={{display:'grid',gridTemplateColumns:'repeat(2,1fr)',gap:8,marginBottom:22}}>
        {[
          {l:'TOTAL STAFF',v:users.length,c:T.gold},
          {l:'ON SHIFT',v:clockedInNow.length,c:T.green},
          {l:'OPEN TASKS',v:openT,c:T.blue},
          {l:'HIGH PRIORITY',v:highT,c:T.red},
        ].map(s=>(
          <div key={s.l} style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:'14px 16px'}}>
            <div className="syne" style={{fontSize:26,fontWeight:800,color:s.c}}>{s.v}</div>
            <div style={{fontSize:9,color:'#444',letterSpacing:'0.08em',marginTop:3}}>{s.l}</div>
          </div>
        ))}
      </div>

      {/* Tabs */}
      <div style={{display:'flex',gap:6,marginBottom:20,flexWrap:'wrap'}}>
        {[['staff','STAFF'],['attendance','ATTENDANCE'],['announcements','ANNOUNCEMENTS']].map(([id,lbl])=>(
          <button key={id} style={tabBtn(id,lbl)} onClick={()=>setTab(id)}>{lbl}</button>
        ))}
      </div>

      {/* ── STAFF ── */}
      {tab==='staff'&&(
        <div>
          {/* Live status */}
          <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:16,marginBottom:16}}>
            <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:12}}>CURRENT STATUS</div>
            <div style={{display:'flex',gap:8,flexWrap:'wrap'}}>
              {users.map(u=>{
                const isIn=ClockOps.isIn(clockRecords,u.id);
                return (
                  <div key={u.id} style={{display:'flex',alignItems:'center',gap:8,padding:'7px 12px',
                    background:T.s2,border:`1px solid ${isIn?'#5db37e33':T.b1}`,borderRadius:5}}>
                    <Avatar user={u} size={22}/>
                    <div>
                      <div style={{fontSize:11,color:T.text}}>{u.name.split(' ')[0]}</div>
                      <div style={{fontSize:9,color:isIn?T.green:'#444',letterSpacing:'0.06em'}}>{isIn?'● IN':'○ OUT'}</div>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:12}}>
            <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em'}}>{users.length} MEMBER{users.length!==1?'S':''}</div>
            <button onClick={()=>setShowAdd(!showAdd)} style={{...BTNG,padding:'8px 14px'}}>+ ADD STAFF</button>
          </div>

          {/* Add form */}
          {showAdd&&(
            <div style={{background:T.s2,border:`1px solid ${T.b2}`,borderRadius:6,padding:20,marginBottom:14}}>
              <div className="syne" style={{fontSize:11,fontWeight:700,color:T.gold,marginBottom:14,letterSpacing:'0.1em'}}>NEW STAFF MEMBER</div>
              <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:12,marginBottom:12}}>
                <FField label="FULL NAME *"><input style={INP} value={form.name} onChange={e=>setForm({...form,name:e.target.value})} placeholder="Jane Smith" onFocus={focusGold} onBlur={blurLine}/></FField>
                <FField label="EMAIL *"><input type="email" style={INP} value={form.email} onChange={e=>setForm({...form,email:e.target.value})} placeholder="jane@co.com" onFocus={focusGold} onBlur={blurLine}/></FField>
                <FField label="PASSWORD *"><input type="text" style={INP} value={form.password} onChange={e=>setForm({...form,password:e.target.value})} placeholder="Set login password" onFocus={focusGold} onBlur={blurLine}/></FField>
                <FField label="ROLE">
                  <select style={{...INP,cursor:'pointer'}} value={form.role} onChange={e=>setForm({...form,role:e.target.value})}>
                    <option value="staff">Staff</option>
                    <option value="admin">Admin</option>
                  </select>
                </FField>
              </div>
              {fErr&&<div style={{fontSize:10,color:T.red,marginBottom:10}}>{fErr}</div>}
              <div style={{display:'flex',gap:8}}>
                <button style={BTNGH} onClick={()=>{setShowAdd(false);setFErr('');}}>CANCEL</button>
                <button style={BTNG} onClick={addStaff}>CREATE ACCOUNT</button>
              </div>
            </div>
          )}

          {/* Staff list */}
          <div style={{display:'flex',flexDirection:'column',gap:8}}>
            {users.map(u=>{
              const myT=tasks.filter(t=>t.assignedTo===u.id).length;
              const isOwner=u.id===SYSTEM_ADMIN.id;
              return (
                <div key={u.id} style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:14,display:'flex',alignItems:'center',gap:12,flexWrap:'wrap'}}>
                  <Avatar user={u} size={40}/>
                  <div style={{flex:1,minWidth:150}}>
                    <div style={{display:'flex',alignItems:'center',gap:8,marginBottom:3,flexWrap:'wrap'}}>
                      <span style={{fontSize:13,color:T.text,fontWeight:500}}>{u.name}</span>
                      <span style={{fontSize:9,padding:'1px 6px',borderRadius:2,fontWeight:700,letterSpacing:'0.08em',
                        background:u.role==='admin'?T.goldDim:'rgba(85,146,212,0.1)',
                        color:u.role==='admin'?T.gold:T.blue}}>
                        {u.role.toUpperCase()}{isOwner?' · OWNER':''}
                      </span>
                    </div>
                    <div style={{fontSize:10,color:T.muted}}>{u.email}</div>
                    <div style={{fontSize:9,color:'#444',marginTop:2}}>{myT} task{myT!==1?'s':''}{u.joinedAt?` · Joined ${new Date(u.joinedAt).toLocaleDateString('en-AU',{month:'short',year:'numeric'})}`:''}</div>
                  </div>
                  {!isOwner&&(
                    <div style={{display:'flex',gap:6,flexWrap:'wrap'}}>
                      <button onClick={()=>promoteToggle(u.id)}
                        style={{padding:'5px 10px',fontSize:9,letterSpacing:'0.06em',borderRadius:4,cursor:'pointer',
                          background:u.role==='admin'?'rgba(224,85,85,0.07)':T.goldDim,
                          border:`1px solid ${u.role==='admin'?`${T.red}33`:`${T.gold}33`}`,
                          color:u.role==='admin'?T.red:T.gold}}>
                        {u.role==='admin'?'DEMOTE':'MAKE ADMIN'}
                      </button>
                      <button onClick={()=>removeStaff(u.id)}
                        style={{padding:'5px 10px',fontSize:9,background:'transparent',border:`1px solid ${T.b2}`,color:T.muted,borderRadius:4,cursor:'pointer'}}
                        onMouseEnter={e=>{e.currentTarget.style.color=T.red;e.currentTarget.style.borderColor=`${T.red}33`;}}
                        onMouseLeave={e=>{e.currentTarget.style.color=T.muted;e.currentTarget.style.borderColor=T.b2;}}>
                        REMOVE
                      </button>
                    </div>
                  )}
                </div>
              );
            })}
          </div>
        </div>
      )}

      {/* ── ATTENDANCE ── */}
      {tab==='attendance'&&(
        <div>
          <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:14,gap:10,flexWrap:'wrap'}}>
            <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em'}}>CLOCK RECORDS</div>
            <div style={{display:'flex',gap:8,flexWrap:'wrap'}}>
              <input type="date" style={{...INP,width:'auto',padding:'7px 10px',fontSize:11}} value={dateF} onChange={e=>setDateF(e.target.value)}/>
              <select style={{...INP,width:'auto',padding:'7px 10px',fontSize:11,cursor:'pointer'}} value={userF} onChange={e=>setUserF(e.target.value)}>
                <option value="all">All staff</option>
                {users.map(u=><option key={u.id} value={u.id}>{u.name}</option>)}
              </select>
            </div>
          </div>
          {filtClock.length===0?<EmptyState icon="📋" text="No clock records" sub="Try changing date or staff filter"/>:(
            <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6}} className="table-wrap">
              <table style={{width:'100%',borderCollapse:'collapse',minWidth:580}}>
                <thead>
                  <tr>{['STAFF','DATE','TIME','TYPE','GPS LOCATION','ACCURACY'].map(h=>(
                    <th key={h} style={{padding:'10px 14px',textAlign:'left',fontSize:9,color:'#444',letterSpacing:'0.1em',borderBottom:`1px solid ${T.b1}`,whiteSpace:'nowrap'}}>{h}</th>
                  ))}</tr>
                </thead>
                <tbody>
                  {filtClock.map(r=>{
                    const u=getU(r.userId);
                    return (
                      <tr key={r.id} style={{borderBottom:`1px solid ${T.b1}`}}>
                        <td style={{padding:'9px 14px'}}><div style={{display:'flex',alignItems:'center',gap:7}}>{u&&<Avatar user={u} size={20}/>}<span style={{fontSize:11,color:T.text}}>{r.userName}</span></div></td>
                        <td style={{padding:'9px 14px',fontSize:11,color:T.muted,whiteSpace:'nowrap'}}>{r.date}</td>
                        <td style={{padding:'9px 14px',fontSize:11,color:T.sub,whiteSpace:'nowrap'}}>{r.time}</td>
                        <td style={{padding:'9px 14px'}}><span style={{fontSize:9,padding:'2px 7px',borderRadius:2,fontWeight:700,letterSpacing:'0.1em',background:r.type==='in'?'rgba(93,179,126,0.1)':'rgba(224,85,85,0.1)',color:r.type==='in'?T.green:T.red}}>{r.type.toUpperCase()}</span></td>
                        <td style={{padding:'9px 14px',fontSize:10,color:T.muted}}>{r.lat?<a href={`https://maps.google.com/?q=${r.lat},${r.lng}`} target="_blank" rel="noopener" style={{color:T.blue}}>{r.lat.toFixed(4)}, {r.lng.toFixed(4)}</a>:<span style={{color:'#333'}}>{r.locationError||'—'}</span>}</td>
                        <td style={{padding:'9px 14px',fontSize:10,color:'#444'}}>{r.accuracy?`±${r.accuracy}m`:'—'}</td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            </div>
          )}
        </div>
      )}

      {/* ── ANNOUNCEMENTS ── */}
      {tab==='announcements'&&(
        <div>
          <div style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:8,padding:20,marginBottom:20}}>
            <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:14}}>SEND ANNOUNCEMENT</div>
            <div style={{fontSize:11,color:T.muted,marginBottom:14,lineHeight:1.6}}>
              Send a message to all team members — it appears in their notifications feed instantly.
            </div>
            <FField label="MESSAGE">
              <textarea style={{...INP,resize:'vertical',minHeight:80,lineHeight:1.6}} value={ann} onChange={e=>setAnn(e.target.value)} placeholder="Type your announcement..."/>
            </FField>
            <div style={{marginTop:12,display:'flex',alignItems:'center',gap:12}}>
              <button onClick={sendAnn} style={{...BTNG,padding:'9px 20px'}}>SEND TO ALL STAFF</button>
              {annMsg&&<div style={{fontSize:10,color:T.green}}>{annMsg}</div>}
            </div>
          </div>
          <div className="syne" style={{fontSize:10,fontWeight:700,color:'#888',letterSpacing:'0.1em',marginBottom:12}}>SENT ANNOUNCEMENTS</div>
          {notifs.filter(n=>n.type==='announcement').sort((a,b)=>new Date(b.createdAt)-new Date(a.createdAt)).length===0
            ?<EmptyState icon="📣" text="No announcements yet"/>
            :notifs.filter(n=>n.type==='announcement').sort((a,b)=>new Date(b.createdAt)-new Date(a.createdAt)).map(n=>(
            <div key={n.id} style={{background:T.s1,border:`1px solid ${T.b1}`,borderRadius:6,padding:14,marginBottom:8}}>
              <div style={{fontSize:12,color:T.text,marginBottom:4,lineHeight:1.5}}>{n.body}</div>
              <div style={{fontSize:9,color:'#444'}}>{new Date(n.createdAt).toLocaleString('en-AU',{day:'numeric',month:'short',hour:'2-digit',minute:'2-digit'})} · by {n.sentBy||'Admin'}</div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

/* ══════════════════════════════════════════════════════════════════════
   PILLAR 7 — APP SHELL / NAVIGATION / PWA LAYER
   Auth guard, responsive shell, bottom/sidebar nav, PWA install.
══════════════════════════════════════════════════════════════════════ */

/* ── Login ── */
function LoginPage({ onLogin }) {
  const [email,setEmail]=useState('');
  const [pass,setPass]=useState('');
  const [err,setErr]=useState('');
  const [loading,setLoading]=useState(false);

  const submit=async()=>{
    setLoading(true);setErr('');
    const ok=await onLogin(email,pass);
    if(!ok){setErr('Invalid email or password');setLoading(false);}
  };

  return (
    <div style={{minHeight:'100vh',background:T.bg,display:'flex',alignItems:'center',justifyContent:'center',padding:20}}>
      <div className="fade-in" style={{width:'100%',maxWidth:360}}>
        <div style={{textAlign:'center',marginBottom:48}}>
          <div className="syne" style={{fontSize:34,fontWeight:800,letterSpacing:'-0.02em',color:T.text}}>
            TASK<span style={{color:T.gold}}>FLOW</span>
          </div>
          <div style={{fontSize:10,color:'#444',letterSpacing:'0.2em',marginTop:5}}>TEAM OPERATIONS PLATFORM</div>
        </div>

        <div style={{background:T.s1,border:`1px solid ${T.b2}`,borderRadius:10,padding:30}}>
          <div style={{fontSize:10,color:'#555',letterSpacing:'0.15em',marginBottom:24}}>SIGN IN TO CONTINUE</div>
          <div style={{display:'flex',flexDirection:'column',gap:14}}>
            <FField label="EMAIL">
              <input style={INP} type="email" value={email} onChange={e=>setEmail(e.target.value)}
                placeholder="you@company.com" onFocus={focusGold} onBlur={blurLine}
                onKeyDown={e=>e.key==='Enter'&&submit()}/>
            </FField>
            <FField label="PASSWORD">
              <input style={INP} type="password" value={pass} onChange={e=>setPass(e.target.value)}
                onFocus={focusGold} onBlur={blurLine} onKeyDown={e=>e.key==='Enter'&&submit()}/>
            </FField>
            {err&&<div style={{fontSize:11,color:T.red,padding:'8px 12px',background:'rgba(224,85,85,0.08)',borderRadius:4,border:`1px solid rgba(224,85,85,0.2)`}}>{err}</div>}
            <button onClick={submit} disabled={loading}
              style={{...BTNG,width:'100%',marginTop:4,padding:'13px',opacity:loading?0.6:1}}>
              {loading?'AUTHENTICATING...':'SIGN IN'}
            </button>
          </div>
        </div>

        <div style={{textAlign:'center',marginTop:20,fontSize:9,color:'#333',letterSpacing:'0.08em'}}>
          CONTACT YOUR ADMIN FOR ACCESS CREDENTIALS
        </div>
      </div>
    </div>
  );
}

/* ── App Shell ── */
function AppShell({ children, page, setPage }) {
  const { currentUser, notifs, dispatch } = useApp();
  const unread=NotifOps.unread(notifs,currentUser.id);
  const logout=()=>dispatch({type:'LOGOUT'});

  const NAV=[
    {id:'calendar', icon:'▦', label:'CALENDAR'},
    {id:'tasks',    icon:'✓', label:'TASKS'},
    {id:'clock',    icon:'◎', label:'CLOCK'},
    {id:'notifications', icon:'🔔', label:'NOTIFS', badge:unread>0?unread:null},
    {id:'profile',  icon:'◉', label:'PROFILE'},
    ...(currentUser.role==='admin'?[{id:'admin',icon:'⊞',label:'ADMIN'}]:[]),
  ];

  const sideBtn=(n)=>({
    width:'100%',padding:'9px 12px',borderRadius:5,display:'flex',alignItems:'center',gap:10,
    background:page===n.id?'#1e1e1e':'transparent',
    border:`1px solid ${page===n.id?T.b2:'transparent'}`,
    color:page===n.id?T.gold:T.muted,
    fontSize:10,fontWeight:page===n.id?700:400,letterSpacing:'0.06em',
    cursor:'pointer',transition:'all 0.14s',position:'relative',
  });

  return (
    <div className="app-wrap">
      {/* Sidebar (desktop) */}
      <aside className="sidebar">
        <div style={{padding:'0 16px 24px'}}>
          <div className="syne" style={{fontSize:18,fontWeight:800,color:T.text}}>TASK<span style={{color:T.gold}}>FLOW</span></div>
          <div style={{fontSize:9,color:'#3a3a3a',letterSpacing:'0.14em',marginTop:2}}>OPERATIONS PLATFORM</div>
        </div>

        <div style={{flex:1,padding:'0 8px',display:'flex',flexDirection:'column',gap:2}}>
          {NAV.map(n=>(
            <button key={n.id} style={sideBtn(n)} onClick={()=>setPage(n.id)}
              onMouseEnter={e=>page!==n.id&&(e.currentTarget.style.background='#141414')}
              onMouseLeave={e=>page!==n.id&&(e.currentTarget.style.background='transparent')}>
              <span style={{fontSize:15}}>{n.icon}</span>
              <span>{n.label}</span>
              {n.badge&&<span style={{marginLeft:'auto',background:T.green,color:'#0c0c0c',borderRadius:10,
                minWidth:18,height:18,fontSize:9,fontWeight:700,display:'flex',alignItems:'center',justifyContent:'center',padding:'0 4px'}}>{n.badge}</span>}
            </button>
          ))}
        </div>

        <div style={{padding:'16px 12px',borderTop:`1px solid ${T.b1}`}}>
          <div style={{display:'flex',alignItems:'center',gap:10,marginBottom:12}}>
            <Avatar user={currentUser} size={34}/>
            <div style={{minWidth:0}}>
              <div style={{fontSize:11,color:T.text,fontWeight:700,overflow:'hidden',textOverflow:'ellipsis',whiteSpace:'nowrap'}}>{currentUser.name}</div>
              <div style={{fontSize:9,color:'#555',letterSpacing:'0.06em'}}>{currentUser.role.toUpperCase()}</div>
            </div>
          </div>
          <button onClick={logout} style={{width:'100%',padding:'8px',background:'transparent',
            border:`1px solid ${T.b2}`,color:T.muted,borderRadius:4,fontSize:9,letterSpacing:'0.1em',cursor:'pointer',transition:'all 0.14s'}}
            onMouseEnter={e=>{e.currentTarget.style.borderColor=T.red;e.currentTarget.style.color=T.red;}}
            onMouseLeave={e=>{e.currentTarget.style.borderColor=T.b2;e.currentTarget.style.color=T.muted;}}>
            SIGN OUT
          </button>
        </div>
      </aside>

      {/* Main column */}
      <div className="main-col">
        {/* Top bar */}
        <div style={{height:50,borderBottom:`1px solid ${T.b1}`,display:'flex',alignItems:'center',
          justifyContent:'space-between',padding:'0 18px',flexShrink:0,background:'#0a0a0a'}}>
          <div style={{display:'flex',alignItems:'center',gap:14}}>
            {/* Mobile logo */}
            <div className="syne" style={{fontSize:14,fontWeight:800,color:T.text,display:'none'}} id="mob-logo">
              TASK<span style={{color:T.gold}}>FLOW</span>
            </div>
            <style>{`@media(max-width:767px){#mob-logo{display:block!important}}`}</style>
            <span className="syne" style={{fontSize:13,fontWeight:700,color:T.text}}>
              {NAV.find(n=>n.id===page)?.label}
            </span>
          </div>
          <div style={{display:'flex',alignItems:'center',gap:10}}>
            <div className="syne" style={{fontSize:11,fontWeight:700,color:T.gold}}>{currentUser.name.split(' ')[0].toUpperCase()}</div>
            <Avatar user={currentUser} size={28}/>
          </div>
        </div>

        {/* Page body */}
        <div className="page-body slide-in">{children}</div>
      </div>

      {/* Bottom nav (mobile) */}
      <nav className="bot-nav">
        {NAV.slice(0,6).map(n=>(
          <button key={n.id} onClick={()=>setPage(n.id)}
            style={{flex:1,display:'flex',flexDirection:'column',alignItems:'center',gap:3,
              background:'transparent',border:'none',color:page===n.id?T.gold:'#555',cursor:'pointer',
              padding:'2px 0',position:'relative',transition:'color 0.14s'}}>
            <span style={{fontSize:18,lineHeight:1}}>{n.icon}</span>
            <span style={{fontSize:8,letterSpacing:'0.04em',fontWeight:page===n.id?700:400}}>{n.label}</span>
            {n.badge&&<span style={{position:'absolute',top:-1,right:'50%',marginRight:-16,
              background:T.green,color:'#0c0c0c',borderRadius:8,minWidth:14,height:14,
              fontSize:8,fontWeight:700,display:'flex',alignItems:'center',justifyContent:'center',padding:'0 3px'}}>{n.badge}</span>}
            {page===n.id&&<div style={{position:'absolute',top:0,left:'50%',transform:'translateX(-50%)',
              width:20,height:2,background:T.gold,borderRadius:1}}/>}
          </button>
        ))}
      </nav>
    </div>
  );
}

/* ── Root ── */
function Root() {
  const [state, dispatch] = useReducer(reducer, {
    currentUser:null, users:[], tasks:[], clockRecords:[], notifs:[], ready:false
  });
  const [page, setPage] = useState('calendar');

  // PWA manifest inject
  useEffect(()=>{
    const manifest={
      name:'TaskFlow',short_name:'TaskFlow',description:'Team Operations Platform',
      start_url:'./',display:'standalone',background_color:'#0c0c0c',theme_color:'#0c0c0c',
      orientation:'any',
      icons:[
        {src:"data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 192 192'%3E%3Crect width='192' height='192' fill='%230c0c0c'/%3E%3Ctext x='96' y='130' font-size='100' text-anchor='middle' fill='%23c8a96e' font-family='sans-serif' font-weight='bold'%3ETF%3C/text%3E%3C/svg%3E",sizes:'192x192',type:'image/svg+xml'},
        {src:"data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 512 512'%3E%3Crect width='512' height='512' fill='%230c0c0c'/%3E%3Ctext x='256' y='340' font-size='260' text-anchor='middle' fill='%23c8a96e' font-family='sans-serif' font-weight='bold'%3ETF%3C/text%3E%3C/svg%3E",sizes:'512x512',type:'image/svg+xml'},
      ]
    };
    const blob=new Blob([JSON.stringify(manifest)],{type:'application/json'});
    const url=URL.createObjectURL(blob);
    const link=document.createElement('link');
    link.rel='manifest'; link.href=url;
    document.head.appendChild(link);

    // Service Worker
    if('serviceWorker' in navigator){
      const sw=`
        const CACHE='taskflow-v1';
        const ASSETS=['./', location.href];
        self.addEventListener('install',e=>{ e.waitUntil(caches.open(CACHE).then(c=>c.addAll(ASSETS))); self.skipWaiting(); });
        self.addEventListener('activate',e=>{ e.waitUntil(caches.keys().then(k=>Promise.all(k.filter(n=>n!==CACHE).map(n=>caches.delete(n))))); self.clients.claim(); });
        self.addEventListener('fetch',e=>{ e.respondWith(caches.match(e.request).then(r=>r||fetch(e.request).catch(()=>caches.match('./')))); });
      `;
      const swBlob=new Blob([sw],{type:'application/javascript'});
      const swUrl=URL.createObjectURL(swBlob);
      navigator.serviceWorker.register(swUrl).catch(()=>{});
    }
  },[]);

  useEffect(()=>{
    const users=Repo.getUsers();
    const tasks=Repo.getTasks();
    const clock=Repo.getClock();
    const notifs=Repo.getNotifs();
    const session=Repo.getSession();
    const liveUser=session?users.find(u=>u.id===session.id):null;
    dispatch({type:'BOOT',payload:{users,tasks,clockRecords:clock,notifs,currentUser:liveUser||null}});
  },[]);

  const login=async(email,password)=>{
    const users=Repo.getUsers();
    const found=users.find(u=>u.email===email&&u.password===password);
    if(!found)return false;
    const {password:_,  ...safe}=found;
    dispatch({type:'SET_USER',payload:safe});
    return true;
  };

  if(!state.ready){
    return (
      <div style={{background:T.bg,height:'100vh',display:'flex',alignItems:'center',justifyContent:'center'}}>
        <div className="pulse syne" style={{fontSize:11,color:'#333',letterSpacing:'0.2em'}}>LOADING TASKFLOW...</div>
      </div>
    );
  }

  if(!state.currentUser){
    return (
      <Ctx.Provider value={{...state,dispatch}}>
        <LoginPage onLogin={login}/>
      </Ctx.Provider>
    );
  }

  return (
    <Ctx.Provider value={{...state,dispatch}}>
      <AppShell page={page} setPage={setPage}>
        <div style={{height:'100%',overflow:page==='calendar'?'hidden':'auto'}}>
          {page==='calendar'      && <CalendarPage/>}
          {page==='tasks'         && <TasksPage/>}
          {page==='clock'         && <ClockPage/>}
          {page==='notifications' && <NotificationsPage/>}
          {page==='profile'       && <ProfilePage setPage={setPage}/>}
          {page==='admin'         && state.currentUser.role==='admin'&&<AdminPage/>}
        </div>
      </AppShell>
    </Ctx.Provider>
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(<Root/>);
</script>
</body>
</html>
