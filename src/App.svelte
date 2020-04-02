<script>
  
  import { scaleLinear } from "d3-scale";
  // import { Date } from "d3-time"
  import Chart from './Chart.svelte';
  import { onMount } from 'svelte';
  import { selectAll } from 'd3-selection'
  import { drag } from 'd3-drag';
  import queryString from "query-string";
  import Checkbox from './Checkbox.svelte';
  import Arrow from './Arrow.svelte';
  import { format } from 'd3-format'
  import { event } from 'd3-selection'

  import katex from 'katex';

  const legendheight = 67 

  function range(n){
    return Array(n).fill().map((_, i) => i);
  }

  function formatNumber(num) {
    return num.toString().replace(/(\d)(?=(\d{3})+(?!\d))/g, '$1,')
  }

  var sum = function(arr, bools){
    var x = 0
    for (var i = 0; i < arr.length; i++) {
      x = x + arr[i]*(bools[i] ? 1 : 0)
    }
    return x
  }

  var Integrators = {
    Euler    : [[1]],
    Midpoint : [[.5,.5],[0, 1]],
    Heun     : [[1, 1],[.5,.5]],
    Ralston  : [[2/3,2/3],[.25,.75]],
    K3       : [[.5,.5],[1,-1,2],[1/6,2/3,1/6]],
    SSP33    : [[1,1],[.5,.25,.25],[1/6,1/6,2/3]],
    SSP43    : [[.5,.5],[1,.5,.5],[.5,1/6,1/6,1/6],[1/6,1/6,1/6,1/2]],
    RK4      : [[.5,.5],[.5,0,.5],[1,0,0,1],[1/6,1/3,1/3,1/6]],
    RK38     : [[1/3,1/3],[2/3,-1/3,1],[1,1,-1,1],[1/8,3/8,3/8,1/8]]
  };

  // f is a func of time t and state y
  // y is the initial state, t is the time, h is the timestep
  // updated y is returned.
  var integrate=(m,f,y,t,h)=>{
    for (var k=[],ki=0; ki<m.length; ki++) {
      var _y=y.slice(), dt=ki?((m[ki-1][0])*h):0;
      for (var l=0; l<_y.length; l++) for (var j=1; j<=ki; j++) _y[l]=_y[l]+h*(m[ki-1][j])*(k[ki-1][l]);
      k[ki]=f(t+dt,_y,dt); 
    }
    for (var r=y.slice(),l=0; l<_y.length; l++) for (var j=0; j<k.length; j++) r[l]=r[l]+h*(k[j][l])*(m[ki-1][j]);
    return r;
  }


  $: Time_to_death     = 32
  $: logN              = Math.log(1.85e6)
  $: N                 = Math.exp(logN)
  $: I0                = 1
  $: R0                = 3
  $: D_incbation       = 7       
  $: D_infectious      = 2.9 
  $: D_recovery_mild   = (14 - D_infectious)  
  $: D_recovery_severe = (31.5 - D_infectious)
  $: D_hospital_lag    = 5
  $: D_death           = Time_to_death - D_infectious 
  $: CFR               = 0.03  
  $: InterventionTime  = 35  
  $: OMInterventionAmt = 0.6
  $: InterventionAmt   = 1 - OMInterventionAmt
  $: InterventionTime1  = 90  
  $: OMInterventionAmt1 = 0.8
  $: InterventionAmt1   = 1 - OMInterventionAmt1
  $: InterventionTime2  = 150  
  $: OMInterventionAmt2 = 0.6
  $: InterventionAmt2   = 1 - OMInterventionAmt2
  $: Time              = 220
  $: Xmax              = 110000
  $: dt                = 2
  $: P_SEVERE          = 0.2
  $: duration          = 7*12*1e10

  $: state = location.protocol + '//' + location.host + location.pathname + "?" + queryString.stringify({"Time_to_death":Time_to_death,
               "logN":logN,
               "I0":I0,
               "R0":R0,
               "D_incbation":D_incbation,
               "D_infectious":D_infectious,
               "D_recovery_mild":D_recovery_mild,
               "D_recovery_severe":D_recovery_severe,
               "CFR":CFR,
               "InterventionTime":InterventionTime,
               "InterventionAmt":InterventionAmt,
               "InterventionTime1":InterventionTime1,
               "InterventionAmt1":InterventionAmt1,
               "InterventionTime2":InterventionTime2,
               "InterventionAmt2":InterventionAmt2,
               "D_hospital_lag":D_hospital_lag,
               "P_SEVERE": P_SEVERE})
			   
   
