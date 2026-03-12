import { useState, useEffect, useCallback } from "react";

// ════════════════════════════════════════════════════════════════
// 1. CONSTANTES & CONFIGURATION
// ════════════════════════════════════════════════════════════════

const PHI = 1.6180339887;

const FIB_RATIOS = [
  { ratio: "0.236", value: 0.236, importance: 1 },
  { ratio: "0.382", value: 0.382, importance: 3 },
  { ratio: "0.500", value: 0.500, importance: 2 },
  { ratio: "0.618", value: 0.618, importance: 3 },
  { ratio: "0.786", value: 0.786, importance: 1 },
];

const TIMEFRAMES = ["W", "D", "H4"];

const COLORS = {
  gold:       "#d4af37",
  goldDark:   "#c9971c",
  green:      "#22c55e",
  red:        "#ef4444",
  bg:         "#070710",
  bgCard:     "#0d0d1a",
  bgCard2:    "#111128",
  border:     "#1e1e3a",
  text:       "#e0e0ff",
  muted:      "#666",
  faint:      "#555",
};

// ════════════════════════════════════════════════════════════════
// 2. LOGIQUE MÉTIER — CALCULS PURS
// ════════════════════════════════════════════════════════════════

/**
 * Calcule les niveaux de retracement Fibonacci
 * entre un point haut et un point bas.
 */
function calcFibonacciLevels(high, low) {
  if (isNaN(high) || isNaN(low) || high <= low) return null;
  const diff = high - low;
  return FIB_RATIOS.map(({ ratio, value, importance }) => ({
    ratio,
    importance,
    price: +(high - diff * value).toFixed(2),
  }));
}

/**
 * Calcule le Stop-Loss et les 3 Take-Profits
 * en utilisant les puissances successives du Ratio d'Or φ.
 */
function calcTradePlan(entry, riskPercent) {
  if (isNaN(entry) || isNaN(riskPercent) || entry <= 0 || riskPercent <= 0) return null;
  const sl      = +(entry * (1 - riskPercent / 100)).toFixed(2);
  const riskAmt = entry - sl;
  return {
    entry,
    sl,
    tp1: +(entry + riskAmt * PHI).toFixed(2),
    tp2: +(entry + riskAmt * PHI ** 2).toFixed(2),
    tp3: +(entry + riskAmt * PHI ** 3).toFixed(2),
    rr1: +PHI.toFixed(2),
    rr2: +(PHI ** 2).toFixed(2),
    rr3: +(PHI ** 3).toFixed(2),
  };
}

/**
 * Calcule le score Neo-Convergence (0–100)
 * selon 4 critères pondérés.
 */
function calcNeoScore({ ema, rsi, priceAboveEma50H4 }) {
  const bullTFCount = TIMEFRAMES.filter(tf => ema[tf].ema50 && ema[tf].ema200).length;
  const rsiScore    = rsi < 40 ? 25 : rsi < 55 ? 10 : 0;
  const emaScore    = priceAboveEma50H4 ? 20 : 0;
  const crossScore  = ema.H4.cross ? 20 : 0;
  const tfScore     = Math.round((bullTFCount / 3) * 35);
  return Math.min(100, tfScore + rsiScore + emaScore + crossScore);
}

/**
 * Détermine le label et la couleur d'un score Neo.
 */
function scoreLabel(score) {
  if (score >= 75) return { label: "CONVERGENCE", color: COLORS.green };
  if (score >= 45) return { label: "PARTIEL",     color: COLORS.gold };
  return               { label: "FAIBLE",         color: COLORS.red };
}

// ════════════════════════════════════════════════════════════════
// 3. SIMULATION DE DONNÉES MARCHÉ (remplace un WebSocket réel)
// ════════════════════════════════════════════════════════════════

function randomEMAState() {
  return Object.fromEntries(
    TIMEFRAMES.map(tf => [
      tf,
      {
        ema50:  Math.random() > 0.4,
        ema200: Math.random() > 0.45,
        cross:  Math.random() > 0.55,
      },
    ])
  );
}

