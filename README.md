<!DOCTYPE html>
<html lang="hy">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>City Battle</title>

<style>
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    overflow: hidden;
    background: #101317;
    font-family: Arial, sans-serif;
    color: white;
}

canvas {
    display: block;
    background: #1b2025;
    cursor: crosshair;
}

#ui {
    position: fixed;
    top: 15px;
    left: 15px;
    z-index: 10;
    pointer-events: none;
}

.panel {
    background: rgba(10, 13, 17, .88);
    border: 1px solid rgba(255,255,255,.12);
    border-radius: 12px;
    padding: 12px 15px;
    margin-bottom: 8px;
    box-shadow: 0 5px 20px rgba(0,0,0,.35);
}

#healthOuter {
    width: 220px;
    height: 15px;
    background: #30353b;
    border-radius: 20px;
    overflow: hidden;
    margin-top: 6px;
}

#health {
    width: 100%;
    height: 100%;
    background: #37d66b;
    transition: width .15s;
}

#weapon {
    font-weight: bold;
    color: #ffd35a;
}

#help {
    position: fixed;
    bottom: 15px;
    left: 15px;
    z-index: 10;
    background: rgba(10,13,17,.8);
    border-radius: 10px;
    padding: 10px 14px;
    font-size: 13px;
    color: #c9d0d8;
    pointer-events: none;
}

#message {
    position: fixed;
    inset: 0;
    z-index: 20;
    display: none;
    align-items: center;
    justify-content: center;
    background: rgba(0,0,0,.65);
}

.messageBox {
    text-align: center;
    background: #151a20;
    padding: 35px 50px;
    border-radius: 18px;
    border: 1px solid #3a414a;
    box-shadow: 0 20px 60px rgba(0,0,0,.6);
}

.messageBox h1 {
    font-size: 42px;
    margin-bottom: 12px;
}

.messageBox p {
    color: #bbc3cc;
    margin-bottom: 20px;
}

button {
    border: 0;
    background: #3b82f6;
    color: white;
    font-size: 16px;
    padding: 12px 25px;
    border-radius: 9px;
    cursor: pointer;
}

button:hover {
    background: #2563eb;
}
</style>
</head>

<body>

<canvas id="game"></canvas>

<div id="ui">

    <div class="panel">
        <div>❤️ Առողջություն</div>
        <div id="healthOuter">
            <div id="health"></div>
        </div>
    </div>

    <div class="panel">
        🔫 Զենք՝ <span id="weapon">Pistol</span><br>
        🎯 Սպանություններ՝ <span id="kills">0</span><br>
        👥 Թշնամիներ՝ <span id="enemies">0</span>
    </div>

</div>

<div id="help">
    WASD — շարժվել &nbsp; | &nbsp;
    Մկնիկ — ուղղություն &nbsp; | &nbsp;
    Ձախ click — կրակել &nbsp; | &nbsp;
    1/2/3 — զենք
</div>

<div id="message">
    <div class="messageBox">
        <h1 id="messageTitle">Game Over</h1>
        <p id="messageText"></p>
        <button onclick="restartGame()">Կրկին խաղալ</button>
    </div>
</div>

<script>
"use strict";

const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let W = window.innerWidth;
let H = window.innerHeight;

canvas.width = W;
canvas.height = H;

window.addEventListener("resize", () => {
    W = window.innerWidth;
    H = window.innerHeight;
    canvas.width = W;
    canvas.height = H;
});

const keys = {};
const mouse = {
    x: W / 2,
    y: H / 2,
    down: false
};

document.addEventListener("keydown", e => {
    keys[e.key.toLowerCase()] = true;

    if (e.key === "1") player.weapon = "pistol";
    if (e.key === "2") player.weapon = "rifle";
    if (e.key === "3") player.weapon = "shotgun";

    if (e.key.toLowerCase() === "r" && gameOver) {
        restartGame();
    }
});

document.addEventListener("keyup", e => {
    keys[e.key.toLowerCase()] = false;
});

canvas.addEventListener("mousemove", e => {
    mouse.x = e.clientX;
    mouse.y = e.clientY;
});

canvas.addEventListener("mousedown", e => {
    if (e.button === 0) mouse.down = true;
});

canvas.addEventListener("mouseup", e => {
    if (e.button === 0) mouse.down = false;
});

canvas.addEventListener("mouseleave", () => {
    mouse.down = false;
});

