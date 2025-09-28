<!--
Greenhouse Environmental Control – Siemens TIA (LAD + ST)
Drop this directly into your README.md (GitHub supports inline HTML).
-->

<h1 align="center">🌱 Greenhouse Environmental Control (Siemens TIA) — <em>LAD () + ST() </em></h1>

<p align="center">
  <img alt="PLC" src="https://img.shields.io/badge/PLC-Siemens%20TIA-0b70f2?logo=siemens&logoColor=white">
  <img alt="Languages" src="https://img.shields.io/badge/Languages-LAD%20%2B%20ST-7952b3">
  <img alt="Status" src="https://img.shields.io/badge/Status-Working-brightgreen">
</p>

<p align="center">
  <b>Goal:</b> Maintain optimal <em>temperature</em>, <em>humidity</em>, and <em>light</em> in a greenhouse using
  <b>Structured Text</b> for logic and <b>Ladder</b> only for block calls & I/O mapping. ⚙️
</p>

<hr/>

<h2>🧭 Architecture Overview</h2>

<ul>
  <li><b>FC_SensorScaling (ST)</b> → Converts raw analog inputs (0–27648) into engineering units (°C, %RH, Lux).</li>
  <li><b>FB_ClimateControl (ST)</b> → Decides actuators (Fan, Heater, Humidifier, GrowLight) using setpoints, hysteresis, and optional min on/off times.</li>
  <li><b>DB_Greenhouse (Global DB)</b> → Central parameters & runtime values (setpoints, scaled values, outputs, enables).</li>
  <li><b>OB1 (LAD)</b> → Minimal: call FC ➜ call FB ➜ map outputs to digital outputs.</li>
</ul>

<h3>📈 Dataflow (SVG Block Diagram)</h3>

<!-- Simple inline SVG, no external files needed -->
<div align="center">
<svg width="780" height="270" xmlns="http://www.w3.org/2000/svg">
  <rect x="10" y="20" width="190" height="230" rx="12" fill="#f1f5f9" stroke="#cbd5e1"/>
  <text x="105" y="45" text-anchor="middle" font-family="sans-serif" font-size="14">🌡️ Raw Sensors</text>
  <text x="105" y="70" text-anchor="middle" font-family="monospace" font-size="12">IW64 → TempRaw</text>
  <text x="105" y="90" text-anchor="middle" font-family="monospace" font-size="12">IW66 → HumRaw</text>
  <text x="105" y="110" text-anchor="middle" font-family="monospace" font-size="12">IW68 → LightRaw</text>

  <rect x="230" y="20" width="180" height="230" rx="12" fill="#ecfeff" stroke="#67e8f9"/>
  <text x="320" y="45" text-anchor="middle" font-family="sans-serif" font-size="14">🧮 FC_SensorScaling</text>
  <text x="320" y="70" text-anchor="middle" font-family="monospace" font-size="12">Raw → °C / %RH / Lux</text>
  <line x1="200" y1="120" x2="230" y2="120" stroke="#94a3b8" marker-end="url(#arrow)"/>

  <rect x="440" y="20" width="180" height="230" rx="12" fill="#fef9c3" stroke="#fde047"/>
  <text x="530" y="45" text-anchor="middle" font-family="sans-serif" font-size="14">🧠 FB_ClimateControl</text>
  <text x="530" y="70" text-anchor="middle" font-family="monospace" font-size="12">Setpoints + Hysteresis</text>
  <line x1="410" y1="120" x2="440" y2="120" stroke="#94a3b8" marker-end="url(#arrow)"/>

  <rect x="650" y="20" width="120" height="230" rx="12" fill="#e7ffe7" stroke="#86efac"/>
  <text x="710" y="45" text-anchor="middle" font-family="sans-serif" font-size="14">⚡ Actuators</text>
  <text x="710" y="70" text-anchor="middle" font-family="monospace" font-size="12">Q0.0 Fan</text>
  <text x="710" y="90" text-anchor="middle" font-family="monospace" font-size="12">Q0.1 Heater</text>
  <text x="710" y="110" text-anchor="middle" font-family="monospace" font-size="12">Q0.2 Humid.</text>
  <text x="710" y="130" text-anchor="middle" font-family="monospace" font-size="12">Q0.3 Light</text>
  <line x1="620" y1="120" x2="650" y2="120" stroke="#94a3b8" marker-end="url(#arrow)"/>

  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L6,3 z" fill="#94a3b8"/>
    </marker>
  </defs>
</svg>
</div>

<hr/>

<h2>🧩 Components & Working Principles</h2>

<h3>1) <code>FC_SensorScaling</code> — Function (ST)</h3>
<p>
  Converts unipolar analog inputs (0…27648) into engineering units using linear scaling.
  This keeps <em>all math</em> isolated and re-usable.
</p>

<ul>
  <li><b>Inputs:</b> <code>TempRaw</code>, <code>HumRaw</code>, <code>LightRaw</code> (+ optional max EU ranges)</li>
  <li><b>Outputs:</b> <code>TempValue (°C)</code>, <code>HumValue (%RH)</code>, <code>LightValue (Lux)</code></li>
