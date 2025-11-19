# Bomber-2d-
Hehe



<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Jogo P2P - Bomber Friends (Rosa Oculto)</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
  /* Estilos Gerais */
  body {
    font-family: Arial, sans-serif;
    background-color: #0d1117; 
    color: white;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    height: 100vh; 
    margin: 0;
    overflow: hidden; 
  }
  h2 { 
    color: #58a6ff; 
    margin-bottom: 5px; 
    font-size: 1.2em; 
  }
  
  /* Contêiner de ID e Conexão (Topo) */
  #connectionPanel {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 10px;
  }
  #idContainer {
    display: flex;
    align-items: center;
    margin-bottom: 5px;
  }
  #meuId {
    font-size: 12px; 
    background: #21262d;
    padding: 4px 8px; 
    border-radius: 5px 0 0 5px; 
    user-select: text; 
    margin-right: -1px; 
  }
  #copyButton {
    background: #58a6ff; 
    color: black; 
    padding: 4px 8px; 
    border: none;
    border-radius: 0 5px 5px 0; 
    cursor: pointer; 
    font-size: 12px;
    font-weight: bold;
    transition: background 0.1s;
    height: 28px; 
    display: flex;
    align-items: center;
  }
  #copyButton:hover {
    background: #4688e3;
  }

  input, button { 
    padding: 6px; 
    margin: 3px; 
    border-radius: 5px; 
    border: none; 
    font-size: 0.9em;
  }
  .action-button { 
    background: #238636; 
    color: white; 
    cursor: pointer; 
    transition: background 0.1s;
  }
  #gameCanvas {
    background: #161b22; 
    border: 2px solid #58a6ff; 
    margin-top: 10px;
    outline: none; 
  }

  /* Controles Responsivos (Joystick/D-Pad) */
  .controls-container {
    margin-top: 10px;
    display: flex;
    justify-content: center;
    gap: 20px; 
    width: 100%;
    max-width: 500px;
  }
  .d-pad {
    display: grid;
    grid-template-areas: ". up ." "left . right" ". down .";
    grid-template-columns: 1fr 1fr 1fr;
    width: 150px; 
    height: 150px; 
  }
  .control-button {
    width: 100%;
    height: 100%;
    font-size: 1.5em; 
    margin: 0;
    background: #58a6ff; 
    color: black; 
    cursor: pointer; 
    user-select: none;
    border-radius: 5px;
    -webkit-tap-highlight-color: transparent; 
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .control-button.up    { grid-area: up;    }
  .control-button.down  { grid-area: down;  }
  .control-button.left  { grid-area: left;  }
  .control-button.right { grid-area: right; }

  #bombButton {
    align-self: center; 
    height: 70px; 
    width: 70px; 
    padding: 0;
    font-size: 2.5em; 
    background: #FF0077;
    border-radius: 50%; 
    display: flex;
    justify-content: center;
    align-items: center;
    user-select: none;
    -webkit-user-select: none; 
    -webkit-tap-highlight-color: transparent;
  }
</style>
</head>
<body>
  <h2>💣 Jogo P2P (Mobile Otimizado)</h2>
  
  <div id="connectionPanel">
      <div id="idContainer">
        <div id="meuId">Conectando...</div>
        <button id="copyButton" onclick="copyId()">Copiar ID</button>
      </div>
      <div>
        <input id="conectarId" placeholder="ID do amigo">
        <button class="action-button" onclick="conectar()">Conectar</button>
      </div>
  </div>

  <canvas id="gameCanvas" tabindex="0"></canvas>

  <div class="controls-container">
      <div class="d-pad">
        <button class="control-button up" 
                ontouchstart="move('up', true)" ontouchend="move('up', false)" 
                onmousedown="move('up', true)" onmouseup="move('up', false)">↑</button>
        <button class="control-button left" 
                ontouchstart="move('left', true)" ontouchend="move('left', false)" 
                onmousedown="move('left', true)" onmouseup="move('left', false)">←</button>
        <button class="control-button right" 
                ontouchstart="move('right', true)" ontouchend="move('right', false)" 
                onmousedown="move('right', true)" onmouseup="move('right', false)">→</button>
        <button class="control-button down" 
                ontouchstart="move('down', true)" ontouchend="move('down', false)" 
                onmousedown="move('down', true)" onmouseup="move('down', false)">↓</button>
      </div>
      
      <button id="bombButton" 
              ontouchstart="placeBombIntent()" onmousedown="placeBombIntent()">💣</button>
  </div>