const weapons = {
    pistol: {
        name: "Pistol",
        damage: 28,
        speed: 9,
        cooldown: 330,
        bullets: 1,
        spread: 0
    },

    rifle: {
        name: "Rifle",
        damage: 16,
        speed: 13,
        cooldown: 110,
        bullets: 1,
        spread: 0.035
    },

    shotgun: {
        name: "Shotgun",
        damage: 12,
        speed: 10,
        cooldown: 650,
        bullets: 7,
        spread: 0.23
    }
};

let player;
let enemies = [];
let bullets = [];
let particles = [];
let buildings = [];
let kills = 0;
let gameOver = false;
let lastTime = 0;

function resetPlayer() {
    player = {
        x: W / 2,
        y: H / 2,
        r: 17,
        speed: 3.4,
        hp: 100,
        maxHp: 100,
        weapon: "pistol",
        lastShot: 0,
        color: "#45a3ff"
    };
}

function createBuildings() {

    buildings = [];

    const list = [
        [80, 80, 230, 120],
        [430, 70, 180, 150],
        [760, 100, 270, 110],

        [90, 350, 210, 150],
        [420, 330, 260, 120],
        [800, 350, 210, 170],

        [180, 650, 250, 130],
        [560, 620, 190, 150],
        [850, 650, 260, 120]
    ];

    for (const b of list) {

        let [x,y,w,h] = b;

        if (x + w < W && y + h < H) {
            buildings.push({
                x, y, w, h
            });
        }
    }
}

function randomEnemyPosition() {

    for (let i = 0; i < 100; i++) {

        const x = 30 + Math.random() * (W - 60);
        const y = 30 + Math.random() * (H - 60);

        if (
            distance(x, y, player.x, player.y) > 350 &&
            !circleHitsBuildings(x, y, 20)
        ) {
            return {x, y};
        }
    }

    return {
        x: 50 + Math.random() * (W - 100),
        y: 50 + Math.random() * (H - 100)
    };
}

function spawnEnemies(count = 10) {

    enemies = [];

    for (let i = 0; i < count; i++) {

        const p = randomEnemyPosition();

        const types = ["soldier", "fast", "heavy"];
        const type = types[Math.floor(Math.random() * types.length)];

        let enemy = {
            x: p.x,
            y: p.y,
            r: 16,
            hp: 60,
            maxHp: 60,
            speed: 1.15,
            damage: 8,
            cooldown: 850,
            lastShot: Math.random() * 1000,
            type,
            color: "#ef5350",
            angle: 0
        };

        if (type === "fast") {
            enemy.hp = 40;
            enemy.maxHp = 40;
            enemy.speed = 1.9;
            enemy.damage = 6;
            enemy.color = "#ff9f43";
        }

        if (type === "heavy") {
            enemy.hp = 110;
            enemy.maxHp = 110;
            enemy.speed = .65;
            enemy.damage = 13;
            enemy.color = "#c85cff";
            enemy.r = 20;
        }

        enemies.push(enemy);
    }
}

function distance(x1,y1,x2,y2) {
    return Math.hypot(x2-x1, y2-y1);
}

function circleRectCollision(cx, cy, r, rect) {

    const closestX = Math.max(rect.x, Math.min(cx, rect.x + rect.w));
    const closestY = Math.max(rect.y, Math.min(cy, rect.y + rect.h));

    const dx = cx - closestX;
    const dy = cy - closestY;

    return dx * dx + dy * dy < r * r;
}

function circleHitsBuildings(x,y,r) {

    for (const b of buildings) {
        if (circleRectCollision(x,y,r,b)) {
            return true;
        }
    }

    return false;
}

function moveEntity(entity, dx, dy) {

    let nx = entity.x + dx;

    if (
        nx > entity.r &&
        nx < W - entity.r &&
        !circleHitsBuildings(nx, entity.y, entity.r)
    ) {
        entity.x = nx;
    }

    let ny = entity.y + dy;

    if (
        ny > entity.r &&
        ny < H - entity.r &&
        !circleHitsBuildings(entity.x, ny, entity.r)
    ) {
        entity.y = ny;
    }
}