function randomMarket(prev) {
  const price  = prev ? +(prev.price + (Math.random() - 0.49) * 8).toFixed(2) : +(4200 + Math.random() * 800).toFixed(2);
  const rsi    = prev ? Math.max(10, Math.min(90, +(prev.rsi + (Math.random() - 0.5) * 3).toFixed(1))) : +(25 + Math.random() * 60).toFixed(1);
  const change = +((Math.random() - 0.45) * 6).toFixed(2);
  const volume = +(1.2 + Math.random() * 3).toFixed(2);
  return { price, rsi, change, volume };
}

function buildSparklinePoints(count = 20, positive = true) {
  return Array.from({ length: count }, (_, i) => {
    const base = positive ? 50 - i * 1.5 : 50 + i * 1.5;
    return base + (Math.random() - 0.5) * 12;
  }).reverse();
}

function sparklineToSVGPath(points) {
  const min  = Math.min(...points);
  const max  = Math.max(...points);
  const norm = points.map(p => ((p - min) / (max - min || 1)) * 36);
  return norm.map((y, i) => `${i === 0 ? "M" : "L"} ${(i / (norm.length - 1)) * 100} ${38 - y}`).join(" ");
}

// ════════════════════════════════════════════════════════════════
// 4. SOUS-COMPOSANTS UI (purs, sans logique métier)
// ════════════════════════════════════════════════════════════════

function Sparkline({ positive }) {
  const pts  = buildSparklinePoints(20, positive);
  const path = sparklineToSVGPath(pts);
  const clr  = positive ? COLORS.gold : COLORS.red;
  return (
    <svg viewBox="0 0 100 40" className="w-full h-8" preserveAspectRatio="none">
      <defs>
        <linearGradient id="sg" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%"   stopColor={clr} stopOpacity="0.3" />
          <stop offset="100%" stopColor={clr} stopOpacity="0"   />
        </linearGradient>
      </defs>
      <path d={`${path} L 100 40 L 0 40 Z`} fill="url(#sg)" />
      <path d={path} fill="none" stroke={clr} strokeWidth="1.5" />
    </svg>
  );
}

function RadialScore({ score }) {
  const r     = 54;
  const circ  = 2 * Math.PI * r;
  const { label, color } = scoreLabel(score);
  return (
    <div style={{ position: "relative", width: 140, height: 140, display: "flex", alignItems: "center", justifyContent: "center" }}>
      <svg width="140" height="140" style={{ position: "absolute" }}>
        <circle cx="70" cy="70" r={r} fill="none" stroke="#1a1a2e" strokeWidth="10" />
        <circle cx="70" cy="70" r={r} fill="none" stroke={color} strokeWidth="10"
          strokeDasharray={circ}
          strokeDashoffset={circ - (score / 100) * circ}
          strokeLinecap="round"
          style={{ transformOrigin: "70px 70px", transform: "rotate(-90deg)", transition: "stroke-dashoffset 1s ease, stroke 0.5s ease" }}
        />
        {[...Array(12)].map((_, i) => {
          const a   = ((i / 12) * 360 - 90) * Math.PI / 180;
          return <line key={i} x1={70 + 48 * Math.cos(a)} y1={70 + 48 * Math.sin(a)} x2={70 + 44 * Math.cos(a)} y2={70 + 44 * Math.sin(a)} stroke="#2a2a4a" strokeWidth="1" />;
        })}
      </svg>
      <div style={{ display: "flex", flexDirection: "column", alignItems: "center", zIndex: 10 }}>
        <span style={{ fontSize: 28, fontFamily: "'Orbitron', monospace", color, fontWeight: 700, lineHeight: 1 }}>{score}</span>
        <span style={{ fontSize: 9,  letterSpacing: "0.15em", color: "#888", marginTop: 2 }}>{label}</span>
      </div>
    </div>
  );
}

