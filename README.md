
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#101827">
<title>Boom & Crash Signal AI</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#0b1220;color:#eef3ff;
font:15px system-ui,sans-serif;padding:18px}
main{max-width:650px;margin:auto}
h1{font-size:25px;margin-bottom:5px}
.sub,.muted{color:#a7b5cc}
.card{background:#141f31;border:1px solid #293850;
border-radius:15px;padding:16px;margin:14px 0}
label{display:block;margin:12px 0 6px}
select,button{width:100%;padding:13px;border-radius:9px;
font-size:15px}
select{background:#0b1220;color:white;border:1px solid #40516b}
button{background:#36d399;color:#081b16;font-weight:700;
border:0;margin-top:12px}
.stat{display:flex;justify-content:space-between;gap:10px;
padding:12px 0;border-bottom:1px solid #293850}
.value{font-weight:700;text-align:right;overflow-wrap:anywhere}
.signal{font-size:26px;font-weight:800;margin:10px 0}
.green{color:#36d399}.red{color:#ff7777}.yellow{color:#ffd166}
.small{font-size:12px;line-height:1.6}
canvas{width:100%;height:150px}
</style>
<style>
/* Styles for the starter page. They only apply to its own classes. */
.hello-page {
  margin: 0;
  min-height: 100vh;
  display: grid;
  place-items: center;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  background: #f5f2ed;
  color: #1b1817;
}

.hello-card {
  max-width: 28rem;
  margin: 1.5rem;
  padding: 2rem;
  border-radius: 16px;
  background: #fff;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.08);
}

.hello-card h1 {
  margin-top: 0;
  color: #c2560c;
}

.hello-card button {
  padding: 0.6rem 1.2rem;
  border: 0;
  border-radius: 8px;
  background: #fe9615;
  color: #1b1817;
  font: inherit;
  font-weight: 600;
  cursor: pointer;
}

</style>
</head>
<body>
<main>
<h1>⚡ Boom & Crash Signal AI</h1>
<div class="sub">Live market data · Technical analysis · Demo only</div>

<div class="card">
<div class="stat"><span>Connection</span><b id="status">Connecting…</b></div>
<label for="market">Select a market</label>
<select id="market"><option>Loading markets…</option></select>
<button id="start">Connect / Refresh analysis</button>
<div class="small muted">Uses public Deriv market data. No trading account or password required.</div>
</div>

<div class="card">
<div class="muted">Selected market · Latest price</div>
<div id="price" class="signal">Waiting for data</div>
<div id="updated" class="small muted">No data received yet</div>
<canvas id="chart" width="600" height="180"></canvas>
</div>

<div class="card">
<div class="muted">Technical signal (not a prediction)</div>
<div id="signal" class="signal yellow">WAIT</div>
<div id="reason">Collecting price history…</div>
<div class="stat"><span>Trend · EMA 9 / 21</span><b id="trend">—</b></div>
<div class="stat"><span>RSI (14)</span><b id="rsi">—</b></div>
<div class="stat"><span>ATR (14)</span><b id="atr">—</b></div>
<div class="stat"><span>Illustrative stop-loss</span><b id="sl">—</b></div>
<div class="stat"><span>Illustrative target (2R)</span><b id="tp">—</b></div>
<div class="stat"><span>Setup agreement</span><b id="agree">—</b></div>
</div>

<div class="card">
<h3>Paper-trading journal</h3>
<div class="muted small">Record hypothetical signals to review the strategy. No real trades are placed.</div>
<button id="record">Record current signal</button>
<div id="journal"></div>
<button id="clear" style="background:#34445d;color:white">Clear journal</button>
</div>

<div class="card small">
<b>Risk warning</b>
<p>Boom and Crash indices can make sudden price spikes or drops. EMA, RSI and ATR cannot reliably predict when a spike will occur. BUY and SELL indicate technical bias, not guaranteed profitable entries. WAIT means the rules do not agree.</p>
<p>Stop-loss and target are estimates for a hypothetical directional trade. Actual execution, contract type and risk can differ. Test with historical data and a demo account before risking money.</p>
</div>
</main>
<script>
const $=id=>document.getElementById(id);
let ws, symbols=[], candles=[], selected="", lastTick=null;
let tickSub=null, requestId=0, pending={};
let journal=[];

function send(obj){
  if(ws && ws.readyState===WebSocket.OPEN){
    obj.req_id=++requestId;
    pending[obj.req_id]=obj;
    ws.send(JSON.stringify(obj));
  }
}
function connect(){
  if(ws) try{ws.close()}catch(e){}
  $("status").textContent="Connecting…";
  $("start").disabled=true;
  ws=new WebSocket("wss://ws.derivws.com/websockets/v3?app_id=1089");
  ws.onopen=()=>{
    $("status").textContent="Connected";
    $("start").disabled=false;
    send({active_symbols:"brief",product_type:"basic"});
  };
  ws.onmessage=e=>{
    const d=JSON.parse(e.data);
    if(d.error){
      $("status").textContent="API error: "+d.error.message;
      $("start").disabled=false;
      return;
    }
    if(d.msg_type==="active_symbols"){
      symbols=(d.active_symbols||[]).filter(s=>
        /boom|crash/i.test(s.display_name||"")
      ).sort((a,b)=>(a.display_name||"").localeCompare(b.display_name||""));
      const sel=$("market");
      sel.innerHTML="";
      if(!symbols.length){
        sel.innerHTML='<option value="">No Boom/Crash symbols found</option>';
        $("status").textContent="No matching symbols";
        return;
      }
      symbols.forEach(s=>{
        const o=document.createElement("option");
        o.value=s.symbol;
        o.textContent=s.display_name+" ("+s.symbol+")";
        sel.appendChild(o);
      });
      const first=symbols.find(s=>/boom/i.test(s.display_name))||symbols[0];
      selected=sel.value||first.symbol;
      sel.value=selected;
      loadMarket();
    }
    if(d.msg_type==="candles"||d.msg_type==="history"){
      const rows=d.candles||[];
      if(rows.length){
        candles=rows.map(c=>({
          epoch:+c.epoch,open:+c.open,high:+c.high,
          low:+c.low,close:+c.close
        })).filter(c=>[c.open,c.high,c.low,c.close].every(Number.isFinite));
        analyze();
        drawChart();
      }
    }
    if(d.msg_type==="tick"){
      const t=d.tick;
      if(!t)return;
      lastTick=+t.quote;
      const epoch=+t.epoch;
      if(!Number.isFinite(lastTick))return;
      const minute=Math.floor(epoch/60)*60;
      const last=candles[candles.length-1];
      if(!last || minute>last.epoch){
        candles.push({epoch:minute,open:lastTick,high:lastTick,
          low:lastTick,close:lastTick});
        if(candles.length>150)candles.shift();
      }else{
        last.high=Math.max(last.high,lastTick);
        last.low=Math.min(last.low,lastTick);
        last.close=lastTick;
      }
      $("price").textContent=lastTick.toFixed(priceDigits(lastTick));
      $("updated").textContent="Updated "+new Date(epoch*1000).toLocaleTimeString();
      analyze();
      drawChart();
    }
  };
  ws.onerror=()=>{$("status").textContent="Connection error — retry"};
  ws.onclose=()=>{
    $("start").disabled=false;
    if($("status").textContent==="Connected")
      $("status").textContent="Disconnected — reconnect";
  };
}
function priceDigits(n){
  if(!Number.isFinite(n))return 2;
  return Math.min(6,Math.max(2,(String(n).split(".")[1]||"").length));
}
function loadMarket(){
  selected=$("market").value;
  if(!selected)return;
  candles=[];lastTick=null;
  $("signal").textContent="WAIT";
  $("reason").textContent="Loading historical candles…";
  $("price").textContent="Loading…";
  $("trend").textContent=$("rsi").textContent=
  $("atr").textContent=$("sl").textContent=$("tp").textContent="—";
  $("agree").textContent="—";
  send({ticks_history:selected,count:100,end:"latest",
        style:"candles",granularity:60});
  send({ticks:selected,subscribe:1});
}
function ema(values,period){
  if(values.length<period)return null;
  let v=values.slice(0,period).reduce((a,b)=>a+b,0)/period;
  const k=2/(period+1);
  for(let i=period;i<values.length;i++)v=values[i]*k+v*(1-k);
  return v;
}
function calcRSI(values,p=14){
  if(values.length<=p)return null;
  let gain=0,loss=0;
  for(let i=values.length-p;i<values.length;i++){
    const d=values[i]-values[i-1];
    if(d>0)gain+=d;else loss-=d;
  }
  if(loss===0)return gain===0?50:100;
  return 100-100/(1+gain/loss);
}
function calcATR(cs,p=14){
  if(cs.length<=p)return null;
  const trs=[];
  for(let i=1;i<cs.length;i++){
    const c=cs[i],prev=cs[i-1];
    trs.push(Math.max(c.high-c.low,
      Math.abs(c.high-prev.close),Math.abs(c.low-prev.close)));
  }
  return trs.slice(-p).reduce((a,b)=>a+b,0)/p;
}
function analyze(){
  if(candles.length<25)return;
  const closes=candles.map(c=>c.close);
  const price=closes[closes.length-1];
  const e9=ema(closes,9),e21=ema(closes,21);
  const r=calcRSI(closes),a=calcATR(candles);
  if([price,e9,e21,r,a].some(v=>v===null||!Number.isFinite(v)))return;
  const up=e9>e21,down=e9<e21;
  // Conservative illustrative rules: trend + momentum agreement.
  const buy=up&&r>=50&&r<68;
  const sell=down&&r<=50&&r>32;
  const sig=buy?"BUY BIAS":sell?"SELL BIAS":"WAIT";
  $("signal").textContent=sig;
  $("signal").className="signal "+(buy?"green":sell?"red":"yellow");
  $("reason").textContent=buy?
    "EMA trend is upward and RSI confirms positive momentum.":
    sell?"EMA trend is downward and RSI confirms negative momentum.":
    "Trend and momentum do not agree, or momentum is extended. No setup.";
  $("trend").textContent=up?"UPTREND":down?"DOWNTREND":"FLAT";
  $("rsi").textContent=r.toFixed(1);
  $("atr").textContent=a.toFixed(priceDigits(price));
  const sl=buy?price-1.5*a:sell?price+1.5*a:null;
  const tp=buy?price+3*a:sell?price-3*a:null;
  $("sl").textContent=sl===null?"N/A":sl.toFixed(priceDigits(price));
  $("tp").textContent=tp===null?"N/A":tp.toFixed(priceDigits(price));
  $("agree").textContent=(buy||sell)?"2/2 indicators":"Mixed / no setup";
  $("price").textContent=price.toFixed(priceDigits(price));
  $("updated").textContent="Analysis uses "+candles.length+" one-minute candles.";
}
function drawChart(){
  const c=$("chart"),ctx=c.getContext("2d");
  ctx.clearRect(0,0,c.width,c.height);
  if(candles.length<2)return;
  const data=candles.slice(-60),lo=Math.min(...data.map(x=>x.low)),
    hi=Math.max(...data.map(x=>x.high)),range=hi-lo||1;
  ctx.strokeStyle="#36d399";ctx.lineWidth=2;ctx.beginPath();
  data.forEach((d,i)=>{
    const x=i*(c.width-8)/(data.length-1)+4;
    const y=c.height-8-(d.close-lo)/range*(c.height-16);
    if(i===0)ctx.moveTo(x,y);else ctx.lineTo(x,y);
  });
  ctx.stroke();
}
$("start").onclick=connect;
$("market").onchange=loadMarket;
$("record").onclick=()=>{
  if(!candles.length){alert("Wait for market data first.");return}
  journal.unshift({market:$("market").selectedOptions[0].textContent,
    signal:$("signal").textContent,price:candles[candles.length-1].close,
    time:new Date().toLocaleString()});
  renderJournal();
};
function renderJournal(){
  $("journal").innerHTML="";
  journal.slice(0,30).forEach(j=>{
    const d=document.createElement("div");d.className="stat";
    const l=document.createElement("span");
    l.textContent=j.market+" · "+j.signal+" · "+j.time;
    const r=document.createElement("b");r.textContent=String(j.price);
    d.append(l,r);$("journal").appendChild(d);
  });
}
$("clear").onclick=()=>{journal=[];renderJournal()};
connect();
</script>
<script>
// Script for the starter page. It does nothing if the button isn't there.
const button = document.getElementById('greet');
const message = document.getElementById('message');
let clicks = 0;

if (button && message) {
  button.addEventListener('click', () => {
    clicks += 1;
    message.textContent = `Hello! You clicked ${clicks} time${clicks === 1 ? '' : 's'}.`;
    console.log('Button clicked', clicks);
  });
}

</script>
</body>
</html>