function shoot(shooter, isPlayer) {

    const now = performance.now();

    const weapon = isPlayer
        ? weapons[player.weapon]
        : {
            damage: shooter.damage,
            speed: 7,
            cooldown: shooter.cooldown,
            bullets: 1,
            spread: .04
        };

    if (now - shooter.lastShot < weapon.cooldown) return;

    shooter.lastShot = now;

    let targetX = mouse.x;
    let targetY = mouse.y;

    if (!isPlayer) {
        targetX = player.x;
        targetY = player.y;
    }

    const baseAngle = Math.atan2(
        targetY - shooter.y,
        targetX - shooter.x
    );

    for (let i = 0; i < weapon.bullets; i++) {

        const angle =
            baseAngle +
            (Math.random() - .5) * weapon.spread;

        bullets.push({
            x: shooter.x + Math.cos(angle) * 22,
            y: shooter.y + Math.sin(angle) * 22,
            vx: Math.cos(angle) * weapon.speed,
            vy: Math.sin(angle) * weapon.speed,
            damage: weapon.damage,
            owner: isPlayer ? "player" : "enemy",
            life: 1000
        });
    }

    createMuzzleFlash(
        shooter.x + Math.cos(baseAngle) * 22,
        shooter.y + Math.sin(baseAngle) * 22
    );
}

function createMuzzleFlash(x,y) {

    for (let i=0; i<5; i++) {

        particles.push({
            x,
            y,
            vx: (Math.random()-.5)*3,
            vy: (Math.random()-.5)*3,
            life: 180,
            maxLife: 180,
            size: 2 + Math.random()*3,
            color: "#ffd35a"
        });
    }
}

function createExplosion(x,y,color="#fff") {

    for (let i=0; i<14; i++) {

        const a = Math.random() * Math.PI * 2;
        const s = 1 + Math.random() * 4;

        particles.push({
            x,
            y,
            vx: Math.cos(a)*s,
            vy: Math.sin(a)*s,
            life: 500 + Math.random()*300,
            maxLife: 800,
            size: 2 + Math.random()*4,
            color
        });
    }
}

function updatePlayer(dt) {

    let dx = 0;
    let dy = 0;

    if (keys["w"] || keys["arrowup"]) dy--;
    if (keys["s"] || keys["arrowdown"]) dy++;
    if (keys["a"] || keys["arrowleft"]) dx--;
    if (keys["d"] || keys["arrowright"]) dx++;

    if (dx !== 0 || dy !== 0) {

        const len = Math.hypot(dx,dy);

        dx /= len;
        dy /= len;

        moveEntity(
            player,
            dx * player.speed * dt / 16,
            dy * player.speed * dt / 16
        );
    }

    if (mouse.down) {
        shoot(player,true);
    }
}

function hasLineOfSight(a,b) {

    const steps = Math.ceil(distance(a.x,a.y,b.x,b.y) / 12);

    for (let i=0; i<=steps; i++) {

        const t = i / steps;

        const x = a.x + (b.x-a.x)*t;
        const y = a.y + (b.y-a.y)*t;

        if (circleHitsBuildings(x,y,3)) {
            return false;
        }
    }

    return true;
}

function updateEnemies(dt) {

    for (const enemy of enemies) {

        const d = distance(
            enemy.x,
            enemy.y,
            player.x,
            player.y
        );

        enemy.angle = Math.atan2(
            player.y-enemy.y,
            player.x-enemy.x
        );

        if (d > 260) {

            let dx = Math.cos(enemy.angle);
            let dy = Math.sin(enemy.angle);

            moveEntity(
                enemy,
                dx * enemy.speed * dt / 16,
                dy * enemy.speed * dt / 16
            );

        } else if (d < 130) {

            let dx = -Math.cos(enemy.angle);
            let dy = -Math.sin(enemy.angle);

            moveEntity(
                enemy,
                dx * enemy.speed * .7 * dt / 16,
                dy * enemy.speed * .7 * dt / 16
            );
        }

        if (
            d < 650 &&
            hasLineOfSight(enemy,player)
        ) {
            shoot(enemy,false);
        }
    }
}