function Card({ children, style = {}, highlight = false }) {
  return (
    <div style={{
      background: `linear-gradient(135deg, ${COLORS.bgCard} 0%, ${COLORS.bgCard2} 100%)`,
      border: `1px solid ${highlight ? "rgba(212,175,55,0.3)" : COLORS.border}`,
      borderRadius: 14, padding: 16,
      animation: highlight ? "pulse-gold 2s ease-in-out infinite" : "none",
      ...style,
    }}>{children}</div>
  );
}

function CardTitle({ icon, children }) {
  return (
    <div style={{ fontSize: 11, fontFamily: "'Orbitron', monospace", color: COLORS.gold, letterSpacing: "0.12em", marginBottom: 12, display: "flex", alignItems: "center", gap: 6 }}>
      {icon && <span>{icon}</span>}
      {children}
    </div>
  );
}

function StatusPill({ ok, children }) {
  return (
    <span style={{
      fontSize: 9, padding: "2px 7px", borderRadius: 4, fontWeight: 700,
      background: ok ? "rgba(34,197,94,0.15)" : "rgba(239,68,68,0.15)",
      color:      ok ? COLORS.green            : COLORS.red,
      border:    `1px solid ${ok ? "rgba(34,197,94,0.3)" : "rgba(239,68,68,0.3)"}`,
    }}>{children}</span>
  );
}

function SignalRow({ label, ok, detail }) {
  return (
    <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", padding: "5px 10px", borderRadius: 7, background: ok ? "rgba(34,197,94,0.07)" : "rgba(239,68,68,0.05)", border: `1px solid ${ok ? "rgba(34,197,94,0.2)" : "rgba(239,68,68,0.1)"}` }}>
      <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
        <div style={{ width: 6, height: 6, borderRadius: "50%", background: ok ? COLORS.green : COLORS.red }} />
        <span style={{ fontSize: 10, color: ok ? "#ccc" : COLORS.muted }}>{label}</span>
      </div>
      <span style={{ fontSize: 9, color: ok ? COLORS.green : COLORS.red }}>{detail}</span>
    </div>
  );
}

function GoldButton({ onClick, children }) {
  return (
    <button onClick={onClick} style={{
      alignSelf: "flex-end", background: `linear-gradient(135deg, ${COLORS.goldDark}, ${COLORS.gold})`,
      border: "none", borderRadius: 7, padding: "7px 16px",
      color: "#000", fontWeight: 700, fontSize: 11, cursor: "pointer",
      fontFamily: "'Orbitron', monospace",
    }}>{children}</button>
  );
}

function NumberInput({ label, value, onChange, color }) {
  return (
    <div style={{ flex: 1 }}>
      <label style={{ fontSize: 9, color: "#888", letterSpacing: "0.1em", display: "block", marginBottom: 4 }}>{label}</label>
      <input value={value} onChange={e => onChange(e.target.value)} style={{
        width: "100%", background: COLORS.bgCard, border: `1px solid ${COLORS.border}`,
        borderRadius: 7, padding: "7px 10px", color: color || COLORS.text,
        fontSize: 14, fontFamily: "'Orbitron', monospace", outline: "none",
      }} />
    </div>
  );
}

// ════════════════════════════════════════════════════════════════
// 5. COMPOSANTS SECTIONS (logique locale + rendu)
// ════════════════════════════════════════════════════════════════

function EMAVisualizer({ ema }) {
  return (
    <Card>
      <CardTitle icon="📊">VISUALISEUR EMA — 3 TIMEFRAMES</CardTitle>
      <div style={{ display: "flex", gap: 8, overflowX: "auto" }}>
        {TIMEFRAMES.map(tf => (
          <div key={tf} style={{ background: `linear-gradient(135deg, #0d0d1a, #13132a)`, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "10px 14px", minWidth: 110 }}>
            <div style={{ fontSize: 11, color: COLORS.gold, fontFamily: "'Orbitron', monospace", letterSpacing: "0.12em", marginBottom: 8 }}>{tf}</div>
            {["ema50", "ema200"].map(k => (
              <div key={k} style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 4 }}>
                <span style={{ fontSize: 10, color: COLORS.muted }}>{k === "ema50" ? "EMA 50" : "EMA 200"}</span>
                <StatusPill ok={ema[tf][k]}>{ema[tf][k] ? "▲ BULL" : "▼ BEAR"}</StatusPill>
              </div>
            ))}
            <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginTop: 6, paddingTop: 6, borderTop: `1px solid ${COLORS.border}` }}>
              <span style={{ fontSize: 10, color: COLORS.muted }}>Cross</span>
              <span style={{ fontSize: 9, color: ema[tf].cross ? COLORS.gold : COLORS.faint }}>{ema[tf].cross ? "⚡ ACTIF" : "— OFF"}</span>
            </div>
          </div>
        ))}
      </div>
    </Card>
  );
}

