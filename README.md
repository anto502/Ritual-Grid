# Ritual-Grid
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Photo Match Ultra Pro Fixed</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@700&display=swap');
        
        :root { 
            --bg: #020a04; 
            --card: #08160b; 
            --accent-green: #00ff88;
            --error-red: #ff2d55;
            --neon-blue: #00d4ff;
        }
        
        * { -webkit-tap-highlight-color: transparent; box-sizing: border-box; }

        body { 
            background: var(--bg); 
            color: #fff; 
            font-family: 'Orbitron', sans-serif; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
            min-height: 100vh; 
            margin: 0; 
            overflow: hidden; 
            user-select: none;
            touch-action: manipulation;
        }
        
        .game-container { 
            background: var(--card); 
            padding: clamp(10px, 4vw, 20px); 
            border-radius: 24px; 
            width: 95vw;
            max-width: 360px; 
            text-align: center; 
            border: 2px solid #1a3a20; 
            box-shadow: 0 0 30px rgba(0, 255, 136, 0.15);
            position: relative;
        }

        .stats-row { display: flex; justify-content: space-between; margin-bottom: 8px; font-weight: bold; font-size: 0.85rem; }
        #lives { color: var(--error-red); letter-spacing: 2px; }
        #score { color: var(--accent-green); }
        #cps { color: var(--neon-blue); }
        
        .timer-row { font-size: 1.4rem; color: #fff; margin-bottom: 12px; text-shadow: 0 0 10px var(--neon-blue); }

        .grid { 
            display: grid; 
            grid-template-columns: repeat(3, 1fr); 
            gap: clamp(8px, 2vw, 12px); 
            margin-bottom: 15px;
            transition: opacity 0.3s;
        }
        
        .box { 
            aspect-ratio: 1 / 1;
            background: rgba(13, 40, 20, 0.9); 
            border-radius: clamp(10px, 3vw, 16px); 
            cursor: pointer; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
            border: 1px solid #2d5a36; 
            transition: transform 0.1s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            padding: 12%;
            position: relative;
        }
        
        .box.selected { border: 3px solid #fff; transform: scale(0.92); background: #163a1f; }
        .box img { width: 100%; height: 100%; object-fit: contain; pointer-events: none; }
        .box.correct { border-color: var(--accent-green) !important; box-shadow: 0 0 20px var(--accent-green); }
        .box.wrong { border-color: var(--error-red) !important; animation: boxShake 0.2s ease-in-out; }

        @keyframes boxShake {
            0%, 100% { transform: translateX(0); }
            25% { transform: translateX(-4px); }
            75% { transform: translateX(4px); }
        }

        #msg { height: 40px; color: var(--error-red); font-weight: bold; font-size: clamp(1.1rem, 5vw, 1.5rem); display: flex; align-items: center; justify-content: center; }

        #end-menu { 
            display: none; 
            flex-direction: column; 
            gap: 12px; 
            margin-top: 10px;
            width: 100%;
        }

        button { 
            padding: 16px; 
            border-radius: 14px; 
            border: none; 
            font-weight: bold; 
            cursor: pointer; 
            font-family: 'Orbitron', sans-serif; 
            font-size: 1rem; 
            transition: background 0.2s, transform 0.1s;
            width: 100%;
        }
        .btn-retry { background: var(--accent-green); color: #000; }
        .btn-share { background: transparent; color: #fff; border: 1px solid #333; }
        button:active { transform: scale(0.98); }
    </style>
</head>
<body>

<div class="game-container">
    <div class="stats-row">
        <div>PTS: <span id="score">0</span></div>
        <div>CPS: <span id="cps">0.0</span></div>
        <div>LIFE: <span id="lives">❤️❤️❤️</span></div>
    </div>
    <div class="timer-row">TIME: <span id="timer">60</span>s</div>
    
    <div id="msg"></div>
    <div class="grid" id="grid"></div>

    <div id="end-menu">
        <button class="btn-retry" onclick="startGame()">RESTART</button>
        <button class="btn-share" onclick="shareOnX()">SHARE ON X</button>
    </div>
</div>

<script>
    let audioCtx;
    let clickTimes = [];
    const images = [
        "https://cdn-icons-png.flaticon.com/512/625/625150.png",
        "https://cdn-icons-png.flaticon.com/512/2103/2103633.png",
        "https://cdn-icons-png.flaticon.com/512/2991/2991148.png",
        "https://cdn-icons-png.flaticon.com/512/911/911409.png",
        "https://cdn-icons-png.flaticon.com/512/3655/3655591.png",
        "https://cdn-icons-png.flaticon.com/512/2592/2592317.png",
        "https://cdn-icons-png.flaticon.com/512/2097/2097276.png",
        "https://cdn-icons-png.flaticon.com/512/1067/1067256.png"
    ];

    let score = 0, timeLeft = 60, lives = 3, selected = [], isPlaying = false, timer;

    function initAudio() {
        if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        if (audioCtx.state === 'suspended') audioCtx.resume();
    }

    function playSound(freq, type, duration, vol=0.1) {
        if (!audioCtx || isPlaying === false) return;
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = type;
        osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
        gain.gain.setValueAtTime(vol, audioCtx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start();
        osc.stop(audioCtx.currentTime + duration);
    }

    const sounds = {
        correct: () => { playSound(700, 'sine', 0.1); setTimeout(() => playSound(1000, 'sine', 0.1), 50); },
        wrong: () => playSound(180, 'triangle', 0.15, 0.08),
        gameOver: () => { 
            const osc = audioCtx.createOscillator();
            osc.frequency.setValueAtTime(150, audioCtx.currentTime);
            osc.connect(audioCtx.destination);
            osc.start(); osc.stop(audioCtx.currentTime + 0.3);
        }
    };

    function updateCPS() {
        const now = Date.now();
        clickTimes.push(now);
        clickTimes = clickTimes.filter(t => now - t < 1000);
        document.getElementById('cps').innerText = clickTimes.length.toFixed(1);
    }

    function renderGrid() {
        const grid = document.getElementById('grid');
        grid.innerHTML = '';
        let pool = [...images].sort(() => 0.5 - Math.random());
        let pair = pool[0];
        let others = pool.slice(1, 8);
        let items = [pair, pair, ...others].sort(() => 0.5 - Math.random());

        items.forEach(imgUrl => {
            const div = document.createElement('div');
            div.className = 'box';
            div.dataset.img = imgUrl;
            div.innerHTML = `<img src="${imgUrl}" draggable="false">`;
            div.onpointerdown = (e) => {
                initAudio();
                if (isPlaying) {
                    updateCPS();
                    handleSelect(div);
                }
            };
            grid.appendChild(div);
        });
    }

    function startGame() {
        // Reset Variables
        score = 0; timeLeft = 60; lives = 3; 
        isPlaying = true; clickTimes = []; selected = [];
        
        // UI Reset
        const grid = document.getElementById('grid');
        grid.style.opacity = '1';
        grid.style.pointerEvents = 'auto';
        document.getElementById('score').innerText = score;
        document.getElementById('lives').innerText = "❤️❤️❤️";
        document.getElementById('timer').innerText = timeLeft;
        document.getElementById('msg').innerText = "";
        document.getElementById('end-menu').style.display = 'none';
        
        renderGrid();
        
        clearInterval(timer);
        timer = setInterval(() => {
            if(!isPlaying) return;
            timeLeft--;
            document.getElementById('timer').innerText = timeLeft;
            if(timeLeft <= 0) endGame("TIME EXPIRED");
            
            const now = Date.now();
            clickTimes = clickTimes.filter(t => now - t < 1000);
            document.getElementById('cps').innerText = clickTimes.length.toFixed(1);
        }, 1000);
    }

    function handleSelect(div) {
        if(!isPlaying || selected.length >= 2 || div.classList.contains('selected')) return;
        div.classList.add('selected');
        selected.push(div);
        if(selected.length === 2) checkMatch();
    }

    function checkMatch() {
        const [a, b] = selected;
        if(a.dataset.img === b.dataset.img) {
            score += 25;
            document.getElementById('score').innerText = score;
            a.classList.add('correct'); b.classList.add('correct');
            sounds.correct();
            setTimeout(() => { if(isPlaying) renderGrid(); }, 100); 
        } else {
            lives--; 
            updateLivesDisplay();
            a.classList.add('wrong'); b.classList.add('wrong');
            sounds.wrong();
            if(lives <= 0) endGame("TERMINATED");
            else setTimeout(() => {
                a.classList.remove('selected', 'wrong');
                b.classList.remove('selected', 'wrong');
            }, 150);
        }
        selected = [];
    }

    function updateLivesDisplay() {
        let l = "";
        for(let i=0; i < 3; i++) l += (i < lives) ? "❤️" : "🖤";
        document.getElementById('lives').innerText = l;
    }

    function endGame(m) { 
        isPlaying = false; 
        clearInterval(timer); 
        document.getElementById('msg').innerText = m; 
        document.getElementById('end-menu').style.display = 'flex';
        
        const grid = document.getElementById('grid');
        grid.style.opacity = '0.2';
        grid.style.pointerEvents = 'none';
        
        if(audioCtx) sounds.gameOver();
    }

    function shareOnX() {
        const shareText = `NEON MATCH: I scored ${score} PTS with ${document.getElementById('cps').innerText} CPS! ⚡ Beat me!`;
        window.open(`https://twitter.com/intent/tweet?text=${encodeURIComponent(shareText)}`, '_blank');
    }

    // First Start
    startGame();
</script>
</body>
</html>