function updateBullets(dt) {

    for (let i = bullets.length-1; i>=0; i--) {

        const b = bullets[i];

        b.x += b.vx * dt / 16;
        b.y += b.vy * dt / 16;

        b.life -= dt;

        let remove = false;

        if (
            b.x < 0 ||
            b.x > W ||
            b.y < 0 ||
            b.y > H ||
            b.life <= 0
        ) {
            remove = true;
        }

        if (!remove && circleHitsBuildings(b.x,b.y,3)) {

            createExplosion(b.x,b.y,"#b9c1ca");
            remove = true;
        }

        if (!remove && b.owner === "player") {

            for (let j=enemies.length-1; j>=0; j--) {

                const e = enemies[j];

                if (
                    distance(b.x,b.y,e.x,e.y)
                    < e.r + 4
                ) {

                    e.hp -= b.damage;

                    createExplosion(
                        b.x,
                        b.y,
                        "#ff6666"
                    );

                    remove = true;

                    if (e.hp <= 0) {

                        createExplosion(
                            e.x,
                            e.y,
                            e.color
                        );

                        enemies.splice(j,1);

                        kills++;
                    }

                    break;
                }
            }
        }

        if (!remove && b.owner === "enemy") {

            if (
                distance(
                    b.x,b.y,
                    player.x,player.y
                ) < player.r + 4
            ) {

                player.hp -= b.damage;

                createExplosion(
                    b.x,
                    b.y,
                    "#ff5555"
                );

                remove = true;

                if (player.hp <= 0) {
                    player.hp = 0;
                    endGame(false);
                }
            }
        }

        if (remove) {
            bullets.splice(i,1);
        }
    }
}

function updateParticles(dt) {

    for (let i=particles.length-1; i>=0; i--) {

        const p = particles[i];

        p.x += p.vx * dt/16;
        p.y += p.vy * dt/16;

        p.vx *= .97;
        p.vy *= .97;

        p.life -= dt;

        if (p.life <= 0) {
            particles.splice(i,1);
        }
    }
}

function drawBackground() {

    ctx.fillStyle = "#20262c";
    ctx.fillRect(0,0,W,H);

    const grid = 40;

    ctx.strokeStyle = "rgba(255,255,255,.035)";
    ctx.lineWidth = 1;

    for (let x=0; x<W; x+=grid) {

        ctx.beginPath();
        ctx.moveTo(x,0);
        ctx.lineTo(x,H);
        ctx.stroke();
    }

    for (let y=0; y<H; y+=grid) {

        ctx.beginPath();
        ctx.moveTo(0,y);
        ctx.lineTo(W,y);
        ctx.stroke();
    }
}

function drawBuildings() {

    for (const b of buildings) {

        ctx.fillStyle = "#343c44";
        ctx.fillRect(b.x,b.y,b.w,b.h);

        ctx.strokeStyle = "#58626d";
        ctx.lineWidth = 2;
        ctx.strokeRect(b.x,b.y,b.w,b.h);

        const windowSize = 13;

        for (
            let x=b.x+20;
            x<b.x+b.w-10;
            x+=32
        ) {

            for (
                let y=b.y+20;
                y<b.y+b.h-10;
                y+=32
            ) {

                ctx.fillStyle = "rgba(255,215,100,.38)";
                ctx.fillRect(
                    x,
                    y,
                    windowSize,
                    windowSize
                );
            }
        }

        ctx.fillStyle = "rgba(0,0,0,.12)";
        ctx.fillRect(
            b.x,
            b.y+b.h-10,
            b.w,
            10
        );
    }
}

function drawHealthBar(entity) {

    const width = 42;
    const height = 5;

    const hpRatio =
        Math.max(0,entity.hp/entity.maxHp);

    ctx.fillStyle = "rgba(0,0,0,.65)";
    ctx.fillRect(
        entity.x-width/2,
        entity.y-entity.r-13,
        width,
        height
    );

    ctx.fillStyle =
        entity === player
        ? "#39e36b"
        : "#ff5353";

    ctx.fillRect(
        entity.x-width/2,
        entity.y-entity.r-13,
        width*hpRatio,
        height
    );
}

function drawPerson(entity,isPlayer=false) {

    ctx.save();

    ctx.translate(entity.x,entity.y);

    let angle;

    if (isPlayer) {

        angle = Math.atan2(
            mouse.y-entity.y,
            mouse.x-entity.x
        );

    } else {
        angle = entity.angle;
    }

    ctx.rotate(angle);

    // shadow
    ctx.fillStyle = "rgba(0,0,0,.3)";
    ctx.beginPath();
    ctx.ellipse(
        0,
        entity.r*.75,
        entity.r*1.1,
        entity.r*.5,
        0,
        0,
        Math.PI*2
    );
    ctx.fill();

    // body
    ctx.fillStyle = entity.color;

    ctx.beginPath();
    ctx.arc(
        0,
        0,
        entity.r,
        0,
        Math.PI*2
    );
    ctx.fill();

    // outline
    ctx.strokeStyle =
        isPlayer ? "#bfe3ff" : "#ffb4b4";

    ctx.lineWidth = 2;
    ctx.stroke();

    // weapon
    ctx.fillStyle = "#15191d";

    ctx.fillRect(
        5,
        -4,
        entity.r+13,
        8
    );

    // head/helmet marker
    ctx.fillStyle = isPlayer
        ? "#d7edff"
        : "#ffd0d0";

    ctx.beginPath();
    ctx.arc(
        -5,
        0,
        entity.r*.42,
        0,
        Math.PI*2
    );
    ctx.fill();

    ctx.restore();

    drawHealthBar(entity);
}

