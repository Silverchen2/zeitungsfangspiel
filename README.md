# Zeitungsfangspiel
<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <title>Fang die Zeitung! (Gamified)</title>
  <style>
    /* === Grundlayout === */
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      font-family: sans-serif;
      background: #e6f0f7;
      overflow: hidden;
    }
    #gameArea {
      position: relative;
      width: 100%;
      height: 100vh;
      background: linear-gradient(to top, #d0e6f0, #ffffff);
    }

    /* === Start- und Game-Over-Bildschirm === */
    .overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.6);
      color: #fff;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      z-index: 10;
    }
    .overlay.hidden {
      display: none;
    }
    .overlay h1 {
      font-size: 3em;
      margin-bottom: 0.5em;
    }
    .overlay button {
      padding: 0.8em 1.5em;
      font-size: 1.2em;
      margin-top: 1em;
      border: none;
      border-radius: 8px;
      background: #ffcc00;
      cursor: pointer;
      transition: background 0.2s;
    }
    .overlay button:hover {
      background: #e6b800;
    }

    /* === Spieler (Zeitungswagen) === */
    #player {
      width: 100px;
      height: 40px;
      background: #444;
      position: absolute;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      border-radius: 10px;
    }

    /* === Zeitung (fallendes Objekt) === */
    .newspaper {
      width: 40px;
      height: 40px;
      background: url('https://cdn-icons-png.flaticon.com/512/21/21601.png') no-repeat center;
      background-size: contain;
      position: absolute;
      top: 0;
    }

    /* === Anzeige für Punkte, Leben, Level === */
    #hud {
      position: absolute;
      top: 10px;
      left: 50%;
      transform: translateX(-50%);
      display: flex;
      gap: 2em;
      font-size: 1.2em;
      font-weight: bold;
      color: #333;
    }
    #hud span {
      background: rgba(255,255,255,0.8);
      padding: 0.3em 0.6em;
      border-radius: 5px;
    }
  </style>