</ul>

<details>
<summary>🔎 Scaling Logic (pseudocode)</summary>
<pre><code>// 0..27648 → 0..MaxEU
TempValue  := (REAL(TempRaw)  / 27648.0) * TempMaxEU;
HumValue   := (REAL(HumRaw)   / 27648.0) * HumMaxEU;
LightValue := (REAL(LightRaw) / 27648.0) * LightMaxEU;
</code></pre>
</details>

<h3>2) <code>FB_ClimateControl</code> — Function Block (ST)</h3>

<p>
  The FB is the “brain”: it decides actuator states based on setpoints, with
  <b>hysteresis</b> to prevent chatter and optional <b>min on/off</b> times using TON timers.
</p>

<ul>
  <li><b>VAR_INPUT:</b> <code>TempValue, HumValue, LightValue</code>, <code>TempSet, HumSet, LightSet</code>, feature enables, min on/off times</li>
  <li><b>VAR_OUTPUT:</b> <code>Fan, Heater, Humidifier, GrowLight</code></li>
  <li><b>VAR (Static):</b> hysteresis values, request latches, <code>TON</code> instances, last states</li>
  <li><b>VAR_TEMP:</b> scratch bools like <code>WantFan</code>, <code>WantHeater</code>…</li>
</ul>

<p><b>Control strategy:</b></p>
<ul>
  <li><b>Temperature:</b> If above <code>TempSet + TempHyst</code> → <code>Fan=ON</code>, below <code>TempSet - TempHyst</code> → <code>Heater=ON</code>, else both OFF.</li>
  <li><b>Humidity:</b> If below <code>HumSet - HumHyst</code> → <code>Humidifier=ON</code>, if ≥ <code>HumSet</code> → OFF, else hold previous request.</li>
  <li><b>Light:</b> If <code>LightValue &lt; LightSet</code> → <code>GrowLight=ON</code> else OFF.</li>
  <li><b>Min On/Off:</b> Optional TON gates ensure equipment runs/rests for a minimum time to protect hardware and avoid short cycling.</li>
</ul>

<details>
<summary>🧠 Hysteresis & Min-Time Concept (visual)</summary>
<div align="center">
<svg width="640" height="180" xmlns="http://www.w3.org/2000/svg">
  <rect x="15" y="20" width="610" height="140" rx="10" fill="#f8fafc" stroke="#e2e8f0"/>
  <text x="320" y="40" text-anchor="middle" font-family="sans-serif" font-size="13">Temp Control with Deadband</text>
  <line x1="60" y1="120" x2="580" y2="120" stroke="#94a3b8"/>
  <text x="60" y="135" font-family="sans-serif" font-size="11">Low</text>
  <text x="560" y="135" font-family="sans-serif" font-size="11">High</text>

  <!-- Setpoint and bands -->
  <line x1="200" y1="60" x2="200" y2="140" stroke="#60a5fa"/>
  <text x="200" y="55" text-anchor="middle" font-size="11" fill="#2563eb">TempSet</text>

  <line x1="240" y1="60" x2="240" y2="140" stroke="#86efac"/>
  <text x="240" y="55" text-anchor="middle" font-size="11" fill="#16a34a">TempSet + Hyst</text>

  <line x1="160" y1="60" x2="160" y2="140" stroke="#fca5a5"/>
  <text x="160" y="55" text-anchor="middle" font-size="11" fill="#dc2626">TempSet - Hyst</text>

  <!-- Labels -->
  <text x="120" y="90" font-size="11" fill="#dc2626">Heater ON</text>
  <text x="280" y="90" font-size="11" fill="#16a34a">Fan ON</text>
  <text x="200" y="130" font-size="11">Deadband → both OFF</text>
</svg>
</div>
</details>

<h3>3) <code>DB_Greenhouse</code> — Global Data Block</h3>

<p>Single source of truth for HMI and diagnostics: setpoints, enables, scaled values, output states, min on/off times.</p>

<table>
  <thead><tr><th>Category</th><th>Tags</th></tr></thead>
  <tbody>
    <tr>
      <td><b>Setpoints</b></td>
      <td><code>TempSet</code> (°C), <code>HumSet</code> (%RH), <code>LightSet</code> (Lux)</td>
    </tr>
    <tr>
      <td><b>Scaled Values</b></td>
      <td><code>TempValue</code>, <code>HumValue</code>, <code>LightValue</code></td>
    </tr>
    <tr>
      <td><b>Enables</b></td>
      <td><code>EnTempCtrl</code>, <code>EnHumCtrl</code>, <code>EnLightCtrl</code></td>
    </tr>
    <tr>
      <td><b>Min On/Off</b></td>
      <td><code>MinOnSec_*</code>, <code>MinOffSec_*</code> (TIME)</td>
    </tr>
    <tr>
      <td><b>Outputs</b></td>
      <td><code>Fan</code>, <code>Heater</code>, <code>Humidifier</code>, <code>GrowLight</code></td>
    </tr>
  </tbody>
