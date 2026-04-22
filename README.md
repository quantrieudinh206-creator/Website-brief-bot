# Website-brief-bot
Bot can make morning brief
import { useState, useEffect, useRef } from "react";

const CLAUDE_MODEL = "claude-sonnet-4-20250514";

// ── Simulated real-time market data ──────────────────────────────────────────
function generateMarketData() {
  const base = {
    vnindex: 1247.35,
    vn30: 1312.8,
    hnx: 218.45,
    sp500: 5432.1,
    nasdaq: 17821.3,
    dowjones: 40123.5,
    oil: 82.45,
    gold: 2341.2,
    silver: 27.85,
    copper: 4.32,
    usdvnd: 25112,
    btc: 68420,
    eth: 3821,
    dxy: 104.2,
  };

  return Object.fromEntries(
    Object.entries(base).map(([k, v]) => [
      k,
      {
        value: v + (Math.random() - 0.5) * v * 0.003,
        change: (Math.random() - 0.48) * 2.5,
      },
    ])
  );
}

const INITIAL_NEWS = [
  { id: 1, region: "US", title: "Fed giữ nguyên lãi suất, tín hiệu cắt giảm vào Q3/2025", impact: "HIGH", sentiment: "positive", category: "Macro", time: "07:32" },
  { id: 2, region: "VN", title: "Khối ngoại mua ròng 420 tỷ VND phiên sáng, tập trung VHM và MWG", impact: "HIGH", sentiment: "positive", category: "Market", time: "09:15" },
  { id: 3, region: "CN", title: "PMI Sản xuất Trung Quốc giảm xuống 49.2, dưới ngưỡng kỳ vọng", impact: "MEDIUM", sentiment: "negative", category: "Macro", time: "08:45" },
  { id: 4, region: "EU", title: "ECB phát tín hiệu cắt giảm lãi suất vào tháng 6 nếu lạm phát hạ nhiệt", impact: "MEDIUM", sentiment: "positive", category: "Macro", time: "06:20" },
  { id: 5, region: "VN", title: "Giải ngân FDI 4 tháng đầu năm đạt 6.28 tỷ USD, tăng 7.4% YoY", impact: "MEDIUM", sentiment: "positive", category: "Macro", time: "08:00" },
  { id: 6, region: "US", title: "Dầu thô WTI +1.2% sau báo cáo tồn kho giảm mạnh hơn dự báo", impact: "MEDIUM", sentiment: "neutral", category: "Market", time: "07:10" },
  { id: 7, region: "VN", title: "NHNN bơm 15,000 tỷ VND qua kênh OMO, hỗ trợ thanh khoản hệ thống", impact: "HIGH", sentiment: "positive", category: "Macro", time: "09:30" },
  { id: 8, region: "GEO", title: "Căng thẳng Trung Đông leo thang, giá dầu có thể test $85-87", impact: "HIGH", sentiment: "negative", category: "Geopolitics", time: "05:45" },
];

const RISK_EVENTS = [
  { event: "Fed Minutes Release", date: "Thứ 4, 22h ET", impact: "HIGH" },
  { event: "US CPI (tháng 4)", date: "Thứ 5, 21h30 ET", impact: "HIGH" },
  { event: "VN GDP Q1 Official", date: "Thứ 6, 9h00 VN", impact: "MEDIUM" },
  { event: "ECB Interest Rate", date: "06/06, 20h15 ET", impact: "HIGH" },
];

// ── Helper components ─────────────────────────────────────────────────────────
const ImpactBadge = ({ impact }) => {
  const styles = {
    HIGH: "bg-red-500/20 text-red-400 border border-red-500/40",
    MEDIUM: "bg-amber-500/20 text-amber-400 border border-amber-500/40",
    LOW: "bg-slate-600/40 text-slate-400 border border-slate-600",
  };
  return (
    <span className={`text-[10px] font-bold px-1.5 py-0.5 rounded ${styles[impact]}`}>
      {impact}
    </span>
  );
};

