# index.html5
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>Dino Evolution - Assets Edition</title>
    <style>
        body { margin: 0; background: #1a1a1a; display: flex; justify-content: center; align-items: center; height: 100vh; overflow: hidden; color: white; font-family: 'Segoe UI', sans-serif; }
        #game-container { position: relative; box-shadow: 0 20px 50px rgba(0,0,0,0.5); background: #87CEEB; border: 4px solid #333; }
        canvas { display: block; image-rendering: pixelated; } /* 保持像素清晰 */
        .hud { position: absolute; top: 15px; left: 20px; pointer-events: none; }
        .score { font-size: 32px; font-family: 'Courier New', monospace; font-weight: bold; color: #535353; text-shadow: 1px 1px 0px white; }
        .loading { position: absolute; width: 100%; height: 100%; background: #1a1a1a; display: flex; justify-content: center; align-items: center; z-index: 10; }
    </style>
</head>
<body>

<div id="game-container">
    <div id="loading" class="loading">正在讀取 assets 圖片資源...</div>
    <div class="hud">
        <div id="score" class="score">00000</div>
        <div id="status" style="color: #444; font-size: 14px; font-weight: bold;">STATUS: INITIAL</div>
    </div>
    <canvas id="gameCanvas"></canvas>
</div>

<script>
/**
 * 資源定義 - 使用你指定的 dino.png, cactus.png, bird.png
 */
const Assets = {
    images: {},
    manifest: {
        dino: './assets/dino.png',
        cactus: './assets/cactus.png',
        bird: './assets/bird.png'
    },
    async load() {
        const loads = Object.entries(this.manifest).map(([key, src]) => {
            return new Promise(resolve => {
                const img = new Image();
                img.src = src;
                img.onload = () => { 
                    this.images[key] = img; 
                    console.log(`已載入: ${src}`);
                    resolve(true); 
                };
                img.onerror = () => {
                    console.error(`載入失敗: ${src} (請檢查檔名大小寫)`);
                    resolve(false);
                };
            });
        });
        return Promise.all(loads);
    }
};

class Dino {
    constructor() {
        this.x = 50;
        this.w = 50; this.h = 50;
        this.baseY = 200 - this.h;
        this.y = this.baseY;
        this.dy = 0;
        this.gravity = 2200;
        this.jumpForce = -750;
        this.isGrounded = true;
    }
    jump() {
        if (this.isGrounded) {
            this.dy = this.jumpForce;
            this.isGrounded = false;
        }
    }
    update(dt) {
        this.dy += this.gravity * dt;
        this.y += this.dy * dt;
        if (this.y > this.baseY) {
            this.y = this.baseY;
            this.dy = 0;
            this.isGrounded = true;
        }
    }
    draw(ctx) {
        const img = Assets.images.dino;
        if (img) {
            ctx.drawImage(img, this.x, this.y, this.w, this.h);
        } else {
            ctx.fillStyle = '#535353'; // 沒圖時畫灰色塊
            ctx.fillRect(this.x, this.y, this.w, this.h);
        }
    }
}

class Engine {
    constructor() {
        this.canvas = document.getElementById('gameCanvas');
        this.ctx = this.canvas.getContext('2d');
        this.canvas.width = 800; this.canvas.height = 250;
        
        this.score = 0;
        this.gameSpeed = 400;
        this.isGameOver = false;
        this.lastTime = performance.now();
        
        this.dino = new Dino();
        this.obstacles = [];
        this.spawnTimer = 0;

        this.initInput();
        requestAnimationFrame(t => this.loop(t));
    }

    initInput() {
        const action = () => { if(this.isGameOver) location.reload(); else this.dino.jump(); };
        window.addEventListener('keydown', e => { if(e.code === 'Space' || e.code === 'ArrowUp') action(); });
        this.canvas.addEventListener('touchstart', (e) => { e.preventDefault(); action(); });
    }

    update(dt) {
        this.score += dt * 10;
        this.gameSpeed = 400 + Math.pow(this.score, 0.6) * 6;
        this.dino.update(dt);

        this.spawnTimer += dt;
        if (this.spawnTimer > 1.8 * (400 / this.gameSpeed)) {
            const isBird = Math.random() > 0.7;
            this.obstacles.push({
                x: 800,
                y: isBird ? 100 : 160,
                w: isBird ? 50 : 30,
                h: 40,
                type: isBird ? 'bird' : 'cactus'
            });
            this.spawnTimer = 0;
        }

        this.obstacles.forEach((obs, i) => {
            obs.x -= this.gameSpeed * dt;
            if (this.checkCollision(this.dino, obs)) this.isGameOver = true;
            if (obs.x < -100) this.obstacles.splice(i, 1);
        });

        document.getElementById('score').innerText = Math.floor(this.score).toString().padStart(5, '0');
        document.getElementById('status').innerText = `EVO: ${this.score > 500 ? 'GOLD' : 'NORMAL'} | SPEED: ${Math.floor(this.gameSpeed)}`;
    }

    checkCollision(d, o) {
        return d.x + 10 < o.x + o.w && d.x + d.w - 10 > o.x && d.y + 10 < o.y + o.h && d.y + d.h - 10 > o.y;
    }

    draw() {
        this.ctx.clearRect(0, 0, 800, 250);
        
        // 畫地面線
        this.ctx.strokeStyle = '#535353';
        this.ctx.beginPath(); this.ctx.moveTo(0, 200); this.ctx.lineTo(800, 200); this.ctx.stroke();

        this.dino.draw(this.ctx);

        this.obstacles.forEach(o => {
            const img = Assets.images[o.type];
            if (img) {
                this.ctx.drawImage(img, o.x, o.y, o.w, o.h);
            } else {
                this.ctx.fillStyle = o.type === 'bird' ? '#ff00ff' : '#228B22';
                this.ctx.fillRect(o.x, o.y, o.w, o.h);
            }
        });

        if (this.isGameOver) {
            this.ctx.fillStyle = 'rgba(0,0,0,0.7)';
            this.ctx.fillRect(0,0,800,250);
            this.ctx.fillStyle = 'white';
            this.ctx.textAlign = 'center';
            this.ctx.font = '30px Courier New';
            this.ctx.fillText("EVOLUTION FAILED", 400, 110);
            this.ctx.fillText("PRESS SPACE TO RESTART", 400, 150);
        }
    }

    loop(t) {
        const dt = (t - this.lastTime) / 1000;
        this.lastTime = t;
        if (!this.isGameOver) this.update(dt);
        this.draw();
        requestAnimationFrame(t => this.loop(t));
    }
}

// 啟動引擎
Assets.load().then(results => {
    document.getElementById('loading').style.display = 'none';
    new Engine();
});
</script>
</body>
</html>
