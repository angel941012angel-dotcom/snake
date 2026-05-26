<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// --- 結構型別定義 (TypeScript) ---
interface Point {
    x: number;
    y: number;
}

interface Guard {
    id: number;
    x: number;
    y: number;
    alive: boolean;
    side: string;
}

interface Laser {
    guardId: number;
    x1: number;
    y1: number;
    x2: number;
    y2: number;
    cells: Point[];
    startTime: number;
    warnEndTime: number;
    expireTime: number;
}

interface Bullet {
    x: number;
    y: number;
    dx: number;
    dy: number;
}

// --- 遊戲參數配置 ---
const GRID_SIZE = 25;         
const CELL_SIZE = 20;         
const GAME_SPEED = 95;        
const BULLET_SPEED = 0.5;     

const LASER_INTERVAL = 2500;  
const LASER_WARN_TIME = 1200; 
const LASER_ACTIVE_TIME = 400;
const KILL_TIMEOUT = 30000;   

// --- HTML 元素引用 ---
const gameCanvas = ref<HTMLCanvasElement | null>(null);

// --- 響應式 UI 數據 ---
const ammo = ref<number>(10);
const timeLeftStr = ref<string>("30.0");
const remainingGuards = ref<number>(0);
const statusText = ref<string>("等待開局");
const statusColor = ref<string>("#ffcc00");
const actionHint = ref<string>("請按 【R】 鍵正式開始遊戲");
const isHighlight = ref<boolean>(true);

// --- 內部遊戲狀態變數 ---
let snake: Point[] = [];
let dir: Point = { x: 1, y: 0 };     
let nextDir: Point = { x: 1, y: 0 };
let foodList: Point[] = [];            
let guards: Guard[] = [];              
let activeLasers: Laser[] = [];        
let bullets: Bullet[] = [];             
let gameOver = false;
let gameWin = false;
let gameStarted = false; 
let canShoot = true;          

let gameInterval: any;
let laserInterval: any;
let bulletInterval: any;           
let lastKillTime = 0;             
let guardIdCounter = 0;       

// --- 靜態配置舞台 ---
function setupStage() {
    snake = [];
    for(let i = 0; i < 10; i++) {
        snake.push({ x: 13 - i, y: 12 });
    }
    dir = { x: 1, y: 0 };
    nextDir = { x: 1, y: 0 };
    ammo.value = 10;
    bullets = [];
    activeLasers = [];
    guardIdCounter = 0;
    canShoot = true;
    foodList = []; 

    guards = [];
    for(let x = 0; x < GRID_SIZE; x++) {
        for(let y = 0; y < GRID_SIZE; y++) {
            if(x === 0 || x === GRID_SIZE - 1 || y === 0 || y === GRID_SIZE - 1) {
                let side = 'top';
                if (x === 0 && y === 0) side = 'corner-tl';
                else if (x === 0 && y === GRID_SIZE - 1) side = 'corner-bl';
                else if (x === GRID_SIZE - 1 && y === 0) side = 'corner-tr';
                else if (x === GRID_SIZE - 1 && y === GRID_SIZE - 1) side = 'corner-br';
                else if (x === 0) side = 'left';
                else if (x === GRID_SIZE - 1) side = 'right';
                else if (y === GRID_SIZE - 1) side = 'bottom';
                
                guards.push({ id: guardIdCounter++, x, y, alive: true, side: side });
            }
        }
    }

    maintainFoodSupply(); 
    
    remainingGuards.value = guards.length;
    timeLeftStr.value = "30.0";
    statusText.value = "等待開局";
    statusColor.value = "#ffcc00";
    actionHint.value = "請按 【R】 鍵正式開始遊戲";
    isHighlight.value = true;

    draw(); 
}

// --- 正式啟動核心計時器 ---
function startGame() {
    clearInterval(gameInterval);
    clearInterval(laserInterval);
    clearInterval(bulletInterval);

    setupStage();
    
    gameOver = false;
    gameWin = false;
    gameStarted = true;
    lastKillTime = Date.now(); 

    gameInterval = setInterval(gameLoop, GAME_SPEED);
    laserInterval = setInterval(triggerLaser, LASER_INTERVAL);
    bulletInterval = setInterval(updateBullets, 16); 
    
    statusText.value = "進行中";
    statusColor.value = "#00ffcc";
    actionHint.value = "戰鬥進行中...";
    isHighlight.value = false; 
    
    draw();
}

