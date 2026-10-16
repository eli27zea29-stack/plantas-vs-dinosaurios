<h1>Plants vs Dinos</h1>

<div id="hud">
    Sol: <span id="puntos">150</span>
</div>

<div id="panel-superior">
    <div id="cartaLanzador" class="carta seleccionada" onclick="seleccionarCarta('lanzador')">
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
                    this.timerDisparo = 0;
                }
                this.timerSol++;
                if (this.timerSol >= 200) {
                    puntos += 15;
                    document.getElementById("puntos").innerText = puntos;
                    this.timerSol = 0;
                }
            }
        }
    }

    class Dinosaurio {
        constructor(row) {
            this.row = row;
            this.x = canvas.width;
            this.y = row * CELL_SIZE;
            this.width = 60;
            this.height = CELL_SIZE;
            this.velocidade = 0.6;
            this.vida = 80;
        }

        draw() {
            let cx = this.x + 30;
            let cy = this.y + 40;

            ctx.fillStyle = "#e84118"; // Cuerpo
            ctx.beginPath();
            ctx.ellipse(cx, cy, 22, 14, 0, 0, Math.PI * 2);
            ctx.fill();

            ctx.fillStyle = "#c23616"; // Cabeza
            ctx.beginPath();
            ctx.arc(cx + 20, cy - 10, 10, 0, Math.PI * 2);
            ctx.fill();

            // Cola
            ctx.strokeStyle = "#e84118";
            ctx.lineWidth = 5;
            ctx.beginPath();
            ctx.moveTo(cx - 10, cy);
            ctx.lineTo(cx - 25, cy - 8);
            ctx.stroke();

            // Ojo
            ctx.fillStyle = "#fbc531";
            ctx.fillRect(cx + 22, cy - 13, 3, 3);
        }

        update() {
            this.x -= this.velocidade;
        }
    }

    class Proyectil {
        constructor(x, y, velocidad, esFusion, row) {
            this.x = x;
            this.y = y;
            this.velocidade = velocidad;
            this.esFusion = esFusion;
            this.row = row;
            this.radius = esFusion ? 7 : 5;
        }

        draw() {
            ctx.fillStyle = this.esFusion ? "#fffa65" : "#4cd137";
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fill();
        }

        update() {
            this.x += this.velocidade;
        }
    }

    // Generar dinosaurio en filas aleatorias
    setInterval(() => {
        let filaAleatoria = Math.floor(Math.random() * ROWS);
        dinosaurios.push(new Dinosaurio(filaAleatoria));
    }, 5000);

    // Clic directo y sin fallos por casillas exactas
    canvas.addEventListener("click", (e) => {
        const rect = canvas.getBoundingClientRect();
        let mouseX = e.clientX - rect.left;
        let mouseY = e.clientY - rect.top;

        let col = Math.floor(mouseX / CELL_SIZE);
        let row = Math.floor(mouseY / CELL_SIZE);

        if (col >= 0 && col < COLS && row >= 0 && row < ROWS) {
            let costo = cartaSeleccionada === 'lanzador' ? 50 : 25;

            if (puntos >= costo) {
                let plantaExistente = plantas.find(p => p.col === col && p.row === row);

                if (plantaExistente) {
                    plantaExistente.tipo = 'fusion';
                    puntos -= costo;
                    document.getElementById("puntos").innerText = puntos;
                } else {
                    plantas.push(new Planta(col, row, cartaSeleccionada));
                    puntos -= costo;
                    document.getElementById("puntos").innerText = puntos;
                }
            }
        }
    });

    // Bucle principal
    void function gameLoop() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // Dibujar tablero tipo celdas PvZ
        for (let r = 0; r < ROWS; r++) {
            for (let c = 0; c < COLS; c++) {
                ctx.fillStyle = (r + c) % 2 === 0 ? "#26af5f" : "#229e55";
                ctx.fillRect(c * CELL_SIZE, r * CELL_SIZE, CELL_SIZE, CELL_SIZE);
                
                ctx.strokeStyle = "rgba(0,0,0,0.15)";
                ctx.strokeRect(c * CELL_SIZE, r * CELL_SIZE, CELL_SIZE, CELL_SIZE);
            }
        }

        // Actualizar plantas
        plantas.forEach(planta => {
            planta.update();
            planta.draw();
        });

        // Actualizar proyectiles
        proyectiles.forEach((proyectil, index) => {
            proyectil.update();
            proyectil.draw();

            if (proyectil.x > canvas.width) {
                proyectiles.splice(index, 1);
            }
        });

        // Actualizar dinosaurios y colisiones
        dinosaurios.forEach((dino, indexDino) => {
            dino.update();
            dino.draw();

            proyectiles.forEach((proyectil, indexProyectil) => {
                if (
                    proyectil.row === dino.row &&
                    proyectil.x >= dino.x &&
                    proyectil.x <= dino.x + dino.width &&
                    proyectil.y >= dino.y &&
                    proyectil.y <= dino.y + dino.height
                ) {
                    let daño = proyectil.esFusion ? 40 : 25;
                    dino.vida -= daño;
                    proyectiles.splice(indexProyectil, 1);

                    if (dino.vida <= 0) {
                        dinosaurios.splice(indexDino, 1);
                        puntos += 20;
                        document.getElementById("puntos").innerText = puntos;
                    }
                }
            });

            if (dino.x < 0) {
                dinosaurios.splice(indexDino, 1);
                puntos -= 25;
                document.getElementById("puntos").innerText = puntos;
            }
        });

        requestAnimationFrame(gameLoop);
    }();
</script>