const RegionTag = ({ region }) => {
  const colors = {
    US: "text-blue-400 bg-blue-400/10",
    VN: "text-red-400 bg-red-400/10",
    EU: "text-indigo-400 bg-indigo-400/10",
    CN: "text-yellow-400 bg-yellow-400/10",
    GEO: "text-orange-400 bg-orange-400/10",
    JP: "text-pink-400 bg-pink-400/10",
  };
  return (
    <span className={`text-[10px] font-bold px-1.5 py-0.5 rounded ${colors[region] || "text-slate-400 bg-slate-700"}`}>
      {region}
    </span>
  );
};

const Ticker = ({ label, data, prefix = "" }) => {
  const isUp = data.change >= 0;
  return (
    <div className="flex flex-col items-center px-4 py-1.5 border-r border-slate-700/50 last:border-r-0 min-w-[90px]">
      <span className="text-[10px] text-slate-500 font-mono uppercase tracking-widest">{label}</span>
      <span className="text-sm font-bold text-slate-100 font-mono">
        {prefix}{label === "USD/VND" ? Math.round(data.value).toLocaleString() : data.value.toFixed(2)}
      </span>
      <span className={`text-[11px] font-mono font-semibold ${isUp ? "text-emerald-400" : "text-red-400"}`}>
        {isUp ? "▲" : "▼"} {Math.abs(data.change).toFixed(2)}%
      </span>
    </div>
  );
};

const MarketCard = ({ label, data, prefix = "", large = false }) => {
  const isUp = data.change >= 0;
  return (
    <div className={`bg-slate-800/60 rounded border border-slate-700/50 p-3 hover:border-slate-500/50 transition-all duration-300 ${large ? "col-span-2" : ""}`}>
      <div className="text-[10px] text-slate-500 font-mono uppercase tracking-widest mb-1">{label}</div>
      <div className="text-xl font-bold text-slate-100 font-mono">
        {prefix}{label.includes("VND") ? Math.round(data.value).toLocaleString() : data.value.toFixed(2)}
      </div>
      <div className={`text-sm font-mono font-semibold flex items-center gap-1 mt-0.5 ${isUp ? "text-emerald-400" : "text-red-400"}`}>
        <span>{isUp ? "▲" : "▼"}</span>
        <span>{Math.abs(data.change).toFixed(2)}%</span>
        <div className={`ml-auto w-1.5 h-1.5 rounded-full animate-pulse ${isUp ? "bg-emerald-400" : "bg-red-400"}`} />
      </div>
    </div>
  );
};