</table>

<h3>4) <code>OB1</code> — Main (LAD, minimal)</h3>

<p>Only three networks:</p>
<ol>
  <li><b>CALL FC_SensorScaling</b>: IW64/IW66/IW68 → DB_Greenhouse.TempValue/HumValue/LightValue</li>
  <li><b>CALL FB_ClimateControl</b> (with <em>DB_ClimateControl</em> instance): inputs from DB_Greenhouse; outputs back to DB_Greenhouse</li>
  <li><b>I/O Mapping</b>: DB_Greenhouse.* → Q0.0..Q0.3</li>
</ol>

<details>
<summary>🪜 Ladder Sketch (conceptual)</summary>
<pre><code>// Net1: FC call
[ FC_SensorScaling ]
  TempRaw := IW64
  HumRaw  := IW66
  LightRaw:= IW68
  --&gt; DB_Greenhouse.TempValue, HumValue, LightValue

// Net2: FB call (Instance DB_ClimateControl)
[ FB_ClimateControl (DB_ClimateControl) ]
  TempValue := DB_Greenhouse.TempValue
  HumValue  := DB_Greenhouse.HumValue
  LightValue:= DB_Greenhouse.LightValue
  TempSet   := DB_Greenhouse.TempSet
  HumSet    := DB_Greenhouse.HumSet
  LightSet  := DB_Greenhouse.LightSet
  EnTempCtrl:= DB_Greenhouse.EnTempCtrl
  EnHumCtrl := DB_Greenhouse.EnHumCtrl
  EnLightCtrl:= DB_Greenhouse.EnLightCtrl
  ...
  --&gt; DB_Greenhouse.Fan/Heater/Humidifier/GrowLight

// Net3: Coils / assignments
Q0.0 := DB_Greenhouse.Fan
Q0.1 := DB_Greenhouse.Heater
Q0.2 := DB_Greenhouse.Humidifier
Q0.3 := DB_Greenhouse.GrowLight
</code></pre>
</details>

<hr/>

<h2>🛠️ I/O Map</h2>

<table>
  <thead><tr><th>Signal</th><th>Address</th><th>Description</th></tr></thead>
  <tbody>
    <tr><td>Temp Raw AI</td><td><code>IW64</code></td><td>0–27648 → 0–50 °C</td></tr>
    <tr><td>Hum Raw AI</td><td><code>IW66</code></td><td>0–27648 → 0–100 %RH</td></tr>
    <tr><td>Light Raw AI</td><td><code>IW68</code></td><td>0–27648 → 0–1000 Lux</td></tr>
    <tr><td>Fan DO</td><td><code>Q0.0</code></td><td>Exhaust / circulation fan</td></tr>
    <tr><td>Heater DO</td><td><code>Q0.1</code></td><td>Space heater contactor</td></tr>
    <tr><td>Humidifier DO</td><td><code>Q0.2</code></td><td>Mist / vapor unit</td></tr>
    <tr><td>Grow Light DO</td><td><code>Q0.3</code></td><td>Supplemental lighting</td></tr>
  </tbody>
</table>

<hr/>

<h2>🚀 Commissioning Steps</h2>

<ol>
  <li>Create <code>DB_Greenhouse</code> with all tags (setpoints, enables, min times, scaled values, outputs).</li>
  <li>Add <code>FC_SensorScaling</code> (ST) and paste scaling code.</li>
  <li>Add <code>FB_ClimateControl</code> (ST) and paste logic (hysteresis + min-times).</li>
  <li>In <code>OB1</code> (LAD):
    <ul>
      <li>Net1: Call FC → write scaled values to DB.</li>
      <li>Net2: Call FB with <em>DB_ClimateControl</em> instance → write outputs to DB.</li>
      <li>Net3: Map DB outputs to Q0.x.</li>
    </ul>
  </li>
  <li>Download, go online, tune <code>DB_Greenhouse.TempSet/HumSet/LightSet</code>, and watch the magic ✨.</li>
</ol>

<hr/>

<h2>🧪 Tuning Tips</h2>

<ul>
  <li>Start with <code>TempHyst = 2.0 °C</code> and <code>HumHyst = 5 %RH</code>, adjust to reduce cycling.</li>
  <li>Use <em>Min On/Off</em> (e.g., Heater: 10s) for equipment protection.</li>
  <li>Expose <code>DB_Greenhouse</code> to your HMI for runtime tweaking and status LEDs.</li>
</ul>

<hr/>

<h2>📦 Extras (Optional)</h2>

<ul>
  <li>Swap temperature hysteresis with a PID block (FB41 / PID_Compact) for tighter control.</li>
  <li>Add safety interlocks in LAD before coils (overtemp, door switches, E-stop).</li>
  <li>Log trends (Temp, Hum, Light, outputs) in a separate DB for analysis.</li>
</ul>

<p align="center"><i>Built with 💚 for learning and rapid commissioning.</i></p>