function drawBullets() {

    for (const b of bullets) {

        ctx.fillStyle =
            b.owner === "player"
            ? "#ffe36e"
            : "#ff6767";

        ctx.beginPath();

        ctx.arc(
            b.x,
            b.y,
            3,
            0,
            Math.PI*2
        );

        ctx.fill();

        ctx.strokeStyle =
            b.owner === "player"
            ? "rgba(255,220,70,.25)"
            : "rgba(255,70,70,.25)";

        ctx.lineWidth = 4;

        ctx.beginPath();

        ctx.moveTo(
            b.x-b.vx*.8,
            b.y-b.vy*.8
        );

        ctx.lineTo(b.x,b.y);

        ctx.stroke();
    }
}

function drawParticles() {

    for (const p of particles) {

        const alpha =
            Math.max(0,p.life/p.maxLife);

        ctx.globalAlpha = alpha;

        ctx.fillStyle = p.color;

        ctx.beginPath();

        ctx.arc(
            p.x,
            p.y,
            p.size,
            0,
            Math.PI*2
        );

        ctx.fill();
    }

    ctx.globalAlpha = 1;
}

function drawCrosshair() {

    ctx.strokeStyle = "rgba(255,255,255,.75)";
    ctx.lineWidth = 1.5;

    const s = 9;

    ctx.beginPath();

    ctx.moveTo(mouse.x-s,mouse.y);
    ctx.lineTo(mouse.x-3,mouse.y);

    ctx.moveTo(mouse.x+3,mouse.y);
    ctx.lineTo(mouse.x+s,mouse.y);

    ctx.moveTo(mouse.x,mouse.y-s);
    ctx.lineTo(mouse.x,mouse.y-3);

    ctx.moveTo(mouse.x,mouse.y+3);
    ctx.lineTo(mouse.x,mouse.y+s);

    ctx.stroke();

    ctx.beginPath();

    ctx.arc(
        mouse.x,
        mouse.y,
        3,
        0,
        Math.PI*2
    );

    ctx.stroke();
}

function draw() {

    drawBackground();
    drawBuildings();

    drawBullets();

    for (const enemy of enemies) {
        drawPerson(enemy,false);
    }

    drawPerson(player,true);

    drawParticles();

    drawCrosshair();
}

function updateUI() {

    document.getElementById("health").style.width =
        player.hp + "%";

    document.getElementById("kills").textContent =
        kills;

    document.getElementById("enemies").textContent =
        enemies.length;

    document.getElementById("weapon").textContent =
        weapons[player.weapon].name;

    if (player.hp < 35) {
        document.getElementById("health").style.background =
            "#ff4d4d";
    } else {
        document.getElementById("health").style.background =
            "#37d66b";
    }
}

function endGame(win) {

    if (gameOver) return;

    gameOver = true;

    const box = document.getElementById("message");

    box.style.display = "flex";

    document.getElementById("messageTitle").textContent =
        win ? "🏆 Հաղթանակ!" : "💀 Game Over";

    document.getElementById("messageText").textContent =
        win
        ? "Դու ոչնչացրեցիր բոլոր հակառակորդներին։"
        : "Քո կերպարը պարտվեց։ Փորձիր կրկին։";
}

function restartGame() {

    gameOver = false;

    document.getElementById("message").style.display =
        "none";

    kills = 0;
    bullets = [];
    particles = [];

    resetPlayer();
    createBuildings();
    spawnEnemies(10);

    lastTime = performance.now();
}

function gameLoop(time) {

    const dt = Math.min(
        40,
        time-lastTime
    );

    lastTime = time;

    if (!gameOver) {

        updatePlayer(dt);
        updateEnemies(dt);
        updateBullets(dt);
        updateParticles(dt);

        if (enemies.length === 0) {
            endGame(true);
        }

        updateUI();
    }

    draw();

    requestAnimationFrame(gameLoop);
}

restartGame();
requestAnimationFrame(gameLoop);

</script>

</body>
</html>
