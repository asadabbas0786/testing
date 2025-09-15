import React, { useEffect, useMemo, useRef, useState } from "react";
import { useNavigate, useParams, useLocation } from "react-router-dom";

const DEFAULT_STREAM_URL = "https://onelearning.org.in/pi-stream/";
const DEFAULT_POSTURE_API = "http://127.0.0.1:8000/posture";

const Positioning = () => {
  const { topicId, courseId } = useParams();
  const navigate = useNavigate();
  const location = useLocation();

  // incoming state
  const registrationId   = location.state?.registration_id || "";
  const reportId         = location.state?.reportId;
  const assignmentId     = location.state?.assignmentId;
  const teacher_username = location.state?.teacher_username;

  const initialStreamUrl = useMemo(
    () => location.state?.piStreamUrl || DEFAULT_STREAM_URL,
    [location.state?.piStreamUrl]
  );
  const postureApi = useMemo(
    () => location.state?.postureApi || DEFAULT_POSTURE_API,
    [location.state?.postureApi]
  );

  // state
  const [streamUrl, setStreamUrl] = useState(initialStreamUrl);
  const [isStreaming, setIsStreaming] = useState(true);
  const [error, setError] = useState("");
  const [streamHealthy, setStreamHealthy] = useState("checking"); // ok | bad | checking
  const [posture, setPosture] = useState({ position: "Detecting…", confidence: 0, suggestions: [], keypoints: null, connections: null });
  const [lastUpdated, setLastUpdated] = useState(null);
  const [history, setHistory] = useState([]);
  const [fps, setFps] = useState(0);
  const [resolution, setResolution] = useState({ w: 0, h: 0 });
  const [showOverlay, setShowOverlay] = useState(true);
  const [showGrid, setShowGrid] = useState(false);

  // refs
  const imgRef = useRef(null);
  const canvasRef = useRef(null);
  const rafRef = useRef(null);
  const boxRef = useRef(null);

  // cover-fit helper
  function coverFit(imgW, imgH, boxW, boxH) {
    if (!imgW || !imgH || !boxW || !boxH) return { dx: 0, dy: 0, dw: boxW, dh: boxH };
    const s  = Math.max(boxW / imgW, boxH / imgH);
    const dw = Math.round(imgW * s);
    const dh = Math.round(imgH * s);
    const dx = Math.floor((boxW - dw) / 2);
    const dy = Math.floor((boxH - dh) / 2);
    return { dx, dy, dw, dh };
  }

  // stream draw (compact, with eslint-friendly cleanup)
  useEffect(() => {
    if (!isStreaming) return;

    setError("");
    setStreamHealthy("checking");

    const img = new Image();
    img.crossOrigin = "anonymous";
    imgRef.current = img;
    img.src = streamUrl;

    const canvas = canvasRef.current;
    const ctx = canvas ? canvas.getContext("2d") : null;
    const observedEl = boxRef.current;

    const syncCanvasSize = () => {
      if (!observedEl || !canvas) return;
      const { clientWidth: bw, clientHeight: bh } = observedEl;
      if (bw && bh && (canvas.width !== bw || canvas.height !== bh)) {
        canvas.width = bw;
        canvas.height = bh;
      }
    };
    syncCanvasSize();

    const ro = new ResizeObserver(syncCanvasSize);
    if (observedEl) ro.observe(observedEl);

    let frames = 0;
    let tick = Date.now();
    let lastNW = 0, lastNH = 0;

    const draw = () => {
      if (!ctx || !imgRef.current) return;

      const nW = imgRef.current.naturalWidth || 1280;
      const nH = imgRef.current.naturalHeight || 720;
      if (nW !== lastNW || nH !== lastNH) {
        lastNW = nW; lastNH = nH;
        setResolution({ w: nW, h: nH });
      }

      syncCanvasSize();
      const cw = canvas.width || 1;
      const ch = canvas.height || 1;

      try {
        ctx.clearRect(0, 0, cw, ch);
        const { dx, dy, dw, dh } = coverFit(nW, nH, cw, ch);
        ctx.drawImage(imgRef.current, 0, 0, nW, nH, dx, dy, dw, dh);

        if (showGrid) drawGrid(ctx, cw, ch);
        if (showOverlay && posture?.keypoints?.length) {
          drawOverlayFitted(ctx, cw, ch, { keypoints: posture.keypoints, connections: posture.connections }, { dx, dy, dw, dh });
        }

        frames++;
        const now = Date.now();
        if (now - tick >= 1000) {
          setFps(frames); frames = 0; tick = now; setStreamHealthy("ok");
        }
      } catch {}

      rafRef.current = requestAnimationFrame(draw);
    };

    const onLoad = () => { if (!rafRef.current) rafRef.current = requestAnimationFrame(draw); };
    const onError = () => { setError("Unable to load stream. Check the URL or CORS on the Pi server."); setStreamHealthy("bad"); };

    img.addEventListener("load", onLoad);
    img.addEventListener("error", onError);

    return () => {
      img.removeEventListener("load", onLoad);
      img.removeEventListener("error", onError);
      try { img.src = ""; } catch {}
      imgRef.current = null;
      if (rafRef.current) cancelAnimationFrame(rafRef.current);
      rafRef.current = null;
      try { if (observedEl) ro.unobserve(observedEl); } catch {}
      try { ro.disconnect(); } catch {}
    };
  }, [streamUrl, isStreaming, showOverlay, showGrid, posture.keypoints, posture.connections]);

  // posture polling (1s)
  useEffect(() => {
    let stop = false, timer;
    const fetchPosture = async () => {
      try {
        const res = await fetch(postureApi, { cache: "no-store" });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        if (stop) return;

        const next = {
          position: json.position ?? "Unknown",
          confidence: Number.isFinite(json.confidence) ? json.confidence : 0,
          suggestions: Array.isArray(json.suggestions) ? json.suggestions : [],
          keypoints: Array.isArray(json.keypoints) ? json.keypoints : null,
          connections: Array.isArray(json.connections) ? json.connections : null,
        };
        setPosture(next);
        setLastUpdated(new Date());
        setHistory(h => (h[0]?.position === next.position ? h : [{ position: next.position, ts: new Date() }, ...h].slice(0,5)));
      } catch {/* keep last */}
      finally { if (!stop) timer = setTimeout(fetchPosture, 1000); }
    };
    fetchPosture();
    return () => { stop = true; if (timer) clearTimeout(timer); };
  }, [postureApi]);

  const handleToggle   = () => setIsStreaming(s => !s);
  const handleSnapshot = () => {
    const canvas = canvasRef.current; if (!canvas) return;
    const a = document.createElement("a");
    a.href = canvas.toDataURL("image/png"); a.download = `snapshot_${Date.now()}.png`; a.click();
  };
  const handleNext = () => {
    if (!topicId || !courseId) return;
    navigate(`/student-dashboard/courses/ongoing/${topicId}/protocols/image-acquisition/${courseId}`, {
      state: { registration_id: registrationId, reportId, assignmentId, teacher_username, piStreamUrl: streamUrl, postureApi }
    });
  };

  const aspect = resolution.w && resolution.h ? `${gcdRatio(resolution.w, resolution.h)} (${resolution.w}×${resolution.h})` : "—";
  const updatedText = lastUpdated
    ? new Intl.DateTimeFormat(undefined, { hour: "2-digit", minute: "2-digit", second: "2-digit" }).format(lastUpdated)
    : "—";

  return (
    <div className="h-full overflow-hidden bg-gradient-hero text-white p-2 md:p-3">
      <div className="max-w-6xl mx-auto h-full">
        {/* Card, fixed to viewport height */}
        <div className="h-[calc(100vh-16px)] bg-[#151a1e]/90 rounded-xl shadow-2xl border border-[#26313b] overflow-hidden">
          <div className="grid grid-cols-1 lg:grid-cols-2 h-full">
            {/* LEFT: media fills column */}
            <div className="relative bg-black">
              <div
                ref={boxRef}
                className="absolute inset-0"
                style={{ display: "flex", alignItems: "center", justifyContent: "center" }}
              >
                <canvas ref={canvasRef} className="block w-full h-full" />
              </div>

              {/* top badges */}
              <div className="absolute top-0 left-0 right-0 px-2 py-2 flex items-center justify-between bg-gradient-to-b from-black/40 to-transparent">
                <h2 className="text-xs md:text-sm font-semibold">Camera • Gantry-Top View</h2>
                <div className="flex items-center gap-1.5">
                  <Badge color={streamHealthy === "ok" ? "emerald" : streamHealthy === "bad" ? "red" : "slate"} label={streamHealthy === "ok" ? "Live" : streamHealthy === "bad" ? "Offline" : "Checking…"} />
                  <Badge color="slate" label={`${fps} fps`} />
                  <Badge color="slate" label={resolution.w ? `${resolution.w}×${resolution.h}` : "—"} />
                </div>
              </div>

              {/* bottom controls (compact) */}
              <div className="absolute bottom-3 left-3 flex flex-wrap gap-1.5">
                <button
                  className={`px-3 py-1.5 rounded-md text-xs font-semibold shadow ${isStreaming ? "bg-red-600 hover:bg-red-700" : "bg-green-600 hover:bg-green-700"}`}
                  onClick={handleToggle}
                >
                  {isStreaming ? "Stop" : "Start"}
                </button>
                <button className="px-3 py-1.5 rounded-md text-xs font-semibold shadow bg-indigo-600 hover:bg-indigo-700" onClick={handleSnapshot}>
                  Snapshot
                </button>
                <label className="flex items-center gap-1.5 bg-[#0f1316]/80 px-2 py-1.5 rounded-md text-[11px] border border-[#2a3642]">
                  <input type="checkbox" checked={showOverlay} onChange={(e)=>setShowOverlay(e.target.checked)} />
                  Overlay
                </label>
                <label className="flex items-center gap-1.5 bg-[#0f1316]/80 px-2 py-1.5 rounded-md text-[11px] border border-[#2a3642]">
                  <input type="checkbox" checked={showGrid} onChange={(e)=>setShowGrid(e.target.checked)} />
                  Grid
                </label>
              </div>

              {error && <div className="absolute bottom-3 right-3 text-[11px] bg-red-600/90 px-2.5 py-1.5 rounded">{error}</div>}
            </div>

            {/* RIGHT: info; scrolls inside only if needed */}
            <div className="h-full overflow-auto p-4 md:p-5 bg-gradient-hero">
              <h1 className="text-2xl md:text-3xl font-extrabold tracking-tight">Patient Positioning</h1>

              <div className="mt-4 space-y-4">
                {/* position */}
                <div>
                  <p className="text-slate-300/90 text-xs">Detected Position</p>
                  <p className="text-xl md:text-2xl font-bold mt-0.5">{posture.position}</p>
                </div>

                {/* confidence */}
                <div>
                  <p className="text-slate-300/90 text-xs mb-1">Confidence</p>
                  <div className="w-full h-2.5 bg-[#0f1316] rounded-full border border-[#2a3642] overflow-hidden">
                    <div className="h-full bg-emerald-500 transition-all" style={{ width: `${Math.max(0, Math.min(100, Math.round(posture.confidence)))}%` }} />
                  </div>
                  <p className="mt-1 text-xs font-semibold">{Math.round(posture.confidence)}%</p>
                </div>

                {/* suggestions */}
                <div>
                  <p className="text-slate-300/90 text-xs mb-1.5">Suggestions</p>
                  {posture.suggestions?.length ? (
                    <ul className="space-y-1.5">
                      {posture.suggestions.map((s, i) => (
                        <li key={i} className="flex items-start gap-2">
                          <span className="mt-[6px] inline-block w-1.5 h-1.5 rounded-full bg-emerald-400" />
                          <span className="text-[13px]">{s}</span>
                        </li>
                      ))}
                    </ul>
                  ) : (
                    <p className="text-slate-400 text-xs">No suggestions</p>
                  )}
                </div>

                {/* stream info */}
                <div className="bg-[#0f1316]/70 border border-[#2a3642] rounded-lg p-3">
                  <p className="text-xs font-semibold mb-2">Stream Info</p>
                  <div className="grid grid-cols-2 gap-2 text-xs">
                    <Info label="Resolution" value={resolution.w ? `${resolution.w} × ${resolution.h}` : "—"} />
                    <Info label="Aspect" value={aspect} />
                    <Info label="FPS" value={`${fps}`} />
                    <Info label="Last Update" value={updatedText} />
                  </div>
                </div>

                {/* history */}
                <div>
                  <p className="text-slate-300/90 text-xs mb-1">Recent detections</p>
                  {history.length ? (
                    <div className="flex flex-wrap gap-1.5">
                      {history.map((h, i) => (
                        <Badge key={i} color="slate" label={`${h.position} • ${timeAgo(h.ts)}`} />
                      ))}
                    </div>
                  ) : (
                    <p className="text-slate-500 text-xs">—</p>
                  )}
                </div>

                {/* actions */}
                <div className="pt-1 flex gap-2">
                  <button className="px-3 py-1.5 rounded-lg bg-[#2d6dff] hover:bg-[#2357cc] text-sm font-semibold" onClick={() => window.location.reload()}>
                    Retry Scan
                  </button>
                  <button className="px-3 py-1.5 rounded-lg bg-emerald-600 hover:bg-emerald-700 text-sm font-semibold" onClick={handleNext}>
                    Next
                  </button>
                </div>

                {/* stream url */}
                <div className="mt-3">
                  <label className="block text-[11px] text-slate-400 mb-1">Stream URL (MJPEG)</label>
                  <div className="flex gap-2">
                    <input
                      type="text"
                      className="w-full bg-[#0f1316] border border-[#2a3642] rounded-md px-3 py-2 text-sm"
                      value={streamUrl}
                      onChange={(e) => setStreamUrl(e.target.value)}
                      placeholder="http://<pi-ip>:<port>/video_feed"
                    />
                    <button className="px-3 py-2 rounded-md bg-slate-700 hover:bg-slate-600 text-sm font-semibold" onClick={() => setIsStreaming(true)} title="Reconnect">
                      Apply
                    </button>
                  </div>
                  <p className="text-[11px] text-slate-400 mt-1">
                    Video fills the pane (no stretching). Excess edges are safely cropped.
                  </p>
                </div>
              </div>
            </div>
            {/* END RIGHT */}
          </div>
        </div>
      </div>
    </div>
  );
};