// ── AI Report Panel ───────────────────────────────────────────────────────────
function AIReportPanel({ marketData, news }) {
  const [report, setReport] = useState("");
  const [loading, setLoading] = useState(false);
  const [reportType, setReportType] = useState("morning");
  const [error, setError] = useState("");
  const abortRef = useRef(null);

  const generateReport = async (type) => {
    setLoading(true);
    setReport("");
    setError("");
    setReportType(type);

    const newsContext = news
      .slice(0, 6)
      .map((n) => `[${n.region}][${n.impact}] ${n.title}`)
      .join("\n");

    const marketContext = `
VN-Index: ${marketData.vnindex?.value?.toFixed(2)} (${marketData.vnindex?.change?.toFixed(2)}%)
VN30: ${marketData.vn30?.value?.toFixed(2)} (${marketData.vn30?.change?.toFixed(2)}%)
S&P 500: ${marketData.sp500?.value?.toFixed(2)} (${marketData.sp500?.change?.toFixed(2)}%)
Dầu WTI: $${marketData.oil?.value?.toFixed(2)} (${marketData.oil?.change?.toFixed(2)}%)
Vàng: $${marketData.gold?.value?.toFixed(2)} (${marketData.gold?.change?.toFixed(2)}%)
USD/VND: ${Math.round(marketData.usdvnd?.value)?.toLocaleString()}
BTC: $${marketData.btc?.value?.toFixed(0)}`;

    const prompts = {
      morning: `Bạn là một Macro Strategist & Quant Analyst cao cấp tại một quỹ đầu tư lớn.

Dữ liệu thị trường hiện tại:
${marketContext}

Tin tức vĩ mô nổi bật:
${newsContext}

Hãy viết MORNING BRIEF cho trader Việt Nam. Format:
---
🌅 MORNING BRIEF — ${new Date().toLocaleDateString("vi-VN", { weekday: "long", day: "2-digit", month: "2-digit", year: "numeric" })}

📊 DIỄN BIẾN QUA ĐÊM
[2-3 câu tóm tắt thị trường Mỹ, châu Âu, hàng hóa qua đêm]

🧭 NHẬN ĐỊNH CHÍNH
Bias hôm nay: [BULLISH / NEUTRAL / BEARISH]
[1 câu lý giải ngắn gọn]

🎯 KEY DRIVERS
• [Driver 1: yếu tố quan trọng nhất]
• [Driver 2]
• [Driver 3]

📈 VN-INDEX HÔM NAY
Kịch bản base: [Mô tả ngắn]
Hỗ trợ: [mức] — Kháng cự: [mức]
Kịch bản tích cực: [Điều kiện & mục tiêu]
Kịch bản tiêu cực: [Điều kiện & mức nguy hiểm]

⚡ SECTORS NÊN THEO DÕI
• [Sector 1] — [Lý do ngắn]
• [Sector 2] — [Lý do ngắn]

⚠️ RỦI RO HÔM NAY
[1-2 rủi ro cụ thể cần watch]

💡 ACTIONABLE INSIGHT
[1 câu hành động cụ thể nhất cho trader ngày hôm nay]
---
Viết súc tích, dữ liệu cụ thể, không lan man. Phong cách Bloomberg Terminal.`,

      eod: `Bạn là một Macro Strategist & Quant Analyst cao cấp.

Dữ liệu thị trường hiện tại:
${marketContext}

Tin tức trong ngày:
${newsContext}

Hãy viết END-OF-DAY REPORT. Format:
---
🌆 END-OF-DAY REPORT — ${new Date().toLocaleDateString("vi-VN", { weekday: "long", day: "2-digit", month: "2-digit", year: "numeric" })}

📊 TỔNG KẾT PHIÊN
[2-3 câu diễn biến chính phiên hôm nay]

💰 DÒNG TIỀN
Khối ngoại: [Mua/Bán ròng và giá trị ước tính]
Tự doanh: [Nhận định]
Dòng tiền ngành nổi bật: [1-2 ngành]

🏆 SECTOR PERFORMANCE
• Mạnh nhất: [Sector + lý do]
• Yếu nhất: [Sector + lý do]

📌 THỐNG KÊ KỸ THUẬT
Market breadth: [Tăng/Giảm/Tham chiếu]
Thanh khoản: [So sánh bình quân]
Volume signal: [Tích lũy / Phân phối / Trung tính]

🔮 NHẬN ĐỊNH PHIÊN SAU
Bias: [BULLISH / NEUTRAL / BEARISH]
Kịch bản: [Mô tả ngắn]
Cần theo dõi: [1-2 catalyst]

💡 TRADE IDEA
[1 ý tưởng cụ thể có cơ sở dữ liệu]
---
Viết súc tích, không cảm tính, dựa trên data.`,

      macro: `Bạn là Macro Strategist với 20 năm kinh nghiệm.

Dữ liệu thị trường:
${marketContext}

Tin tức vĩ mô:
${newsContext}

Viết MACRO IMPACT ANALYSIS ngắn gọn:
---
🌍 MACRO IMPACT ANALYSIS

🔗 IMPACT MAPPING
[Vẽ chuỗi tác động: Tin tức → Chỉ số → Hệ quả]
Ví dụ: [Tin X] → [Tác động Y] → [Ảnh hưởng Z đến thị trường VN]

📉 RỦI RO HỆ THỐNG
Risk-on / Risk-off: [Đánh giá hiện tại]
Global risk appetite: [Score 0-100 và lý do]

🇻🇳 TÁC ĐỘNG LÊN VN
VN-Index: [Tác động và mức độ]
USD/VND: [Áp lực và xu hướng]
Lãi suất nội địa: [Nhận định]
Dòng vốn ngoại: [Dự báo ngắn hạn]

⚡ SIGNAL NHANH
[3 bullet points: tín hiệu quan trọng nhất từ dữ liệu macro]
---`,
    };

    try {
      const resp = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: CLAUDE_MODEL,
          max_tokens: 1000,
          messages: [{ role: "user", content: prompts[type] }],
        }),
      });

      const data = await resp.json();
      if (data.content?.[0]?.text) {
        setReport(data.content[0].text);
      } else {
        setError("Không thể tạo báo cáo. Vui lòng thử lại.");
      }
    } catch (e) {
      setError("Lỗi kết nối API. Vui lòng thử lại.");
    }
    setLoading(false);
  };

  const btnClass = (type) =>
    `px-3 py-1.5 text-xs font-bold rounded transition-all duration-200 font-mono uppercase tracking-wider ${
      reportType === type && report
        ? "bg-cyan-500 text-slate-900"
        : "bg-slate-700/60 text-slate-300 hover:bg-slate-600/60 border border-slate-600/50"
    }`;

  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center gap-2 mb-3 flex-wrap">
        <button className={btnClass("morning")} onClick={() => generateReport("morning")}>
          🌅 Morning Brief
        </button>
        <button className={btnClass("eod")} onClick={() => generateReport("eod")}>
          🌆 End-of-Day
        </button>
        <button className={btnClass("macro")} onClick={() => generateReport("macro")}>
          🌍 Macro Analysis
        </button>
        {loading && (
          <div className="ml-auto flex items-center gap-2 text-cyan-400 text-xs font-mono">
            <div className="flex gap-0.5">
              <div className="w-1 h-3 bg-cyan-400 animate-pulse rounded" style={{ animationDelay: "0ms" }} />
              <div className="w-1 h-3 bg-cyan-400 animate-pulse rounded" style={{ animationDelay: "150ms" }} />
              <div className="w-1 h-3 bg-cyan-400 animate-pulse rounded" style={{ animationDelay: "300ms" }} />
            </div>
            AI đang phân tích...
          </div>
        )}
      </div>

      <div className="flex-1 bg-slate-900/80 rounded border border-slate-700/50 p-4 overflow-y-auto min-h-[300px] font-mono text-sm">
        {!report && !loading && !error && (
          <div className="flex flex-col items-center justify-center h-full text-slate-600 gap-3">
            <div className="text-4xl">⚡</div>
            <div className="text-center">
              <div className="text-slate-400 font-semibold mb-1">AI Market Intelligence</div>
              <div className="text-xs text-slate-600">Chọn loại báo cáo để AI phân tích thị trường</div>
            </div>
          </div>
        )}
        {error && <div className="text-red-400 text-sm">{error}</div>}
        {report && (
          <pre className="whitespace-pre-wrap text-slate-200 leading-relaxed text-xs">
            {report}
          </pre>
        )}
      </div>
    </div>
  );
}

