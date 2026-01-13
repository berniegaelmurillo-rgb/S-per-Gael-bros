<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Super Gael Bros - 1984 Edition</title>
    <style>
        /* Fuente estilo Retro */
        @font-face {
            font-family: 'Retro';
            src: url('https://fonts.cdnfonts.com/s/14352/PressStart2P-Regular.woff') format('woff');
        }

        body {
            background-color: #111;
            margin: 0;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            font-family: 'Press Start 2P', cursive, sans-serif;
            color: white;
            touch-action: none;
        }

        #game-container {
            position: relative;
            width: 512px;
            height: 448px;
            image-rendering: pixelated;
            border: 6px solid #333;
            background: #5c94fc; /* Color inicial del cielo */
            transition: background 1s;
        }

        canvas {
            width: 100%;
            height: 100%;
            display: block;
        }

        #ui {
            position: absolute;
            top: 15px;
            width: 100%;
            display: flex;
            justify-content: space-around;
            font-size: 10px;
            text-shadow: 2px 2px #000;
            pointer-events: none;
            z-index: 10;
        }

        #controls {
            display: flex;
            justify-content: space-between;
            width: 100%;
            max-width: 512px;
            margin-top: 25px;
            padding: 0 20px;
            box-sizing: border-box;
        }

        .btn-group {
            display: flex;
            gap: 15px;
        }

        .btn {
            width: 70px;
            height: 70px;
            background: #444;
            border: 4px solid #000;
            border-radius: 12px;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 24px;
            user-select: none;
            cursor: pointer;
            -webkit-tap-highlight-color: transparent;
            box-shadow: 0 4px 0 #111;
        }

        .btn:active {
            transform: translateY(4px);
            box-shadow: none;
            background: #666;
        }

        .btn-jump {
            width: 90px;
            height: 90px;
            background: #a00;
            border-radius: 50%;
            border-color: #500;
            box-shadow: 0 4px 0 #400;
        }

        .btn-jump:active {
            background: #f00;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="ui">
            <div>GAEL<br><span id="score">000000</span></div>
            <div>MUNDO<br><span id="world-num">1-1</span></div>
            <div>DIST<br><span id="dist">0</span>m</div>
        </div>
        <canvas id="gameCanvas" width="256" height="224"></canvas>
    </div>

    <div id="controls">
        <div class="btn-group">
            <div class="btn" id="left">←</div>
            <div class="btn" id="right">→</div>
        </div>
        <div class="btn btn-jump" id="jump">A</div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Definición de Mundos
        const worlds = [
            { name: "1-1", bg: "#5c94fc", ground: "#943a00", top: "#ff9452", brick: "#943a00" }, // Cielo
            { name: "1-2", bg: "#000000", ground: "#0000aa", top: "#5c94fc", brick: "#444444" }, // Subterráneo
            { name: "1-3", bg: "#220022", ground: "#440044", top: "#ff52ff", brick: "#220022" }  // Noche / Castillo
        ];

        let currentW = 0;

        // Dibujado de Sprites Pixel Art 1984
        const sprites = {
            ground: (x, y, wIdx) => {
                const w = worlds[wIdx];
                ctx.fillStyle = w.ground; ctx.fillRect(x, y, 16, 16);
                ctx.fillStyle = w.top; ctx.fillRect(x, y, 16, 1); ctx.fillRect(x, y, 1, 16);
                ctx.fillStyle = '#000'; ctx.fillRect(x+15, y, 1, 16); ctx.fillRect(x, y+15, 16, 1);
            },
            brick: (x, y, wIdx) => {
                const w = worlds[wIdx];
                ctx.fillStyle = w.brick; ctx.fillRect(x, y, 16, 16);
                ctx.fillStyle = '#000'; ctx.fillRect(x, y+7, 16, 1); ctx.fillRect(x+8, y, 1, 7);
                ctx.fillStyle = w.top; ctx.fillRect(x, y, 16, 1);
            },
            q: (x, y, hit) => {
                ctx.fillStyle = hit ? '#777' : '#ff9452'; ctx.fillRect(x, y, 16, 16);
                if(!hit) {
                    ctx.fillStyle = '#fff'; ctx.font = '8px monospace';
                    ctx.fillText('?', x+5, y+11);
                }
            },
            hero: (x, y, shirt, dir, frame, name) => {
                const walk = Math.floor(frame/8) % 2;
                // Pelo
                ctx.fillStyle = '#402000'; ctx.fillRect(x+2, y, 5, 2);
                // Cara
                ctx.fillStyle = '#ffccaa'; ctx.fillRect(x+2, y+2, 5, 4);
                // Camisa
                ctx.fillStyle = shirt; ctx.fillRect(x+2, y+6, 5, 8);
                // Pantalón / Pies
                ctx.fillStyle = '#000';
                ctx.fillRect(x + (walk ? 1 : 2), y+14, 2, 2);
                ctx.fillRect(x + (walk ? 5 : 4), y+14, 2, 2);
                // Nombre
                if(name) {
                    ctx.fillStyle = 'white'; ctx.font = '4px monospace';
                    ctx.fillText(name, x-2, y-2);
                }
            }
        };

        // Estado
        let player = { x: 40, y: 150, vx: 0, vy: 0, dir: 1, friends: [] };
        let cameraX = 0;
        let score = 0;
        let frame = 0;
        let worldData = [];
        let nextX = 0;
        let keys = {};

        function addSegment() {
            let r = Math.random();
            if(r < 0.6) {
                worldData.push({ x: nextX, y: 192, w: 128, type: 'floor' });
                if(Math.random() > 0.8) worldData.push({ x: nextX + 64, y: 128, type: 'q', item: 'IAN' });
                nextX += 128;
            } else if(r < 0.8) {
                worldData.push({ x: nextX, y: 192, w: 64, type: 'floor' });
                nextX += 128; // Hueco
            } else {
                worldData.push({ x: nextX, y: 192, w: 128, type: 'floor' });
                worldData.push({ x: nextX + 32, y: 144, type: 'brick' });
                worldData.push({ x: nextX + 48, y: 144, type: 'q', item: 'JOSEP' });
                worldData.push({ x: nextX + 64, y: 144, type: 'brick' });
                nextX += 128;
            }
        }

        for(let i=0; i<6; i++) addSegment();

        function update() {
            frame++;
            if(keys.left) { player.vx = -2.5; player.dir = -1; }
            else if(keys.right) { player.vx = 2.5; player.dir = 1; }
            else { player.vx *= 0.8; }

            player.vy += 0.45;
            player.x += player.vx;
            player.y += player.vy;

            // Lógica de mundos
            let dist = Math.floor(player.x / 10);
            if(dist > 150 && currentW === 0) currentW = 1;
            if(dist > 350 && currentW === 1) currentW = 2;

            let grounded = false;
            worldData.forEach(obj => {
                if(obj.type === 'floor') {
                    if(player.x+8 > obj.x && player.x < obj.x+obj.w && player.y+16 > obj.y && player.y+16 < obj.y+10) {
                        player.y = obj.y-16; player.vy = 0; grounded = true;
                    }
                }
                if(obj.type === 'q' || obj.type === 'brick') {
                    if(player.vy < 0 && player.x+8 > obj.x && player.x < obj.x+16 && player.y < obj.y+16 && player.y > obj.y+8) {
                        player.vy = 2;
                        if(obj.type === 'q' && !obj.hit) {
                            obj.hit = true; score += 100;
                            player.friends.push({ name: obj.item, color: obj.item === 'IAN' ? '#00f' : '#0f0' });
                        }
                    }
                }
            });

            if(keys.jump && grounded) player.vy = -8.5;
            if(player.x > nextX - 400) addSegment();

            cameraX = Math.max(cameraX, player.x - 80);

            // Reinicio si cae
            if(player.y > 224) {
                player.x = cameraX + 10; player.y = 100; player.vy = 0;
                player.friends = [];
                score = Math.max(0, score - 200);
            }

            // UI
            document.getElementById('score').innerText = score.toString().padStart(6, '0');
            document.getElementById('dist').innerText = dist;
            document.getElementById('world-num').innerText = worlds[currentW].name;
            document.getElementById('game-container').style.background = worlds[currentW].bg;
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.save();
            ctx.translate(-cameraX, 0);

            worldData.forEach(obj => {
                if(obj.type === 'floor') {
                    for(let i=0; i<obj.w; i+=16) sprites.ground(obj.x + i, obj.y, currentW);
                } else if(obj.type === 'brick') {
                    sprites.brick(obj.x, obj.y, currentW);
                } else if(obj.type === 'q') {
                    sprites.q(obj.x, obj.y, obj.hit);
                }
            });

            player.friends.forEach((f, i) => {
                sprites.hero(player.x - (i+1)*14, player.y, f.color, player.dir, frame, f.name);
            });

            sprites.hero(player.x, player.y, '#f00', player.dir, Math.abs(player.vx)>0.2?frame:0, 'GAEL');

            ctx.restore();
        }

        function loop() {
            update();
            draw();
            requestAnimationFrame(loop);
        }

        // Controles Táctiles y Teclado
        const bind = (id, k) => {
            const el = document.getElementById(id);
            el.onmousedown = el.ontouchstart = (e) => { e.preventDefault(); keys[k] = true; };
            el.onmouseup = el.onmouseleave = el.ontouchend = (e) => { e.preventDefault(); keys[k] = false; };
        };
        bind('left', 'left'); bind('right', 'right'); bind('jump', 'jump');

        window.onkeydown = e => {
            if(e.key === 'ArrowLeft') keys.left = true;
            if(e.key === 'ArrowRight') keys.right = true;
            if(e.key === ' ' || e.key === 'ArrowUp') keys.jump = true;
        };
        window.onkeyup = e => {
            if(e.key === 'ArrowLeft') keys.left = false;
            if(e.key === 'ArrowRight') keys.right = false;
            if(e.key === ' ' || e.key === 'ArrowUp') keys.jump = false;
        };

        loop();
    </script>
</body>
</html>