function maintainFoodSupply() {
    let currentLength = snake.length;
    let targetFoodCount = 1;

    if (currentLength >= 10) targetFoodCount = 1;
    else if (currentLength > 5) targetFoodCount = 2;
    else if (currentLength > 2) targetFoodCount = 3;
    else targetFoodCount = 4;

    while (foodList.length < targetFoodCount) {
        let fx = Math.floor(Math.random() * (GRID_SIZE - 6)) + 3; 
        let fy = Math.floor(Math.random() * (GRID_SIZE - 6)) + 3; 
        let onSnake = snake.some(segment => segment.x === fx && segment.y === fy);
        let onFood = foodList.some(f => f.x === fx && f.y === fy);

        if (!onSnake && !onFood) {
            foodList.push({ x: fx, y: fy });
        }
    }

    while (foodList.length > targetFoodCount) {
        foodList.pop();
    }
}

function triggerLaser() {
    if (gameOver || gameWin || !gameStarted) return;

    let aliveGuards = guards.filter(g => g.alive);
    if (aliveGuards.length === 0) return;

    let shooter = aliveGuards[Math.floor(Math.random() * aliveGuards.length)];
    if (!shooter) return; 

    let now = Date.now();
    let laser: Laser = {
        guardId: shooter.id, 
        x1: 0, y1: 0, x2: 0, y2: 0,
        cells: [], 
        startTime: now,
        warnEndTime: now + LASER_WARN_TIME,     
        expireTime: now + LASER_WARN_TIME + LASER_ACTIVE_TIME 
    };

    if (shooter.side === 'left' || shooter.side === 'right' || shooter.side === 'corner-tl' || shooter.side === 'corner-tr') {
        laser.x1 = 0; laser.y1 = shooter.y;
        laser.x2 = GRID_SIZE - 1; laser.y2 = shooter.y;
        for(let x = 0; x < GRID_SIZE; x++) laser.cells.push({x: x, y: shooter.y});
    } else {
        laser.x1 = shooter.x; laser.y1 = 0;
        laser.x2 = shooter.x; laser.y2 = GRID_SIZE - 1;
        for(let y = 0; y < GRID_SIZE; y++) laser.cells.push({x: shooter.x, y: y});
    }

    activeLasers.push(laser);
}

function checkLaserDamage(laser: Laser) {
    if (gameOver || gameWin || snake.length === 0) return;
    
    let now = Date.now();
    if (now < laser.warnEndTime) return;

    let head = snake[0]!; // 補上型別非空斷言
    if (laser.cells.some(c => c.x === head.x && c.y === head.y)) {
        endGame(false, "蛇頭被高能激光蒸發！");
        return;
    }

    for (let i = 1; i < snake.length; i++) {
        let segment = snake[i];
        if (segment && laser.cells.some(c => c.x === segment.x && c.y === segment.y)) {
            snake = snake.slice(0, i); 
            ammo.value = snake.length;       
            break; 
        }
    }
}

function fireBullet(bulletDir: Point) {
    if (gameOver || gameWin || !gameStarted || snake.length === 0) return;
    if (ammo.value <= 1 || snake.length <= 1 || !canShoot) return;

    canShoot = false; 
    ammo.value--;
    snake.pop(); 

    let head = snake[0]!;

    bullets.push({
        x: head.x,
        y: head.y,
        dx: bulletDir.x,
        dy: bulletDir.y
    });

    maintainFoodSupply(); 
}

function updateBullets() {
    if (gameOver || gameWin || !gameStarted) return;

    for (let i = bullets.length - 1; i >= 0; i--) {
        let b = bullets[i];
        if (!b) continue;

        b.x += b.dx * BULLET_SPEED;
        b.y += b.dy * BULLET_SPEED;

        let gridX = Math.floor(b.x + 0.5);
        let gridY = Math.floor(b.y + 0.5);

        if (gridX < 0 || gridX >= GRID_SIZE || gridY < 0 || gridY >= GRID_SIZE) {
            bullets.splice(i, 1);
            continue;
        }

        let hitGuard = guards.find(g => g.x === gridX && g.y === gridY && g.alive);
        if (hitGuard) {
            hitGuard.alive = false; 
            let targetId = hitGuard.id;
            activeLasers = activeLasers.filter(l => l.guardId !== targetId); 
            lastKillTime = Date.now();
            bullets.splice(i, 1); 
            continue;
        }
    }
}