<script>
  // --- Configurações do Jogo ---
  const CANVAS = document.getElementById("gameCanvas");
  const CTX = CANVAS.getContext("2d");
  const GRID_SIZE = 13; 
  const BOMB_TIMER = 3000;
  const EXPLOSION_RANGE = 2; // O alcance da explosão em tiles

  let TILE_SIZE;
  let PLAYER_SIZE;
  let SPEED;
  
  let myPlayer;
  let otherPlayer;

  // Posições de spawn fixas
  const SPAWN_TILE_A = { col: 1, row: 1 }; 
  const SPAWN_TILE_B = { col: GRID_SIZE - 2, row: GRID_SIZE - 2 }; 
  
  // Cores fixas
  const COLOR_A = '#238636'; // Verde
  const COLOR_B = '#FF0077'; // Rosa

  let gameMap = [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 2, 2, 2, 0, 2, 2, 2, 0, 0, 1],
    [1, 0, 1, 2, 1, 0, 1, 0, 1, 2, 1, 0, 1],
    [1, 2, 2, 2, 0, 2, 2, 2, 0, 2, 2, 2, 1],
    [1, 2, 1, 0, 1, 0, 1, 0, 1, 0, 1, 2, 1],
    [1, 2, 0, 2, 2, 2, 0, 2, 2, 2, 0, 2, 1],
    [1, 0, 1, 2, 1, 0, 1, 0, 1, 2, 1, 0, 1],
    [1, 2, 2, 2, 0, 2, 2, 2, 0, 2, 2, 2, 1],
    [1, 2, 1, 0, 1, 0, 1, 0, 1, 0, 1, 2, 1],
    [1, 2, 0, 2, 2, 2, 0, 2, 2, 2, 0, 2, 1],
    [1, 0, 1, 2, 1, 0, 1, 0, 1, 2, 1, 0, 1],
    [1, 0, 0, 2, 2, 2, 0, 2, 2, 2, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ];
  
  let bombs = [];
  let keys = { up: false, down: false, left: false, right: false };
  let bombIntent = false;
  let explosionEffectTiles = [];
  let animationFrameId; 

  let isPlayerA = null; 
  
  // --- Lógica Responsiva e Atribuição de Papel ---

  function setPlayerRole(isInitiator) {
      const playerOffset = (TILE_SIZE - PLAYER_SIZE) / 2;
      
      if (isInitiator) {
          isPlayerA = true;
          myPlayer = { 
              x: SPAWN_TILE_A.col * TILE_SIZE + playerOffset, 
              y: SPAWN_TILE_A.row * TILE_SIZE + playerOffset, 
              color: COLOR_A, 
              currentTile: SPAWN_TILE_A
          };
          otherPlayer = { 
              x: SPAWN_TILE_B.col * TILE_SIZE + playerOffset, 
              y: SPAWN_TILE_B.row * TILE_SIZE + playerOffset, 
              color: COLOR_B,
              currentTile: SPAWN_TILE_B,
              // *** ADICIONADO: Estado de conexão inicial (Oculto) ***
              isConnected: false 
          };
      } 
      else {
          isPlayerA = false;
          myPlayer = { 
              x: SPAWN_TILE_B.col * TILE_SIZE + playerOffset, 
              y: SPAWN_TILE_B.row * TILE_SIZE + playerOffset, 
              color: COLOR_B, 
              currentTile: SPAWN_TILE_B
          };
          otherPlayer = { 
              x: SPAWN_TILE_A.col * TILE_SIZE + playerOffset, 
              y: SPAWN_TILE_A.row * TILE_SIZE + playerOffset, 
              color: COLOR_A, 
              currentTile: SPAWN_TILE_A,
              // *** ADICIONADO: Estado de conexão inicial (Oculto) ***
              isConnected: false 
          };
      }
      
      if (!animationFrameId) {
          gameLoop();
      }
  }

  function setupResponsiveCanvas() {
    const availableSize = Math.min(window.innerWidth, window.innerHeight) - 120;
    const size = Math.min(availableSize, 520);
    CANVAS.width = size;
    CANVAS.height = size;
    
    TILE_SIZE = CANVAS.width / GRID_SIZE;
    PLAYER_SIZE = TILE_SIZE * 0.75; 
    SPEED = TILE_SIZE * 0.05;

    if (!myPlayer) {
        setPlayerRole(false); 
        myPlayer.color = COLOR_A; 
        otherPlayer.color = COLOR_B; 
    } else {
        const playerOffset = (TILE_SIZE - PLAYER_SIZE) / 2;
        
        const myGrid = pixelToGrid(myPlayer.x + PLAYER_SIZE / 2, myPlayer.y + PLAYER_SIZE / 2);
        myPlayer.x = myGrid.col * TILE_SIZE + playerOffset;
        myPlayer.y = myGrid.row * TILE_SIZE + playerOffset;

        // A posição do outro jogador só é ajustada se ele estiver conectado
        if (otherPlayer.isConnected) {
            const otherGrid = pixelToGrid(otherPlayer.x + PLAYER_SIZE / 2, otherPlayer.y + PLAYER_SIZE / 2);
            otherPlayer.x = otherGrid.col * TILE_SIZE + playerOffset;
            otherPlayer.y = otherGrid.row * TILE_SIZE + playerOffset;
        }
    }
  }
  
  // --- Funções Auxiliares e Bomba/Explosão ---
  
  function pixelToGrid(x, y) {
      return {
          col: Math.floor(x / TILE_SIZE),
          row: Math.floor(y / TILE_SIZE)
      };
  }
  
  function isBlocked(col, row) {
      if (row < 0 || row >= GRID_SIZE || col < 0 || col >= GRID_SIZE) {
          return true;
      }
      const tile = gameMap[row][col];
      return tile === 1 || tile === 2;
  }
  
  function placeBombIntent() {
      if (!bombIntent) {
          bombIntent = true;
      }
  }

  function placeBomb() {
      const { col, row } = pixelToGrid(myPlayer.x + PLAYER_SIZE / 2, myPlayer.y + PLAYER_SIZE / 2);
      
      const existingBomb = bombs.find(b => b.col === col && b.row === row);
      
      if (!existingBomb) {
          const newBomb = {
              col: col,
              row: row,
              time: Date.now() + BOMB_TIMER,
              ownerId: peer.id, 
              ownerTile: { col: col, row: row } 
          };
          bombs.push(newBomb);
          enviarBomba(newBomb);
      }
      bombIntent = false;
  }

  // Função central que calcula as tiles afetadas pela explosão
  function calculateExplosionTiles(col, row) {
      let tilesToClear = [{ col, row }]; 
      const directions = [{ dc: 0, dr: -1 }, { dc: 0, dr: 1 }, { dc: -1, dr: 0 }, { dc: 1, dr: 0 }];
      
      directions.forEach(dir => {
          for (let i = 1; i <= EXPLOSION_RANGE; i++) {
              const nextCol = col + dir.dc * i;
              const nextRow = row + dir.dr * i;
              
              if (nextCol < 0 || nextCol >= GRID_SIZE || nextRow < 0 || nextRow >= GRID_SIZE) {
                  break; 
              }
              
              const tile = gameMap[nextRow][nextCol];
              
              if (tile === 1) { // Muro sólido, para o fogo
                  break; 
              }
              
              tilesToClear.push({ col: nextCol, row: nextRow });
              
              if (tile === 2) { // Muro destrutível, destrói e para o fogo nessa direção
                  break; 
              }
          }
      });
      
      return tilesToClear;
  }

  // Função que lida com a explosão (recursiva para reação em cadeia)
  function handleExplosion(bomb) {
      const { col, row } = bomb;
      
      bombs = bombs.filter(b => b !== bomb);
      const tilesAffected = calculateExplosionTiles(col, row);
      
      let newlyExplodedTiles = [];
      tilesAffected.forEach(tile => {
          const { col: tCol, row: tRow } = tile;
          
          if (gameMap[tRow] && gameMap[tRow][tCol] === 2) {
              gameMap[tRow][tCol] = 0; 
              enviarDestruicaoMuro(tCol, tRow);
          }
          newlyExplodedTiles.push(tile);
      });
      
      showExplosionEffect(newlyExplodedTiles);

      // VERIFICA IMPACTO
      checkPlayerHit(myPlayer, newlyExplodedTiles);
      
      // Só checa o outro jogador se ele estiver conectado
      if (otherPlayer.isConnected) {
          checkPlayerHit(otherPlayer, newlyExplodedTiles);
      }


      // REAÇÃO EM CADEIA: Verifica outras bombas atingidas
      let chainBombs = [];
      for (let i = bombs.length - 1; i >= 0; i--) {
          const otherBomb = bombs[i];
          const isHit = newlyExplodedTiles.some(tile => 
              tile.col === otherBomb.col && tile.row === otherBomb.row
          );

          if (isHit) {
              chainBombs.push(otherBomb);
          }
      }
      
      chainBombs.forEach(handleExplosion); // Chama recursivamente para a reação em cadeia
  }
  
  function showExplosionEffect(tiles) {
      explosionEffectTiles.push({ tiles: tiles, endTime: Date.now() + 200 }); 
  }

  // FUNÇÃO DE COLISÃO DO JOGADOR (Mantida sem teleporte)
  function checkPlayerHit(player, explosionTiles) {
      const { col, row } = player.currentTile;

      const isHit = explosionTiles.some(tile => tile.col === col && tile.row === row);
      
      if (isHit) {
          
          if (player === myPlayer) {
              
              // 1. Reseta o movimento para o jogador parar imediatamente
              keys.up = keys.down = keys.left = keys.right = false;
              
              // 2. Não move para o spawn, apenas para o centro do tile
              const playerOffset = (TILE_SIZE - PLAYER_SIZE) / 2;
              player.x = col * TILE_SIZE + playerOffset;
              player.y = row * TILE_SIZE + playerOffset;
              
              // 3. Informa ao outro jogador que fui atingido e parei.
              enviarPosicao(); 
              
          } else {
             // O outro jogador será sincronizado pela rede
          }
      }
  }

  // --- Game Loop (Lógica + Desenho) ---
  
  function update() {
    let moved = false;
    let newX = myPlayer.x;
    let newY = myPlayer.y;
    
    myPlayer.currentTile = pixelToGrid(myPlayer.x + PLAYER_SIZE / 2, myPlayer.y + PLAYER_SIZE / 2);

    // Se o jogador não estiver imobilizado (keys false), processa o movimento
    if (keys.up || keys.down || keys.left || keys.right) {
        // 1. Movimento (Apenas do myPlayer)
        if (keys.up) newY -= SPEED;
        if (keys.down) newY += SPEED;
        if (keys.left) newX -= SPEED;
        if (keys.right) newX += SPEED;
    }


    // 2. Colisão
    
    const { col: targetCol, row: targetRow } = pixelToGrid(newX + PLAYER_SIZE / 2, newY + PLAYER_SIZE / 2);

    let collision = false;
    
    // Atualiza o status de posse da bomba
    bombs.forEach(bomb => {
        if (bomb.ownerTile && bomb.ownerId === peer.id) {
             const { col, row } = myPlayer.currentTile;
             if (bomb.ownerTile.col !== col || bomb.ownerTile.row !== row) {
                 bomb.ownerTile = null; 
             }
        }
    });

    if (isBlocked(targetCol, targetRow)) {
        collision = true;
    }
    
    // Colisão com bombas
    const collidingBomb = bombs.find(b => b.col === targetCol && b.row === targetRow);

    if (collidingBomb) {
        if (collidingBomb.ownerId === peer.id && collidingBomb.ownerTile) {
            // Permite a passagem se for sua bomba e você ainda estiver em cima
        } else {
            collision = true;
        }
    }


    if (!collision) {
        if (newX !== myPlayer.x || newY !== myPlayer.y) {
            moved = true;
        }
        myPlayer.x = newX;
        myPlayer.y = newY;
    } 

    myPlayer.x = Math.max(0, Math.min(CANVAS.width - PLAYER_SIZE, myPlayer.x));
    myPlayer.y = Math.max(0, Math.min(CANVAS.height - PLAYER_SIZE, myPlayer.y));
    
    // 4. Colocação de Bomba
    if (bombIntent) {
        placeBomb();
    }

    // 5. Verificar Explosões (APENAS pelo tempo)
    const now = Date.now();
    for (let i = bombs.length - 1; i >= 0; i--) {
        const bomb = bombs[i];
        if (now >= bomb.time) {
            handleExplosion(bomb);
        }
    }
    
    // 6. Sincronizar Posição
    if (moved || (keys.up === false && keys.down === false && keys.left === false && keys.right === false)) {
        enviarPosicao();
    }
  }

  // --- Funções de Desenho ---

  function draw() {
    CTX.clearRect(0, 0, CANVAS.width, CANVAS.height);
    
    drawMap();
    drawExplosion();
    bombs.forEach(drawBomb);

    // *** ALTERADO: Só desenha o outro jogador se ele estiver conectado ***
    if (otherPlayer.isConnected) {
        drawPlayer(otherPlayer); 
    }

    drawPlayer(myPlayer);
  }
  
  function gameLoop() {
    update();
    draw();
    animationFrameId = requestAnimationFrame(gameLoop);
  }
  
  function drawMap() {
      for (let row = 0; row < GRID_SIZE; row++) {
          for (let col = 0; col < GRID_SIZE; col++) {
              const tile = gameMap[row][col];
              if (tile === 1) {
                  drawTile(col, row, '#373e47');
              } else if (tile === 2) {
                  drawTile(col, row, '#795548');
              }
          }
      }
  }
  
  function drawTile(col, row, color) {
    CTX.fillStyle = color;
    CTX.fillRect(col * TILE_SIZE, row * TILE_SIZE, TILE_SIZE, TILE_SIZE);
  }
  
  function drawPlayer(player) {
    CTX.fillStyle = player.color;
    CTX.fillRect(player.x, player.y, PLAYER_SIZE, PLAYER_SIZE);
  }
  
  function drawBomb(bomb) {
      // O pulso da bomba fica mais rápido
      const pulseFactor = 150 - (bomb.time - Date.now()) / 20; 
      const pulse = Math.sin(Date.now() / Math.max(50, pulseFactor)) * (TILE_SIZE * 0.1) + TILE_SIZE * 0.3; 
      
      const cx = bomb.col * TILE_SIZE + TILE_SIZE / 2;
      const cy = bomb.row * TILE_SIZE + TILE_SIZE / 2;

      CTX.fillStyle = 'black'; 
      CTX.beginPath();
      CTX.arc(cx, cy, TILE_SIZE * 0.45, 0, Math.PI * 2);
      CTX.fill();
      
      CTX.fillStyle = 'red';
      CTX.beginPath();
      CTX.arc(cx, cy, pulse, 0, Math.PI * 2);
      CTX.fill();
  }

  function drawExplosion() {
      const now = Date.now();
      explosionEffectTiles = explosionEffectTiles.filter(effect => {
          if (now < effect.endTime) {
              CTX.fillStyle = 'rgba(255, 165, 0, 0.8)'; 
              effect.tiles.forEach(tile => {
                  CTX.fillRect(tile.col * TILE_SIZE, tile.row * TILE_SIZE, TILE_SIZE, TILE_SIZE);
              });
              return true; 
          }
          return false; 
      });
  }

  // --- Controles (Teclado e Toque/Mouse) ---

  document.addEventListener('keydown', (e) => handleKey(e.key, true));
  document.addEventListener('keyup', (e) => handleKey(e.key, false));

  function handleKey(key, isPressed) {
    if (key === 'ArrowUp' || key === 'w') keys.up = isPressed;
    else if (key === 'ArrowDown' || key === 's') keys.down = isPressed;
    else if (key === 'ArrowLeft' || key === 'a') keys.left = isPressed;
    else if (key === 'ArrowRight' || key === 'd') keys.right = isPressed;
    else if (key === ' ' || key === 'b') {
        if (isPressed) {
            placeBombIntent();
        }
    }
  }
  
  function move(direction, isPressed) {
      keys[direction] = isPressed;
  }
  
  window.addEventListener('resize', () => {
      cancelAnimationFrame(animationFrameId);
      setupResponsiveCanvas();
      gameLoop();
  });

  // --- Função Copiar ID ---
  function copyId() {
      const idText = peer.id;
      if (navigator.clipboard) {
          navigator.clipboard.writeText(idText).then(() => {
              const button = document.getElementById('copyButton');
              button.textContent = 'Copiado!';
              setTimeout(() => {
                  button.textContent = 'Copiar ID';
              }, 1000);
          }).catch(err => {
              console.error('Falha ao copiar:', err);
          });
      } else {
          alert("Por favor, selecione e copie o ID: " + idText);
      }
  }


  // --- P2P (PeerJS) ---
  
  const peer = new Peer();
  const meuIdElement = document.getElementById("meuId");
  const conectarInput = document.getElementById("conectarId");

  let conexao;
  
  peer.on("open", id => {
    meuIdElement.innerHTML = `Seu ID: <b>${id}</b>`;
  });

  peer.on("connection", conn => {
    conexao = conn;
    
    // Se recebeu a conexão, ele é o Jogador B
    setPlayerRole(false);
    
    conexao.on("data", receberDados);
    conexao.on("open", () => {
        meuIdElement.innerHTML = `✅ Conectado como Jogador B (Rosa)! ID: <b>${peer.id}</b>`;
        // *** HABILITA O JOGADOR ROS