// dt, N, I0, R0, D_incbation, D_infectious, D_recovery_mild, D_hospital_lag, D_recovery_severe, D_death, P_SEVERE, CFR, InterventionTime, InterventionAmt, duration

  function get_solution(dt, N, I0, R0, D_incbation, D_infectious, D_recovery_mild, D_hospital_lag, D_recovery_severe, D_death, P_SEVERE, CFR, InterventionTime, InterventionAmt, InterventionTime1, InterventionAmt1, InterventionTime2, InterventionAmt2, duration) {
	var interpolation_steps = 40
    var steps = 110*interpolation_steps
    var dt = dt/interpolation_steps
    var sample_step = interpolation_steps

    var method = Integrators["RK4"]
    function f(t, x){

      // SEIR ODE
      if (t > InterventionTime2 && t < InterventionTime2 + duration){
        var beta = (InterventionAmt2)*R0/(D_infectious)
      } else if (t > InterventionTime1 && t < InterventionTime1 + duration){
        var beta = (InterventionAmt1)*R0/(D_infectious)
      } else if (t > InterventionTime && t < InterventionTime + duration){
        var beta = (InterventionAmt)*R0/(D_infectious)
      } else if (t > InterventionTime2 + duration) {
        var beta = 0.5*R0/(D_infectious)        
      } else {
        var beta = R0/(D_infectious)
      }
      var a     = 1/D_incbation
      var gamma = 1/D_infectious
      
      var S        = x[0] // Susectable
      var E        = x[1] // Exposed
      var I        = x[2] // Infectious 
      var Mild     = x[3] // Recovering (Mild)     
      var Severe   = x[4] // Recovering (Severe at home)
      var Severe_H = x[5] // Recovering (Severe in hospital)
      var Fatal    = x[6] // Recovering (Fatal)
      var R_Mild   = x[7] // Recovered
      var R_Severe = x[8] // Recovered
      var R_Fatal  = x[9] // Dead

      var p_severe = P_SEVERE
      var p_fatal  = CFR
      var p_mild   = 1 - P_SEVERE - CFR

      var dS        = -beta*I*S
      var dE        =  beta*I*S - a*E
      var dI        =  a*E - gamma*I
      var dMild     =  p_mild*gamma*I   - (1/D_recovery_mild)*Mild
      var dSevere   =  p_severe*gamma*I - (1/D_hospital_lag)*Severe
      var dSevere_H =  (1/D_hospital_lag)*Severe - (1/D_recovery_severe)*Severe_H
      var dFatal    =  p_fatal*gamma*I  - (1/D_death)*Fatal
      var dR_Mild   =  (1/D_recovery_mild)*Mild
      var dR_Severe =  (1/D_recovery_severe)*Severe_H
      var dR_Fatal  =  (1/D_death)*Fatal

      //      0   1   2   3      4        5          6       7        8          9
      return [dS, dE, dI, dMild, dSevere, dSevere_H, dFatal, dR_Mild, dR_Severe, dR_Fatal]
    }

    var v = [1 - I0/N, 0, I0/N, 0, 0, 0, 0, 0, 0, 0]
    var t = 0

    var P  = []
    var TI = []
    var Iters = []
    while (steps--) { 
      if ((steps+1) % (sample_step) == 0) {
            //    Dead   Hospital          Recovered        Infectious   Exposed
        P.push([ N*v[9], N*(v[5]+v[6]),  N*(v[7] + v[8]), N*v[2],    N*v[1] ])
        Iters.push(v)
        TI.push(N*(1-v[0]))
        // console.log((v[0] + v[1] + v[2] + v[3] + v[4] + v[5] + v[6] + v[7] + v[8] + v[9]))
        // console.log(v[0] , v[1] , v[2] , v[3] , v[4] , v[5] , v[6] , v[7] , v[8] , v[9])
      }
      v =integrate(method,f,v,t,dt); 
      t+=dt
    }
    return {"P": P, 
            "deaths": N*v[6], 
            "total": 1-v[0],
            "total_infected": TI,
            "Iters":Iters,
            "dIters": f}
  }

  function max(P, checked) {
    return P.reduce((max, b) => Math.max(max, sum(b, checked) ), sum(P[0], checked) )
  }

  $: Sol            = get_solution(dt, N, I0, R0, D_incbation, D_infectious, D_recovery_mild, D_hospital_lag, D_recovery_severe, D_death, P_SEVERE, CFR, InterventionTime, InterventionAmt, InterventionTime1, InterventionAmt1, InterventionTime2, InterventionAmt2, duration)
  $: P              = Sol["P"].slice(0,100)
  $: timestep       = dt
  $: tmax           = dt*100
  $: deaths         = Sol["deaths"]
  $: total          = Sol["total"]
  $: total_infected = Sol["total_infected"].slice(0,100)
  $: Iters          = Sol["Iters"]
  $: dIters         = Sol["dIters"]
  $: Pmax           = max(P, checked)
  $: lock           = false

  var colors = [ "#386cb0", "#8da0cb", "#4daf4a", "#f0027f", "#fdc086"]

  var Plock = 1

  var drag_y = function (){
    var dragstarty = 0
    var Pmaxstart = 0

    var dragstarted = function (d) {
      dragstarty = event.y  
      Pmaxstart  = Pmax
    }

    var dragged = function (d) {
      Pmax = Math.max( (Pmaxstart*(1 + (event.y - dragstarty)/500)), 10)
    }

    return drag().on("drag", dragged).on("start", dragstarted)
  }

  var drag_x = function (){
    var dragstartx = 0
    var dtstart = 0
    var Pmaxstart = 0
    var dragstarted = function (d) {
      dragstartx = event.x
      dtstart  = dt
      Plock = Pmax
      lock = true
    }
    var dragged = function (d) {
      dt = dtstart - 0.0015*(event.x - dragstartx)
    }
    var dragend = function (d) {
      lock = false
    }
    return drag().on("drag", dragged).on("start", dragstarted).on("end", dragend)
  }

  var drag_intervention = function (){
    var dragstarty = 0
    var InterventionTimeStart = 0

    var dragstarted = function (d) {
      dragstarty = event.x  
      InterventionTimeStart = InterventionTime
      Plock = Pmax
      lock = true
    }

    var dragged = function (d) {
      // InterventionTime = Math.max( (*(1 + (event.x - dragstarty)/500)), 10)
      // console.log(event.x)
      InterventionTime = Math.min(tmax-1, Math.max(0, InterventionTimeStart + xScaleTimeInv(event.x - dragstarty)), InterventionTime1)
    }

    var dragend = function (d) {
      lock = false
    }

    return drag().on("drag", dragged).on("start", dragstarted).on("end", dragend)
  }
  
  var drag_intervention1 = function (){
    var dragstarty = 0
    var InterventionTimeStart = 0

    var dragstarted = function (d) {
      dragstarty = event.x  
      InterventionTimeStart = InterventionTime1
      Plock = Pmax
      lock = true
    }

    var dragged = function (d) {
      // InterventionTime = Math.max( (*(1 + (event.x - dragstarty)/500)), 10)
      // console.log(event.x)
      InterventionTime1 = Math.min(tmax-1, Math.max(InterventionTime, InterventionTimeStart + xScaleTimeInv(event.x - dragstarty)), InterventionTime2)
    }

    var dragend = function (d) {
      lock = false
    }

    return drag().on("drag", dragged).on("start", dragstarted).on("end", dragend)
  }
  
  var drag_intervention2 = function (){
    var dragstarty = 0
    var InterventionTimeStart = 0

    var dragstarted = function (d) {
      dragstarty = event.x  
      InterventionTimeStart = InterventionTime2
      Plock = Pmax
      lock = true
    }

    var dragged = function (d) {
      // InterventionTime = Math.max( (*(1 + (event.x - dragstarty)/500)), 10)
      // console.log(event.x)
      InterventionTime2 = Math.min(tmax-1, Math.max(InterventionTime1, InterventionTimeStart + xScaleTimeInv(event.x - dragstarty)))
    }

    var dragend = function (d) {
      lock = false
    }

    return drag().on("drag", dragged).on("start", dragstarted).on("end", dragend)
  }


  var drag_intervention_end = function (){
  console.log('dragend');
    var dragstarty = 0
    var durationStart = 0

    var dragstarted = function (d) {
      dragstarty = event.x  
      durationStart = duration
      Plock = Pmax
      lock = true
    }

    var dragged = function (d) {
      // InterventionTime = Math.max( (*(1 + (event.x - dragstarty)/500)), 10)
      // console.log(event.x)
      duration = Math.min(tmax-1, Math.max(0, durationStart + xScaleTimeInv(event.x - dragstarty)))
    }

    var dragend = function (d) {
      lock = false
    }

    return drag().on("drag", dragged).on("start", dragstarted).on("end", dragend)
  }


  $: parsed = "";
  onMount(async () => {
    var drag_callback_y = drag_y()
    drag_callback_y(selectAll("#yAxisDrag"))
    var drag_callback_x = drag_x()
    drag_callback_x(selectAll("#xAxisDrag"))
    var drag_callback_intervention = drag_intervention()
    drag_callback_intervention(selectAll("#intervention"))
	var drag_callback_intervention1 = drag_intervention1()
    drag_callback_intervention1(selectAll("#intervention1"))
	var drag_callback_intervention2 = drag_intervention2()
    drag_callback_intervention2(selectAll("#intervention2"))
	
    // var drag_callback_intervention_end = drag_intervention_end()
    // drag_callback_intervention_end(selectAll("#dottedline2"))

    if (typeof window !== 'undefined') {
      parsed = queryString.parse(window.location.search)
	  
      if (!(parsed.logN === undefined)) {logN = parsed.logN}
      if (!(parsed.I0 === undefined)) {I0 = parseFloat(parsed.I0)}
      if (!(parsed.R0 === undefined)) {R0 = parseFloat(parsed.R0)}
      if (!(parsed.D_incbation === undefined)) {D_incbation = parseFloat(parsed.D_incbation)}
      if (!(parsed.D_infectious === undefined)) {D_infectious = parseFloat(parsed.D_infectious)}
      if (!(parsed.D_recovery_mild === undefined)) {D_recovery_mild = parseFloat(parsed.D_recovery_mild)}
      if (!(parsed.D_recovery_severe === undefined)) {D_recovery_severe = parseFloat(parsed.D_recovery_severe)}
      if (!(parsed.CFR === undefined)) {CFR = parseFloat(parsed.CFR)}
      if (!(parsed.InterventionTime === undefined)) {InterventionTime = parseFloat(parsed.InterventionTime)}
      if (!(parsed.InterventionAmt === undefined)) {InterventionAmt = parseFloat(parsed.InterventionAmt)}
	  if (!(parsed.InterventionTime1 === undefined)) {InterventionTime1 = parseFloat(parsed.InterventionTime1)}
      if (!(parsed.InterventionAmt1 === undefined)) {InterventionAmt1 = parseFloat(parsed.InterventionAmt1)}
	  if (!(parsed.InterventionTime2 === undefined)) {InterventionTime2 = parseFloat(parsed.InterventionTime2)}
      if (!(parsed.InterventionAmt2 === undefined)) {InterventionAmt2 = parseFloat(parsed.InterventionAmt2)}
      if (!(parsed.D_hospital_lag === undefined)) {D_hospital_lag = parseFloat(parsed.D_hospital_lag)}
      if (!(parsed.P_SEVERE === undefined)) {P_SEVERE = parseFloat(parsed.P_SEVERE)}
      if (!(parsed.Time_to_death === undefined)) {Time_to_death = parseFloat(parsed.Time_to_death)}

    }
  });

  function lock_yaxis(){
    Plock = Pmax
    lock  = true
  }

  function unlock_yaxis(){
    lock = false
  }

  const padding = { top: 20, right: 0, bottom: 20, left: 25 };

  let width  = 750;
  let height = 400;

  $: xScaleTime = scaleLinear()
    .domain([0, tmax])
    .range([padding.left, width - padding.right]);

  $: xScaleTimeInv = scaleLinear()
    .domain([0, width])
    .range([0, tmax]);

  $: indexToTime = scaleLinear()
    .domain([0, P.length])
    .range([0, tmax])

  window.addEventListener('mouseup', unlock_yaxis);

  $: checked = [true, true, false, true, true]
  $: active  = 0
  $: active_ = active >= 0 ? active : Iters.length - 1

  var Tinc_s = "\\color{#CCC}{T^{-1}_{\\text{inc}}} "
  var Tinf_s = "\\color{#CCC}{T^{-1}_{\\text{inf}}}"
  var Rt_s   = "\\color{#CCC}{\\frac{\\mathcal{R}_{t}}{T_{\\text{inf}}}} "
  $: ode_eqn = katex.renderToString("\\frac{d S}{d t}=-" +Rt_s +"\\cdot IS,\\qquad \\frac{d E}{d t}=" +Rt_s +"\\cdot IS- " + Tinc_s + " E,\\qquad \\frac{d I}{d t}=" + Tinc_s + "E-" + Tinf_s+ "I, \\qquad \\frac{d R}{d t}=" + Tinf_s+ "I", {
    throwOnError: false,
    displayMode: true,
    colorIsTextColor: true
  });

  function math_inline(str) {
    return katex.renderToString(str, {
    throwOnError: false,
    displayMode: false,
    colorIsTextColor: true
    });
  }

  function math_display(str) {
    return katex.renderToString(str, {
    throwOnError: false,
    displayMode: true,
    colorIsTextColor: true
    });
  }
  
  $: p_num_ind = 40

  $: get_d = function(i){
    return dIters(indexToTime(i), Iters[i])
  }

  function get_milestones(P){

    function argmax(x, index) {
      return x.map((x, i) => [x[index], i]).reduce((r, a) => (a[0] > r[0] ? a : r))[1];
    }

     //    Dead   Hospital          Recovered        Infectious   Exposed
	 console.log(P);
    var milestones = []
	var firtDeathFound = false;
	var infectionEndDateFound = false;
    for (var i = 0; i < P.length; i++) {
      if (!firtDeathFound && P[i][0] >= 0.5) {
        milestones.push([i*dt, "Первая смерть"]);
		firtDeathFound = true;
      }
	  
	  if (!infectionEndDateFound && P[i][3] < 0.5) {
        milestones.push([i*dt, "Инфицирование остановлено"])
		infectionEndDateFound = true;
      }
    }

    var i = argmax(P, 1)
    milestones.push([i*dt, "Пик: " + format(",")(Math.round(P[i][1])) + " госпитализировано"])

    return milestones
  }

  $: milestones = get_milestones(P)
  $: log = true