function gameLoop() {
    if (gameOver || gameWin || !gameStarted || snake.length === 0) return;

    dir = nextDir; 
    canShoot = true; 

    let timeElapsed = Date.now() - lastKillTime;
    let timeLeft = Math.max(0, (KILL_TIMEOUT - timeElapsed) / 1000);
    timeLeftStr.value = timeLeft.toFixed(1);

    if (timeLeft <= 0) {
        endGame(false, "超時未擊殺守衛！系統處死。");
        return;
    }

    let head = snake[0]!;
    let nextX = head.x + dir.x;
    let nextY = head.y + dir.y;

    let guardAtTarget = guards.find(g => g.x === nextX && g.y === nextY && g.alive);
    if (guardAtTarget) {
        endGame(false, "正面撞上守衛防線！");
        return;
    }

    if (nextX < 0 || nextX >= GRID_SIZE || nextY < 0 || nextY >= GRID_SIZE) {
        endGame(false, "正面撞擊外牆邊界！");
        return;
    }

    if (snake.some(seg => seg.x === nextX && seg.y === nextY)) {
        endGame(false, "咬到自己了！");
        return;
    }

    let newHead = { x: nextX, y: nextY };
    snake.unshift(newHead);

    let foodIndex = foodList.findIndex(f => f.x === nextX && f.y === nextY);
    if (foodIndex !== -1) {
        ammo.value++; 
        foodList.splice(foodIndex, 1); 
    } else {
        snake.pop(); 
    }

    maintainFoodSupply();

    activeLasers.forEach(laser => checkLaserDamage(laser));
    activeLasers = activeLasers.filter(l => Date.now() < l.expireTime);

    let aliveCount = guards.filter(g => g.alive).length;
    remainingGuards.value = aliveCount;
    ammo.value = snake.length;

    if (aliveCount === 0) {
        endGame(true, "成功殲滅所有守衛！你贏了！");
    }

    draw();
}

function endGame(isWin: boolean, message: string) {
    if(isWin) {
        gameWin = true;
        statusText.value = "獲勝！";
        statusColor.value = "#00ff00";
    } else {
        gameOver = true;
        statusText.value = "遊戲結束";
        statusColor.value = "#ff3366";
    }

    clearInterval(gameInterval);
    clearInterval(laserInterval);
    clearInterval(bulletInterval);
    
    actionHint.value = `【${message}】— 請按 【R】 鍵重新開始新局`;
    isHighlight.value = true;
}

function draw() {
    if (!gameCanvas.value) return;
    const ctx = gameCanvas.value.getContext('2d');
    if (!ctx) return;

    ctx.fillStyle = '#050508';
    ctx.fillRect(0, 0, gameCanvas.value.width, gameCanvas.value.height);

    guards.forEach(g => {
        ctx.fillStyle = g.alive ? '#ff2a4b' : '#12121a';
        ctx.fillRect(g.x * CELL_SIZE, g.y * CELL_SIZE, CELL_SIZE - 1, CELL_SIZE - 1);
    });

    ctx.fillStyle = '#ffcc00'; 
    foodList.forEach(food => {
        ctx.fillRect(food.x * CELL_SIZE, food.y * CELL_SIZE, CELL_SIZE - 1, CELL_SIZE - 1);
    });

    ctx.fillStyle = '#00ffff'; 
    bullets.forEach(b => {
        ctx.fillRect(b.x * CELL_SIZE + 5, b.y * CELL_SIZE + 5, 10, 10);
    });

    snake.forEach((segment, index) => {
        ctx.fillStyle = (index === 0) ? '#00ff66' : '#00aa44';
        ctx.fillRect(segment.x * CELL_SIZE, segment.y * CELL_SIZE, CELL_SIZE - 1, CELL_SIZE - 1);
    });

    let now = Date.now();
    activeLasers.forEach(l => {
        ctx.save();
        let isWarning = gameStarted ? (now < l.warnEndTime) : true; 
        
        if (gameOver || gameWin) {
            isWarning = (l.expireTime - now > LASER_ACTIVE_TIME);
        }

        if (isWarning) {
            ctx.strokeStyle = '#ff3366';
            ctx.lineWidth = 2;
            ctx.setLineDash([6, 6]); 
        } else {
            ctx.strokeStyle = '#ffffff'; 
            ctx.shadowColor = '#ff0055'; 
            ctx.shadowBlur = 10;
            ctx.lineWidth = 5;
            ctx.setLineDash([]); 
        }
        
        ctx.beginPath();
        ctx.moveTo(l.x1 * CELL_SIZE + CELL_SIZE/2, l.y1 * CELL_SIZE + CELL_SIZE/2);
        ctx.lineTo(l.x2 * CELL_SIZE + CELL_SIZE/2, l.y2 * CELL_SIZE + CELL_SIZE/2);
        ctx.stroke();
        ctx.restore();
    });
}