/* Small bits */
function Badge({ color = "slate", label = "" }) {
  const map = {
    slate: "bg-slate-700/60 text-slate-200 border border-slate-600/60",
    emerald: "bg-emerald-600/70 text-emerald-50 border border-emerald-500/70",
    red: "bg-red-600/70 text-red-50 border border-red-500/70",
  };
  return <span className={`px-2 py-1 rounded-md text-[11px] font-semibold ${map[color] || map.slate}`}>{label}</span>;
}

function Info({ label, value }) {
  return (
    <div className="bg-[#0b0f13] rounded-md border border-[#2a3642] px-2.5 py-2">
      <p className="text-[10px] text-slate-400">{label}</p>
      <p className="text-xs font-semibold">{value}</p>
    </div>
  );
}

function drawOverlayFitted(ctx, W, H, { keypoints = [], connections = null }, fit) {
  const { dx, dy, dw, dh } = fit;
  if (connections?.length) {
    ctx.lineWidth = Math.max(2, Math.round(Math.min(W, H) * 0.003));
    ctx.strokeStyle = "rgba(0, 200, 255, 0.9)";
    ctx.beginPath();
    for (const [a, b] of connections) {
      const pa = keypoints[a], pb = keypoints[b];
      if (!pa || !pb) continue;
      ctx.moveTo(dx + pa.x * dw, dy + pa.y * dh);
      ctx.lineTo(dx + pb.x * dw, dy + pb.y * dh);
    }
    ctx.stroke();
  }
  const r = Math.max(3, Math.round(Math.min(W, H) * 0.004));
  for (const kp of keypoints) {
    ctx.fillStyle = "#30e88c";
    ctx.beginPath();
    ctx.arc(dx + kp.x * dw, dy + kp.y * dh, r, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawGrid(ctx, W, H) {
  ctx.save();
  ctx.strokeStyle = "rgba(255,255,255,0.15)";
  ctx.lineWidth = 1;
  const thirdsX = [W/3, (2*W)/3];
  const thirdsY = [H/3, (2*H)/3];
  ctx.beginPath();
  thirdsX.forEach(x => { ctx.moveTo(x, 0); ctx.lineTo(x, H); });
  thirdsY.forEach(y => { ctx.moveTo(0, y); ctx.lineTo(W, y); });
  ctx.stroke();
  ctx.restore();
}

function gcdRatio(w, h) {
  const g = (a, b) => (b ? g(b, a % b) : a);
  if (!w || !h) return "—";
  const d = g(w, h);
  return `${w/d}:${h/d}`;
}
function timeAgo(ts) {
  const diff = Math.max(0, (Date.now() - new Date(ts).getTime())/1000);
  if (diff < 60) return `${Math.round(diff)}s ago`;
  if (diff < 3600) return `${Math.round(diff/60)}m ago`;
  return `${Math.round(diff/3600)}h ago`;
}

export default Positioning;