</script>

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.11.1/dist/katex.css" integrity="sha384-bsHo4/LA+lkZv61JspMDQB9QP1TtO4IgOf2yYS+J6VdAYLVyx1c3XKcsHh0Vy8Ws" crossorigin="anonymous">

<style>
  .small { font: italic 6px Source Code Pro; }
  @import url('https://fonts.googleapis.com/css?family=Source+Code+Pro&display=swap');


  h2 {
    margin: auto;
    width: 950px;
    font-size: 40px;
    padding-top: 20px;
    padding-bottom: 20px;
    font-weight: 300;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    padding-bottom: 30px
  }

  .center {
    margin: auto;
    width: 950px;
    padding-bottom: 20px;
    font-weight: 300;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    color:#666;
    font-size: 16.5px;
    text-align: justify;
    line-height: 24px
  }

  .ack {
    margin: auto;
    width: 950px;
    padding-bottom: 20px;
    font-weight: 300;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    color:#333;
    font-size: 13px;
  }

  .row {
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    margin: auto;
    display: flex;
    width: 948px;
    font-size: 13px;
  }

  .caption {
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    font-size: 13px;    
  }

  .column {
    flex: 158px;
    padding: 0px 5px 5px 0px;
    margin: 0px 5px 5px 5px;
    /*border-top: 2px solid #999*/
  }

  .minorTitle {
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    margin: auto;
    display: flex;
    width: 950px;
    font-size: 17px;
    color: #666;
  }

  .minorTitleColumn{
    flex: 60px;
    padding: 3px;
    border-bottom: 2px solid #999;
  }


  .paneltext{
    position:relative;
    height:130px;
  }

  .paneltitle{
    color:#777; 
    line-height: 17px; 
    padding-bottom: 4px;
    font-weight: 700;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
  }

  .paneldesc{
    color:#888; 
    text-align: left;
    font-weight: 300;
  }

  .slidertext{
    color:#555; 
    line-height: 7px; 
    padding-bottom: 0px; 
    padding-top: 7px;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    font-family: 'Source Code Pro', monospace;
    font-size: 10px;
    text-align: right;
    /*font-weight: bold*/
  }
    
  .range {
    width: 100%;
  }

  .chart {
    width: 100%;
    margin: 0 auto;
    padding-top:0px;
    padding-bottom:10px;
  }

  .legend {
    color: #888;
    font-family: Helvetica, Arial;
    font-size: .725em;
    font-weight: 200;
    height: 100px;
    left: 20px;
    top: 4px;
    position: absolute;
  }

  .legendtitle {
    color:#777; 
    font-size:13px;
    padding-bottom: 6px;
    font-weight: 600;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
  }


  .legendtext{
    color:#888; 
    font-size:13px;
    padding-bottom: 5px;
    font-weight: 300;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    line-height: 14px;
  }

  .legendtextnum{
    color:#888; 
    font-size:13px;
    padding-bottom: 5px;
    font-weight: 300;
    line-height: 12px;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    left: -3px;
    position: relative;
  }

  .tick {
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    font-size: .725em;
    font-weight: 200;
    font-size: 13px
  }

  td { 
    text-align: left;
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    border-bottom: 1px solid #DDD;
    border-collapse: collapse;
    padding: 3px;
    /*font-size: 14px;*/
  }

  tr {
    border-collapse: collapse;
    border-spacing: 15px;
  }

  .eqn {
    font-family: nyt-franklin,helvetica,arial,sans-serif;
    margin: auto;
    display: flex;
    flex-flow: row wrap;
    width: 950px;
    column-count: 4;
    font-weight: 300;
    color:#666;
    font-size: 16.5px;
  }

  th { font-weight: 500; text-align: left; padding-bottom: 5px; vertical-align: text-top;     border-bottom: 1px solid #DDD; }

  a:link { color: grey; }
  a:visited { color: grey; }

</style>

<h2>Прогноз распространения Коронавируса COVID-19 с учетом карантинных мер</h2>

<div class="chart" style="display: flex; max-width: 1120px">

  <div style="flex: 0 0 270px; width:270px;">
    <div style="position:relative; top:48px; right:-115px">
      <div class="legendtext" style="position:absolute; left:-16px; top:-34px; width:50px; height: 100px; font-size: 13px; line-height:16px; font-weight: normal; text-align: center"><b>День</b><br> {Math.round(indexToTime(active_))}</div>

      <!-- Susceptible -->
      <div style="position:absolute; left:0px; top:0px; width: 180px; height: 100px">

        <span style="pointer-events: none"><Checkbox color="#CCC"/></span>
        <Arrow height="41"/>

        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Восприимчивы</div>
          <div style="padding-top: 5px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*Iters[active_][0]))} 
                                  ({ (100*Iters[active_][0]).toFixed(2) }%)</i></div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*get_d(active_)[0]))} / день</i>
                                 </div>
          </div>
        </div>
          <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 4px; position:relative;">Нет иммунитета к вирусу</div>

      </div>

      <!-- Exposed -->
      <div style="position:absolute; left:0px; top:{legendheight*1}px; width: 180px; height: 100px">

        <Checkbox color="{colors[4]}" bind:checked={checked[4]}/>      
        <Arrow height="41"/>

        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Заражены</div>

          <div style="padding-top: 5px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*Iters[active_][1]))} 
                                  ({ (100*Iters[active_][1]).toFixed(2) }%)</div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*get_d(active_)[1])) } / день</i>
                                 </div>
          </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 4px; position:relative;">Заражены, но еще не заразны</div>

      </div>

      <!-- Infectious -->
      <div style="position:absolute; left:0px; top:{legendheight*2}px; width: 180px; height: 100px">

        <Checkbox color="{colors[3]}" bind:checked={checked[3]}/>
        <Arrow height="41"/>   

        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Заразны</div>
          <div style="padding-top: 5px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*Iters[active_][2]))} 
                                  ({ (100*Iters[active_][2]).toFixed(2) }%)</div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*get_d(active_)[2])) } / день</i>
                                 </div>
          </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 4px; position:relative;">Число <i>активно</i> разносящих инфекцию</div>


      </div>

      <!-- Removed -->
      <div style="position:absolute; left:0px; top:{legendheight*3}px; width: 180px; height: 100px">

        <Checkbox color="grey" callback={(s) => {checked[1] = s; checked[0] = s; checked[2] = s} }/>
        <Arrow height="56" arrowhead="" dasharray="3 2"/>

        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Не подвержены</div>
          <div style="padding-top: 10px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N* (1-Iters[active_][0]-Iters[active_][1]-Iters[active_][2]) ))} 
                                  ({ ((100*(1-Iters[active_][0]-Iters[active_][1]-Iters[active_][2]))).toFixed(2) }%)</div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*(get_d(active_)[3]+get_d(active_)[4]+get_d(active_)[5]+get_d(active_)[6]+get_d(active_)[7]) )) } / день</i>
                                 </div>
          </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 4x; position:relative;">Число людей с иммунитетом или изолированы</div>

      </div>

      <!-- Recovered -->
      <div style="position:absolute; left:0px; top:{legendheight*4+14-3}px; width: 180px; height: 100px">
        <Checkbox color="{colors[2]}" bind:checked={checked[2]}/>
        <Arrow height="23" arrowhead="" dasharray="3 2"/>
        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Вылечились</div>

          <div style="padding-top: 3px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*(Iters[active_][7]+Iters[active_][8]) ))} 
                                  ({ (100*(Iters[active_][7]+Iters[active_][8])).toFixed(2) }%)</div>
          </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 8px; position:relative;">Полностью вылечились</div>

      </div>

      <!-- Hospitalized -->
      <div style="position:absolute; left:0px; top:{legendheight*4+57}px; width: 180px; height: 100px">
        <Arrow height="43" arrowhead="" dasharray="3 2"/>
        <Checkbox color="{colors[1]}" bind:checked={checked[1]}/>
        <div class="legend" style="position:absolute;">
          <div class="legendtitle">В больнице</div>
          <div style="padding-top: 3px; padding-bottom: 1px">
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*(Iters[active_][5]+Iters[active_][6]) ))} 
                                  ({ (100*(Iters[active_][5]+Iters[active_][6])).toFixed(2) }%)</div>
          </div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*(get_d(active_)[5]+get_d(active_)[6]))) } / день</i>
                                 </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 10px; position:relative;">Активные госпитализации</div>

      </div>

      <div style="position:absolute; left:0px; top:{legendheight*4 + 120+2}px; width: 180px; height: 100px">
        <Arrow height="40" arrowhead="" dasharray="3 2"/>

        <Checkbox color="{colors[0]}" bind:checked={checked[0]}/>

        <div class="legend" style="position:absolute;">
          <div class="legendtitle">Погибшие</div>
          <div style="padding-top: 3px; padding-bottom: 1px">          
          <div class="legendtextnum"><span style="font-size:12px; padding-right:3px; color:#CCC">∑</span> <i>{formatNumber(Math.round(N*Iters[active_][9]))} 
                                  ({ (100*Iters[active_][9]).toFixed(2) }%)</div>
          <div class="legendtextnum"><span style="font-size:12px; padding-right:2px; color:#CCC">Δ</span> <i>{formatNumber(Math.round(N*get_d(active_)[9])) } / день</i>
                                 </div>
          </div>
        </div>
        <div class="legendtext" style="text-align: right; width:105px; left:-111px; top: 10px; position:relative;">Число смертей</div>
      </div>
    </div>
  </div>

  <div style="flex: 0 0 890px; width:890px; height: {height+128}px; position:relative;">

    <div style="position:relative; top:60px; left: 10px">
      <Chart bind:checked={checked}
             bind:active={active}
             y = {P} 
             xmax = {Xmax} 
             total_infected = {total_infected} 
             deaths = {deaths} 
             total = {total} 
             timestep={timestep}
             tmax={tmax}
             N={N}
             ymax={lock ? Plock: Pmax}
             InterventionTime={InterventionTime}
             colors={colors}
             log={!log}/>
      </div>

      <div id="xAxisDrag"
           style="pointer-events: all;
                  position: absolute;
                  top:{height+80}px;
                  left:{0}px;
                  width:{780}px;
                  background-color:#222;
                  opacity: 0;
                  height:25px;
                  cursor:col-resize">
      </div>

      <div id="yAxisDrag"
           style="pointer-events: all;
                  position: absolute;
                  top:{55}px;
                  left:{0}px;
                  width:{20}px;
                  background-color:#222;
                  opacity: 0;
                  height:425px;
                  cursor:row-resize">
      </div>

      <!-- Intervention Line -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:100px; left:10px; pointer-events: none">
        <div class="dottedline" id="intervention"  style="pointer-events: all;
                    position: absolute;
                    top:-38px;
                    left:{xScaleTime(InterventionTime)}px;
                    visibility: {(xScaleTime(InterventionTime) < (width - padding.right)) ? 'visible':'hidden'};
                    width:2px;
                    background-color:#FFF;
                    border-right: 1px dashed black;
                    pointer-events: all;
                    cursor:col-resize;
                    height:{height+19}px">

			<div style="position:absolute; opacity: 0.5; top:5px; left:10px; width: 120px">
				<span style="font-size: 13px">{@html math_inline("\\mathcal{R}_t=" + (R0*InterventionAmt).toFixed(2) )}</span> ⟶ 
			</div>

			{#if xScaleTime(InterventionTime) >= 100}
			  <div style="position:absolute; opacity: 0.5; top: 8px; left:-97px; width: 120px">
			  <span style="font-size: 13px">⟵ {@html math_inline("\\mathcal{R}_0=" + (R0).toFixed(2) )}</span>
			  </div>      
			{/if}

			<div id="interventionDrag" class="legendtext" style="flex: 0 0 160px; width:80px; position:relative;  top:-70px; height: 60px; padding-right: 15px; left: -85px; pointer-events: all;cursor:col-resize;" >
			  <div class="paneltitle" style="top:9px; position: relative; text-align: right">Карантин 1 день {format("d")(InterventionTime)}</div>
			  <span></span><div style="top:9px; position: relative; text-align: right"></div>
			  <div style="top:43px; left:40px; position: absolute; text-align: right; width: 20px; height:20px; opacity: 0.3">
				<svg width="20" height="20">
				  <g transform="rotate(90)">
					<g transform="translate(0,-20)">
					  <path d="M2 11h16v2H2zm0-4h16v2H2zm8 11l3-3H7l3 3zm0-16L7 5h6l-3-3z"/>
					 </g>  
				  </g>
				</svg>
			  </div>
			</div>


			<div style="width:150px; position:relative; top:-85px; height: 80px; padding-right: 15px; left: 0px; ;cursor:col-resize; background-color: white; position:absolute" >

			</div>


        </div>
      </div>

      <!-- Intervention Line slider -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:120px; left:10px; pointer-events: none">
        <div style="
            position: absolute;
            top:-38px;
            left:{xScaleTime(InterventionTime)}px;
            visibility: {(xScaleTime(InterventionTime) < (width - padding.right)) ? 'visible':'hidden'};
            width:2px;
            background-color:#FFF;
            border-right: 1px dashed black;
            cursor:col-resize;
            height:{height}px">
            <div style="flex: 0 0 160px; width:100px; position:relative; top:-125px; left: 1px" >
              <div class="caption" style="pointer-events: none; position: absolute; left:0; top:40px; width:100px; border-left: 2px solid #777; padding: 5px 7px 7px 7px; ">      
				  <div class="paneltext"  style="height:30px; text-align: right">
					<div class="paneldesc">Эффективность<br></div>
				  </div>
				  <div style="pointer-events: all">
					  <div class="slidertext" on:mousedown={lock_yaxis}>{(100*(1-InterventionAmt)).toFixed(1)}%</div>
					  <input class="range" type=range bind:value={OMInterventionAmt} min=0 max=1 step=0.01 on:mousedown={lock_yaxis}>
				  </div>
              </div>
            </div>
          </div>
      </div>
	  
	  <!-- Intervention Line 1 -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:100px; left:10px; pointer-events: none">
        <div class="dottedline" id="intervention1"  style="pointer-events: all;
                    position: absolute;
                    top:-38px;
                    left:{xScaleTime(InterventionTime1)}px;
                    visibility: {(xScaleTime(InterventionTime1) < (width - padding.right)) ? 'visible':'hidden'};
                    width:2px;
                    background-color:#FFF;
                    border-right: 1px dashed black;
                    pointer-events: all;
                    cursor:col-resize;
                    height:{height+19}px">

			<div style="position:absolute; opacity: 0.5; top: 5px; left:10px; width: 120px">
				<span style="font-size: 13px">{@html math_inline("\\mathcal{R}_t=" + (R0*InterventionAmt1).toFixed(2) )}</span> ⟶ 
			</div>

			<div id="interventionDrag1" class="legendtext" style="flex: 0 0 160px; width:80px; position:relative;  top:-70px; height: 60px; padding-right: 15px; left: -85px; pointer-events: all;cursor:col-resize;" >
			  <div class="paneltitle" style="top:9px; position: relative; text-align: right">Карантин 2 день {format("d")(InterventionTime1)}</div>
			  <span></span><div style="top:9px; position: relative; text-align: right"></div>
			  <div style="top:43px; left:40px; position: absolute; text-align: right; width: 20px; height:20px; opacity: 0.3">
				<svg width="20" height="20">
				  <g transform="rotate(90)">
					<g transform="translate(0,-20)">
					  <path d="M2 11h16v2H2zm0-4h16v2H2zm8 11l3-3H7l3 3zm0-16L7 5h6l-3-3z"/>
					 </g>  
				  </g>
				</svg>
			  </div>
			</div>


			<div style="width:150px; position:relative; top:-85px; height: 80px; padding-right: 15px; left: 0px; ;cursor:col-resize; background-color: white; position:absolute" >

			</div>


        </div>
      </div>

      <!-- Intervention Line slider 1 -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:120px; left:10px; pointer-events: none">
        <div style="
            position: absolute;
            top:-38px;
            left:{xScaleTime(InterventionTime1)}px;
            visibility: {(xScaleTime(InterventionTime1) < (width - padding.right)) ? 'visible':'hidden'};
            width:2px;
            background-color:#FFF;
            border-right: 1px dashed black;
            cursor:col-resize;
            height:{height}px">
            <div style="flex: 0 0 160px; width:100px; position:relative; top:-125px; left: 1px" >
              <div class="caption" style="pointer-events: none; position: absolute; left:0; top:40px; width:100px; border-left: 2px solid #777; padding: 5px 7px 7px 7px; ">      
				  <div class="paneltext"  style="height:30px; text-align: right">
					<div class="paneldesc">Эффективность<br></div>
				  </div>
				  <div style="pointer-events: all">
					  <div class="slidertext" on:mousedown={lock_yaxis}>{(100*(1-InterventionAmt1)).toFixed(1)}%</div>
					  <input class="range" type=range bind:value={OMInterventionAmt1} min=0 max=1 step=0.01 on:mousedown={lock_yaxis}>
				  </div>
              </div>
            </div>
          </div>
      </div>
	  
	  <!-- Intervention Line 2 -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:100px; left:10px; pointer-events: none">
        <div class="dottedline" id="intervention2"  style="pointer-events: all;
                    position: absolute;
                    top:-38px;
                    left:{xScaleTime(InterventionTime2)}px;
                    visibility: {(xScaleTime(InterventionTime2) < (width - padding.right)) ? 'visible':'hidden'};
                    width:2px;
                    background-color:#FFF;
                    border-right: 1px dashed black;
                    pointer-events: all;
                    cursor:col-resize;
                    height:{height+19}px">

			<div style="position:absolute; opacity: 0.5; top: 5px; left:10px; width: 120px">
				<span style="font-size: 13px">{@html math_inline("\\mathcal{R}_t=" + (R0*InterventionAmt2).toFixed(2) )}</span> ⟶ 
			</div>

			<div id="interventionDrag2" class="legendtext" style="flex: 0 0 160px; width:80px; position:relative;  top:-70px; height: 60px; padding-right: 15px; left: -85px; pointer-events: all;cursor:col-resize;" >
			  <div class="paneltitle" style="top:9px; position: relative; text-align: right">Карантин 3 день {format("d")(InterventionTime2)}</div>
			  <span></span><div style="top:9px; position: relative; text-align: right"></div>
			  <div style="top:43px; left:40px; position: absolute; text-align: right; width: 20px; height:20px; opacity: 0.3">
				<svg width="20" height="20">
				  <g transform="rotate(90)">
					<g transform="translate(0,-20)">
					  <path d="M2 11h16v2H2zm0-4h16v2H2zm8 11l3-3H7l3 3zm0-16L7 5h6l-3-3z"/>
					 </g>  
				  </g>
				</svg>
			  </div>
			</div>


			<div style="width:150px; position:relative; top:-85px; height: 80px; padding-right: 15px; left: 0px; ;cursor:col-resize; background-color: white; position:absolute" >

			</div>

        </div>
      </div>

      <!-- Intervention Line slider 2 -->
      <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:120px; left:10px; pointer-events: none">
        <div style="
            position: absolute;
            top:-38px;
            left:{xScaleTime(InterventionTime2)}px;
            visibility: {(xScaleTime(InterventionTime2) < (width - padding.right)) ? 'visible':'hidden'};
            width:2px;
            background-color:#FFF;
            border-right: 1px dashed black;
            cursor:col-resize;
            height:{height}px">
            <div style="flex: 0 0 160px; width:100px; position:relative; top:-125px; left: 1px" >
              <div class="caption" style="pointer-events: none; position: absolute; left:0; top:40px; width:100px; border-left: 2px solid #777; padding: 5px 7px 7px 7px; ">      
				  <div class="paneltext"  style="height:30px; text-align: right">
					<div class="paneldesc">Эффективность<br></div>
				  </div>
				  <div style="pointer-events: all">
					  <div class="slidertext" on:mousedown={lock_yaxis}>{(100*(1-InterventionAmt2)).toFixed(1)}%</div>
					  <input class="range" type=range bind:value={OMInterventionAmt2} min=0 max=1 step=0.01 on:mousedown={lock_yaxis}>
				  </div>
              </div>
            </div>
          </div>
      </div>

<!-- 
      {#if xScaleTime(InterventionTime+duration) < (width - padding.right)}
        <div id="dottedline2" style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:105px; left:10px; pointer-events: none;">
          <div style="
              position: absolute;
              top:-38px;
              left:{xScaleTime(InterventionTime+duration)}px;
              visibility: {(xScaleTime(InterventionTime+duration) < (width - padding.right)) ? 'visible':'hidden'};
              width:3px;
              background-color:white;
              border-right: 1px dashed black;
              cursor:col-resize;
              opacity: 0.3;
              pointer-events: all;
              height:{height+13}px">
            <div style="position:absolute; opacity: 0.5; top:-10px; left:10px; width: 120px">
            <span style="font-size: 13px">{@html math_inline("\\mathcal{R}_t=" + (R0*InterventionAmt).toFixed(2) )}</span> ⟶ 
            </div>
          </div>
        </div>

        <div style="position: absolute; width:{width+15}px; height: {height}px; position: absolute; top:120px; left:10px; pointer-events: none">
          <div style="
              opacity: 0.5;
              position: absolute;
              top:-38px;
              left:{xScaleTime(InterventionTime+duration)}px;
              visibility: {(xScaleTime(InterventionTime+duration) < (width - padding.right)) ? 'visible':'hidden'};
              width:2px;
              background-color:#FFF;
              cursor:col-resize;
              height:{height}px">
              <div style="flex: 0 0 160px; width:200px; position:relative; top:-125px; left: 1px" >
                <div class="caption" style="pointer-events: none; position: absolute; left:0; top:40px; width:150px; border-left: 2px solid #777; padding: 5px 7px 7px 7px; ">      
                <div class="paneltext"  style="height:20px; text-align: right">
                <div class="paneldesc">decrease transmission by<br></div>
                </div>
                <div style="pointer-events: all">
                <div class="slidertext" on:mousedown={lock_yaxis}>{(InterventionAmt).toFixed(2)}</div>
                <input class="range" type=range bind:value={InterventionAmt} min=0 max=1 step=0.01 on:mousedown={lock_yaxis}>
                </div>
                </div>
              </div>
            </div>
        </div>
      {/if} -->


      <div style="pointer-events: none;
                  position: absolute;
                  top:{height+84}px;
                  left:{0}px;
                  width:{780}px;
                  opacity: 1.0;
                  height:25px;
                  cursor:col-resize">
            {#each milestones as milestone}
              <div style="position:absolute; left: {xScaleTime(milestone[0])+8}px; top: -30px;">
                <span style="opacity: 0.3"><Arrow height=30 arrowhead="#circle" dasharray = "2 1"/></span>
                  <div class="tick" style="position: relative; left: 0px; top: 35px; max-width: 130px; color: #BBB; background-color: white; padding-left: 4px; padding-right: 4px">{@html milestone[1]}</div>
              </div>
            {/each}
      </div>
    
    <div style="opacity:{xScaleTime(InterventionTime) >= 192? 1.0 : 0.2}">
      <div class="tick" style="color: #AAA; position:absolute; pointer-events:all; left:10px; top: 10px">
        <Checkbox color="#CCC" bind:checked={log}/><div style="position: relative; top: 4px; left:20px; width: 50px;">линейный масштаб</div>
      </div>
    </div>

   </div>

</div>


<div style="height:220px;">
  <div class="minorTitle">
    <div style="margin: 0px 0px 5px 4px" class="minorTitleColumn">Динамика распространения</div>
    <div style="flex: 0 0 20; width:20px"></div>
    <div style="margin: 0px 4px 5px 0px" class="minorTitleColumn">Клиническая динамика</div>
  </div>
  <div class = "row">

    <div class="column">
      <div class="paneltitle">Данные населеннности</div>
      <div class="paneldesc" style="height:30px">Размер популяции<br></div>
      <div class="slidertext">{format(",")(Math.round(N))}</div>
      <input class="range" style="margin-bottom: 8px"type=range bind:value={logN} min={5} max=25 step=0.01>
      <div class="paneldesc" style="height:29px; border-top: 1px solid #EEE; padding-top: 10px">Стартовое число заразных<br></div>
      <div class="slidertext">{I0}</div>
      <input class="range" type=range bind:value={I0} min={1} max=10000 step=1>
    </div>

    <div class="column">
      <div class="paneltext">
      <div class="paneltitle">Basic Reproduction Number (Заразность) {@html math_inline("\\mathcal{R}_0")} </div>
      <div class="paneldesc">Число людей, которых заражает один заразный<br></div>
      </div>
      <div class="slidertext">{R0}</div>
      <input class="range" type=range bind:value={R0} min=0.01 max=10 step=0.01> 
    </div> 

    <div class="column">
      <div class="paneltitle">Сроки распространения</div>
      <div class="paneldesc" style="height:30px">Инкубационный период, {@html math_inline("T_{\\text{inc}}")}.<br></div>
      <div class="slidertext">{(D_incbation).toFixed(2)} дней</div>
      <input class="range" style="margin-bottom: 8px"type=range bind:value={D_incbation} min={0.15} max=24 step=0.0001>
      <div class="paneldesc" style="height:29px; border-top: 1px solid #EEE; padding-top: 10px">Период заразности, {@html math_inline("T_{\\text{inf}}")}.<br></div>
      <div class="slidertext">{D_infectious} дней</div>
      <input class="range" type=range bind:value={D_infectious} min={0} max=24 step=0.01>
    </div>

    <div style="flex: 0 0 20; width:20px"></div>

    <div class="column">
      <div class="paneltitle">Статистика смертности</div>
      <div class="paneldesc" style="height:30px">Доля смертельных случаев<br></div>
      <div class="slidertext">{(CFR*100).toFixed(2)} %</div>
      <input class="range" style="margin-bottom: 8px" type=range bind:value={CFR} min={0} max=1 step=0.0001>
      <div class="paneldesc" style="height:29px; border-top: 1px solid #EEE; padding-top: 10px">Время от заболевания до смерти<br></div>
      <div class="slidertext">{Time_to_death} дней</div>
      <input class="range" type=range bind:value={Time_to_death} min={(D_infectious)+0.1} max=100 step=0.01>
    </div>

    <div class="column">
      <div class="paneltitle">Выздоровление</div>
      <div class="paneldesc" style="height:30px">Длительность госпитализации<br></div>
      <div class="slidertext">{D_recovery_severe} дней</div>
      <input class="range" style="margin-bottom: 8px" type=range bind:value={D_recovery_severe} min={0.1} max=100 step=0.01>
      <div class="paneldesc" style="height:29px; border-top: 1px solid #EEE; padding-top: 10px">Время восстановления легких случаев<br></div>
      <div class="slidertext">{D_recovery_mild} дней</div>
      <input class="range" type=range bind:value={D_recovery_mild} min={0.5} max=100 step=0.01>
    </div>

    <div class="column">
      <div class="paneltitle">Статистика госпитализации</div>
      <div class="paneldesc" style="height:30px">Доля госпитализаций<br></div>
      <div class="slidertext">{(P_SEVERE*100).toFixed(2)} %</div>
      <input class="range" style="margin-bottom: 8px"type=range bind:value={P_SEVERE} min={0} max=1 step=0.0001>      
      <div class="paneldesc" style="height:29px; border-top: 1px solid #EEE; padding-top: 10px">Время госпитализации<br></div>
      <div class="slidertext">{D_hospital_lag} дней</div>
      <input class="range" type=range bind:value={D_hospital_lag} min={0.5} max=100 step=0.01>
    </div>

  </div>
</div>

<div style="position: relative; height: 12px"></div>

<p class = "center">
	Калькулятор реализовал Gabriel Goh, он занимается исследованиями в области машинного обучения, его <a href="http://gabgoh.github.io/">сайт</a>. Исходный код оригинальной версии калькулятора <a href="https://github.com/gabgoh/epcalc">здесь</a>. Данный калькулятор реализован на основе модели <a href="https://en.wikipedia.org/wiki/Compartmental_models_in_epidemiology#The_SEIR_model">SEIR (Susceptible → Exposed → Infected → Removed)</a>. В этой модели предполагается, что инфицированные люди после болезни приобретают иммунитет.
</p>

<p class = "center">
	Меня зовут Михаил, я перевел калькулятор на русский язык, и немного расширил функционал. Вопросы <a href="https://facebook.com/goldenalfer">сюда</a>. Исходный код моей версии калькулятора <a href="https://github.com/goldenalfer/epcalc">здесь</a>.
</p>

<p class = "center">С помощью данного калькулятор можно смоделировать распространение вируса COVID-19 с учетом введения карантина, а также изменения условий карантина.</p>

<p class = "center">
	<b> Acknowledgements </b><br>
	<a href = "https://enkimute.github.io/">Steven De Keninck</a> for RK4 Integrator. <a href="https://twitter.com/ch402">Chris Olah</a>, <a href="https://twitter.com/shancarter">Shan Carter
	</a> and <a href="https://twitter.com/ludwigschubert">Ludwig Schubert
	</a> wonderful feedback. <a href="https://twitter.com/NikitaJer">Nikita Jerschov</a> for improving clarity of text. Charie Huang for context and discussion.
</p>

<!-- Input data -->
<div style="margin-bottom: 30px">

  <div class="center" style="padding: 10px; margin-top: 3px; width: 925px">
    <div class="legendtext">Ссылка на калькулятор с вашими параметрами:</div>
    <form>
      <textarea type="textarea" rows="1" cols="5000" style="white-space: nowrap;  overflow: auto; width:100%; text-align: left" id="fname" name="fname">{state}</textarea>
    </form>
  </div>
</div>