function NeoConvergence({ ema, market }) {
  const priceAboveEma50H4 = ema.H4.ema50;
  const score   = calcNeoScore({ ema, rsi: market.rsi, priceAboveEma50H4 });
  const signals = [
    { label: "RSI < 40",       ok: market.rsi < 40,    detail: `RSI = ${market.rsi}` },
    { label: "Prix > EMA50 H4", ok: priceAboveEma50H4, detail: priceAboveEma50H4 ? "Confirmé" : "Invalide" },
    { label: "Cross EMA actif", ok: ema.H4.cross,      detail: ema.H4.cross ? "Actif" : "Inactif" },
    { label: "Bull 3 TF",       ok: TIMEFRAMES.filter(tf => ema[tf].ema50 && ema[tf].ema200).length >= 2, detail: `${TIMEFRAMES.filter(tf => ema[tf].ema50 && ema[tf].ema200).length}/3 TF` },
  ];
  return (
    <Card highlight={score >= 75}>
      <CardTitle icon="◈">INDICATEUR NEO CONVERGENCE</CardTitle>
      <div style={{ display: "flex", alignItems: "center", gap: 16 }}>
        <RadialScore score={score} />
        <div style={{ flex: 1, display: "flex", flexDirection: "column", gap: 6 }}>
          {signals.map(s => <SignalRow key={s.label} {...s} />)}
        </div>
      </div>
    </Card>
  );
}

function FibonacciEngine() {
  const [high,   setHigh]   = useState("4800");
  const [low,    setLow]    = useState("4000");
  const [levels, setLevels] = useState(null);

  const handleCalc = useCallback(() => {
    setLevels(calcFibonacciLevels(parseFloat(high), parseFloat(low)));
  }, [high, low]);

  return (
    <Card>
      <CardTitle icon="φ">MOTEUR FIBONACCI</CardTitle>
      <div style={{ display: "flex", gap: 10, marginBottom: 14 }}>
        <NumberInput label="HIGH" value={high} onChange={setHigh} color={COLORS.green} />
        <NumberInput label="LOW"  value={low}  onChange={setLow}  color={COLORS.red}   />
        <GoldButton onClick={handleCalc}>CALC</GoldButton>
      </div>

      {levels ? (
        <div style={{ display: "flex", flexDirection: "column", gap: 5 }}>
          {levels.map(({ ratio, price, importance }) => (
            <div key={ratio} style={{ display: "flex", alignItems: "center", justifyContent: "space-between", background: importance === 3 ? "rgba(212,175,55,0.07)" : "transparent", borderRadius: 6, padding: "5px 8px", border: `1px solid ${importance === 3 ? "rgba(212,175,55,0.2)" : "transparent"}` }}>
              <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                <span style={{ fontSize: 10, color: COLORS.faint, width: 36 }}>{ratio}</span>
                <div style={{ display: "flex", gap: 3 }}>
                  {[...Array(importance)].map((_, i) => <div key={i} style={{ width: 5, height: 5, borderRadius: 2, background: COLORS.gold, opacity: 0.8 }} />)}
                </div>
              </div>
              <span style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color: importance === 3 ? COLORS.gold : "#aaa", fontWeight: importance === 3 ? 700 : 400 }}>${price.toLocaleString()}</span>
            </div>
          ))}
        </div>
      ) : (
        <div style={{ textAlign: "center", padding: "16px 0", color: "#333", fontSize: 11 }}>Entrez un HIGH et un LOW, puis CALC</div>
      )}

      <div style={{ marginTop: 14, padding: 12, background: "#07070f", borderRadius: 10, border: `1px solid ${COLORS.border}` }}>
        <div style={{ fontSize: 10, color: COLORS.gold, fontFamily: "'Orbitron', monospace", marginBottom: 8 }}>GUIDE</div>
        {[["0.382", "Premier rebond possible"], ["0.500", "Zone médiane psychologique"], ["0.618 ★", "Golden Ratio — zone principale"], ["0.786", "Dernier support avant invalidation"]].map(([r, d]) => (
          <div key={r} style={{ display: "flex", gap: 10, marginBottom: 5 }}>
            <span style={{ fontSize: 10, color: r.includes("★") ? COLORS.gold : COLORS.faint, minWidth: 52 }}>{r}</span>
            <span style={{ fontSize: 10, color: COLORS.muted }}>{d}</span>
          </div>
        ))}
      </div>
    </Card>
  );
}

