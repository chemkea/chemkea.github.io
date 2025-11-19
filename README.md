
html = """<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Interactive B68 plot — final (Total full-width behind)</title>
  <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    .controls { display:flex; gap:20px; align-items:center; margin-bottom:12px; flex-wrap:wrap; }
    .control-block { display:flex; flex-direction:column; align-items:center; min-width:260px; }
    .small { font-size:0.9em; color:#333; }
    #chi2box { font-weight:600; }
    button { padding:6px 10px; }
  </style>
</head>
<body>

<h3>Interactive H₂ lines — CR / UV contribution</h3>

<div class="controls">

  <div class="control-block">
    <label class="small">ζ (cosmic-ray ionization rate) — log10</label>
    <input id="zeta_slider" type="range" min="-18" max="-14" step="0.01" value="-16" style="width:260px;">
    <div class="small">ζ = <span id="zeta_val">1.70e-16</span> s⁻¹</div>
  </div>

  <div class="control-block">
    <label class="small">χ<sub>UV</sub> (UV scaling) — log10</label>
    <input id="chi_slider" type="range" min="-1" max="2" step="0.01" value="0" style="width:260px;">
    <div class="small">χ<sub>UV</sub> = <span id="chi_val">1.89</span></div>
  </div>

  <div style="display:flex; flex-direction:column; gap:6px; align-items:flex-start;">
    <button id="reset_btn">Reset parameters</button>
    <div class="small">Reduced χ²: <span id="chi2box">--</span></div>
  </div>

</div>

<div id="plot" style="width:100%; max-width:1100px; height:560px;"></div>

<script>
// categorical labels (strings)
const lines = ["2.122","2.407","2.413","2.424","2.627","2.803","3.004"];

// data
const obs = [3.99051950537949e-8,3.93307058041730e-8,7.04487696318364e-8,2.77024579226435e-8,1.25250295746157e-7,2.57088306318885e-8,3.91265628413995e-8];
const obs_err = [5.69727386022035e-9,8.43024067979104e-9,8.51332635996861e-9,8.53930518136776e-9,5.99428060019090e-9,5.39582642915556e-9,4.95728518392377e-9];

const BAE = [
 [0.00010937,0.42,0.58],
 [0.00023958,0.5,0.52],
 [1.77981459,0.36,0.51],
 [0.00010937,0.33,0.51],
 [1.56983646,1.0,0.47],
 [0.00023958,0.5,0.44],
 [1.77981459,0.34,0.41]
];

const f_UV = [0.0186,0.0205,0.0101,0.0131,0.0094,0.0174,0.0078];
const prefactor = 1/(4*Math.PI);
const const_factor = 0.6541e22;
const eV_to_erg = 1.60217663e-12;

function compute_CR(zeta){
  let out = [];
  for(let i=0;i<BAE.length;i++){
    const energy = BAE[i][2]*eV_to_erg;
    out.push(prefactor*const_factor*BAE[i][0]*BAE[i][1]*energy*zeta);
  }
  return out;
}
function compute_UV(chi){
  return f_UV.map(v=>9.6e-7*chi*v);
}
function reduced_chi2(obs_arr, model_arr, err_arr, p=2){
  let chi2=0;
  for(let i=0;i<obs_arr.length;i++){
    let d=(obs_arr[i]-model_arr[i])/err_arr[i];
    chi2+=d*d;
  }
  return chi2/(obs_arr.length - p);
}
function fmt(x){ return x.toExponential(2).replace('e+','e'); }

// initial parameters
let zeta = 1.7e-16;
let chi = 1.89;

let CR = compute_CR(zeta);
let UV = compute_UV(chi);
let TOTAL = CR.map((v,i)=>v+UV[i]);

// traces: total first (full-width behind)
const total_trace = {
  x: lines,
  y: TOTAL,
  type: 'bar',
  name: 'Total model',
  marker: {color: '#bbbbbb'},
  opacity: 0.35,
  offsetgroup: 'TOTAL',
  width: 0.8
};
const cr_trace = {
  x: lines,
  y: CR,
  type: 'bar',
  name: 'CR contribution',
  marker: {color: '#1f77b4'},
  width: 0.32,
  offsetgroup: 'CR',
  alignmentgroup: 'group1'
};
const uv_trace = {
  x: lines,
  y: UV,
  type: 'bar',
  name: 'UV contribution',
  marker: {color: '#ff7f0e'},
  width: 0.32,
  offsetgroup: 'UV',
  alignmentgroup: 'group1'
};
const obs_trace = {
  x: lines,
  y: obs,
  mode: 'markers',
  type: 'scatter',
  name: 'Observations',
  marker: {color: 'black', size: 10},
  error_y: {type: 'data', array: obs_err, visible: true, thickness:1.5, width:4}
};

const layout = {
  barmode: 'group',
  xaxis: {
    title: 'Wavelength (μm)',
    type: 'category',
    categoryorder: 'array',
    categoryarray: lines
  },
  yaxis: {title: "Intensity (erg cm⁻² s⁻¹ sr⁻¹)", showexponent: "all", exponentformat: "power"},
  legend: {orientation: 'h', y: 1.12},
  margin: {t:60}
};

Plotly.newPlot('plot', [total_trace, cr_trace, uv_trace, obs_trace], layout, {responsive:true});

// DOM
const zeta_slider = document.getElementById('zeta_slider');
const chi_slider = document.getElementById('chi_slider');
const zeta_val = document.getElementById('zeta_val');
const chi_val = document.getElementById('chi_val');
const chi2box = document.getElementById('chi2box');
const reset_btn = document.getElementById('reset_btn');

function update_chi2_display(){
  chi2box.textContent = reduced_chi2(obs, TOTAL, obs_err, 2).toFixed(3);
}
update_chi2_display();

// IMPORTANT: use Plotly.update and always resend x & layout categoryarray
zeta_slider.addEventListener('input', function(){
  const logz = parseFloat(this.value);
  zeta = Math.pow(10, logz);
  zeta_val.textContent = fmt(zeta);
  CR = compute_CR(zeta);
  TOTAL = CR.map((v,i)=>v+UV[i]);
  Plotly.update('plot',
    {'x': [lines, lines], 'y': [TOTAL, CR]},
    {'xaxis': {type:'category', categoryorder:'array', categoryarray: lines}},
    [0,1]
  ).then(update_chi2_display);
});

chi_slider.addEventListener('input', function(){
  const logc = parseFloat(this.value);
  chi = Math.pow(10, logc);
  chi_val.textContent = chi.toFixed(2);
  UV = compute_UV(chi);
  TOTAL = CR.map((v,i)=>v+UV[i]);
  Plotly.update('plot',
    {'x': [lines, lines], 'y': [TOTAL, UV]},
    {'xaxis': {type:'category', categoryorder:'array', categoryarray: lines}},
    [0,2]
  ).then(update_chi2_display);
});

reset_btn.addEventListener('click', function(){
  zeta = 1.7e-16; chi = 1.89;
  zeta_slider.value = -16; chi_slider.value = 0;
  zeta_val.textContent = fmt(zeta);
  chi_val.textContent = chi.toFixed(2);
  CR = compute_CR(zeta); UV = compute_UV(chi);
  TOTAL = CR.map((v,i)=>v+UV[i]);
  Plotly.update('plot',
    {'x': [lines, lines, lines], 'y': [TOTAL, CR, UV]},
    {'xaxis': {type:'category', categoryorder:'array', categoryarray: lines}},
    [0,1,2]
  ).then(update_chi2_display);
});

</script>
</body>
</html>
"""
