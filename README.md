<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>호빵맨: 기운 백배 대결!</title>
    <style>
        body { margin: 0; background: #222; overflow: hidden; font-family: 'Arial', sans-serif; touch-action: none; }
        canvas { background: #87CEEB; display: block; margin: 0 auto; border: 4px solid #444; }
        #ui-layer { position: absolute; top: 0; left: 50%; transform: translateX(-50%); width: 400px; height: 100%; pointer-events: none; }
        
        .hp-bar-container { width: 90%; margin: 10px auto; display: flex; justify-content: space-between; }
        .hp-wrapper { width: 45%; background: #eee; height: 20px; border: 2px solid #000; border-radius: 10px; overflow: hidden; position: relative; }
        .hp-fill { height: 100%; transition: width 0.3s; }
        #player-hp-fill { background: #ff4757; width: 100%; }
        #boss-hp-fill { background: #6a5acd; width: 100%; }
        .hp-label { position: absolute; width: 100%; text-align: center; font-size: 11px; font-weight: bold; line-height: 20px; color: #000; }

        .fever-info { color: #fff; text-align: center; font-size: 18px; font-weight: bold; text-shadow: 2px 2px 4px #000; }
        #fever-active { color: #ff0; display: none; font-size: 20px; animation: blink 0.5s infinite; }
        @keyframes blink { 50% { opacity: 0; } }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>
    <div id="ui-layer">
        <div class="hp-bar-container">
            <div class="hp-wrapper">
                <div id="player-hp-fill" class="hp-fill"></div>
                <div class="hp-label">호빵맨</div>
            </div>
            <div class="hp-wrapper">
                <div id="boss-hp-fill" class="hp-fill"></div>
                <div class="hp-label">세균맨</div>
            </div>
        </div>
        <div class="fever-info">
            구출한 아이들: <span id="rescue-count">0</span> / 5
            <div id="fever-active">🔥 기운 백배! 피버 타임! 🔥</div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = 400; canvas.height = 600;

        let playerHP = 100, bossHP = 100, rescueCount = 0, isFever = false, gameActive = true;
        let bgOffset = 0;

        const player = { x: 200, y: 500, radius: 25, history: [] };
        const boss = { x: 200, y: 100, radius: 35, vx: 3, cooldown: 0 };
        const playerProjectiles = []; 
        const bossPunches = [];
        const items = []; // 잼아저씨의 얼굴 & 짤랑이의 하트
        const trappedKids = [];
        const followers = [];

        const joy = { base: {x: 80, y: 500}, stick: {x: 80, y: 500}, active: false, id: null, dir: {x:0, y:0} };
        const btnP = { x: 320, y: 520, r: 40 };

        function drawAnpanman(x, y, r) {
            ctx.fillStyle = "#F4A460"; ctx.strokeStyle = "#8B4513"; ctx.lineWidth = 2;
            ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI*2); ctx.fill(); ctx.stroke();
            ctx.fillStyle = "#FF0000"; ctx.beginPath(); ctx.arc(x, y + 2, r/4, 0, Math.PI*2); ctx.fill(); ctx.stroke();
            ctx.fillStyle = "#FF6347"; ctx.beginPath(); ctx.arc(x - r/1.8, y, r/4, 0, Math.PI*2); ctx.fill(); ctx.stroke();
            ctx.beginPath(); ctx.arc(x + r/1.8, y, r/4, 0, Math.PI*2); ctx.fill(); ctx.stroke();
            ctx.fillStyle = "#000"; ctx.beginPath(); ctx.arc(x - r/3, y - r/4, 3, 0, Math.PI*2); ctx.fill();
            ctx.beginPath(); ctx.arc(x + r/3, y - r/4, 3, 0, Math.PI*2); ctx.fill();
            ctx.beginPath(); ctx.arc(x, y + 5, r/2, 0.2 * Math.PI, 0.8 * Math.PI); ctx.stroke();
        }

        function update() {
            if (!gameActive) return;
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            bgOffset += 2;
            ctx.fillStyle = "#87CEEB"; ctx.fillRect(0,0,400,600);
            
            // 호빵맨 이동
            player.x += joy.dir.x * 7; player.y += joy.dir.y * 7;
            player.x = Math.max(25, Math.min(375, player.x));
            player.y = Math.max(200, Math.min(575, player.y));
            player.history.unshift({x: player.x, y: player.y});
            if(player.history.length > 60) player.history.pop();

            // 세균맨 이동
            boss.x += boss.vx;
            if (boss.x > 350 || boss.x < 50) boss.vx *= -1;

            // --- 아이템 생성 로직 ---
            if (Math.random() < 0.005) { // 잼아저씨의 새 얼굴
                items.push({ x: Math.random()*360+20, y: -30, type: 'anpan', img: '🍞' });
            }
            if (Math.random() < 0.003) { // 짤랑이의 응원
                items.push({ x: boss.x, y: boss.y, type: 'dokin', img: '🍓', vy: -2 });
            }

            // 아이템 처리
            items.forEach((it, i) => {
                if(it.type === 'anpan') {
                    it.y += 3;
                    ctx.font = "30px Arial"; ctx.fillText(it.img, it.x-15, it.y+10);
                    if (Math.hypot(player.x - it.x, player.y - it.y) < 40) {
                        playerHP = Math.min(100, playerHP + 5); updateHP();
                        items.splice(i, 1);
                    }
                } else { // 짤랑이 응원 (세균맨에게 날아감)
                    it.y += it.vy;
                    ctx.font = "25px Arial"; ctx.fillText("💕", it.x-12, it.y);
                    if (it.y < 0) { // 세균맨이 받음
                        bossHP = Math.min(100, bossHP + 5);
                        document.getElementById('boss-hp-fill').style.width = bossHP + "%";
                        items.splice(i, 1);
                    }
                }
                if (it.y > 600) items.splice(i, 1);
            });

            // 공격 로직
            if (!isFever) {
                boss.cooldown++;
                if (boss.cooldown > 50) { bossPunches.push({ x: boss.x, y: boss.y, vy: 6 }); boss.cooldown = 0; }
            }

            // 구출 & 투사체 드로잉 (이전 코드 동일)
            processProjectiles();
            drawEntities();
            
            drawUI();
            requestAnimationFrame(update);
        }

        function processProjectiles() {
            // 호빵 펀치
            playerProjectiles.forEach((p, i) => {
                p.y -= 10;
                ctx.fillStyle = "#F4A460"; ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI*2); ctx.fill();
                ctx.font = p.r + "px Arial"; ctx.fillText("👊", p.x-p.r/1.5, p.y+p.r/2);
                if (Math.hypot(p.x - boss.x, p.y - boss.y) < boss.radius + p.r) {
                    bossHP -= isFever ? 4 : 2;
                    document.getElementById('boss-hp-fill').style.width = Math.max(0, bossHP) + "%";
                    playerProjectiles.splice(i, 1);
                    if (bossHP <= 0) endGame(true);
                }
                if (p.y < -20) playerProjectiles.splice(i, 1);
            });

            // 세균맨 주먹
            bossPunches.forEach((p, i) => {
                p.y += 6;
                ctx.fillStyle = "#6A5ACD"; ctx.beginPath(); ctx.arc(p.x, p.y, 12, 0, Math.PI*2); ctx.fill();
                ctx.fillText("⚡", p.x-6, p.y+5);
                if (Math.hypot(player.x - p.x, player.y - p.y) < 35) {
                    playerHP -= 5; updateHP(); bossPunches.splice(i, 1);
                }
                if (p.y > 600) bossPunches.splice(i, 1);
            });
        }

        function drawEntities() {
            // 감옥/아이들
            if (Math.random() < 0.01 && trappedKids.length < 1 && followers.length < 5) trappedKids.push({ x: Math.random()*300+50, y: -50 });
            trappedKids.forEach((k, i) => {
                k.y += 3;
                ctx.fillStyle = "#555"; ctx.fillRect(k.x-20, k.y-20, 40, 40);
                if (Math.hypot(player.x - k.x, player.y - k.y) < 45) {
                    trappedKids.splice(i, 1); rescueCount++; followers.push({});
                    document.getElementById('rescue-count').innerText = rescueCount;
                    if(rescueCount >= 5) startFever();
                }
                if (k.y > 600) trappedKids.splice(i, 1);
            });

            followers.forEach((f, i) => {
                let pos = player.history[(i + 1) * 10] || player.history[player.history.length-1];
                if(pos) { ctx.font = "12px Arial"; ctx.fillText("😊", pos.x-6, pos.y+44); }
            });

            ctx.font = "40px Arial"; ctx.fillText("😈", boss.x-20, boss.y+15);
            drawAnpanman(player.x, player.y, player.radius);
        }

        function drawUI() {
            ctx.globalAlpha = 0.5;
            ctx.fillStyle = "#fff"; ctx.beginPath(); ctx.arc(joy.base.x, joy.base.y, 45, 0, Math.PI*2); ctx.fill();
            ctx.fillStyle = "#444"; ctx.beginPath(); ctx.arc(joy.stick.x, joy.stick.y, 25, 0, Math.PI*2); ctx.fill();
            ctx.fillStyle = "#ff4757"; ctx.beginPath(); ctx.arc(btnP.x, btnP.y, btnP.r, 0, Math.PI*2); ctx.fill();
            ctx.globalAlpha = 1; ctx.fillStyle = "#fff"; ctx.font = "bold 14px Arial"; ctx.fillText("PUNCH", btnP.x-25, btnP.y+5);
        }

        function launchPunch() { if(!gameActive) return; playerProjectiles.push({ x: player.x, y: player.y - 20, r: isFever ? 25 : 15 }); }
        function startFever() { isFever = true; document.getElementById('fever-active').style.display = 'block'; setTimeout(() => { isFever = false; rescueCount = 0; followers.length = 0; document.getElementById('rescue-count').innerText = 0; document.getElementById('fever-active').style.display = 'none'; }, 7000); }
        function updateHP() { document.getElementById('player-hp-fill').style.width = Math.max(0, playerHP) + "%"; if (playerHP <= 0) endGame(false); }
        function endGame(win) { gameActive = false; setTimeout(() => { alert(win ? "✨ 기운 백배~!! 이겼다!! ✨" : "💀 하히후헤호!! (세균맨의 승리) 💀"); location.reload(); }, 100); }

        canvas.addEventListener('touchstart', (e) => {
            const rect = canvas.getBoundingClientRect();
            for (let t of e.changedTouches) {
                const tx = (t.clientX - rect.left) * (400 / rect.width);
                const ty = (t.clientY - rect.top) * (600 / rect.height);
                if (tx > 200) { if (Math.hypot(tx-btnP.x, ty-btnP.y) < btnP.r) launchPunch(); } 
                else { joy.active = true; joy.id = t.identifier; updateJoy(tx, ty); }
            }
        });

        canvas.addEventListener('touchmove', (e) => {
            e.preventDefault();
            const rect = canvas.getBoundingClientRect();
            for (let t of e.changedTouches) {
                if (t.identifier === joy.id) {
                    const tx = (t.clientX - rect.left) * (400 / rect.width);
                    const ty = (t.clientY - rect.top) * (600 / rect.height);
                    updateJoy(tx, ty);
                }
            }
            if(isFever && Math.random() < 0.2) launchPunch();
        }, {passive: false});

        canvas.addEventListener('touchend', (e) => { for (let t of e.changedTouches) { if (t.identifier === joy.id) { joy.active = false; joy.stick.x = joy.base.x; joy.stick.y = joy.base.y; joy.dir = {x:0, y:0}; } } });

        function updateJoy(tx, ty) {
            const dist = Math.hypot(tx - joy.base.x, ty - joy.base.y);
            const angle = Math.atan2(ty - joy.base.y, tx - joy.base.x);
            const mDist = Math.min(dist, 45);
            joy.stick.x = joy.base.x + Math.cos(angle) * mDist;
            joy.stick.y = joy.base.y + Math.sin(angle) * mDist;
            joy.dir = { x: Math.cos(angle) * (mDist/45), y: Math.sin(angle) * (mDist/45) };
        }

        update();
    </script>
</body>
</html># ANPANMAN