function TradePlanEngine({ currentPrice }) {
  const [entry, setEntry] = useState(currentPrice?.toFixed(2) || "4500");
  const [risk,  setRisk]  = useState("1.5");
  const [plan,  setPlan]  = useState(null);

  const handleGen = useCallback(() => {
    setPlan(calcTradePlan(parseFloat(entry), parseFloat(risk)));
  }, [entry, risk]);

  return (
    <Card>
      <CardTitle icon="📋">PLAN DE TRADE</CardTitle>
      <div style={{ display: "flex", gap: 10, marginBottom: 14 }}>
        <NumberInput label="ENTRY $" value={entry} onChange={setEntry} />
        <NumberInput label="RISK %"  value={risk}  onChange={setRisk}  />
        <GoldButton onClick={handleGen}>GEN</GoldButton>
      </div>

      {plan ? (
        <div style={{ display: "flex", flexDirection: "column", gap: 5 }}>
          {/* Stop-Loss */}
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", padding: "6px 10px", background: "rgba(239,68,68,0.1)", borderRadius: 8, border: "1px solid rgba(239,68,68,0.25)" }}>
            <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <div style={{ width: 8, height: 8, borderRadius: "50%", background: COLORS.red }} />
              <span style={{ fontSize: 10, color: COLORS.red, fontFamily: "'Orbitron', monospace" }}>STOP-LOSS</span>
            </div>
            <span style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color: COLORS.red, fontWeight: 700 }}>${plan.sl.toLocaleString()}</span>
          </div>

          {/* Entry */}
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", padding: "5px 10px", background: COLORS.bgCard, borderRadius: 7, border: `1px solid ${COLORS.border}` }}>
            <span style={{ fontSize: 10, color: COLORS.muted }}>ENTRY</span>
            <span style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color: COLORS.text }}>${plan.entry.toLocaleString()}</span>
          </div>

          {/* Take-Profits */}
          {[["TP1", plan.tp1, plan.rr1, "φ¹", "rgba(34,197,94,0.08)", "rgba(34,197,94,0.2)"],
            ["TP2", plan.tp2, plan.rr2, "φ²", "rgba(34,197,94,0.13)","rgba(34,197,94,0.3)"],
            ["TP3", plan.tp3, plan.rr3, "φ³", "rgba(34,197,94,0.18)","rgba(34,197,94,0.4)"]].map(
            ([label, price, rr, phi, bg, border]) => (
              <div key={label} style={{ display: "flex", alignItems: "center", justifyContent: "space-between", padding: "6px 10px", background: bg, borderRadius: 8, border: `1px solid ${border}` }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                  <div style={{ width: 8, height: 8, borderRadius: "50%", background: COLORS.green }} />
                  <span style={{ fontSize: 10, color: COLORS.green, fontFamily: "'Orbitron', monospace" }}>{label}</span>
                  <span style={{ fontSize: 9, color: COLORS.faint }}>{phi} = {rr}R</span>
                </div>
                <span style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color: COLORS.green, fontWeight: 700 }}>${price.toLocaleString()}</span>
              </div>
            )
          )}
        </div>
      ) : (
        <div style={{ textAlign: "center", padding: "16px 0", color: "#333", fontSize: 11 }}>Renseignez l'entrée et le risque, puis GEN</div>
      )}

      <div style={{ marginTop: 14, padding: 12, background: "#07070f", borderRadius: 10, border: `1px solid ${COLORS.border}` }}>
        <div style={{ fontSize: 10, color: COLORS.gold, fontFamily: "'Orbitron', monospace", marginBottom: 6 }}>RATIO D'OR φ = {PHI}</div>
        {[["TP1", "φ¹", PHI.toFixed(2)], ["TP2", "φ²", (PHI**2).toFixed(2)], ["TP3", "φ³", (PHI**3).toFixed(2)]].map(([tp, phi, rr]) => (
          <div key={tp} style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>
            <span style={{ color: "#888" }}>{tp} = Entry + Risk × {phi} ({rr}R)</span>
          </div>
        ))}
      </div>
    </Card>
  );
}

