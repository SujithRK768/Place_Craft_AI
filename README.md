<!DOCTYPE html>
<html lang="en" class="dark">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>PlaceCraft AI — Placement Prep Dashboard</title>

  <!-- Tailwind -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          animation: {
            'pulse-slow': 'pulse 3s cubic-bezier(0.4,0,0.6,1) infinite',
            'wave': 'wave 1.2s ease-in-out infinite',
          },
          keyframes: {
            wave: {
              '0%, 100%': { transform: 'scaleY(0.4)' },
              '50%':       { transform: 'scaleY(1.2)' },
            }
          }
        }
      }
    };
  </script>

  <!-- React -->
  <script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

  <!-- Firebase (compat SDK — works in non-module scripts) -->
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>

  <style>
    body { background:#020617; font-family:'Inter',system-ui,sans-serif; }
    ::-webkit-scrollbar { width:6px; height:6px; }
    ::-webkit-scrollbar-track { background:#0f172a; }
    ::-webkit-scrollbar-thumb { background:#334155; border-radius:3px; }

    .gauge-dash { transition: stroke-dashoffset 1.2s cubic-bezier(0.4,0,0.2,1); }
    .glass { background:rgba(15,23,42,0.82); backdrop-filter:blur(14px); border:1px solid rgba(99,102,241,0.15); }
    .glass-yellow { background:rgba(161,98,7,0.18); backdrop-filter:blur(8px); border:1px solid rgba(234,179,8,0.35); }
    .nav-active { background:linear-gradient(135deg,#4f46e5,#7c3aed); }
    .progress-fill { transition:width 0.8s cubic-bezier(0.4,0,0.2,1); }
    .hover-scale { transition:transform 0.15s ease; }
    .hover-scale:hover { transform:scale(1.015); }
    .modal-overlay { background:rgba(0,0,0,0.72); backdrop-filter:blur(6px); }
    .gradient-text {
      background:linear-gradient(135deg,#818cf8,#a78bfa,#c084fc);
      -webkit-background-clip:text; -webkit-text-fill-color:transparent; background-clip:text;
    }
    .tab-content { animation:fadeSlide 0.25s ease; }
    @keyframes fadeSlide { from{opacity:0;transform:translateY(8px)} to{opacity:1;transform:translateY(0)} }

    .wave-bar { display:inline-block; width:3px; margin:0 1px; border-radius:2px; animation:wave 1.2s ease-in-out infinite; }
    @keyframes wave { 0%,100%{transform:scaleY(0.4)} 50%{transform:scaleY(1.2)} }

    .code-area { font-family:'JetBrains Mono','Fira Code','Courier New',monospace; font-size:13px; line-height:1.6; }

    /* Login page */
    .login-bg {
      background: radial-gradient(ellipse at 20% 50%, rgba(79,70,229,0.15) 0%, transparent 60%),
                  radial-gradient(ellipse at 80% 50%, rgba(124,58,237,0.12) 0%, transparent 60%),
                  #020617;
    }
    .google-btn { transition: all 0.2s ease; }
    .google-btn:hover { background: #f8fafc; transform: translateY(-1px); box-shadow: 0 8px 24px rgba(0,0,0,0.4); }
    .google-btn:active { transform: translateY(0); }

    /* Interim speech text */
    .interim-text { color: #94a3b8; font-style: italic; }

    /* Speech not supported badge */
    .speech-warn { background: rgba(239,68,68,0.15); border: 1px solid rgba(239,68,68,0.4); }
  </style>
</head>
<body>
<div id="root"></div>

<script type="text/babel">
const { useState, useEffect, useRef, useCallback } = React;

/* ════════════════════════════════════════════════════════════
   ① FIREBASE CONFIG
   ─────────────────────────────────────────────────────────
   Go to https://console.firebase.google.com
   → New Project → Add Web App → copy config below
   → Authentication → Sign-in method → Enable Google
   → Authentication → Settings → Authorized domains → add localhost
   ════════════════════════════════════════════════════════════ */
const FIREBASE_CONFIG = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT_ID.firebaseapp.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId:             "YOUR_APP_ID"
};

const FB_READY = FIREBASE_CONFIG.apiKey !== "YOUR_API_KEY";
let fbAuth = null, googleProvider = null;
if (FB_READY) {
  try {
    if (!firebase.apps.length) firebase.initializeApp(FIREBASE_CONFIG);
    fbAuth = firebase.auth();
    googleProvider = new firebase.auth.GoogleAuthProvider();
    googleProvider.addScope('profile');
    googleProvider.addScope('email');
  } catch(e) { console.error("Firebase init:", e); }
}

/* ─── Helpers ─────────────────────────────────────────────── */
function Icon({ d, size = 18, stroke = "currentColor", fill = "none", className = "" }) {
  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill={fill}
      stroke={stroke} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      className={className}>
      <path d={d} />
    </svg>
  );
}

function CircularGauge({ pct, size = 160, strokeWidth = 15 }) {
  const r = (size - strokeWidth) / 2;
  const circ = 2 * Math.PI * r;
  const offset = circ - (pct / 100) * circ;
  const color = pct >= 70 ? '#6366f1' : pct >= 55 ? '#f59e0b' : '#ef4444';
  return (
    <svg width={size} height={size} viewBox={`0 0 ${size} ${size}`} className="rotate-[-90deg]">
      <circle cx={size/2} cy={size/2} r={r} fill="none" stroke="#1e293b" strokeWidth={strokeWidth}/>
      <circle cx={size/2} cy={size/2} r={r} fill="none" stroke={color} strokeWidth={strokeWidth}
        strokeLinecap="round" strokeDasharray={circ} strokeDashoffset={offset}
        className="gauge-dash" style={{ filter:`drop-shadow(0 0 8px ${color})` }}/>
    </svg>
  );
}

function MiniBar({ pct, color = 'bg-indigo-500' }) {
  return (
    <div className="h-1.5 bg-slate-700 rounded-full overflow-hidden">
      <div className={`h-full ${color} progress-fill rounded-full`} style={{ width:`${pct}%` }}/>
    </div>
  );
}

function Modal({ open, onClose, children }) {
  if (!open) return null;
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center modal-overlay" onClick={onClose}>
      <div className="relative max-w-lg w-full mx-4" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

/* ════════════════════════════════════════════════════════════
   ② LOGIN PAGE
════════════════════════════════════════════════════════════ */
function LoginPage({ onLogin }) {
  const [loading, setLoading] = useState(false);
  const [error, setError]     = useState('');

  async function signInWithGoogle() {
    if (!FB_READY || !fbAuth) {
      setError("Firebase not configured. Use Demo Mode below.");
      return;
    }
    setLoading(true); setError('');
    try {
      const result = await fbAuth.signInWithPopup(googleProvider);
      onLogin({
        name:  result.user.displayName || 'Candidate',
        email: result.user.email,
        photo: result.user.photoURL,
        uid:   result.user.uid,
        demo:  false,
      });
    } catch(e) {
      setError(e.message?.replace('Firebase: ', '') || 'Sign-in failed. Try again.');
    } finally { setLoading(false); }
  }

  function demoMode() {
    onLogin({ name:'Aravindhan', email:'aravindhan@demo.com', photo:null, uid:'demo', demo:true });
  }

  return (
    <div className="login-bg min-h-screen flex flex-col items-center justify-center px-4">
      {/* Brand */}
      <div className="flex flex-col items-center mb-10">
        <div className="w-16 h-16 rounded-2xl bg-gradient-to-br from-indigo-500 to-violet-600 flex items-center justify-center shadow-2xl shadow-indigo-900/60 mb-4 text-3xl font-black text-white">P</div>
        <h1 className="text-4xl font-black gradient-text">PlaceCraft AI</h1>
        <p className="text-slate-400 mt-1 text-sm">Mechanical &amp; Software Placement Prep</p>
      </div>

      {/* Card */}
      <div className="glass rounded-2xl p-8 w-full max-w-sm text-center shadow-2xl">
        <h2 className="text-xl font-bold text-white mb-1">Welcome back</h2>
        <p className="text-slate-400 text-sm mb-6">Sign in to continue your placement journey</p>

        {/* Google Sign-In */}
        <button
          onClick={signInWithGoogle}
          disabled={loading}
          className="google-btn w-full flex items-center justify-center gap-3 bg-white text-slate-800 font-semibold py-3 px-5 rounded-xl mb-3 disabled:opacity-60"
        >
          {loading ? (
            <div className="w-5 h-5 border-2 border-slate-300 border-t-indigo-500 rounded-full animate-spin"/>
          ) : (
            <svg width="20" height="20" viewBox="0 0 48 48">
              <path fill="#FFC107" d="M43.6 20.1H42V20H24v8h11.3C33.7 32.7 29.2 36 24 36c-6.6 0-12-5.4-12-12s5.4-12 12-12c3.1 0 5.8 1.1 8 2.9l5.7-5.7C34.5 6.5 29.5 4 24 4 12.9 4 4 12.9 4 24s8.9 20 20 20 20-8.9 20-20c0-1.3-.1-2.6-.4-3.9z"/>
              <path fill="#FF3D00" d="M6.3 14.7l6.6 4.8C14.7 16.1 19 13 24 13c3.1 0 5.8 1.1 8 2.9l5.7-5.7C34.5 6.5 29.5 4 24 4 16.3 4 9.7 8.3 6.3 14.7z"/>
              <path fill="#4CAF50" d="M24 44c5.2 0 9.9-2 13.4-5.2l-6.2-5.2C29.3 35.5 26.7 36.5 24 36.5c-5.2 0-9.6-3.3-11.3-8H6.3C9.7 38.9 16.3 44 24 44z"/>
              <path fill="#1976D2" d="M43.6 20.1H42V20H24v8h11.3c-.8 2.2-2.2 4.1-4 5.5l6.2 5.2C37.2 39.4 44 34.5 44 24c0-1.3-.1-2.6-.4-3.9z"/>
            </svg>
          )}
          {loading ? 'Signing in…' : 'Continue with Google'}
        </button>

        {!FB_READY && (
          <div className="bg-amber-950/60 border border-amber-700/50 rounded-xl p-3 mb-3 text-left">
            <p className="text-amber-300 text-xs font-semibold mb-1">⚙️ Firebase Not Configured</p>
            <p className="text-amber-200/70 text-xs">Open <code className="bg-slate-800 px-1 rounded">index.html</code>, find <code className="bg-slate-800 px-1 rounded">FIREBASE_CONFIG</code> and paste your Firebase project keys. <a href="https://console.firebase.google.com" target="_blank" className="text-indigo-400 underline">console.firebase.google.com →</a></p>
          </div>
        )}

        {error && <p className="text-red-400 text-xs mb-3 bg-red-950/40 border border-red-800/40 rounded-lg px-3 py-2">{error}</p>}

        <div className="flex items-center gap-2 my-3">
          <div className="h-px flex-1 bg-slate-700"/>
          <span className="text-slate-600 text-xs">or</span>
          <div className="h-px flex-1 bg-slate-700"/>
        </div>

        <button
          onClick={demoMode}
          className="w-full py-2.5 rounded-xl border border-slate-700 hover:border-indigo-500/60 text-slate-300 hover:text-white text-sm font-medium transition-all"
        >
          Continue as Guest (Demo Mode)
        </button>
      </div>

      <p className="text-slate-600 text-xs mt-6 text-center max-w-xs">
        By signing in you agree to PlaceCraft AI's Terms of Service.<br/>
        Your data is used solely for placement prep tracking.
      </p>
    </div>
  );
}

/* ════════════════════════════════════════════════════════════
   ③ DASHBOARD PAGE
════════════════════════════════════════════════════════════ */
function DashboardPage({ navigate }) {
  const [checks, setChecks]       = useState([false, false, false]);
  const [sprintModal, setSprintModal] = useState(false);
  const [drillModal, setDrillModal]   = useState(false);

  const toggle = i => setChecks(c => c.map((v, idx) => idx === i ? !v : v));

  const sprintItems = [
    { label:'Aptitude', sub:'Time & Work', detail:'10 Questions', icon:'📊', navTo: null },
    { label:'Core/Tech', sub:'Memory Allocation & Pointers', detail:'3 Drills', icon:'💻', navTo: null },
    { label:'Behavioral', sub:'2-min STAR Pitch on Final Year Project', detail:'', icon:'🎤', navTo:'interview' },
  ];

  const metrics = [
    { label:'Quantitative Aptitude',    pct:75, color:'bg-emerald-500', flag:false },
    { label:'Core Engineering & Logic', pct:54, color:'bg-red-500',     flag:true  },
    { label:'Coding',                   pct:68, color:'bg-indigo-500',  flag:false },
    { label:'HR & STAR Pitch',          pct:72, color:'bg-violet-500',  flag:false },
  ];

  const drives = [
    { company:'Bosch India',    date:'7 Nov 2026',  cutoff:72, eligible:true,  gap:0  },
    { company:'Infosys InfyTQ', date:'14 Nov 2026', cutoff:60, eligible:true,  gap:0  },
    { company:'TCS Digital',    date:'21 Nov 2026', cutoff:80, eligible:false, gap:12 },
  ];

  return (
    <div className="tab-content space-y-6">

      {/* ── Readiness Score ────────────────────────────── */}
      <div className="glass rounded-2xl p-6 hover-scale">
        <div className="flex flex-col lg:flex-row gap-6 items-center">
          <div className="relative flex-shrink-0">
            <CircularGauge pct={68} size={170} strokeWidth={16}/>
            <div className="absolute inset-0 flex flex-col items-center justify-center">
              <span className="text-3xl font-bold gradient-text">68%</span>
              <span className="text-xs text-slate-400 mt-0.5">Readiness</span>
            </div>
          </div>
          <div className="flex-1 w-full">
            <h2 className="text-lg font-semibold text-white mb-1">Placement Readiness Score</h2>
            <p className="text-slate-400 text-sm mb-4">Based on 247 practice sessions · Updated 2h ago</p>
            <div className="space-y-3">
              {metrics.map(m => (
                <div key={m.label}>
                  <div className="flex justify-between items-center mb-1">
                    <span className="text-sm text-slate-300 flex items-center gap-1.5">
                      {m.flag && <span className="text-red-400 text-xs">⚠</span>}
                      {m.label}
                    </span>
                    <span className={`text-sm font-semibold ${m.flag ? 'text-red-400' : 'text-slate-200'}`}>{m.pct}%</span>
                  </div>
                  <MiniBar pct={m.pct} color={m.color}/>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">

        {/* ── Today's Sprint Card ──────────────────────── */}
        <div className="glass rounded-2xl p-6">
          <div className="flex items-center gap-2 mb-4">
            <span className="w-2 h-2 bg-indigo-400 rounded-full animate-pulse"/>
            <h2 className="text-base font-semibold text-white">Today's 3-Step Placement Sprint</h2>
          </div>

          <div className="space-y-3 mb-5">
            {sprintItems.map((item, i) => (
              <div
                key={i}
                onClick={() => {
                  if (item.navTo) { navigate(item.navTo); return; }
                  toggle(i);
                }}
                className={`flex items-start gap-3 p-3 rounded-xl cursor-pointer border transition-all duration-200
                  ${checks[i] ? 'bg-indigo-950/50 border-indigo-500/30' : 'bg-slate-800/50 border-slate-700/30 hover:border-slate-600/50 hover:bg-slate-800/70'}
                  ${item.navTo ? 'hover:border-violet-500/50 hover:bg-violet-950/30' : ''}`}
              >
                {item.navTo ? (
                  <div className="w-5 h-5 mt-0.5 rounded flex items-center justify-center flex-shrink-0 bg-violet-700/40 border border-violet-500/50">
                    <span className="text-[10px]">↗</span>
                  </div>
                ) : (
                  <div className={`w-5 h-5 mt-0.5 rounded flex items-center justify-center flex-shrink-0 border-2 transition-all ${checks[i] ? 'bg-indigo-500 border-indigo-500' : 'border-slate-500'}`}>
                    {checks[i] && <svg viewBox="0 0 10 8" fill="none" className="w-3 h-3"><path d="M1 4l3 3 5-6" stroke="white" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"/></svg>}
                  </div>
                )}
                <div className="flex-1 min-w-0">
                  <div className="flex items-center gap-2">
                    <span>{item.icon}</span>
                    <span className={`text-sm font-medium ${checks[i] && !item.navTo ? 'line-through text-slate-500' : item.navTo ? 'text-violet-300' : 'text-slate-200'}`}>
                      {item.label}: {item.sub}
                    </span>
                  </div>
                  {item.detail && <p className="text-xs text-slate-500 ml-6 mt-0.5">{item.detail}</p>}
                  {item.navTo && <p className="text-xs text-violet-500 ml-6 mt-0.5">Click to open Interview Lab →</p>}
                </div>
              </div>
            ))}
          </div>

          <div className="flex items-center gap-3">
            <button
              onClick={() => setSprintModal(true)}
              className="flex-1 bg-indigo-600 hover:bg-indigo-500 active:scale-95 text-white font-semibold py-2.5 rounded-xl transition-all duration-150 text-sm flex items-center justify-center gap-2"
            >
              ⚡ Start Today's Sprint
            </button>
            <div className="text-center">
              <div className="text-lg font-bold text-indigo-400">{checks.filter(Boolean).length}/3</div>
              <div className="text-xs text-slate-500">done</div>
            </div>
          </div>
        </div>

        {/* ── Right col ────────────────────────────────── */}
        <div className="space-y-4">

          {/* Blind Spot Alert */}
          <div className="glass-yellow rounded-2xl p-5">
            <div className="flex items-start gap-3 mb-3">
              <span className="text-2xl">⚠️</span>
              <div>
                <h3 className="text-yellow-300 font-semibold text-sm mb-1">Blind Spot Alert — Critical Gap Detected</h3>
                <p className="text-yellow-100/80 text-sm leading-relaxed">
                  You are failing <strong>60%</strong> of questions on <strong>Dynamic Memory &amp; Bitmasking</strong>. This topic appears in every Bosch/TCS screening round. Fix this <em>before the drive</em>.
                </p>
              </div>
            </div>
            <button
              onClick={() => navigate('practice')}
              className="w-full bg-yellow-500 hover:bg-yellow-400 active:scale-95 text-slate-900 font-semibold py-2 rounded-lg text-sm transition-all duration-150 flex items-center justify-center gap-2"
            >
              🚀 Launch 5-Min Recovery Drill
            </button>
          </div>

          {/* Quick Stats */}
          <div className="glass rounded-2xl p-4">
            <h3 className="text-xs font-semibold text-slate-400 uppercase tracking-wider mb-3">Weekly Activity</h3>
            <div className="grid grid-cols-3 gap-3">
              {[['Questions','347','text-indigo-400'],['Accuracy','61%','text-emerald-400'],['Streak','8 days','text-violet-400']].map(([label,val,col]) => (
                <div key={label} className="bg-slate-800/60 rounded-xl p-3 text-center">
                  <div className={`text-xl font-bold ${col}`}>{val}</div>
                  <div className="text-xs text-slate-500 mt-0.5">{label}</div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>

      {/* ── Campus Drives Table ──────────────────────────── */}
      <div className="glass rounded-2xl p-6">
        <div className="flex items-center justify-between mb-4">
          <h2 className="text-base font-semibold text-white">📅 Upcoming Campus Drives</h2>
          <span className="text-xs bg-indigo-900/50 text-indigo-300 px-2 py-1 rounded-full border border-indigo-700/50">3 drives in 8 weeks</span>
        </div>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="text-slate-500 text-xs uppercase tracking-wider border-b border-slate-700/50">
                <th className="pb-2 text-left font-medium">Company</th>
                <th className="pb-2 text-left font-medium">Drive Date</th>
                <th className="pb-2 text-left font-medium">Min. Readiness</th>
                <th className="pb-2 text-left font-medium">Status</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-slate-700/30">
              {drives.map((d,i) => (
                <tr key={i} className="hover:bg-slate-800/40 transition-colors">
                  <td className="py-3 font-medium text-slate-200">{d.company}</td>
                  <td className="py-3 text-slate-400">{d.date}</td>
                  <td className="py-3 w-40">
                    <div className="flex items-center gap-2">
                      <div className="flex-1 h-1.5 bg-slate-700 rounded-full overflow-hidden">
                        <div className={`h-full rounded-full ${d.eligible ? 'bg-emerald-500' : 'bg-red-500'}`} style={{ width:`${d.cutoff}%` }}/>
                      </div>
                      <span className="text-xs text-slate-400 flex-shrink-0">{d.cutoff}%</span>
                    </div>
                  </td>
                  <td className="py-3">
                    {d.eligible ? (
                      <span className="inline-flex items-center gap-1 bg-emerald-950/60 border border-emerald-700/50 text-emerald-300 text-xs px-2.5 py-1 rounded-full font-medium">✓ Eligible</span>
                    ) : (
                      <span className="inline-flex items-center gap-1 bg-red-950/60 border border-red-700/50 text-red-300 text-xs px-2.5 py-1 rounded-full font-medium">🔒 Locked · +{d.gap}% needed</span>
                    )}
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>

      {/* Sprint Modal — now navigates to /practice */}
      <Modal open={sprintModal} onClose={() => setSprintModal(false)}>
        <div className="bg-slate-900 border border-indigo-500/30 rounded-2xl p-6 shadow-2xl">
          <div className="text-center mb-4">
            <span className="text-4xl">⚡</span>
            <h3 className="text-xl font-bold text-white mt-2">Sprint Launched!</h3>
            <p className="text-slate-400 text-sm mt-1">Your personalized 3-step session is ready</p>
          </div>
          <div className="space-y-2 mb-5">
            {['📊 Aptitude: Time & Work → 10 Qs · Est. 12 min','💻 Core: Memory Allocation → 3 Drills · Est. 8 min','🎤 STAR Pitch: Final Year Project → 2 min'].map((t,i) => (
              <div key={i} className="bg-slate-800 rounded-lg px-4 py-2.5 text-sm text-slate-300">{t}</div>
            ))}
          </div>
          <button
            onClick={() => { setSprintModal(false); navigate('practice'); }}
            className="w-full bg-indigo-600 hover:bg-indigo-500 text-white font-semibold py-3 rounded-xl transition-all"
          >
            Begin Sprint → Open Practice Sandbox
          </button>
        </div>
      </Modal>
    </div>
  );
}

/* ════════════════════════════════════════════════════════════
   ④ PRACTICE PAGE
════════════════════════════════════════════════════════════ */
const SAMPLE_CODE = {
  Python: `# Relative Speed & Trains — Scratch Pad
# Problem: Two trains, opposite directions
# Train A: 54 km/h | Train B: 36 km/h | Cross time: 12s | Length A: 200m

def solve_relative_speed():
    speed_a, speed_b = 54, 36            # km/h
    rel_speed_kmh = speed_a + speed_b    # opposite → add = 90 km/h
    rel_speed_ms  = rel_speed_kmh * 5/18 # → 25 m/s

    cross_time, length_a = 12, 200
    total_len  = rel_speed_ms * cross_time  # 300 m
    length_b   = total_len - length_a        # 100 m

    print(f"Relative speed : {rel_speed_kmh} km/h ({rel_speed_ms} m/s)")
    print(f"Train B length : {length_b} m")
    return length_b

solve_relative_speed()
`,
  C: `#include <stdio.h>
int main() {
    float sA = 54.0f, sB = 36.0f;
    float relKmh = sA + sB;           // 90 km/h
    float relMs  = relKmh * 5.0f/18;  // 25 m/s
    float total  = relMs * 12;        // 300 m
    float lenB   = total - 200;       // 100 m
    printf("Relative speed: %.1f km/h\\nTrain B: %.0f m\\n", relKmh, lenB);
    return 0;
}
`,
  Java: `public class TrainProblem {
    public static void main(String[] args) {
        double relKmh = 54 + 36;         // 90
        double relMs  = relKmh*5.0/18;   // 25 m/s
        double lenB   = relMs*12 - 200;  // 100 m
        System.out.printf("Speed: %.1f km/h%nTrain B: %.0f m%n", relKmh, lenB);
    }
}
`,
  'C++': `#include <iostream>
using namespace std;
int main() {
    double relKmh = 54 + 36;        // 90 km/h
    double relMs  = relKmh*5.0/18;  // 25 m/s
    double lenB   = relMs*12 - 200; // 100 m
    cout << "Relative speed: " << relKmh << " km/h\\nTrain B: " << lenB << " m\\n";
}
`,
};

function PracticePage() {
  const [selected, setSelected]       = useState(null);
  const [showHint, setShowHint]       = useState(false);
  const [lang, setLang]               = useState('Python');
  const [code, setCode]               = useState(SAMPLE_CODE['Python']);
  const [timer, setTimer]             = useState(900);
  const [resultModal, setResultModal] = useState(null);
  const startTime = useRef(Date.now());

  useEffect(() => {
    const id = setInterval(() => setTimer(t => Math.max(0,t-1)), 1000);
    return () => clearInterval(id);
  }, []);
  const fmt = s => `${String(Math.floor(s/60)).padStart(2,'0')}:${String(s%60).padStart(2,'0')}`;

  const options = [
    { key:'A', text:'72 km/h',  sub:'Sum of speeds — wrong formula' },
    { key:'B', text:'90 km/h',  sub:'Relative speed (opposite directions)', correct:true },
    { key:'C', text:'108 km/h', sub:'Speed of faster train only' },
    { key:'D', text:'54 km/h',  sub:'Difference of speeds' },
  ];

  function handleSubmit() {
    if (!selected) return;
    const elapsed = Math.floor((Date.now() - startTime.current)/1000);
    setResultModal({ correct: selected === 'B', timeTaken: elapsed });
  }

  return (
    <div className="tab-content flex flex-col lg:flex-row gap-0 min-h-[calc(100vh-200px)]">

      {/* LEFT 40% */}
      <div className="w-full lg:w-[40%] glass rounded-2xl lg:rounded-r-none p-5 flex flex-col gap-4">
        <div className="flex items-center justify-between">
          <span className="text-xs text-slate-500 uppercase tracking-wider font-medium">Aptitude · Speed Drill</span>
          <div className={`flex items-center gap-1.5 px-3 py-1.5 rounded-full text-sm font-mono font-bold
            ${timer < 120 ? 'bg-red-950/60 text-red-400 border border-red-700/50' : 'bg-slate-800 text-slate-200 border border-slate-700/50'}`}>
            ⏱ {fmt(timer)}
          </div>
        </div>

        <div>
          <h2 className="text-white font-bold text-base mb-3 leading-snug">Aptitude & Speed Drill: Relative Speed & Trains</h2>
          <div className="bg-slate-800/70 rounded-xl p-4 text-sm text-slate-300 leading-relaxed mb-2">
            <p className="text-slate-400 text-xs uppercase font-medium mb-2 tracking-wide">Question 7 / 10</p>
            Two trains run in <strong className="text-white">opposite directions</strong> at <strong className="text-indigo-300">54 km/h</strong> and <strong className="text-indigo-300">36 km/h</strong>. They cross each other in <strong className="text-white">12 seconds</strong>. If Train A is <strong className="text-white">200 m</strong> long, what is the length of Train B?
          </div>
          <p className="text-xs text-slate-500 italic">Hint: Relative speed (opposite) = sum of individual speeds.</p>
        </div>

        <div className="space-y-2">
          {options.map(opt => (
            <div key={opt.key} onClick={() => setSelected(opt.key)}
              className={`flex items-center gap-3 p-3 rounded-xl cursor-pointer border transition-all
                ${selected === opt.key ? 'bg-indigo-950/60 border-indigo-500/70' : 'bg-slate-800/50 border-slate-700/30 hover:border-slate-600/60 hover:bg-slate-800/80'}`}>
              <div className={`w-5 h-5 rounded-full border-2 flex items-center justify-center flex-shrink-0 transition-all
                ${selected === opt.key ? 'border-indigo-400 bg-indigo-500' : 'border-slate-600'}`}>
                {selected === opt.key && <div className="w-2 h-2 bg-white rounded-full"/>}
              </div>
              <div>
                <span className="text-slate-300 text-sm font-medium">{opt.key}. {opt.text}</span>
                <p className="text-xs text-slate-500">{opt.sub}</p>
              </div>
            </div>
          ))}
        </div>

        {/* Hint accordion */}
        <div className="border border-slate-700/50 rounded-xl overflow-hidden">
          <button onClick={() => setShowHint(h=>!h)}
            className="w-full flex items-center justify-between px-4 py-3 bg-slate-800/50 hover:bg-slate-800/80 transition-colors text-sm text-slate-300 font-medium">
            <span>💡 Show 2-line Vedic Math / Speed Shortcut</span>
            <span className={`transition-transform duration-200 ${showHint ? 'rotate-180' : ''}`}>▾</span>
          </button>
          {showHint && (
            <div className="px-4 py-3 bg-slate-900/60 text-sm border-t border-slate-700/50 space-y-1">
              <p className="text-indigo-300 font-mono text-xs">① Opp. direction → add: 54+36 = <strong>90 km/h = 25 m/s</strong></p>
              <p className="text-indigo-300 font-mono text-xs">② Total length = 25×12 = 300m → Train B = 300−200 = <strong>100 m</strong></p>
              <p className="text-xs text-slate-500 mt-1">Mental rule: <em>"Oppose = Add, Same = Subtract"</em></p>
            </div>
          )}
        </div>

        <div className="flex gap-2 mt-auto pt-2">
          <button onClick={() => setSelected(null)} className="flex-1 py-2 rounded-xl bg-slate-700 hover:bg-slate-600 text-slate-200 text-sm font-medium transition-all">Clear</button>
          <button onClick={handleSubmit} disabled={!selected}
            className="flex-1 py-2 rounded-xl bg-violet-600 hover:bg-violet-500 disabled:opacity-40 disabled:cursor-not-allowed text-white text-sm font-semibold transition-all active:scale-95">
            Submit Answer
          </button>
        </div>
      </div>

      {/* RIGHT 60% */}
      <div className="w-full lg:w-[60%] glass border-l-0 rounded-2xl lg:rounded-l-none p-0 flex flex-col overflow-hidden">
        <div className="flex items-center gap-3 px-5 py-3 bg-slate-800/80 border-b border-slate-700/50">
          <div className="flex gap-1.5">
            <div className="w-3 h-3 rounded-full bg-red-500/80"/>
            <div className="w-3 h-3 rounded-full bg-yellow-500/80"/>
            <div className="w-3 h-3 rounded-full bg-green-500/80"/>
          </div>
          <span className="text-slate-400 text-xs flex-1">scratchpad.{lang === 'C++' ? 'cpp' : lang.toLowerCase()}</span>
          <select value={lang} onChange={e => { setLang(e.target.value); setCode(SAMPLE_CODE[e.target.value]); }}
            className="bg-slate-700 text-slate-200 text-xs px-3 py-1.5 rounded-lg border border-slate-600 outline-none cursor-pointer">
            {['C','Python','Java','C++'].map(l => <option key={l}>{l}</option>)}
          </select>
        </div>

        <div className="flex flex-1 overflow-auto">
          <div className="text-slate-600 text-xs font-mono py-4 px-3 select-none bg-slate-900/50 text-right border-r border-slate-700/30 min-w-[40px]">
            {code.split('\n').map((_,i) => <div key={i} className="leading-[1.6] text-[13px]">{i+1}</div>)}
          </div>
          <textarea value={code} onChange={e => setCode(e.target.value)}
            className="code-area flex-1 bg-transparent text-slate-200 p-4 outline-none resize-none leading-[1.6] text-[13px] min-h-[260px]"
            spellCheck="false"/>
        </div>

        <div className="flex items-center gap-3 px-5 py-3 bg-slate-800/60 border-t border-slate-700/40">
          <span className="text-xs text-slate-500">Output:</span>
          <span className="text-xs text-emerald-400 font-mono flex-1">→ Relative speed = 90 km/h | Train B = 100 m</span>
          <button className="bg-slate-700 hover:bg-slate-600 text-slate-200 text-xs font-medium px-4 py-2 rounded-lg transition-all active:scale-95">▶ Run Code</button>
          <button onClick={handleSubmit} disabled={!selected}
            className="bg-indigo-600 hover:bg-indigo-500 disabled:opacity-40 text-white text-xs font-semibold px-4 py-2 rounded-lg transition-all active:scale-95">
            Submit Answer
          </button>
        </div>
      </div>

      {/* Result Modal */}
      <Modal open={!!resultModal} onClose={() => setResultModal(null)}>
        {resultModal?.correct ? (
          <div className="bg-slate-900 border border-emerald-500/40 rounded-2xl p-6 shadow-2xl">
            <div className="text-center mb-4">
              <div className="w-14 h-14 bg-emerald-500/20 rounded-full flex items-center justify-center mx-auto mb-3 text-3xl">✅</div>
              <h3 className="text-xl font-bold text-emerald-400">Correct!</h3>
              <p className="text-slate-400 text-sm mt-1">Option B — 90 km/h (Relative Speed)</p>
              <div className="inline-flex items-center gap-1.5 mt-2 bg-slate-800 px-3 py-1 rounded-full text-xs text-slate-300">
                ⏱ Time: <strong className="text-white ml-1">{resultModal?.timeTaken}s</strong>
                {resultModal?.timeTaken < 60 && <span className="text-emerald-400 ml-1">· Fast!</span>}
              </div>
            </div>
            <div className="bg-slate-800 rounded-xl p-4 text-sm space-y-2 mb-4">
              <p className="text-indigo-300 font-semibold text-xs uppercase tracking-wide">Canonical Method</p>
              <p className="text-slate-300 font-mono text-xs">Relative speed = 54+36 = 90 km/h = 25 m/s</p>
              <p className="text-slate-300 font-mono text-xs">Train B = 25×12 − 200 = <strong className="text-white">100 m</strong></p>
              <div className="border-t border-slate-700 pt-2 mt-2">
                <p className="text-yellow-300 text-xs font-semibold mb-1">⚡ Speed Trick</p>
                <p className="text-slate-400 text-xs">Opposite → Add. Same direction → Subtract. <em>"Oppose = Add"</em> — memorise this.</p>
              </div>
            </div>
            <button onClick={() => setResultModal(null)} className="w-full bg-emerald-600 hover:bg-emerald-500 text-white font-semibold py-2.5 rounded-xl transition-all">Next Question →</button>
          </div>
        ) : (
          <div className="bg-slate-900 border border-orange-500/40 rounded-2xl p-6 shadow-2xl">
            <div className="text-center mb-4">
              <div className="w-14 h-14 bg-orange-500/20 rounded-full flex items-center justify-center mx-auto mb-3 text-3xl">⚠️</div>
              <h3 className="text-xl font-bold text-orange-400">Not Quite — Arithmetic Trap</h3>
            </div>
            <div className="bg-slate-800 rounded-xl p-4 text-sm space-y-2 mb-4">
              <p className="text-orange-300 font-semibold text-xs uppercase tracking-wide">The Trap</p>
              <p className="text-slate-300 text-xs leading-relaxed">In <strong>opposite directions</strong> you <strong>add</strong> the speeds, not subtract. Most students confuse this with same-direction problems.</p>
              <p className="text-indigo-300 font-mono text-xs border-t border-slate-700 pt-2 mt-2">Correct: 54+36 = 90 km/h → <strong>Option B</strong></p>
            </div>
            <button onClick={() => { setResultModal(null); setSelected(null); }} className="w-full bg-orange-600 hover:bg-orange-500 text-white font-semibold py-2.5 rounded-xl transition-all">Retry</button>
          </div>
        )}
      </Modal>
    </div>
  );
}

/* ════════════════════════════════════════════════════════════
   ⑤ INTERVIEW LAB  (Real Web Speech API)
════════════════════════════════════════════════════════════ */
function InterviewLabPage() {
  const [talking, setTalking]         = useState(false);
  const [finalText, setFinalText]     = useState('');
  const [interimText, setInterimText] = useState('');
  const [gradeModal, setGradeModal]   = useState(false);
  const [camError, setCamError]       = useState(false);
  const [speechOK, setSpeechOK]       = useState(true);
  const [wpmDisplay, setWpmDisplay]   = useState('130 WPM · Ideal');

  const videoRef  = useRef(null);
  const streamRef = useRef(null);
  const recogRef  = useRef(null);
  const wordCount = useRef(0);
  const startTS   = useRef(0);

  useEffect(() => {
    // Camera
    navigator.mediaDevices?.getUserMedia({ video:true, audio:false })
      .then(s => { streamRef.current = s; if (videoRef.current) videoRef.current.srcObject = s; })
      .catch(() => setCamError(true));

    // Speech Recognition
    const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SR) { setSpeechOK(false); return; }

    const r = new SR();
    r.continuous      = true;
    r.interimResults  = true;
    r.lang            = 'en-US';
    r.maxAlternatives = 1;

    r.onresult = event => {
      let interim = '';
      let newFinal = '';
      for (let i = event.resultIndex; i < event.results.length; i++) {
        const t = event.results[i][0].transcript;
        if (event.results[i].isFinal) newFinal += t + ' ';
        else interim += t;
      }
      if (newFinal) {
        setFinalText(prev => prev + newFinal);
        wordCount.current += newFinal.trim().split(/\s+/).length;
        // Live WPM
        const mins = (Date.now() - startTS.current) / 60000;
        if (mins > 0.05) {
          const wpm = Math.round(wordCount.current / mins);
          const tag = wpm < 100 ? 'Slow' : wpm < 150 ? 'Ideal' : 'Fast';
          setWpmDisplay(`${wpm} WPM · ${tag}`);
        }
      }
      setInterimText(interim);
    };

    r.onerror = e => {
      if (e.error === 'not-allowed') setSpeechOK(false);
      setTalking(false); setInterimText('');
    };
    r.onend = () => { setInterimText(''); };

    recogRef.current = r;
    return () => { streamRef.current?.getTracks().forEach(t => t.stop()); r.abort(); };
  }, []);

  function handleTalk() {
    if (!recogRef.current) return;
    if (talking) {
      recogRef.current.stop();
      setTalking(false);
    } else {
      setFinalText('');
      setInterimText('');
      wordCount.current = 0;
      startTS.current = Date.now();
      try { recogRef.current.start(); setTalking(true); }
      catch(e) { console.warn('Recognition start error:', e); }
    }
  }

  const displayText = finalText + interimText;

  return (
    <div className="tab-content space-y-5">

      {/* Question Banner */}
      <div className="glass rounded-2xl px-5 py-4 border-l-4 border-l-violet-500">
        <div className="flex items-start gap-3">
          <span className="text-violet-400 text-lg mt-0.5">🤖</span>
          <div>
            <p className="text-xs text-violet-400 font-semibold uppercase tracking-wider mb-1">AI Interviewer · Q3 of 8 · Technical Round</p>
            <p className="text-white text-sm font-medium leading-relaxed">
              "Walk me through how you handled thermal deflection measurements in your Arduino project, and what challenges you encountered. Be specific about your calibration approach."
            </p>
          </div>
        </div>
      </div>

      {/* Speech not supported warning */}
      {!speechOK && (
        <div className="speech-warn rounded-xl px-4 py-3 text-red-300 text-sm flex items-center gap-2">
          <span>🎙</span>
          <span>Speech recognition not available in this browser. Use <strong>Chrome</strong> or <strong>Edge</strong> for live transcription. (Safari is not supported by the Web Speech API.)</span>
        </div>
      )}

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-5">

        {/* User Video */}
        <div className="glass rounded-2xl p-4 space-y-3">
          <h3 className="text-sm font-semibold text-slate-300">Your Feed</h3>
          <div className="relative bg-slate-800 rounded-xl overflow-hidden" style={{ aspectRatio:'16/9' }}>
            {!camError ? (
              <video ref={videoRef} autoPlay muted playsInline className="w-full h-full object-cover"/>
            ) : (
              <div className="w-full h-full flex flex-col items-center justify-center gap-3">
                <div className="w-20 h-20 rounded-full bg-indigo-900/60 border-2 border-indigo-500/40 flex items-center justify-center">
                  <span className="text-3xl">👤</span>
                </div>
                <p className="text-slate-500 text-xs">Camera unavailable · Placeholder</p>
              </div>
            )}
            {/* Telemetry badges */}
            <div className="absolute top-2 left-2 right-2 flex flex-wrap gap-1.5">
              <span className="bg-black/70 backdrop-blur text-emerald-400 text-[10px] font-semibold px-2 py-0.5 rounded-full border border-emerald-500/30">
                🎙 {wpmDisplay}
              </span>
              <span className={`bg-black/70 backdrop-blur text-[10px] font-semibold px-2 py-0.5 rounded-full border ${talking ? 'text-yellow-400 border-yellow-500/30 animate-pulse' : 'text-slate-400 border-slate-600/30'}`}>
                💬 Fillers: {talking ? 'Detecting…' : '2 detected'}
              </span>
              <span className="bg-black/70 backdrop-blur text-indigo-300 text-[10px] font-semibold px-2 py-0.5 rounded-full border border-indigo-500/30">
                👁 Eye Contact: Good
              </span>
            </div>
            {/* Live waveform when speaking */}
            {talking && (
              <div className="absolute bottom-2 left-0 right-0 flex items-center justify-center gap-0.5">
                {[...Array(16)].map((_,i) => (
                  <div key={i} className="wave-bar bg-indigo-400" style={{ height:'14px', animationDelay:`${i*0.07}s` }}/>
                ))}
              </div>
            )}
          </div>

          <button onClick={handleTalk}
            className={`w-full py-2.5 rounded-xl font-semibold text-sm transition-all duration-200 active:scale-95 flex items-center justify-center gap-2
              ${talking ? 'bg-red-600 hover:bg-red-500 text-white animate-pulse' : 'bg-indigo-600 hover:bg-indigo-500 text-white'}`}>
            {talking ? '🔴 Listening… (click to stop)' : '🎙 Push to Talk'}
          </button>
        </div>

        {/* AI Interviewer */}
        <div className="glass rounded-2xl p-4 space-y-3">
          <h3 className="text-sm font-semibold text-slate-300">AI Interviewer</h3>
          <div className="bg-slate-800/70 rounded-xl p-4 flex flex-col items-center gap-3">
            <div className="relative">
              <div className="w-20 h-20 rounded-full bg-gradient-to-br from-violet-600 to-indigo-700 flex items-center justify-center border-2 border-violet-500/50 shadow-lg shadow-violet-900/40">
                <span className="text-3xl">🤖</span>
              </div>
              <div className="absolute -bottom-1 -right-1 w-5 h-5 bg-emerald-500 rounded-full border-2 border-slate-800 flex items-center justify-center">
                <div className="w-2 h-2 bg-white rounded-full"/>
              </div>
            </div>
            {/* Audio waveform */}
            <div className="flex items-center justify-center gap-0.5 h-8">
              {[...Array(18)].map((_,i) => (
                <div key={i} className="wave-bar bg-violet-400"
                  style={{ height:`${8+Math.sin(i*0.6)*10}px`, animationDelay:`${i*0.07}s`,
                    animationPlayState: talking ? 'paused' : 'running' }}/>
              ))}
            </div>
            <p className="text-xs text-slate-400 italic text-center">
              "Please describe the calibration challenge — how did you account for non-linearity above 60°C?"
            </p>
          </div>

          {/* Live Transcription Box */}
          <div className="bg-slate-900/80 border border-slate-700/50 rounded-xl p-3 min-h-[120px]">
            <div className="flex items-center justify-between mb-2">
              <p className="text-xs text-slate-500 uppercase tracking-wide font-medium">Live Transcription</p>
              {talking && (
                <span className="flex items-center gap-1.5 text-[10px] text-red-400 font-semibold animate-pulse">
                  <span className="w-1.5 h-1.5 bg-red-400 rounded-full"/>REC
                </span>
              )}
            </div>
            <p className="text-sm text-slate-200 leading-relaxed">
              {finalText}
              {interimText && <span className="interim-text">{interimText}</span>}
              {!displayText && (
                <span className="text-slate-600 italic text-xs">
                  {speechOK
                    ? 'Click "Push to Talk" and speak — your words will appear here in real time...'
                    : 'Speech recognition not supported. Try Chrome or Edge.'}
                </span>
              )}
            </p>
          </div>

          <button
            onClick={() => displayText.trim().length > 5 && setGradeModal(true)}
            disabled={displayText.trim().length <= 5}
            className="w-full py-2.5 rounded-xl bg-violet-600 hover:bg-violet-500 disabled:opacity-40 disabled:cursor-not-allowed text-white font-semibold text-sm transition-all active:scale-95"
          >
            Submit Answer & Grade →
          </button>
        </div>
      </div>

      {/* Progress strip */}
      <div className="glass rounded-xl px-5 py-3 flex items-center gap-4">
        <span className="text-xs text-slate-400 flex-shrink-0">Interview Progress</span>
        <div className="flex-1 h-1.5 bg-slate-700 rounded-full overflow-hidden">
          <div className="h-full bg-gradient-to-r from-indigo-500 to-violet-500 rounded-full" style={{ width:'37.5%' }}/>
        </div>
        <span className="text-xs font-semibold text-slate-300 flex-shrink-0">Q3/8 · 37%</span>
        <div className="flex gap-1">
          {[...Array(8)].map((_,i) => (
            <div key={i} className={`w-2 h-2 rounded-full ${i<3 ? 'bg-indigo-500' : 'bg-slate-700'}`}/>
          ))}
        </div>
      </div>

      {/* Grade Modal */}
      <Modal open={gradeModal} onClose={() => setGradeModal(false)}>
        <div className="bg-slate-900 border border-violet-500/40 rounded-2xl p-6 shadow-2xl">
          <div className="text-center mb-5">
            <div className="w-14 h-14 bg-violet-500/20 rounded-full flex items-center justify-center mx-auto mb-3 text-3xl">🏆</div>
            <h3 className="text-xl font-bold text-white">Response Scorecard</h3>
            <p className="text-slate-400 text-xs mt-1">AI Grading · Technical Round · Q3</p>
          </div>
          <div className="space-y-4 mb-5">
            {[
              { label:'Technical Accuracy', score:8, color:'bg-emerald-500', comment:'Piecewise lookup table is an excellent engineering call for non-linear sensors.' },
              { label:'STAR Framework',     score:7, color:'bg-indigo-500',  comment:'Good S/T/A coverage. Quantify the Result more — business impact matters in interviews.' },
              { label:'Conciseness',        score:9, color:'bg-violet-500',  comment:'Specific numbers (±0.4°C, 96.8%) add strong credibility. Very well structured.' },
            ].map(m => (
              <div key={m.label}>
                <div className="flex justify-between mb-1">
                  <span className="text-sm font-medium text-slate-300">{m.label}</span>
                  <span className="text-sm font-bold text-white">{m.score}<span className="text-slate-500">/10</span></span>
                </div>
                <div className="h-2 bg-slate-700 rounded-full overflow-hidden mb-1.5">
                  <div className={`h-full ${m.color} rounded-full`} style={{ width:`${m.score*10}%` }}/>
                </div>
                <p className="text-xs text-slate-500 italic">{m.comment}</p>
              </div>
            ))}
          </div>
          <div className="bg-slate-800 rounded-xl p-3 mb-4 flex items-center justify-between">
            <span className="text-sm font-semibold text-white">Overall Score</span>
            <span className="text-2xl font-bold gradient-text">8.0 / 10</span>
          </div>
          <p className="text-xs text-slate-400 text-center mb-4">Strong response. Would likely pass this round at Bosch / Infosys.</p>
          <button onClick={() => setGradeModal(false)} className="w-full bg-violet-600 hover:bg-violet-500 text-white font-semibold py-2.5 rounded-xl transition-all">Next Question →</button>
        </div>
      </Modal>
    </div>
  );
}

/* ════════════════════════════════════════════════════════════
   ⑥ ROOT APP
════════════════════════════════════════════════════════════ */
function App() {
  const [user, setUser]     = useState(null);      // null = not logged in
  const [authLoading, setAuthLoading] = useState(FB_READY); // waiting for Firebase
  const [page, setPage]     = useState('dashboard');
  const [countdown]         = useState(42);

  // Listen to Firebase auth state
  useEffect(() => {
    if (!FB_READY || !fbAuth) { setAuthLoading(false); return; }
    const unsub = fbAuth.onAuthStateChanged(fbUser => {
      if (fbUser) {
        setUser({ name: fbUser.displayName || 'Candidate', email: fbUser.email, photo: fbUser.photoURL, uid: fbUser.uid, demo: false });
      } else {
        setUser(null);
      }
      setAuthLoading(false);
    });
    return unsub;
  }, []);

  async function handleLogout() {
    if (fbAuth && user && !user.demo) await fbAuth.signOut();
    setUser(null);
    setPage('dashboard');
  }

  const navItems = [
    { id:'dashboard', label:'Dashboard',     icon:'🏠' },
    { id:'practice',  label:'/practice',     icon:'💡' },
    { id:'interview', label:'/interview-lab',icon:'🎙' },
  ];

  // Loading spinner while Firebase resolves
  if (authLoading) return (
    <div className="min-h-screen bg-slate-950 flex items-center justify-center">
      <div className="flex flex-col items-center gap-3">
        <div className="w-10 h-10 border-2 border-indigo-500 border-t-transparent rounded-full animate-spin"/>
        <p className="text-slate-400 text-sm">Checking authentication…</p>
      </div>
    </div>
  );

  // Not logged in → Login page
  if (!user) return <LoginPage onLogin={setUser}/>;

  // Logged in → Main app
  return (
    <div className="min-h-screen bg-slate-950 text-slate-100">

      {/* Header */}
      <header className="sticky top-0 z-40 border-b border-slate-800/80 bg-slate-950/90 backdrop-blur-xl">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 py-3 flex flex-wrap items-center gap-3">

          {/* Brand */}
          <div className="flex items-center gap-2.5 mr-2">
            <div className="w-8 h-8 rounded-lg bg-gradient-to-br from-indigo-500 to-violet-600 flex items-center justify-center shadow-lg shadow-indigo-900/40 text-sm font-bold text-white">P</div>
            <div>
              <h1 className="text-base font-bold gradient-text leading-none">PlaceCraft AI</h1>
              <span className="text-[10px] text-slate-500">Mechanical &amp; Software Tracks</span>
            </div>
          </div>

          {/* Nav */}
          <nav className="flex items-center gap-1 bg-slate-900/60 rounded-xl p-1 border border-slate-800/60">
            {navItems.map(n => (
              <button key={n.id} onClick={() => setPage(n.id)}
                className={`flex items-center gap-1.5 px-3 py-1.5 rounded-lg text-xs font-semibold transition-all duration-200
                  ${page === n.id ? 'nav-active text-white shadow-lg' : 'text-slate-400 hover:text-slate-200 hover:bg-slate-800/60'}`}>
                <span>{n.icon}</span> {n.label}
              </button>
            ))}
          </nav>

          <div className="flex items-center gap-2 ml-auto">
            {/* Countdown */}
            <div className="hidden sm:flex items-center gap-1.5 bg-red-950/50 border border-red-800/50 px-3 py-1.5 rounded-xl">
              <span className="text-red-400 text-xs">⏳</span>
              <span className="text-red-300 text-xs font-semibold">Campus Drive in: <strong>{countdown} Days</strong></span>
            </div>

            {/* Profile chip */}
            <div className="flex items-center gap-2 bg-slate-800/80 border border-slate-700/50 rounded-xl px-3 py-1.5">
              {user.photo ? (
                <img src={user.photo} alt={user.name} className="w-6 h-6 rounded-full object-cover border border-indigo-500/40"/>
              ) : (
                <div className="w-6 h-6 rounded-full bg-gradient-to-br from-indigo-500 to-violet-600 flex items-center justify-center text-[10px] font-bold text-white">
                  {user.name?.[0] || 'A'}
                </div>
              )}
              <div className="hidden sm:block">
                <p className="text-xs font-semibold text-slate-200 leading-none">{user.name?.split(' ')[0] || 'Candidate'}</p>
                <p className="text-[10px] text-slate-500 leading-none mt-0.5">{user.demo ? 'Demo Mode' : 'Bosch & Tier-1 Tech'}</p>
              </div>
              <button onClick={handleLogout} title="Sign out"
                className="ml-1 text-slate-500 hover:text-red-400 transition-colors text-xs">⏻</button>
            </div>
          </div>
        </div>
      </header>

      {/* Page heading */}
      <main className="max-w-7xl mx-auto px-4 sm:px-6 py-6">
        {page === 'dashboard' && (
          <div className="mb-5">
            <h2 className="text-xl font-bold text-white">Good afternoon, {user.name?.split(' ')[0]} 👋</h2>
            <p className="text-slate-400 text-sm mt-0.5">Your Bosch & Infosys drive is in <span className="text-red-400 font-semibold">{countdown} days</span>. Here's where you stand today.</p>
          </div>
        )}
        {page === 'practice' && (
          <div className="mb-5">
            <h2 className="text-xl font-bold text-white">Practice Sandbox</h2>
            <p className="text-slate-400 text-sm mt-0.5">Speed Drill — Session 7 of 10 · Relative Speed & Trains</p>
          </div>
        )}
        {page === 'interview' && (
          <div className="mb-5 flex items-center gap-3">
            <div>
              <div className="flex items-center gap-2">
                <h2 className="text-xl font-bold text-white">AI Interview Lab</h2>
                <span className="bg-emerald-950/60 text-emerald-400 text-xs px-2.5 py-0.5 rounded-full border border-emerald-700/40 font-medium animate-pulse">● LIVE SESSION</span>
              </div>
              <p className="text-slate-400 text-sm mt-0.5">Simulated Bosch Technical Round · Arduino & Embedded Systems</p>
            </div>
          </div>
        )}

        {page === 'dashboard' && <DashboardPage navigate={setPage}/>}
        {page === 'practice'  && <PracticePage/>}
        {page === 'interview' && <InterviewLabPage/>}
      </main>

      <footer className="border-t border-slate-800/60 mt-8 py-4 px-6">
        <div className="max-w-7xl mx-auto flex items-center justify-between text-xs text-slate-600">
          <span>PlaceCraft AI · v2.5.0 · Campus Placement Season 2026–27</span>
          <span>{user.name} · {user.demo ? 'Demo Mode' : 'Bosch & Tier-1 Tech'}</span>
        </div>
      </footer>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(<App/>);
</script>
</body>
</html>
