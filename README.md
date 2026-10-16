<strong>Helecho</strong>
        <span>50 Sol</span>
    </div>
    <div id="cartaGirasol" class="carta" onclick="seleccionarCarta('girasol')">
        <strong>Girasol</strong>
        <span>25 Sol</span>
    </div>
    <div style="font-size: 12px; margin-top: 6px; color: #dcdde1;">
        💡 Coloca una planta sobre otra para **fusionarlas**.
    </div>
</div>
<br>

<canvas id="gameCanvas" width="800" height="400"></canvas>

<script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    let puntos = 150;
    let cartaSeleccionada = "lanzador"; // 'lanzador' o 'girasol'

    let plantas = [];
    let dinosaurios = [];
    let proyectiles = [];

    const COLS = 8;
    const ROWS = 4;
    const CELL_SIZE = 80;

    function seleccionarCarta(tipo) {
        cartaSeleccionada = tipo;
        document.getElementById("cartaLanzador").classList.remove("seleccionada");
        document.getElementById("cartaGirasol").classList.remove("seleccionada");
        if (tipo === 'lanzador') {
            document.getElementById("cartaLanzador").classList.add("seleccionada");
        } else {
            document.getElementById("cartaGirasol").classList.add("seleccionada");
        }
    }

    class Planta {
        constructor(col, row, tipo) {
            this.col = col;
            this.row = row;
            this.x = col * CELL_SIZE;
            this.y = row * CELL_SIZE;
            this.tipo = tipo; // 'lanzador', 'girasol', 'fusion'
            this.timerDisparo = 0;
            this.timerSol = 0;
        }

        draw() {
            let cx = this.x + CELL_SIZE / 2;
            let cy = this.y + CELL_SIZE / 2;

            if (this.tipo === 'girasol') {
                ctx.fillStyle = "#c23616"; // Maceta
                ctx.fillRect(cx - 12, cy + 8, 24, 14);
                ctx.fillStyle = "#fbc531"; // Pétalos
                for(let i = 0; i < 8; i++) {
                    let ang = (Math.PI / 4) * i;
                    ctx.beginPath();
                    ctx.arc(cx + Math.cos(ang) * 12, cy - 4 + Math.sin(ang) * 12, 7, 0, Math.PI * 2);
                    ctx.fill();
                }
                ctx.fillStyle = "#7158e2"; // Centro
                ctx.beginPath();
                ctx.arc(cx, cy - 4, 8, 0, Math.PI * 2);
                ctx.fill();

            } else if (this.tipo === 'lanzador') {
                ctx.fillStyle = "#c23616";
                ctx.fillRect(cx - 12, cy + 8, 24, 14);
                ctx.fillStyle = "#4cd137"; // Cuerpo planta
                ctx.beginPath();
                ctx.arc(cx, cy - 2, 16, 0, Math.PI * 2);
                ctx.fill();
                ctx.fillStyle = "#009432"; // Cañón
                ctx.fillRect(cx, cy - 8, 16, 10);

            } else if (this.tipo === 'fusion') {
                ctx.fillStyle = "#8854d0"; // Maceta especial
                ctx.fillRect(cx - 14, cy + 8, 28, 14);
                ctx.fillStyle = "#fffa65"; // Aura
                ctx.beginPath();
                ctx.arc(cx, cy - 2, 18, 0, Math.PI * 2);
                ctx.fill();
                // Doble cañón
                ctx.fillStyle = "#e84118";
                ctx.fillRect(cx, cy - 12, 18, 7);
                ctx.fillRect(cx, cy + 1, 18, 7);
            }
        }

        update() {
            if (this.tipo === 'lanzador') {
                this.timerDisparo++;
                if (this.timerDisparo >= 70) {
                    proyectiles.push(new Proyectil(this.x + CELL_SIZE, this.y + 35, 4, false, this.row));
                    this.timerDisparo = 0;
                }
            } else if (this.tipo === 'girasol') {
                this.timerSol++;
                if (this.timerSol >= 300) {
                    puntos += 25;
                    document.getElementById("puntos").innerText = puntos;
                    this.timerSol = 0;
                }
            } else if (this.tipo === 'fusion') {
                this.timerDisparo++;
                if (this.timerDisparo >= 35) {
                    proyectiles.push(new Proyectil(this.x + CELL_SIZE, this.y + 25, 6, true, this.row));
                    proyectiles.push(new Proyectil(this.x + CELL_SIZE, this.y + 45, 6, true, this.row));
                    