</head>
<body>

  <div id="gameArea">
    <!-- HUD: Punkte, Leben, Level -->
    <div id="hud">
      <span id="scoreDisplay">Punkte: 0</span>
      <span id="livesDisplay">Leben: 3</span>
      <span id="levelDisplay">Level: 1</span>
    </div>

    <!-- Spieler: der Zeitungswagen -->
    <div id="player"></div>

    <!-- Startbildschirm -->
    <div id="startScreen" class="overlay">
      <h1>Fang die Zeitung!</h1>
      <p>Hilf dem Azubi, so viele Zeitungen wie möglich zu fangen!</p>
      <button id="startButton">Spiel starten</button>
    </div>

    <!-- Game-Over-Bildschirm -->
    <div id="gameOverScreen" class="overlay hidden">
      <h1>Game Over</h1>
      <p id="finalScore">Du hast <span>0</span> Punkte erreicht</p>
      <button id="restartButton">Erneut spielen</button>
    </div>
  </div>

  <script>
    // === Variablen ===
    const gameArea = document.getElementById('gameArea');
    const player = document.getElementById('player');
    const scoreDisplay = document.getElementById('scoreDisplay');
    const livesDisplay = document.getElementById('livesDisplay');
    const levelDisplay = document.getElementById('levelDisplay');
    const startScreen = document.getElementById('startScreen');
    const gameOverScreen = document.getElementById('gameOverScreen');
    const startButton = document.getElementById('startButton');
    const restartButton = document.getElementById('restartButton');
    const finalScoreText = document.querySelector('#finalScore span');

    let score = 0;
    let lives = 3;
    let level = 1;
    let playerX = window.innerWidth / 2 - 50;
    let newspaperInterval;       // Interval zum Erzeugen von Zeitungen
    let animationIntervals = []; // Array für alle laufenden Intervalle

    // === Spiellogik starten ===
    startButton.addEventListener('click', () => {
      startScreen.classList.add('hidden');
      resetGame();
      startGameLoop();
    });

    restartButton.addEventListener('click', () => {
      gameOverScreen.classList.add('hidden');
      resetGame();
      startGameLoop();
    });

    // === Spiel zurücksetzen ===
    function resetGame() {
      // Punkte, Leben, Level zurücksetzen
      score = 0;
      lives = 3;
      level = 1;
      updateHUD();

      // Wenn beim Neustart noch Zeitungen im DOM sind, entfernen
      document.querySelectorAll('.newspaper').forEach(el => el.remove());

      // Alle bestehenden Intervalle beenden
      animationIntervals.forEach(id => clearInterval(id));
      animationIntervals = [];
      clearInterval(newspaperInterval);
    }

    // === HUD (Anzeige) aktualisieren ===
    function updateHUD() {
      scoreDisplay.textContent = 'Punkte: ' + score;
      livesDisplay.textContent = 'Leben: ' + lives;
      levelDisplay.textContent = 'Level: ' + level;
    }

    // === Haupt-Spielschleife: Zeitungen erzeugen ===
    function startGameLoop() {
      // Falls Spiel vorbei ist, beende
      if (lives <= 0) return;

      // Erzeuge zu Beginn und dann in Abständen fallende Zeitungen
      createNewspaper();
      let spawnRate = Math.max(1000 - (level - 1) * 100, 300); 
      // mit jedem Level um 100 ms schneller (min. 300ms)

      newspaperInterval = setInterval(createNewspaper, spawnRate);
    }

    // === Spielersteuerung: Links/Rechts mit Pfeiltasten ===
    document.addEventListener('keydown', e => {
      if (e.key === 'ArrowLeft') {
        playerX -= 30;
      }
      if (e.key === 'ArrowRight') {
        playerX += 30;
      }
      // Spieler innerhalb des Bildschirms halten
      playerX = Math.max(0, Math.min(playerX, window.innerWidth - 100));
      player.style.left = playerX + 'px';
    });

    // === Neue Zeitung erzeugen ===
    function createNewspaper() {
      const newspaper = document.createElement('div');
      newspaper.classList.add('newspaper');
      newspaper.style.left = Math.random() * (window.innerWidth - 40) + 'px';
      gameArea.appendChild(newspaper);

      let y = 0;
      // Fallgeschwindigkeit: mit höherem Level schneller
      let fallSpeed = Math.min(4 + (level - 1) * 0.5, 12);

      // Bewegung per setInterval
      const fallInterval = setInterval(() => {
        y += fallSpeed;
        newspaper.style.top = y + 'px';

        // Kollision mit Spieler prüfen
        const paperRect = newspaper.getBoundingClientRect();
        const playerRect = player.getBoundingClientRect();

        if (
          paperRect.bottom >= playerRect.top &&
          paperRect.left < playerRect.right &&
          paperRect.right > playerRect.left
        ) {
          // Zeitung gefangen
          clearInterval(fallInterval);
          newspaper.remove();
          score++;
          updateHUD();
          checkLevelUp();
        }

        // Wenn Zeitungen unten herausfallen
        if (y > window.innerHeight) {
          clearInterval(fallInterval);
          newspaper.remove();
          loseLife();
        }
      }, 20);

      animationIntervals.push(fallInterval);
    }

    // === Leben verlieren ===
    function loseLife() {
      lives--;
      updateHUD();
      if (lives <= 0) {
        endGame();
      }
    }

    // === Level-Up, wenn z.B. jede 10 Punkte ein neues Level beginnt ===
    function checkLevelUp() {
      const nextLevelThreshold = level * 10;
      if (score >= nextLevelThreshold) {
        level++;
        updateHUD();
        // Neue Spawn-Rate: ältere Intervalle löschen und neu starten
        clearInterval(newspaperInterval);
        startGameLoop();
      }
    }

    // === Spiel beenden (Game Over) ===
    function endGame() {
      // Alle Intervalle beenden
      animationIntervals.forEach(id => clearInterval(id));
      clearInterval(newspaperInterval);
      // Alle bestehenden Zeitungen entfernen
      document.querySelectorAll('.newspaper').forEach(el => el.remove());

      // Game-Over-Bildschirm einblenden
      finalScoreText.textContent = score;
      gameOverScreen.classList.remove('hidden');
    }

    // Damit das Canvas beim Größenwechsel korrekt bleibt
    window.addEventListener('resize', () => {
      playerX = Math.min(playerX, window.innerWidth - 100);
      player.style.left = playerX + 'px';
    });
  </script>
</body>
</html>