// ── Main Dashboard ────────────────────────────────────────────────────────────
export default function MarketDashboard() {
  const [marketData, setMarketData] = useState(generateMarketData());
  const [news, setNews] = useState(INITIAL_NEWS);
  const [time, setTime] = useState(new Date());
  const [activeTab, setActiveTab] = useState("all");
  const [tickerPaused, setTickerPaused] = useState(false);

  useEffect(() => {
    const interval = setInterval(() => {
      setMarketData(generateMarketData());
      setTime(new Date());
    }, 4000);
    return () => clearInterval(interval);
  }, []);

  const filteredNews =
    activeTab === "all" ? news : news.filter((n) => n.category.toLowerCase() === activeTab || n.region === activeTab);

  const sentimentCounts = news.reduce(
    (acc, n) => {
      acc[n.sentiment]++;
      return acc;
    },
    { positive: 0, neutral: 0, negative: 0 }
  );
  const totalNews = news.length;
  const sentimentScore = Math.round(
    ((sentimentCounts.positive * 2 + sentimentCounts.neutral) / (totalNews * 2)) * 100
  );
  const marketBias =
    sentimentScore >= 65 ? { label: "BULLISH", color: "text-emerald-400", dot: "bg-emerald-400" }
    : sentimentScore >= 45 ? { label: "NEUTRAL", color: "text-amber-400", dot: "bg-amber-400" }
    : { label: "BEARISH", color: "text-red-400", dot: "bg-red-400" };

  const tickerItems = [
    { l: "VN-INDEX", d: marketData.vnindex },
    { l: "VN30", d: marketData.vn30 },
    { l: "S&P 500", d: marketData.sp500 },
    { l: "NASDAQ", d: marketData.nasdaq },
    { l: "OIL", d: marketData.oil, p: "$" },
    { l: "GOLD", d: marketData.gold, p: "$" },
    { l: "BTC", d: marketData.btc, p: "$" },
    { l: "USD/VND", d: marketData.usdvnd },
    { l: "DXY", d: marketData.dxy },
  ];

  return (
    <div
      style={{
        fontFamily: "'IBM Plex Mono', 'JetBrains Mono', 'Fira Code', monospace",
        background: "#0a0e1a",
        minHeight: "100vh",
        color: "#e2e8f0",
      }}
    >
      {/* Google Fonts */}
      <link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@300;400;500;600;700&family=Space+Grotesk:wght@400;500;600;700&display=swap" rel="stylesheet" />

      {/* ── TOP NAV ── */}
      <nav style={{ background: "#060810", borderBottom: "1px solid #1e2d40" }} className="sticky top-0 z-50">
        <div className="flex items-center justify-between px-4 py-2">
          <div className="flex items-center gap-3">
            <div className="flex items-center gap-2">
              <div className="w-2 h-2 bg-cyan-400 rounded-full animate-pulse" />
              <span className="text-cyan-400 font-bold text-sm tracking-widest uppercase">AXIOM</span>
              <span className="text-slate-600 text-xs">|</span>
              <span className="text-slate-400 text-xs">Market Intelligence Terminal</span>
            </div>
          </div>

          <div className="flex items-center gap-4 text-xs text-slate-400">
            <div className="flex items-center gap-1.5">
              <div className="w-1.5 h-1.5 bg-emerald-400 rounded-full animate-pulse" />
              <span>LIVE</span>
            </div>
            <span className="text-slate-300 font-semibold">
              {time.toLocaleTimeString("vi-VN", { hour: "2-digit", minute: "2-digit", second: "2-digit" })}
            </span>
            <span className="text-slate-500">
              {time.toLocaleDateString("vi-VN", { weekday: "short", day: "2-digit", month: "2-digit", year: "numeric" })}
            </span>
          </div>
        </div>

        {/* Scrolling Ticker */}
        <div
          style={{ background: "#0d1627", borderTop: "1px solid #1e2d40" }}
          className="overflow-hidden"
          onMouseEnter={() => setTickerPaused(true)}
          onMouseLeave={() => setTickerPaused(false)}
        >
          <div className="flex" style={{ animation: tickerPaused ? "none" : "none" }}>
            <div className="flex overflow-x-auto scrollbar-hide">
              {tickerItems.map((t) => (
                <Ticker key={t.l} label={t.l} data={t.d} prefix={t.p || ""} />
              ))}
            </div>
          </div>
        </div>
      </nav>

      {/* ── MAIN GRID ── */}
      <div className="p-4 grid grid-cols-12 gap-4">

        {/* ── LEFT: News Feed ── */}
        <div className="col-span-12 lg:col-span-4 flex flex-col gap-4">

          {/* Sentiment Overview */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-4">
            <div className="flex items-center justify-between mb-3">
              <span className="text-[10px] font-bold text-slate-500 uppercase tracking-widest">Market Sentiment</span>
              <div className={`flex items-center gap-1.5 ${marketBias.color}`}>
                <div className={`w-2 h-2 rounded-full ${marketBias.dot} animate-pulse`} />
                <span className="text-sm font-bold">{marketBias.label}</span>
              </div>
            </div>
            <div className="flex gap-2 mb-2">
              <div className="flex-1 bg-slate-800 rounded overflow-hidden h-2">
                <div className="h-full bg-emerald-400 transition-all duration-1000" style={{ width: `${(sentimentCounts.positive / totalNews) * 100}%` }} />
              </div>
            </div>
            <div className="flex justify-between text-[10px] text-slate-500">
              <span className="text-emerald-400">● {sentimentCounts.positive} Positive</span>
              <span className="text-amber-400">● {sentimentCounts.neutral} Neutral</span>
              <span className="text-red-400">● {sentimentCounts.negative} Negative</span>
            </div>
            <div className="mt-2 text-center">
              <span className="text-2xl font-bold text-slate-200">{sentimentScore}</span>
              <span className="text-xs text-slate-500"> / 100</span>
            </div>
          </div>

          {/* News Feed */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg flex flex-col">
            <div className="flex items-center justify-between p-3 border-b border-slate-800">
              <span className="text-[10px] font-bold text-slate-400 uppercase tracking-widest">📡 Tin Tức Vĩ Mô</span>
              <div className="flex items-center gap-1 text-[10px] text-emerald-400">
                <div className="w-1 h-1 bg-emerald-400 rounded-full animate-pulse" />
                Real-time
              </div>
            </div>

            <div className="flex gap-1 p-2 border-b border-slate-800 overflow-x-auto">
              {["all", "Macro", "Market", "Geopolitics", "Corporate"].map((tab) => (
                <button
                  key={tab}
                  onClick={() => setActiveTab(tab)}
                  className={`px-2 py-1 text-[10px] rounded font-bold uppercase tracking-wider whitespace-nowrap transition-all ${
                    activeTab === tab
                      ? "bg-cyan-500/20 text-cyan-400 border border-cyan-500/40"
                      : "text-slate-500 hover:text-slate-300"
                  }`}
                >
                  {tab === "all" ? "Tất cả" : tab}
                </button>
              ))}
            </div>

            <div className="overflow-y-auto max-h-[380px]">
              {filteredNews.map((item, i) => (
                <div
                  key={item.id}
                  className="p-3 border-b border-slate-800/60 hover:bg-slate-800/30 transition-all cursor-pointer group"
                >
                  <div className="flex items-center gap-1.5 mb-1">
                    <RegionTag region={item.region} />
                    <ImpactBadge impact={item.impact} />
                    <span className="text-[10px] text-slate-600 ml-auto font-mono">{item.time}</span>
                  </div>
                  <div className="text-xs text-slate-300 leading-relaxed group-hover:text-slate-100 transition-colors">
                    {item.title}
                  </div>
                  <div className="mt-1">
                    <span className={`text-[10px] ${
                      item.sentiment === "positive" ? "text-emerald-500"
                      : item.sentiment === "negative" ? "text-red-500"
                      : "text-slate-500"
                    }`}>
                      {item.sentiment === "positive" ? "↑ Tích cực" : item.sentiment === "negative" ? "↓ Tiêu cực" : "→ Trung tính"}
                    </span>
                  </div>
                </div>
              ))}
            </div>
          </div>

          {/* Risk Events */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">⚠️ Risk Events</div>
            <div className="space-y-2">
              {RISK_EVENTS.map((r, i) => (
                <div key={i} className="flex items-center justify-between">
                  <div>
                    <div className="text-xs text-slate-300">{r.event}</div>
                    <div className="text-[10px] text-slate-600 font-mono">{r.date}</div>
                  </div>
                  <ImpactBadge impact={r.impact} />
                </div>
              ))}
            </div>
          </div>
        </div>

        {/* ── CENTER: Markets ── */}
        <div className="col-span-12 lg:col-span-4 flex flex-col gap-4">

          {/* VN Markets */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">🇻🇳 Thị Trường Việt Nam</div>
            <div className="grid grid-cols-2 gap-2">
              <MarketCard label="VN-INDEX" data={marketData.vnindex} />
              <MarketCard label="VN30" data={marketData.vn30} />
              <MarketCard label="HNX" data={marketData.hnx} />
              <MarketCard label="UPCOM" data={{ value: 95.3 + Math.random() * 0.5, change: (Math.random() - 0.4) * 1.5 }} />
            </div>
          </div>

          {/* Global Markets */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">🌍 Thị Trường Toàn Cầu</div>
            <div className="grid grid-cols-2 gap-2">
              <MarketCard label="S&P 500" data={marketData.sp500} />
              <MarketCard label="NASDAQ" data={marketData.nasdaq} />
              <MarketCard label="DOW JONES" data={marketData.dowjones} />
              <MarketCard label="DAX" data={{ value: 18421.3 + Math.random() * 50, change: (Math.random() - 0.45) * 1.8 }} />
            </div>
          </div>

          {/* Commodities */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">🛢️ Hàng Hóa</div>
            <div className="grid grid-cols-2 gap-2">
              <MarketCard label="OIL (WTI)" data={marketData.oil} prefix="$" />
              <MarketCard label="GOLD" data={marketData.gold} prefix="$" />
              <MarketCard label="SILVER" data={marketData.silver} prefix="$" />
              <MarketCard label="COPPER" data={marketData.copper} prefix="$" />
            </div>
          </div>

          {/* FX & Crypto */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">💱 FX & Crypto</div>
            <div className="grid grid-cols-2 gap-2">
              <MarketCard label="USD/VND" data={marketData.usdvnd} />
              <MarketCard label="DXY" data={marketData.dxy} />
              <MarketCard label="BTC/USD" data={marketData.btc} prefix="$" />
              <MarketCard label="ETH/USD" data={marketData.eth} prefix="$" />
            </div>
          </div>
        </div>

        {/* ── RIGHT: AI Reports ── */}
        <div className="col-span-12 lg:col-span-4 flex flex-col gap-4">
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3 flex-1 flex flex-col">
            <div className="flex items-center justify-between mb-3">
              <span className="text-[10px] font-bold text-slate-400 uppercase tracking-widest">🤖 AI Market Intelligence</span>
              <div className="flex items-center gap-1 text-[10px] text-cyan-400">
                <div className="w-1 h-1 bg-cyan-400 rounded-full animate-pulse" />
                Claude AI
              </div>
            </div>

            <div className="text-[10px] text-slate-600 mb-3 leading-relaxed">
              AI phân tích dựa trên dữ liệu thị trường real-time và tin tức vĩ mô. Không phải khuyến nghị đầu tư.
            </div>

            <AIReportPanel marketData={marketData} news={news} />
          </div>

          {/* Quick Stats */}
          <div style={{ background: "#0d1627", border: "1px solid #1e2d40" }} className="rounded-lg p-3">
            <div className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-3">📊 Market Pulse</div>
            <div className="grid grid-cols-3 gap-2">
              {[
                { label: "Risk-On Score", value: `${sentimentScore}`, unit: "/100", color: sentimentScore >= 60 ? "text-emerald-400" : "text-amber-400" },
                { label: "VN Fear/Greed", value: "62", unit: "Greed", color: "text-amber-400" },
                { label: "Global VIX", value: "14.2", unit: "Low Vol", color: "text-emerald-400" },
              ].map((s) => (
                <div key={s.label} className="bg-slate-800/40 rounded p-2 text-center">
                  <div className="text-[9px] text-slate-600 uppercase mb-1 leading-tight">{s.label}</div>
                  <div className={`text-lg font-bold ${s.color}`}>{s.value}</div>
                  <div className="text-[9px] text-slate-500">{s.unit}</div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>

      {/* Footer */}
      <div style={{ borderTop: "1px solid #1e2d40" }} className="px-4 py-2 text-center text-[10px] text-slate-700 font-mono">
        AXIOM Terminal • Dữ liệu mang tính minh hoạ • Không phải khuyến nghị đầu tư • Powered by Claude AI
      </div>

      <style>{`
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
        * { scrollbar-width: thin; scrollbar-color: #1e2d40 transparent; }
        *::-webkit-scrollbar { width: 4px; }
        *::-webkit-scrollbar-track { background: transparent; }
        *::-webkit-scrollbar-thumb { background: #1e2d40; border-radius: 2px; }
      `}</style>
    </div>
  );
}
