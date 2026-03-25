<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Checkers Pro: Fix Online</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-database-compat.js"></script>
    <style>
        :root { --bg: #0a0a0a; --panel: #1a1a1a; --accent: #00e5ff; --cell-w: #f0d9b5; --cell-b: #b58863; }
        body { margin: 0; background: var(--bg); color: white; font-family: sans-serif; overflow: hidden; height: 100vh; display: flex; flex-direction: column; }
        #main-menu { position: fixed; inset: 0; background: var(--bg); display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 1000; }
        .menu-btns { display: flex; flex-direction: column; gap: 12px; width: 85%; max-width: 300px; }
        .btn { background: var(--panel); color: white; border: 2px solid #333; padding: 15px; border-radius: 12px; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn:active { transform: scale(0.95); border-color: var(--accent); }
        #game-ui { display: none; width: 100%; height: 100%; flex-direction: column; align-items: center; }
        @media (orientation: landscape) { 
            #game-ui { flex-direction: row; justify-content: space-evenly; }
            #board-resizer { width: 75vh !important; }
        }
        .status-bar { padding: 15px; background: var(--panel); width: 100%; text-align: center; box-shadow: 0 4px 15px rgba(0,0,0,0.8); }
        #board-resizer { width: 95vw; max-width: 500px; aspect-ratio: 1/1; margin: auto; background: #222; padding: 5px; border-radius: 8px; }
        #board { display: grid; grid-template-columns: repeat(8, 1fr); width: 100%; height: 100%; cursor: pointer; }
        .cell { position: relative; display: flex; justify-content: center; align-items: center; }
        .cell.w { background: var(--cell-w); }
        .cell.b { background: var(--cell-b); }
        .piece { width: 85%; height: 85%; border-radius: 50%; position: relative; z-index: 5; box-shadow: 0 4px 0 #000; transition: 0.3s; }
        .p-white { background: radial-gradient(circle at 30% 30%, #ffffff, #999); }
        .p-black { background: radial-gradient(circle at 30% 30%, #444, #000); }
        .piece.king::after { content: '👑'; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-size: 26px; }
        .selected { transform: scale(1.1); box-shadow: 0 0 20px var(--accent); z-index: 10; border: 2px solid white; }
        .hint { position: absolute; width: 35%; height: 35%; background: rgba(0, 229, 255, 0.6); border-radius: 50%; z-index: 8; animation: pulse 1s infinite; }
        @keyframes pulse { 0% { opacity: 0.4; } 50% { opacity: 0.8; } 100% { opacity: 0.4; } }
        #win-modal { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.95); z-index: 2000; flex-direction: column; align-items: center; justify-content: center; }
    </style>
</head>
<body>

    <div id="main-menu">
        <h1 style="color:var(--accent); text-shadow: 0 0 10px var(--accent);">SHASHKI PRO</h1>
        <div class="menu-btns" id="base-menu">
            <button class="btn" onclick="startMode('local')">👤 2 Игрока (Локально)</button>
            <button class="btn" onclick="toggleMenu('ai-menu')">🤖 Против Бота</button>
            <button class="btn" onclick="toggleMenu('online-menu')">🌐 Онлайн Игра</button>
        </div>
        <div class="menu-btns" id="ai-menu" style="display:none">
            <button class="btn" style="background:#4caf50" onclick="chooseColor(1)">Новичок</button>
            <button class="btn" style="background:#ff9800" onclick="chooseColor(3)">Мастер</button>
            <button class="btn" style="background:#000; border:2px solid gold" onclick="chooseColor(6)">ГРОССМЕЙСТЕР</button>
            <button class="btn" onclick="toggleMenu('base-menu')">Назад</button>
        </div>
        <div class="menu-btns" id="color-menu" style="display:none">
            <button class="btn" style="background:#fff; color:#000" onclick="startAi('white')">За Белых</button>
            <button class="btn" style="background:#333" onclick="startAi('black')">За Черных</button>
        </div>
        <div class="menu-btns" id="online-menu" style="display:none">
            <input type="text" id="rid" placeholder="Код комнаты" style="padding:15px; border-radius:10px; text-align:center; text-transform:uppercase;">
            <button class="btn" onclick="createRoom()">Создать комнату</button>
            <button class="btn" onclick="joinRoom()">Войти по коду</button>
            <button class="btn" onclick="toggleMenu('base-menu')">Назад</button>
        </div>
    </div>

    <div id="game-ui">
        <div class="status-bar">
            <div id="status">Загрузка...</div>
            <div id="room-display" style="color:var(--accent); font-weight:bold; margin-top:5px;"></div>
        </div>
        <div id="board-resizer"><div id="board"></div></div>
        <button class="btn" style="margin:20px; width:120px;" onclick="location.reload()">Выход</button>
    </div>

    <div id="win-modal">
        <div style="background:#1a1a1a; padding:40px; border-radius:20px; text-align:center; border:2px solid var(--accent);">
            <h2 id="win-title">ПОБЕДА!</h2>
            <p id="win-text"></p>
            <button class="btn" onclick="location.reload()">В МЕНЮ</button>
        </div>
    </div>

    <script>
        const fb = firebase.initializeApp({ apiKey: "AIzaSyCFJQ4OKKStxVW7gUj1lS4KHwxGc1_QIwg", databaseURL: "https://mymobilegame-b5e68-default-rtdb.firebaseio.com" });
        const db = firebase.database();
        let board = [], turn = 'white', selected = null, hints = [], chain = null;
        let mode = 'local', aiLvl = 1, myColor = 'white', roomId = null, isWaiting = false;

        function toggleMenu(id) {
            document.querySelectorAll('.menu-btns').forEach(m => m.style.display = 'none');
            document.getElementById(id).style.display = 'flex';
        }
        function chooseColor(lvl) { aiLvl = lvl; toggleMenu('color-menu'); }
        function startAi(col) { myColor = col; startMode('ai', aiLvl); }

        function startMode(m, lvl = 1) {
            mode = m; aiLvl = lvl;
            document.getElementById('main-menu').style.display = 'none';
            document.getElementById('game-ui').style.display = 'flex';
            if(mode !== 'online') { initBoard(); if(mode==='ai' && myColor==='black') setTimeout(aiMove, 600); }
        }

        function initBoard() {
            board = Array(8).fill(null).map(() => Array(8).fill(null));
            for(let r=0; r<8; r++) for(let c=0; c<8; c++) if((r+c)%2!==0) {
                if(r<3) board[r][c] = { color:'black', king:false };
                else if(r>4) board[r][c] = { color:'white', king:false };
            }
            render();
        }

        function render() {
            const bEl = document.getElementById('board'); bEl.innerHTML = '';
            const st = document.getElementById('status');
            if(isWaiting) { st.innerText = "ОЖИДАНИЕ ИГРОКА..."; return; }
            st.innerText = `Ход: ${turn==='white'?'Белых':'Черных'} ${mode!=='local'?(turn===myColor?'(Ваш)':'(Врага)'):''}`;
            
            board.forEach((row, r) => row.forEach((p, c) => {
                const cell = document.createElement('div');
                cell.className = `cell ${(r+c)%2===0?'w':'b'}`;
                if(p) {
                    const pEl = document.createElement('div');
                    pEl.className = `piece p-${p.color} ${p.king?'king':''}`;
                    if(selected && selected.r===r && selected.c===c) pEl.classList.add('selected');
                    pEl.onclick = () => handleSelect(r, c);
                    cell.appendChild(pEl);
                }
                const h = hints.find(h => h.toR===r && h.toC===c);
                if(h) {
                    const hEl = document.createElement('div'); hEl.className = 'hint';
                    hEl.onclick = () => execMove(h); cell.appendChild(hEl);
                }
                bEl.appendChild(cell);
            }));
            checkGameOver();
        }

        function handleSelect(r, c) {
            if(isWaiting || (mode!=='local' && turn!==myColor) || (chain && (chain.r!==r || chain.c!==c))) return;
            const p = board[r][c]; if(!p || p.color !== turn) return;
            const must = getMust(turn);
            hints = must.length ? must.filter(m => m.fromR===r && m.fromC===c) : getSimple(r, c);
            selected = {r, c}; render();
        }

        function getSimple(r, c) {
            const p = board[r][c], res = [], ds = [[1,1],[1,-1],[-1,1],[-1,-1]];
            for(let [dr, dc] of ds) {
                let nr = r+dr, nc = c+dc;
                if(p.king) while(nr>=0&&nr<8&&nc>=0&&nc<8&&!board[nr][nc]) { res.push({fromR:r,fromC:c,toR:nr,toC:nc}); nr+=dr; nc+=dc; }
                else if((p.color==='white'?dr===-1:dr===1) && nr>=0&&nr<8&&nc>=0&&nc<8&&!board[nr][nc]) res.push({fromR:r,fromC:c,toR:nr,toC:nc});
            }
            return res;
        }

        function getHits(r, c) {
            const p = board[r][c], res = [], ds = [[1,1],[1,-1],[-1,1],[-1,-1]];
            for(let [dr, dc] of ds) {
                let nr = r+dr, nc = c+dc, vic = null;
                if(p.king) while(nr>=0&&nr<8&&nc>=0&&nc<8) {
                    const t = board[nr][nc];
                    if(t) { if(t.color!==p.color && !vic) vic={r:nr,c:nc}; else break; }
                    else if(vic) res.push({fromR:r,fromC:c,toR:nr,toC:nc,vic});
                    nr+=dr; nc+=dc;
                } else {
                    let tr=r+dr*2, tc=c+dc*2;
                    if(tr>=0&&tr<8&&tc>=0&&tc<8) {
                        const v = board[nr][nc];
                        if(v && v.color!==p.color && !board[tr][tc]) res.push({fromR:r,fromC:c,toR:tr,toC:tc,vic:{r:nr,c:nc}});
                    }
                }
            }
            return res;
        }

        function getMust(col) {
            let res = []; for(let r=0;r<8;r++)for(let c=0;c<8;c++)if(board[r][c]&&board[r][c].color===col)res.push(...getHits(r,c));
            return res;
        }

        function execMove(m) {
            const p = board[m.fromR][m.fromC]; board[m.toR][m.toC] = p; board[m.fromR][m.fromC] = null;
            if(m.vic) board[m.vic.r][m.vic.c] = null;
            if(m.vic) {
                const next = getHits(m.toR, m.toC);
                if(next.length > 0) {
                    chain = {r:m.toR, c:m.toC}; selected = chain; hints = next; render(); sync();
                    if(mode==='ai' && turn!==myColor) setTimeout(() => execMove(next[0]), 600);
                    return;
                }
            }
            if((p.color==='white'&&m.toR===0)||(p.color==='black'&&m.toR===7)) p.king = true;
            chain = null; selected = null; hints = []; turn = turn==='white'?'black':'white';
            render(); sync();
            if(mode==='ai' && turn!==myColor) setTimeout(aiMove, 600);
        }

        function aiMove() {
            let ms = getMust(turn); if(!ms.length) for(let r=0;r<8;r++)for(let c=0;c<8;c++)if(board[r][c]&&board[r][c].color===turn)ms.push(...getSimple(r,c));
            if(!ms.length) return;
            if(aiLvl >= 5) ms.sort((a,b) => (b.vic?15:0) + (b.toR===0||b.toR===7?10:0) - (a.vic?15:0));
            execMove(ms[0]);
        }

        function checkGameOver() {
            let canMove = getMust(turn).length;
            if(!canMove) for(let r=0;r<8;r++)for(let c=0;c<8;c++)if(board[r][c]&&board[r][c].color===turn)canMove+=getSimple(r,c).length;
            if(canMove===0) {
                document.getElementById('win-text').innerText = `ПОБЕДИЛИ ${turn==='white'?'ЧЕРНЫЕ':'БЕЛЫЕ'}`;
                document.getElementById('win-modal').style.display = 'flex';
            }
        }

        function sync() { if(mode==='online' && !isWaiting) db.ref('rooms/'+roomId).set({ b: JSON.stringify(board), t: turn, s: 'play' }); }

        function createRoom() {
            roomId = Math.random().toString(36).substr(2,4).toUpperCase();
            myColor = 'white'; isWaiting = true;
            db.ref('rooms/'+roomId).set({ b: JSON.stringify(board), t: 'white', s: 'wait' });
            document.getElementById('room-display').innerText = "КОД: " + roomId;
            db.ref('rooms/'+roomId).on('value', snap => {
                const data = snap.val(); if(!data) return;
                if(data.s === 'play') isWaiting = false;
                board = JSON.parse(data.b); turn = data.t; render();
            });
            startMode('online');
        }

        function joinRoom() {
            roomId = document.getElementById('rid').value.toUpperCase();
            myColor = 'black';
            db.ref('rooms/'+roomId).once('value', snap => {
                if(snap.exists()) {
                    initBoard();
                    db.ref('rooms/'+roomId).update({ s: 'play', b: JSON.stringify(board) });
                    db.ref('rooms/'+roomId).on('value', s => {
                        const d = s.val(); board = JSON.parse(d.b); turn = d.t; render();
                    });
                    startMode('online');
                } else alert("Код не найден!");
            });
        }
    </script>
</body>
</html>
