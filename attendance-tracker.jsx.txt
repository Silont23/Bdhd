import { useState, useEffect, useRef, useCallback } from "react";

// ── Palette & Fonts via CSS variables ──────────────────────────────────────
const GLOBAL_STYLE = `
  @import url('https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=Syne:wght@400;600;800&display=swap');

  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

  :root {
    --bg:       #0a0c10;
    --surface:  #111318;
    --card:     #16191f;
    --border:   #252930;
    --accent:   #00e5a0;
    --accent2:  #ff6b35;
    --text:     #e8eaf0;
    --muted:    #6b7280;
    --danger:   #ff4757;
    --live:     #00e5a0;
  }

  body { background: var(--bg); color: var(--text); font-family: 'Syne', sans-serif; }

  ::-webkit-scrollbar { width: 4px; }
  ::-webkit-scrollbar-track { background: var(--surface); }
  ::-webkit-scrollbar-thumb { background: var(--border); border-radius: 4px; }

  @keyframes pulse {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.5; transform: scale(0.85); }
  }
  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(8px); }
    to   { opacity: 1; transform: translateY(0); }
  }
  @keyframes scanline {
    0%   { top: -4px; }
    100% { top: 100%; }
  }
`;

// ─── Helpers ────────────────────────────────────────────────────────────────
const now = () => new Date().toISOString();
const fmt = (iso) => {
  const d = new Date(iso);
  return d.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit", second: "2-digit" });
};
const fmtDate = (iso) => new Date(iso).toLocaleDateString([], { month: "short", day: "numeric" });

function diffFrames(prev, curr, w, h) {
  let changed = 0;
  for (let i = 0; i < prev.length; i += 4) {
    const dr = Math.abs(prev[i] - curr[i]);
    const dg = Math.abs(prev[i + 1] - curr[i + 1]);
    const db = Math.abs(prev[i + 2] - curr[i + 2]);
    if (dr + dg + db > 60) changed++;
  }
  return changed / (w * h);
}

