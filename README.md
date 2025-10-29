<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>○×ゲーム（消える手ルール） — CPU強化版</title>
<style>
  body{font-family:system-ui,-apple-system,Segoe UI,Roboto,"Hiragino Kaku Gothic ProN",Meiryo,sans-serif;background:#0b1220;color:#e6eef8;display:flex;align-items:center;justify-content:center;height:100vh;margin:0;padding:20px}
  .card{width:min(680px,96vw);background:#0f1724;border-radius:12px;padding:18px;box-shadow:0 8px 30px #0008}
  h1{margin:0 0 8px;font-size:18px}
  .meta{color:#9fb0c8;font-size:13px;margin-bottom:12px}
  .controls{display:flex;gap:8px;flex-wrap:wrap;margin-bottom:12px}
  button{background:#0b2340;border:1px solid #23466a;color:#dff0ff;padding:8px 10px;border-radius:8px;cursor:pointer;font-weight:700}
  button.active{background:#1b6fb3;border-color:#2ea0ff}
  .board{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;width:360px;height:360px;margin:10px 0}
  .cell{background:#071026;border-radius:10px;display:flex;align-items:center;justify-content:center;font-size:64px;font-weight:800;cursor:pointer;user-select:none}
  .cell.taken{cursor:not-allowed;opacity:0.9}
  .status{margin-top:8px;color:#d7eefc;font-weight:700}
  .small{font-size:13px;color:#9fb0c8}
  .row{display:flex;gap:8px;align-items:center}
  .footer{display:flex;justify-content:space-between;align-items:center;margin-top:12px}
  select{padding:8px;border-radius:8px;background:#071026;border:1px solid #233b52;color:#dff0ff}
  .hint{color:#9fb0c8;font-size:13px}
  @media(max-width:520px){.board{width:96vw;height:96vw}.cell{font-size:clamp(36px,8vw,64px)}}
</style>
</head>
<body>
<div class="card">
  <h1>４手目で１手目が消える ○×ゲーム</h1>
  <div class="meta">ルール：各プレイヤーは盤上に最大3個まで保持。4手目を置くとそのプレイヤーの最古の手が消えます。</div>

  <div class="controls">
    <div class="row">
      <button id="pvpBtn" class="active">対人（CPU OFF）</button>
      <button id="mediumBtn">中級CPU</button>
      <button id="strongBtn">最強CPU</button>
    </div>
    <div style="margin-left:auto">
      <button id="resetBtn">リセット</button>
      <button id="swapBtn">先手/後手 入れ替え</button>
    </div>
  </div>

  <div class="board" id="board" role="grid" aria-label="tic tac toe board"></div>

  <div class="status" id="status">先手（X）の番です</div>
  <div class="footer">
    <div class="small">X: 先手　O: 後手（CPUはO）</div>
    <div class="hint">※ モード切替は次のゲームから有効</div>
  </div>
</div>

<script>
/* ========= ゲーム定義 ========= */
const LINES = [[0,1,2],[3,4,5],[6,7,8],[0,3,6],[1,4,7],[2,5,8],[0,4,8],[2,4,6]];
const boardEl = document.getElementById('board');
const statusEl = document.getElementById('status');
const pvpBtn = document.getElementById('pvpBtn');
const mediumBtn = document.getElementById('mediumBtn');
const strongBtn = document.getElementById('strongBtn');
const resetBtn = document.getElementById('resetBtn');
const swapBtn = document.getElementById('swapBtn');

let state = null; // ゲーム状態オブジェクト
let cpuMode = 'OFF'; // 'OFF' | 'MEDIUM' | 'STRONG'
let startingTurn = 'X'; // 初期の先手

function makeEmptyState(start='X'){
  return {
    board: Array(9).fill(null),
    xQueue: [],
    oQueue: [],
    turn: start,
    playing: true
  };
}

/* ======= 初期表示 ======= */
function renderBoardDOM(){
  boardEl.innerHTML = '';
  for(let i=0;i<9;i++){
    const div = document.createElement('div');
    div.className = 'cell';
    div.dataset.index = i;
    div.addEventListener('click', ()=>onCellClick(i));
    boardEl.appendChild(div);
  }
  refreshBoardDOM();
}
function refreshBoardDOM(){
  for(let i=0;i<9;i++){
    const div = boardEl.querySelector(`[data-index="${i}"]`);
    const val = state.board[i];
    div.textContent = val || '';
    div.classList.toggle('taken', !!val);
  }
  updateStatusText();
}

/* ======= 基本操作（配置・消去・勝利判定） ======= */
// place a mark on a copy of state — we will often simulate moves, so create pure functions
function cloneState(s){
  return {
    board: s.board.slice(),
    xQueue: s.xQueue.slice(),
    oQueue: s.oQueue.slice(),
    turn: s.turn,
    playing: s.playing
  };
}
// apply move to state in-place
function applyMoveInPlace(s, idx, player){
  s.board[idx] = player;
  const q = (player==='X') ? s.xQueue : s.oQueue;
  q.push(idx);
  if(q.length > 3){
    const oldest = q.shift();
    s.board[oldest] = null;
    // also remove the oldest index from the other queue? no, it's only own queue
  }
}
// simulate move returns new state object
function simulateMove(s, idx, player){
  const ns = cloneState(s);
  applyMoveInPlace(ns, idx, player);
  ns.turn = (player==='X') ? 'O' : 'X';
  return ns;
}
function isWinBoard(board, player){
  return LINES.some(line => line.every(i => board[i] === player));
}
function stateIsWin(s, player){
  return isWinBoard(s.board, player);
}

/* ======= プレイヤー操作 ======= */
function onCellClick(idx){
  if(!state.playing) return;
  if(state.board[idx]) return;
  if(cpuMode !== 'OFF' && state.turn === 'O') return; // CPUの番に人は打てない
  // place mark in real state
  applyMoveInPlace(state, idx, state.turn);
  refreshBoardDOM();
  if(stateIsWin(state, state.turn)){
    state.playing = false;
    statusEl.textContent = state.turn + ' の勝ち！';
    return;
  }
  // 次の手番
  state.turn = (state.turn === 'X') ? 'O' : 'X';
  refreshBoardDOM();
  // CPUの手番なら遅延して打たせる
  if(cpuMode !== 'OFF' && state.turn === 'O' && state.playing){
    setTimeout(()=>cpuMoveWrapper(cpuMode), 220);
  }
}

/* ======= CPU 呼び出しラッパー ======= */
function cpuMoveWrapper(mode){
  if(mode === 'MEDIUM') cpuMoveMedium();
  else if(mode === 'STRONG') cpuMoveStrong();
}

/* ======= 中級AI（1手先＋消えるを考慮） ======= */
function cpuMoveMedium(){
  // 1) 自分が置いて即勝てるならそれ
  let move = findImmediateWinningMove(state, 'O');
  // 2) なければ相手の即勝をブロック（相手が次に置いて勝てる場所）
  if(move === null){
    move = findImmediateWinningMove(state, 'X');
  }
  // 3) なければ、消える影響を簡易評価して有利な手を探す（たとえば、置いても自分の重要な既存手が消えない手）
  if(move === null){
    move = chooseBySurvivalPriority(state);
  }
  // 4) それでもなければ優先位置（中央→隅→端）
  if(move === null){
    if(state.board[4] === null) move = 4;
  }
  if(move === null){
    const corners = [0,2,6,8].filter(i=>state.board[i]===null);
    if(corners.length) move = corners[Math.floor(Math.random()*corners.length)];
  }
  if(move === null){
    const empties = emptyIndices(state);
    if(empties.length) move = empties[0];
  }

  if(move !== null){
    applyMoveInPlace(state, move, 'O');
    refreshBoardDOM();
    if(stateIsWin(state, 'O')){ state.playing=false; statusEl.textContent = 'O の勝ち！'; return; }
    state.turn = 'X';
    refreshBoardDOM();
  }
}
function findImmediateWinningMove(s, player){
  const empties = emptyIndices(s);
  for(const idx of empties){
    const ns = simulateMove(s, idx, player);
    if(stateIsWin(ns, player)) return idx;
  }
  return null;
}
function emptyIndices(s){
  return s.board.map((v,i)=> v? null : i).filter(v=>v!==null);
}
// choose move that minimizes loss of "important" existing marks: prefer move that doesn't cause removal of a recent critical mark
function chooseBySurvivalPriority(s){
  const empties = emptyIndices(s);
  // score each move: +10 if doesn't remove any of our existing marks, +5 if removes the oldest (worse), -inf if leads to opponent immediate win next turn
  let best = null;
  let bestScore = -Infinity;
  for(const idx of empties){
    const ns = simulateMove(s, idx, 'O');
    // if opponent can immediately win after this simulated move, heavily penalize
    if(findImmediateWinningMove(ns, 'X') !== null){
      continue;
    }
    // score by whether any of O's marks were removed by this move (we prefer not to lose many)
    const afterQueue = ns.oQueue.slice();
    let removed = 0;
    // compare counts: if original had 3 and new has 3 but indices differ -> one removed
    const orig = s.oQueue.slice();
    for(const v of orig){
      if(!afterQueue.includes(v)) removed++;
    }
    let score = 0;
    if(removed === 0) score += 10;
    else if(removed === 1) score += 2;
    // prefer center and corners slightly
    if(idx === 4) score += 3;
    if([0,2,6,8].includes(idx)) score += 1;
    if(score > bestScore){ bestScore = score; best = idx; }
  }
  return best;
}

/* ======= 最強AI：MiniMax（消えるルールを含める） ======= */
/* 状態の表現は board + xQueue + oQueue + turn をキーにしてメモ化 */
function cpuMoveStrong(){
  // Use minimax with memoization and alpha-beta; CPU plays 'O' maximizing score
  const key = stateKey(state);
  // find best move
  const empties = emptyIndices(state);
  if(empties.length === 0) return;
  let bestMove = null;
  let bestVal = -Infinity;
  for(const idx of empties){
    const ns = simulateMove(state, idx, 'O');
    const val = minimax(ns, 0, -Infinity, Infinity, false); // next is X (minimizer)
    if(val > bestVal){ bestVal = val; bestMove = idx; }
  }
  if(bestMove !== null){
    applyMoveInPlace(state, bestMove, 'O');
    refreshBoardDOM();
    if(stateIsWin(state, 'O')){ state.playing=false; statusEl.textContent = 'O の勝ち！'; return; }
    state.turn = 'X';
    refreshBoardDOM();
  }
}

/* 評価関数:
   勝ち: +100 - depth, 負け: -100 + depth, 引き分け: 0
   ここで depth を加味して高速勝利を好む
*/
const memo = new Map();
function stateKey(s){
  // represent queues in order to include removal order
  return JSON.stringify({b:s.board, xq:s.xQueue, oq:s.oQueue, t:s.turn});
}

function terminalScore(s, depth){
  if(stateIsWin(s, 'O')) return 100 - depth;
  if(stateIsWin(s, 'X')) return -100 + depth;
  // if no empties -> draw-ish
  if(emptyIndices(s).length === 0) return 0;
  return null;
}

function minimax(s, depth, alpha, beta, maximizingPlayer){
  const t = terminalScore(s, depth);
  if(t !== null) return t;
  const key = stateKey(s);
  if(memo.has(key)) return memo.get(key);

  // Prevent runaway recursion: optional depth cap (but we prefer full search). We'll cap at 12 plies for safety.
  const MAX_DEPTH = 12;
  if(depth >= MAX_DEPTH){
    // heuristic evaluation: count potential winning lines for O minus for X
    const h = heuristicEval(s);
    memo.set(key, h);
    return h;
  }

  const empties = emptyIndices(s);
  let best;
  if(maximizingPlayer){
    best = -Infinity;
    for(const idx of empties){
      const ns = simulateMove(s, idx, 'O');
      const val = minimax(ns, depth+1, alpha, beta, false);
      if(val > best) best = val;
      if(best > alpha) alpha = best;
      if(beta <= alpha) break; // prune
    }
    memo.set(key, best);
    return best;
  } else {
    best = Infinity;
    for(const idx of empties){
      const ns = simulateMove(s, idx, 'X');
      const val = minimax(ns, depth+1, alpha, beta, true);
      if(val < best) best = val;
      if(best < beta) beta = best;
      if(beta <= alpha) break;
    }
    memo.set(key, best);
    return best;
  }
}

function heuristicEval(s){
  // simple heuristic: count lines where O has advantage minus lines where X has advantage
  let score = 0;
  for(const line of LINES){
    const marks = line.map(i=>s.board[i]);
    const oCount = marks.filter(m=>m==='O').length;
    const xCount = marks.filter(m=>m==='X').length;
    if(xCount === 0 && oCount > 0) score += Math.pow(10, oCount);
    if(oCount === 0 && xCount > 0) score -= Math.pow(10, xCount);
  }
  return score;
}

/* ======= UI / モード切替など ======= */
function setMode(mode){
  cpuMode = mode;
  pvpBtn.classList.toggle('active', mode==='OFF');
  mediumBtn.classList.toggle('active', mode==='MEDIUM');
  strongBtn.classList.toggle('active', mode==='STRONG');
  // if switching to CPU and it's currently O's turn, let CPU play (start fresh)
  // We'll reset game to ensure consistent start
  init(startingTurn);
}

pvpBtn.addEventListener('click', ()=> setMode('OFF'));
mediumBtn.addEventListener('click', ()=> setMode('MEDIUM'));
strongBtn.addEventListener('click', ()=> setMode('STRONG'));
resetBtn.addEventListener('click', ()=> init(startingTurn));
swapBtn.addEventListener('click', ()=>{
  startingTurn = (startingTurn === 'X') ? 'O' : 'X';
  init(startingTurn);
});

/* ======= 補助関数 ======= */
function updateStatusText(){
  if(!state.playing) return;
  statusEl.textContent = (state.turn==='X') ? '先手（X）の番です' : '後手（O）の番です';
}

/* ======= 初期化 ======= */
function init(start='X'){
  state = makeEmptyState(start);
  renderBoardDOM();
  memo.clear();
  // if CPU is ON and CPU is to move immediately (O), trigger CPU move
  if(cpuMode !== 'OFF' && state.turn === 'O'){
    setTimeout(()=>cpuMoveWrapper(cpuMode), 250);
  }
}

/* ======= 公開 helper for medium: find move that leads to no immediate opponent win considering removal ======= */
/* (Already implemented above as chooseBySurvivalPriority) */

/* ======= 起動 ======= */
setMode('OFF'); // defaults; this also calls init
// expose init to console for debug:
// window._game = { init };
</script>
</body>
</html>