// ════════════════════════════════════════════════════════════════
// 6. COMPOSANT RACINE — ÉTAT GLOBAL + RENDU
// ════════════════════════════════════════════════════════════════

export default function NeoTradingDashboard() {
  const [ema,       setEma]    = useState(randomEMAState);
  const [market,    setMarket] = useState(() => randomMarket(null));
  const [tick,      setTick]   = useState(0);
  const [activeTab, setTab]    = useState("overview");

  // Flux de données simulé (remplaçable par WebSocket)
  useEffect(() => {
    const iv = setInterval(() => {
      setMarket(prev => randomMarket(prev));
      setTick(t => t + 1);
    }, 2000);
    return () => clearInterval(iv);
  }, []);

  useEffect(() => {
    if (tick % 15 === 0 && tick > 0) setEma(randomEMAState());
  }, [tick]);

  const pos = market.change >= 0;

  const TABS = [
    { id: "overview", label: "VUE GLOBALE" },
    { id: "fib",      label: "FIBONACCI"   },
    { id: "plan",     label: "TRADE PLAN"  },
  ];

  return (
    <div style={{ minHeight: "100vh", background: COLORS.bg, fontFamily: "'Rajdhani', 'Orbitron', sans-serif", color: COLORS.text, backgroundImage: "radial-gradient(ellipse at 20% 20%, rgba(212,175,55,0.04), transparent 60%), radial-gradient(ellipse at 80% 80%, rgba(30,30,80,0.5), transparent 60%)" }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Rajdhani:wght@300;500;700&display=swap');
        * { box-sizing: border-box; }
        input:focus { border-color: ${COLORS.gold} !important; box-shadow: 0 0 0 2px rgba(212,175,55,0.15); }
        @keyframes pulse-gold { 0%,100%{box-shadow:0 0 8px rgba(212,175,55,0.4)} 50%{box-shadow:0 0 20px rgba(212,175,55,0.8)} }
        @keyframes blink      { 0%,100%{opacity:1} 50%{opacity:0.4} }
        ::-webkit-scrollbar { width: 4px; }
        ::-webkit-scrollbar-thumb { background: #2a2a4a; border-radius: 2px; }
      `}</style>

      {/* ── HEADER ── */}
      <div style={{ borderBottom: `1px solid ${COLORS.border}`, padding: "12px 20px", display: "flex", alignItems: "center", justifyContent: "space-between", background: "rgba(10,10,24,0.9)", position: "sticky", top: 0, zIndex: 50, backdropFilter: "blur(10px)" }}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 32, height: 32, borderRadius: 8, background: `linear-gradient(135deg, ${COLORS.goldDark}, ${COLORS.gold})`, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16 }}>◈</div>
          <div>
            <div style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color: COLORS.gold, letterSpacing: "0.15em", lineHeight: 1 }}>NEO-TRADING</div>
            <div style={{ fontSize: 9, color: COLORS.faint, letterSpacing: "0.2em" }}>SYSTEM v2.0</div>
          </div>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
          <div style={{ textAlign: "right" }}>
            <div style={{ fontSize: 20, fontFamily: "'Orbitron', monospace", fontWeight: 700, color: pos ? COLORS.green : COLORS.red, lineHeight: 1 }}>${market.price.toLocaleString()}</div>
            <div style={{ fontSize: 10, color: pos ? COLORS.green : COLORS.red }}>{pos ? "▲" : "▼"} {Math.abs(market.change)}%</div>
          </div>
          <div style={{ width: 8, height: 8, borderRadius: "50%", background: COLORS.green, animation: "blink 1.5s ease-in-out infinite" }} />
        </div>
      </div>

      {/* ── TABS ── */}
      <div style={{ display: "flex", padding: "10px 16px", gap: 8 }}>
        {TABS.map(({ id, label }) => (
          <button key={id} onClick={() => setTab(id)} style={{
            flex: 1, padding: "8px 0", borderRadius: 8, cursor: "pointer",
            background: activeTab === id ? `linear-gradient(135deg, ${COLORS.goldDark}, ${COLORS.gold})` : COLORS.bgCard,
            color:  activeTab === id ? "#000" : COLORS.muted,
            border: activeTab === id ? "none" : `1px solid ${COLORS.border}`,
            fontSize: 10, fontFamily: "'Orbitron', monospace", letterSpacing: "0.1em",
            fontWeight: activeTab === id ? 700 : 400, transition: "all 0.2s",
          }}>{label}</button>
        ))}
      </div>

      {/* ── CONTENT ── */}
      <div style={{ padding: "0 16px 24px", display: "flex", flexDirection: "column", gap: 12 }}>

        {activeTab === "overview" && (<>
          {/* Market Stats */}
          <div style={{ display: "grid", gridTemplateColumns: "repeat(3,1fr)", gap: 8 }}>
            {[
              ["RSI", market.rsi, market.rsi < 30 ? COLORS.green : market.rsi > 70 ? COLORS.red : COLORS.gold, market.rsi < 30 ? "SURVENTE" : market.rsi > 70 ? "SURACHAT" : "NEUTRE"],
              ["PRIX", `$${market.price}`, pos ? COLORS.green : COLORS.red, pos ? "HAUSSIER" : "BAISSIER"],
              ["VOL",  `${market.volume}B`, "#888", "24H"],
            ].map(([lbl, val, color, sub]) => (
              <div key={lbl} style={{ background: `linear-gradient(135deg, ${COLORS.bgCard}, ${COLORS.bgCard2})`, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "10px 12px" }}>
                <div style={{ fontSize: 9, color: COLORS.faint, letterSpacing: "0.12em", marginBottom: 4 }}>{lbl}</div>
                <div style={{ fontSize: 13, fontFamily: "'Orbitron', monospace", color, fontWeight: 700 }}>{val}</div>
                <div style={{ fontSize: 8, color, opacity: 0.7, marginTop: 2 }}>{sub}</div>
              </div>
            ))}
          </div>

          {/* Sparkline */}
          <Card style={{ padding: "12px 14px" }}>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 6 }}>
              <span style={{ fontSize: 10, color: COLORS.faint, letterSpacing: "0.1em" }}>PRICE ACTION</span>
              <span style={{ fontSize: 9, color: pos ? COLORS.green : COLORS.red }}>{pos ? "+" : ""}{market.change}%</span>
            </div>
            <Sparkline positive={pos} />
          </Card>

          <NeoConvergence ema={ema} market={market} />
          <EMAVisualizer ema={ema} />
        </>)}

        {activeTab === "fib"  && <FibonacciEngine />}
        {activeTab === "plan" && <TradePlanEngine currentPrice={market.price} />}
      </div>
    </div>
  );
}