function handleKeyDown(e: KeyboardEvent) {
    switch(e.key) {
        case 'ArrowUp':
            e.preventDefault(); 
            if (dir.y !== 1) nextDir = { x: 0, y: -1 };
            break;
        case 'ArrowDown':
            e.preventDefault(); 
            if (dir.y !== -1) nextDir = { x: 0, y: 1 };
            break;
        case 'ArrowLeft':
            e.preventDefault(); 
            if (dir.x !== 1) nextDir = { x: -1, y: 0 };
            break;
        case 'ArrowRight':
            e.preventDefault(); 
            if (dir.x !== -1) nextDir = { x: 1, y: 0 };
            break;
        case ' ': 
            e.preventDefault(); 
            fireBullet(dir); 
            break;
        case 'r': case 'R': 
            startGame(); 
            break;
    }
}

onMounted(() => {
    setupStage();
    window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
    clearInterval(gameInterval);
    clearInterval(laserInterval);
    clearInterval(bulletInterval);
    window.removeEventListener('keydown', handleKeyDown);
});
</script>

<template>
    <div class="game-container">
        <h1>科技獵蛇：圍城死鬥版 (25x25)</h1>
        
        <div id="game-info">
            <div>長度/彈藥: <span class="status-val">{{ ammo }}</span></div>
            <div>毀滅倒數: <span id="countdown-val">{{ timeLeftStr }}</span> 秒</div>
            <div>剩餘守衛: <span class="status-val">{{ remainingGuards }}</span></div>
            <div>狀態: <span :style="{ color: statusColor }" style="font-weight: bold;">{{ statusText }}</span></div>
        </div>

        <canvas ref="gameCanvas" width="500" height="500"></canvas>
        
        <div id="msg">
            移動：使用右手 <span class="key-hint">↑ ↓ ← → 方向鍵</span> 控制蛇身（已鎖定網頁防捲動）<br>
            射擊：使用左手 <span class="key-hint">空白鍵 Space</span> 順向發射子彈（擊中守衛即銷毀）<br>
            <span :class="{ 'highlight': isHighlight }">{{ actionHint }}</span>
        </div>
    </div>
</template>

<style scoped>
.game-container {
    background-color: #0d0d11;
    color: #fff;
    font-family: "Microsoft JhengHei", sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 20px;
    min-height: 80vh;
}

h1 {
    margin-top: 0;
    font-size: 28px;
    letter-spacing: 2px;
}

#game-info {
    display: flex;
    gap: 25px;
    font-size: 18px;
    margin-bottom: 15px;
    background: #1a1a24;
    padding: 12px 25px;
    border-radius: 8px;
    border: 1px solid #333344;
}

canvas {
    border: 4px solid #444455;
    background-color: #030305;
    box-shadow: 0 0 25px rgba(0, 255, 102, 0.15);
}

.status-val {
    font-weight: bold;
    color: #00ffcc;
}

#countdown-val {
    color: #ffcc00;
    font-weight: bold;
}

#msg {
    font-size: 16px;
    font-weight: bold;
    margin-top: 15px;
    height: 60px;
    color: #bbbbcc;
    text-align: center;
    line-height: 1.6;
}

.highlight {
    color: #ff3366;
}

.key-hint {
    color: #00ffff;
    background: #222233;
    padding: 2px 6px;
    border-radius: 4px;
    border: 1px solid #444466;
}
</style>