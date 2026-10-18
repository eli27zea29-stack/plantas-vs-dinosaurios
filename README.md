GitHub Copilot Chat Assistant

Pega esto en un archivo index.html y ábrelo en tu editor/navegador. Incluye el grid (casillas) y todos los modelos (lanzador, girasol, hielaguisantes, nuez), además de lógica de spawn, disparos y dinos que comen plantas. Si no ves las casillas, abre la consola (F12 → Console) y pega aquí los errores.

<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8" />
<title>Plants vs Dinos - Versión estable</title>
<style>
  body { background: #2f3640; color: #f5f6fa; font-family: Arial, sans-serif; padding: 16px; }
  h1 { margin: 0 0 10px 0; }
  #hud { margin-bottom: 8px; }
  #panel-superior { display:flex; gap:8px; align-items:center; flex-wrap:wrap; margin-bottom:8px; }
  .carta { background:#3d3d3d; color:#fff; padding:8px 10px; border-radius:6px; cursor:pointer; user-select:none; min-width:100px; text-align:center; }
  .seleccionada { box-shadow:0 0 0 2px rgba(255,255,255,0.12) inset; border:2px solid #fff; }
  #gameCanvas { background:#1e8f57; border-radius:6px; display:block; }
  .nota { font-size:12px; color:#dcdde1; margin-left:8px; }
</style>
</head>
<body>
<h1>Plants vs Dinos</h1>

<div id="hud">
  Sol: <span id="puntos">150</span> &nbsp;|&nbsp; Nivel: <span id="nivelValor">1</span>
</div>

<div id="panel-superior">
  <div id="cartaLanzador" class="carta seleccionada" onclick="seleccionarCarta('lanzador')"><strong>Helecho</strong><div>50 Sol</div></div>
  <div id="cartaGirasol" class="carta" onclick="seleccionarCarta('girasol')"><strong>Girasol</strong><div>25 Sol</div></div>
  <div id="cartaHielo" class="carta" onclick="seleccionarCarta('hielo')"><strong>Hielaguisantes</strong><div>75 Sol</div></div>
  <div id="cartaNuez" class="carta" onclick="seleccionarCarta('nuez')"><strong>Nuez</strong><div>75 Sol</div></div>
  <div class="nota">Coloca una planta sobre otra para fusionarlas. Los dinos comen plantas.</div>
</div>

<canvas id="gameCanvas" width="800" height="400"></canvas>

<script>
document.addEventListener('DOMContentLoaded', () => {
  // Canvas + contexto
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');

  // Grid
  const COLS = 8;
  const ROWS = 4;
  const CELL_SIZE = 80;

  // Costs
  const COSTS = { lanzador: 50, girasol: 25, hielo: 75, nuez: 75 };

  // Game state
  let puntos = 150;
  let cartaSeleccionada = 'lanzador';
  let plantas = [];
  let dinosaurios = [];
  let proyectiles = [];

  // Level & spawn
  let nivel = 1;
  let spawnInterval = 5000;
  let nextSpawnTime = Date.now() + spawnInterval;
  let levelDurationMs = 20000;
  let nextLevelTime = Date.now() + levelDurationMs;

  // Timings (frames)
  const GIRASOL_SUN_FRAMES = 300;
  const HIELO_SUN_FRAMES = 400;
  const LANZADOR_FIRE_FRAMES = 70;
  const FUSION_FIRE_FRAMES = 35;

  // UI references
  const puntosEl = document.getElementById('puntos');
  const nivelEl = document.getElementById('nivelValor');

  // Expose selector for HTML onclicks
  window.seleccionarCarta = function(tipo) {
    cartaSeleccionada = tipo;
    document.getElementById('cartaLanzador').classList.remove('seleccionada');
    document.getElementById('cartaGirasol').classList.remove('seleccionada');
    document.getElementById('cartaHielo').classList.remove('seleccionada');
    document.getElementById('cartaNuez').classList.remove('seleccionada');
    const el = document.getElementById('carta' + tipo[0].toUpperCase() + tipo.slice(1));
    if (el) el.classList.add('seleccionada');
  };

  function updatePuntos() { puntosEl.textContent = puntos; }
  function updateNivel() { nivelEl.textContent = nivel; }

  // Helpers
  function roundRectPath(x,y,w,h,r){
    ctx.beginPath();
    ctx.moveTo(x+r,y);
    ctx.lineTo(x+w-r,y);
    ctx.quadraticCurveTo(x+w,y,x+w,y+r);
    ctx.lineTo(x+w,y+h-r);
    ctx.quadraticCurveTo(x+w,y+h,x+w-r,y+h);
    ctx.lineTo(x+r,y+h);
    ctx.quadraticCurveTo(x,y+h,x,y+h-r);
    ctx.lineTo(x,y+r);
    ctx.quadraticCurveTo(x,y,x+r,y);
    ctx.closePath();
  }

  // Classes
  class Planta {
    constructor(col,row,tipo){
      this.col = col; this.row = row;
      this.x = col * CELL_SIZE; this.y = row * CELL_SIZE;
      this.tipo = tipo;
      this.timerDisparo = 0;
      this.timerSol = 0;
      this.maxVida = this.getBaseVida();
      this.vida = this.maxVida;
    }
    getBaseVida(){
      if(this.tipo==='lanzador') return 100;
      if(this.tipo==='girasol') return 60;
      if(this.tipo==='hielo') return 80;
      if(this.tipo==='nuez') return 300;
      if(this.tipo==='fusion') return 180;
      return 80;
    }
    drawHealthBar(){
      const bx = this.x + 6, by = this.y + 6, bw = CELL_SIZE - 12, bh = 6;
      ctx.fillStyle = '#000'; ctx.fillRect(bx,by,bw,bh);
      const frac = Math.max(0, this.vida / this.maxVida);
      ctx.fillStyle = '#2ecc71'; ctx.fillRect(bx,by,bw*frac,bh);
      ctx.strokeStyle = 'rgba(0,0,0,0.35)'; ctx.strokeRect(bx,by,bw,bh);
    }
    // draw models
    drawLanzador(){
      const cx = this.x + CELL_SIZE/2, cy = this.y + CELL_SIZE/2;
      ctx.fillStyle='#81ecec';
      ctx.beginPath(); ctx.ellipse(cx-18,cy+24,16,8,-0.2,0,Math.PI*2); ctx.ellipse(cx+18,cy+24,16,8,0.2,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#00b894'; ctx.lineWidth=2; ctx.stroke();
      ctx.strokeStyle='#55efc4'; ctx.lineWidth=6; ctx.lineCap='round';
      ctx.beginPath(); ctx.moveTo(cx,cy+20); ctx.quadraticCurveTo(cx-4,cy+2,cx,cy-6); ctx.stroke();
      ctx.fillStyle='#55efc4'; ctx.beginPath(); ctx.arc(cx,cy-8,20,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#00b894'; ctx.lineWidth=2; ctx.stroke();
      ctx.strokeStyle='#55efc4'; ctx.lineWidth=4;
      ctx.beginPath(); ctx.moveTo(cx-12,cy-20); ctx.quadraticCurveTo(cx-22,cy-30,cx-26,cy-24); ctx.stroke();
      ctx.fillStyle='#00b894'; roundRectPath(cx+10,cy-14,16,12,4); ctx.fill();
      ctx.fillStyle='#2d3436'; ctx.fillRect(cx+22,cy-12,4,8);
      ctx.fillStyle='#2d3436'; ctx.beginPath(); ctx.arc(cx-2,cy-9,4.5,0,Math.PI*2); ctx.fill();
      ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(cx-3,cy-10,1.5,0,Math.PI*2); ctx.fill();
    }
    drawGirasol(){
      const cx = this.x + CELL_SIZE/2, cy = this.y + CELL_SIZE/2;
      ctx.fillStyle='#81ecec';
      ctx.beginPath(); ctx.ellipse(cx-18,cy+24,16,8,-0.2,0,Math.PI*2); ctx.ellipse(cx+18,cy+24,16,8,0.2,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#00b894'; ctx.lineWidth=2; ctx.stroke();
      ctx.strokeStyle='#55efc4'; ctx.lineWidth=6; ctx.lineCap='round';
      ctx.beginPath(); ctx.moveTo(cx,cy+20); ctx.quadraticCurveTo(cx-4,cy+2,cx,cy-6); ctx.stroke();
      ctx.fillStyle = '#ffeaa7';
      for(let i=0;i<8;i++){
        const ang = (i*Math.PI)/4;
        const px = cx + Math.cos(ang)*14;
        const py = cy - 8 + Math.sin(ang)*14;
        ctx.beginPath(); ctx.arc(px,py,9,0,Math.PI*2); ctx.fill();
      }
      ctx.fillStyle='#b2bec3'; ctx.beginPath(); ctx.arc(cx,cy-8,16,0,Math.PI*2); ctx.fill(); ctx.strokeStyle='#636e72'; ctx.lineWidth=2; ctx.stroke();
      ctx.fillStyle='#2d3436'; ctx.beginPath(); ctx.arc(cx-5,cy-10,3,0,Math.PI*2); ctx.arc(cx+5,cy-8,3,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#2d3436'; ctx.lineWidth=2.5; ctx.lineCap='round'; ctx.beginPath(); ctx.arc(cx,cy-9,9,0.1*Math.PI,0.9*Math.PI); ctx.stroke();
    }
    drawHielo(){
      const cx = this.x + CELL_SIZE/2, cy = this.y + CELL_SIZE/2;
      ctx.fillStyle='#81ecec';
      ctx.beginPath(); ctx.ellipse(cx-18,cy+24,16,8,-0.2,0,Math.PI*2); ctx.ellipse(cx+18,cy+24,16,8,0.2,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#00b894'; ctx.lineWidth=2; ctx.stroke();
      ctx.strokeStyle='#55efc4'; ctx.lineWidth=6; ctx.lineCap='round';
      ctx.beginPath(); ctx.moveTo(cx,cy+20); ctx.quadraticCurveTo(cx-4,cy+2,cx,cy-6); ctx.stroke();
      ctx.fillStyle='#74b9ff'; ctx.beginPath(); ctx.arc(cx,cy-8,20,0,Math.PI*2); ctx.fill(); ctx.strokeStyle='#0984e3'; ctx.lineWidth=2; ctx.stroke();
      ctx.fillStyle='#0984e3'; ctx.beginPath(); ctx.moveTo(cx-10,cy-24); ctx.lineTo(cx-28,cy-28); ctx.lineTo(cx-16,cy-16); ctx.lineTo(cx-30,cy-8); ctx.lineTo(cx-12,cy+2); ctx.closePath(); ctx.fill();
      ctx.fillStyle='#0984e3'; ctx.fillRect(cx+10,cy-14,16,12);
      ctx.fillStyle='#2d3436'; ctx.fillRect(cx+22,cy-12,4,8);
      ctx.fillStyle='#2d3436'; ctx.beginPath(); ctx.arc(cx-2,cy-9,4.5,0,Math.PI*2); ctx.fill();
      ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(cx-3,cy-10,1.5,0,Math.PI*2); ctx.fill();
    }
    drawNuez(){
      const cx = this.x + CELL_SIZE/2, cy = this.y + CELL_SIZE/2;
      const w=26, h=36, x=cx-w/2, y=cy-h/2+2, r=12;
      ctx.fillStyle='#a39182'; ctx.strokeStyle='#6e5d50'; ctx.lineWidth=3;
      ctx.beginPath(); ctx.moveTo(x+r,y); ctx.lineTo(x+w-r,y); ctx.quadraticCurveTo(x+w,y,x+w,y+r); ctx.lineTo(x+w,y+h-r); ctx.quadraticCurveTo(x+w,y+h,x+w-r,y+h); ctx.lineTo(x+r,y+h); ctx.quadraticCurveTo(x,y+h,x,y+h-r); ctx.lineTo(x,y+r); ctx.quadraticCurveTo(x,y,x+r,y); ctx.closePath(); ctx.fill(); ctx.stroke();
      ctx.fillStyle='#2d3436'; ctx.beginPath(); ctx.arc(cx-6,cy-6,4.5,0,Math.PI*2); ctx.fill(); ctx.beginPath(); ctx.arc(cx+6,cy-4,4.5,0,Math.PI*2); ctx.fill();
      ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(cx-7,cy-7,1.5,0,Math.PI*2); ctx.arc(cx+5,cy-5,1.5,0,Math.PI*2); ctx.fill();
    }
    draw(){
      this.drawHealthBar();
      if(this.tipo==='lanzador') this.drawLanzador();
      else if(this.tipo==='girasol') this.drawGirasol();
      else if(this.tipo==='hielo') this.drawHielo();
      else if(this.tipo==='nuez') this.drawNuez();
      else if(this.tipo==='fusion'){ // simple visual
        const cx = this.x + CELL_SIZE/2, cy = this.y + CELL_SIZE/2;
        ctx.fillStyle='#8854d0'; ctx.fillRect(cx-14,cy+8,28,14);
        ctx.fillStyle='#fffa65'; ctx.beginPath(); ctx.arc(cx,cy-2,18,0,Math.PI*2); ctx.fill();
      }
    }
    update(){
      if(this.tipo==='lanzador'){
        this.timerDisparo++;
        if(this.timerDisparo>=LANZADOR_FIRE_FRAMES){
          proyectiles.push(new Proyectil(this.x+CELL_SIZE, this.y+35, 4, false, false, this.row));
          this.timerDisparo=0;
        }
      } else if(this.tipo==='girasol'){
        this.timerSol++;
        if(this.timerSol>=GIRASOL_SUN_FRAMES){ puntos += 25; updatePuntos(); this.timerSol=0; }
      } else if(this.tipo==='hielo'){
        this.timerDisparo++;
        if(this.timerDisparo>=90){ proyectiles.push(new Proyectil(this.x+CELL_SIZE, this.y+35, 3, false, true, this.row)); this.timerDisparo=0; }
        this.timerSol++;
        if(this.timerSol>=HIELO_SUN_FRAMES){ puntos += 10; updatePuntos(); this.timerSol=0; }
      }
    }
  }

  class Dinosaurio {
    constructor(row, nivelActual=1){
      this.row=row; this.x=canvas.width; this.y=row*CELL_SIZE;
      this.width=60; this.height=CELL_SIZE;
      this.baseVel = 0.6 + (nivelActual-1)*0.08;
      this.vel = this.baseVel; this.vida = 80 + (nivelActual-1)*15;
      this.slow = 1; this.slowTimer = 0;
      this.attacking = false; this.targetPlant = null;
      this.attackTimer = 0; this.attackInterval = 30; this.attackDamage = 10;
    }
    draw(){
      const cx = this.x + 30, cy = this.y + 40;
      ctx.fillStyle='#e84118'; ctx.beginPath(); ctx.ellipse(cx,cy,22,14,0,0,Math.PI*2); ctx.fill();
      ctx.fillStyle='#c23616'; ctx.beginPath(); ctx.arc(cx+20,cy-10,10,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle='#e84118'; ctx.lineWidth=5; ctx.beginPath(); ctx.moveTo(cx-10,cy); ctx.lineTo(cx-25,cy-8); ctx.stroke();
      ctx.fillStyle='#fbc531'; ctx.fillRect(cx+22,cy-13,3,3);
      ctx.fillStyle='black'; ctx.fillRect(this.x,this.y+2,this.width,6);
      const maxVida = 80 + (nivel-1)*15; ctx.fillStyle='#2ecc71'; ctx.fillRect(this.x,this.y+2, Math.max(0,(this.vida/maxVida))*this.width, 6);
      if(this.attacking) { ctx.fillStyle='rgba(0,0,0,0.15)'; ctx.fillRect(this.x,this.y,this.width,this.height); }
    }
    update(){
      if(this.slowTimer>0){ this.slowTimer--; if(this.slowTimer<=0) this.slow=1; }
      this.vel = this.baseVel * this.slow;
      if(this.attacking && this.targetPlant){
        this.attackTimer++;
        if(this.attackTimer>=this.attackInterval){
          this.attackTimer=0;
          this.targetPlant.vida -= this.attackDamage;
          if(this.targetPlant.vida <= 0){
            const idx = plantas.indexOf(this.targetPlant);
            if(idx!==-1) plantas.splice(idx,1);
            this.targetPlant = null; this.attacking = false;
          }
        }
      } else {
        this.x -= this.vel;
      }
    }
  }

  class Proyectil {
    constructor(x,y,velocidad,esFusion,esHielo,row){
      this.x=x; this.y=y; this.velocidad=velocidad; this.esFusion=esFusion; this.esHielo=esHielo; this.row=row;
      this.radius = esFusion?7:(esHielo?6:5);
    }
    draw(){
      if(this.esFusion) ctx.fillStyle='#fffa65';
      else if(this.esHielo) ctx.fillStyle='#74b9ff';
      else ctx.fillStyle='#4cd137';
      ctx.beginPath(); ctx.arc(this.x,this.y,this.radius,0,Math.PI*2); ctx.fill();
      if(this.esHielo){ ctx.strokeStyle='rgba(255,255,255,0.6)'; ctx.beginPath(); ctx.arc(this.x,this.y,this.radius+1.5,0,Math.PI*2); ctx.stroke(); }
    }
    update(){ this.x += this.velocidad; }
  }

  // spawn dinos by time
  function trySpawnDino(){
    if(Date.now() >= nextSpawnTime){
      const fila = Math.floor(Math.random()*ROWS);
      dinosaurios.push(new Dinosaurio(fila, nivel));
      nextSpawnTime = Date.now() + spawnInterval;
    }
  }

  // click to place plant / fusion
  canvas.addEventListener('click', (e) => {
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left;
    const my = e.clientY - rect.top;
    const col = Math.floor(mx / CELL_SIZE);
    const row = Math.floor(my / CELL_SIZE);
    if(col<0||col>=COLS||row<0||row>=ROWS) return;
    const cost = COSTS[cartaSeleccionada] || 50;
    if(puntos < cost) return;
    const existing = plantas.find(p => p.col===col && p.row===row);
    if(existing){
      if(existing.tipo !== 'fusion'){
        existing.tipo = 'fusion';
        existing.maxVida = existing.getBaseVida()*2;
        existing.vida = existing.maxVida;
        puntos -= cost; updatePuntos();
      }
    } else {
      plantas.push(new Planta(col,row,cartaSeleccionada));
      puntos -= cost; updatePuntos();
    }
  });

  // Main loop
  function gameLoop(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // draw grid (each frame)
    for(let r=0;r<ROWS;r++){
      for(let c=0;c<COLS;c++){
        ctx.fillStyle = (r+c)%2===0 ? '#26af5f' : '#229e55';
        ctx.fillRect(c*CELL_SIZE, r*CELL_SIZE, CELL_SIZE, CELL_SIZE);
        ctx.strokeStyle = 'rgba(0,0,0,0.15)';
        ctx.strokeRect(c*CELL_SIZE, r*CELL_SIZE, CELL_SIZE, CELL_SIZE);
      }
    }

    // plants update/draw
    for(let i=0;i<plantas.length;i++){ plantas[i].update(); plantas[i].draw(); }

    // projectiles update & collisions
    for(let i=proyectiles.length-1;i>=0;i--){
      const pr = proyectiles[i];
      pr.update(); pr.draw();
      // check collision with dinos
      for(let d=dinosaurios.length-1; d>=0; d--){
        const din = dinosaurios[d];
        if(pr.row === din.row && pr.x >= din.x && pr.x <= din.x + din.width && pr.y >= din.y && pr.y <= din.y + din.height){
          const daño = pr.esFusion ? 40 : (pr.esHielo ? 15 : 25);
          din.vida -= daño;
          if(pr.esHielo){ din.slow = 0.45; din.slowTimer = 240; }
          proyectiles.splice(i,1);
          break;
        }
      }
      if(proyectiles[i] && proyectiles[i].x > canvas.width + 20) proyectiles.splice(i,1);
    }

    // spawn dinos
    trySpawnDino();

    // dinos behavior: check plants in same row and attack
    for(let i=dinosaurios.length-1;i>=0;i--){
      const d = dinosaurios[i];
      if(!d.attacking){
        // check overlapping plant in front
        let target = null;
        for(const p of plantas){
          if(p.row === d.row){
            // if d is overlapping plant cell horizontally
            if(d.x <= p.x + CELL_SIZE - 10 && d.x + d.width >= p.x + 10){
              target = p; break;
            }
          }
        }
        if(target){ d.attacking = true; d.targetPlant = target; d.attackTimer = 0; }
      } else {
        if(!d.targetPlant || !plantas.includes(d.targetPlant)){ d.attacking = false; d.targetPlant = null; }
      }

      d.update();
      d.draw();

      // if d dead
      if(d.vida <= 0){ dinosaurios.splice(i,1); puntos += 20; updatePuntos(); continue; }
      // if d reached left edge
      if(d.x + d.width < 0){ dinosaurios.splice(i,1); puntos -= 25; updatePuntos(); continue; }
    }

    // keep attacking dinos positioned at plant front to avoid overlap
    for(const d of dinosaurios){
      if(d.attacking && d.targetPlant){
        const desiredX = d.targetPlant.x + CELL_SIZE - d.width + 6;
        if(d.x > desiredX) d.x = Math.max(desiredX, d.x - d.vel);
      }
    }

    // level up
    if(Date.now() >= nextLevelTime){
      nivel++; updateNivel();
      spawnInterval = Math.max(1000, spawnInterval - 600);
      nextLevelTime = Date.now() + levelDurationMs;
      puntos += 50; updatePuntos();
    }

    requestAnimationFrame(gameLoop);
  }

  // Start
  updatePuntos(); updateNivel();
  requestAnimationFrame(gameLoop);
});
</script>
</body>
</html>

Pruebas rápidas si sigue sin verse el grid:
- Guarda exactamente como index.html y abre en navegador (no en el preview del editor que haga transformaciones).
- Abre la consola (F12 → Console). Si hay errores rojos, copia y pégamelos.
- Fuerza recarga (Ctrl+F5) o abre en una ventana de incógnito.
- Prueba en otro navegador (Chrome/Edge/Firefox).

Dime: (1) si el grid aparece ahora, o (2) qué error ves en la consola y en qué navegador lo probaste — lo corrijo inmediatamente.