// ─── Main Component ──────────────────────────────────────────────────────────
export default function AttendanceTracker() {
  const videoRef = useRef(null);
  const canvasRef = useRef(null);
  const prevFrameRef = useRef(null);
  const streamRef = useRef(null);
  const intervalRef = useRef(null);

  const [cameraOn, setCameraOn] = useState(false);
  const [detecting, setDetecting] = useState(false);
  const [motionLevel, setMotionLevel] = useState(0);
  const [events, setEvents] = useState([]);
  const [tab, setTab] = useState("live"); // "live" | "log"
  const [status, setStatus] = useState("idle"); // idle | motion | clear
  const [cooldown, setCooldown] = useState(false);
  const [todayCount, setTodayCount] = useState(0);
  const [sensitivity, setSensitivity] = useState(0.03); // 3% pixel change threshold

  // ── Load from storage ──────────────────────────────────────────────────────
  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get("attendance-events");
        if (res) {
          const loaded = JSON.parse(res.value);
          setEvents(loaded);
          const today = new Date().toDateString();
          setTodayCount(loaded.filter(e => new Date(e.time).toDateString() === today).length);
        }
      } catch (_) {}
    })();
  }, []);

  // ── Save to storage ────────────────────────────────────────────────────────
  const saveEvents = useCallback(async (evts) => {
    try {
      await window.storage.set("attendance-events", JSON.stringify(evts));
    } catch (_) {}
  }, []);

  // ── Log detection event ───────────────────────────────────────────────────
  const logEvent = useCallback((motion) => {
    if (cooldown) return;
    setCooldown(true);
    setTimeout(() => setCooldown(false), 4000);

    const entry = { id: Date.now(), time: now(), motion: Math.round(motion * 1000) / 10 };
    setEvents(prev => {
      const next = [entry, ...prev].slice(0, 200);
      saveEvents(next);
      const today = new Date().toDateString();
      setTodayCount(next.filter(e => new Date(e.time).toDateString() === today).length);
      return next;
    });
  }, [cooldown, saveEvents]);

  // ── Camera ────────────────────────────────────────────────────────────────
  const startCamera = useCallback(async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 640, height: 480 } });
      streamRef.current = stream;
      if (videoRef.current) {
        videoRef.current.srcObject = stream;
        await videoRef.current.play();
      }
      setCameraOn(true);
    } catch (e) {
      alert("Camera access denied or unavailable.");
    }
  }, []);

  const stopCamera = useCallback(() => {
    if (streamRef.current) {
      streamRef.current.getTracks().forEach(t => t.stop());
      streamRef.current = null;
    }
    setCameraOn(false);
    setDetecting(false);
    clearInterval(intervalRef.current);
  }, []);

  // ── Detection Loop ─────────────────────────────────────────────────────────
  const startDetection = useCallback(() => {
    setDetecting(true);
    prevFrameRef.current = null;

    intervalRef.current = setInterval(() => {
      const video = videoRef.current;
      const canvas = canvasRef.current;
      if (!video || !canvas || video.readyState < 2) return;

      const W = 160, H = 120;
      canvas.width = W; canvas.height = H;
      const ctx = canvas.getContext("2d");
      ctx.drawImage(video, 0, 0, W, H);
      const frame = ctx.getImageData(0, 0, W, H).data;

      if (prevFrameRef.current) {
        const diff = diffFrames(prevFrameRef.current, frame, W, H);
        setMotionLevel(diff);
        if (diff > sensitivity) {
          setStatus("motion");
          logEvent(diff);
        } else {
          setStatus("clear");
        }
      }
      prevFrameRef.current = new Uint8ClampedArray(frame);
    }, 300);
  }, [sensitivity, logEvent]);

  const stopDetection = useCallback(() => {
    setDetecting(false);
    clearInterval(intervalRef.current);
    setStatus("idle");
    setMotionLevel(0);
  }, []);

  useEffect(() => () => { stopCamera(); clearInterval(intervalRef.current); }, [stopCamera]);

  const clearLog = async () => {
    if (!confirm("Clear all attendance records?")) return;
    setEvents([]);
    setTodayCount(0);
    try { await window.storage.delete("attendance-events"); } catch (_) {}
  };

  // ── Motion bar width ──────────────────────────────────────────────────────
  const motionPct = Math.min(motionLevel / (sensitivity * 3), 1) * 100;
  const motionColor = motionLevel > sensitivity ? "var(--accent2)" : "var(--accent)";

  return (
    <>
      <style>{GLOBAL_STYLE}</style>
      <div style={{ minHeight: "100vh", padding: "24px 16px", maxWidth: 900, margin: "0 auto" }}>

        {/* ── Header ── */}
        <header style={{ display: "flex", alignItems: "flex-end", justifyContent: "space-between", marginBottom: 32 }}>
          <div>
            <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--accent)", letterSpacing: "0.2em", marginBottom: 4 }}>
              ◈ VISION SENTINEL
            </div>
            <h1 style={{ fontFamily: "'Syne', sans-serif", fontWeight: 800, fontSize: "clamp(24px,5vw,40px)", lineHeight: 1, color: "var(--text)" }}>
              Attendance<br />
              <span style={{ color: "var(--accent)" }}>Tracker</span>
            </h1>
          </div>
          <div style={{ textAlign: "right" }}>
            <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--muted)", marginBottom: 4 }}>TODAY</div>
            <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 36, fontWeight: 700, color: "var(--accent)", lineHeight: 1 }}>
              {String(todayCount).padStart(3, "0")}
            </div>
            <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)" }}>detections</div>
          </div>
        </header>

        {/* ── Stat Row ── */}
        <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 12, marginBottom: 28 }}>
          {[
            { label: "TOTAL LOGGED", value: events.length },
            { label: "CAMERA", value: cameraOn ? "ONLINE" : "OFFLINE", accent: cameraOn },
            { label: "DETECTION", value: detecting ? "ACTIVE" : "PAUSED", accent: detecting },
          ].map(({ label, value, accent }) => (
            <div key={label} style={{
              background: "var(--card)", border: "1px solid var(--border)",
              borderRadius: 12, padding: "14px 16px",
              animation: "fadeIn 0.4s ease both"
            }}>
              <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)", marginBottom: 6, letterSpacing: "0.12em" }}>{label}</div>
              <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 18, fontWeight: 700, color: accent ? "var(--accent)" : "var(--text)" }}>{value}</div>
            </div>
          ))}
        </div>

        {/* ── Main Grid ── */}
        <div style={{ display: "grid", gridTemplateColumns: "1fr 340px", gap: 20, alignItems: "start" }}>

          {/* Camera Panel */}
          <div style={{ background: "var(--card)", border: "1px solid var(--border)", borderRadius: 16, overflow: "hidden" }}>
            {/* Video area */}
            <div style={{ position: "relative", background: "#070810", aspectRatio: "4/3", display: "flex", alignItems: "center", justifyContent: "center", overflow: "hidden" }}>
              <video ref={videoRef} muted playsInline style={{
                width: "100%", height: "100%", objectFit: "cover",
                display: cameraOn ? "block" : "none",
                transform: "scaleX(-1)"
              }} />
              <canvas ref={canvasRef} style={{ display: "none" }} />

              {!cameraOn && (
                <div style={{ textAlign: "center", color: "var(--muted)" }}>
                  <div style={{ fontSize: 48, marginBottom: 12 }}>⬡</div>
                  <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 12 }}>NO SIGNAL</div>
                </div>
              )}

              {/* Scanline effect */}
              {cameraOn && detecting && (
                <div style={{
                  position: "absolute", left: 0, right: 0, height: 2,
                  background: "linear-gradient(transparent, var(--accent), transparent)",
                  opacity: 0.4, animation: "scanline 3s linear infinite", pointerEvents: "none"
                }} />
              )}

              {/* Status badge */}
              {cameraOn && (
                <div style={{
                  position: "absolute", top: 12, left: 12,
                  display: "flex", alignItems: "center", gap: 6,
                  background: "rgba(0,0,0,0.7)", borderRadius: 20, padding: "5px 12px",
                  fontFamily: "'Space Mono', monospace", fontSize: 11
                }}>
                  <span style={{
                    width: 7, height: 7, borderRadius: "50%",
                    background: status === "motion" ? "var(--accent2)" : detecting ? "var(--accent)" : "var(--muted)",
                    animation: detecting ? "pulse 1.2s ease infinite" : "none"
                  }} />
                  {status === "motion" ? "MOTION DETECTED" : detecting ? "MONITORING" : "LIVE"}
                </div>
              )}

              {/* Motion level indicator */}
              {detecting && (
                <div style={{ position: "absolute", bottom: 0, left: 0, right: 0, height: 3, background: "rgba(255,255,255,0.05)" }}>
                  <div style={{ height: "100%", width: `${motionPct}%`, background: motionColor, transition: "width 0.2s, background 0.2s" }} />
                </div>
              )}
            </div>

            {/* Controls */}
            <div style={{ padding: 16 }}>
              <div style={{ display: "flex", gap: 10, marginBottom: 16 }}>
                <button onClick={cameraOn ? stopCamera : startCamera} style={{
                  flex: 1, padding: "10px 0", border: "none", borderRadius: 10, cursor: "pointer",
                  fontFamily: "'Space Mono', monospace", fontSize: 12, fontWeight: 700, letterSpacing: "0.08em",
                  background: cameraOn ? "var(--danger)" : "var(--accent)",
                  color: cameraOn ? "#fff" : "#000", transition: "opacity 0.2s"
                }}>
                  {cameraOn ? "STOP CAMERA" : "START CAMERA"}
                </button>
                <button onClick={detecting ? stopDetection : startDetection} disabled={!cameraOn} style={{
                  flex: 1, padding: "10px 0", border: `1px solid ${detecting ? "var(--accent2)" : "var(--border)"}`,
                  borderRadius: 10, cursor: cameraOn ? "pointer" : "not-allowed",
                  fontFamily: "'Space Mono', monospace", fontSize: 12, fontWeight: 700, letterSpacing: "0.08em",
                  background: detecting ? "rgba(255,107,53,0.15)" : "transparent",
                  color: detecting ? "var(--accent2)" : "var(--muted)", transition: "all 0.2s"
                }}>
                  {detecting ? "STOP DETECT" : "START DETECT"}
                </button>
              </div>

              {/* Sensitivity */}
              <div>
                <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6 }}>
                  <span style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--muted)" }}>SENSITIVITY</span>
                  <span style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--accent)" }}>
                    {sensitivity < 0.02 ? "LOW" : sensitivity < 0.05 ? "MED" : "HIGH"}
                  </span>
                </div>
                <input type="range" min="0.005" max="0.1" step="0.005" value={sensitivity}
                  onChange={e => setSensitivity(parseFloat(e.target.value))}
                  style={{ width: "100%", accentColor: "var(--accent)", cursor: "pointer" }}
                />
              </div>

              {/* Motion meter */}
              {detecting && (
                <div style={{ marginTop: 14 }}>
                  <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)", marginBottom: 5 }}>MOTION LEVEL</div>
                  <div style={{ height: 6, background: "var(--border)", borderRadius: 3, overflow: "hidden" }}>
                    <div style={{ height: "100%", width: `${motionPct}%`, background: motionColor, borderRadius: 3, transition: "width 0.2s, background 0.2s" }} />
                  </div>
                  <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)", marginTop: 4, textAlign: "right" }}>
                    {(motionLevel * 100).toFixed(1)}% pixels changed
                  </div>
                </div>
              )}
            </div>
          </div>

          {/* Log Panel */}
          <div style={{ background: "var(--card)", border: "1px solid var(--border)", borderRadius: 16, overflow: "hidden", maxHeight: 520, display: "flex", flexDirection: "column" }}>
            <div style={{ padding: "14px 16px", borderBottom: "1px solid var(--border)", display: "flex", justifyContent: "space-between", alignItems: "center" }}>
              <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 12, color: "var(--text)", fontWeight: 700 }}>
                EVENT LOG
              </div>
              <button onClick={clearLog} style={{
                background: "transparent", border: "1px solid var(--border)", borderRadius: 6,
                padding: "4px 10px", fontFamily: "'Space Mono', monospace", fontSize: 10,
                color: "var(--muted)", cursor: "pointer"
              }}>CLEAR</button>
            </div>
            <div style={{ overflowY: "auto", flex: 1 }}>
              {events.length === 0 ? (
                <div style={{ padding: 24, textAlign: "center", color: "var(--muted)", fontFamily: "'Space Mono', monospace", fontSize: 12 }}>
                  No events yet.<br />Start the camera &amp; detection.
                </div>
              ) : (
                events.map((e, i) => (
                  <div key={e.id} style={{
                    padding: "10px 16px", borderBottom: "1px solid var(--border)",
                    display: "flex", alignItems: "center", gap: 10,
                    animation: i === 0 ? "fadeIn 0.3s ease both" : "none",
                    background: i === 0 ? "rgba(0,229,160,0.04)" : "transparent"
                  }}>
                    <div style={{
                      width: 8, height: 8, borderRadius: "50%", flexShrink: 0,
                      background: e.motion > sensitivity * 100 * 3 ? "var(--accent2)" : "var(--accent)"
                    }} />
                    <div style={{ flex: 1, minWidth: 0 }}>
                      <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--text)" }}>
                        {fmt(e.time)}
                      </div>
                      <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)" }}>
                        {fmtDate(e.time)} · {e.motion}% motion
                      </div>
                    </div>
                    <div style={{
                      width: 36, height: 4, background: "var(--border)", borderRadius: 2, flexShrink: 0, overflow: "hidden"
                    }}>
                      <div style={{
                        height: "100%", background: "var(--accent)",
                        width: `${Math.min(e.motion / (sensitivity * 3 * 100) * 100, 100)}%`
                      }} />
                    </div>
                  </div>
                ))
              )}
            </div>
          </div>
        </div>

        {/* ── How it works ── */}
        <div style={{ marginTop: 24, background: "var(--card)", border: "1px solid var(--border)", borderRadius: 16, padding: "18px 20px" }}>
          <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 11, color: "var(--accent)", marginBottom: 12, letterSpacing: "0.12em" }}>
            ◈ HOW IT WORKS
          </div>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(180px, 1fr))", gap: 16 }}>
            {[
              ["01", "Start Camera", "Grants webcam access via browser."],
              ["02", "Start Detection", "Analyzes 5 frames/sec for pixel changes."],
              ["03", "Motion Logged", "Every detection logs a timestamped entry."],
              ["04", "Persistent DB", "All records survive page reloads via storage."],
            ].map(([n, title, desc]) => (
              <div key={n}>
                <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--accent)", marginBottom: 4 }}>{n}</div>
                <div style={{ fontWeight: 700, marginBottom: 4, fontSize: 14 }}>{title}</div>
                <div style={{ fontSize: 13, color: "var(--muted)", lineHeight: 1.5 }}>{desc}</div>
              </div>
            ))}
          </div>
        </div>

        <div style={{ marginTop: 20, textAlign: "center", fontFamily: "'Space Mono', monospace", fontSize: 10, color: "var(--muted)" }}>
          ALL DATA STORED LOCALLY · CAMERA NEVER LEAVES YOUR BROWSER
        </div>
      </div>
    </>
  );
}
