<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <title>Space Crossy – Bunte Version mit Münzen & Wischen</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      background: radial-gradient(circle at top, #1d4ed8 0, #020617 45%, #000 100%);
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      color: #e5e7eb;
    }

    .game-wrapper {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 8px;
    }

    h1 {
      font-size: 24px;
      margin-bottom: 4px;
      text-shadow: 0 2px 6px rgba(0,0,0,0.6);
    }

    .info {
      font-size: 14px;
      opacity: 0.9;
      margin-bottom: 4px;
      text-align: center;
    }

    canvas {
      border-radius: 16px;
      box-shadow: 0 10px 28px rgba(0,0,0,0.5);
      background: #020617;
      border: 2px solid #4b5563;
      touch-action: none; /* wichtig für Mobile, damit Wischen erkannt wird */
    }

    .hud {
      display: flex;
      gap: 16px;
      font-size: 14px;
      margin-top: 2px;
    }

    .button-row {
      margin-top: 6px;
    }

    button {
      font-size: 14px;
      padding: 6px 14px;
      border-radius: 999px;
      border: none;
      cursor: pointer;
      background: #10b981;
      color: white;
      font-weight: 600;
      transition: transform 0.05s ease, box-shadow 0.1s ease, background 0.2s ease;
      box-shadow: 0 3px 8px rgba(0,0,0,0.4);
    }

    button:hover { background: #059669; }

    button:active {
      transform: translateY(1px);
      box-shadow: 0 1px 4px rgba(0,0,0,0.4);
    }

    .character-select {
      display: flex;
      gap: 8px;
      margin-bottom: 6px;
      align-items: center;
      justify-content: center;
      flex-wrap: wrap;
    }

    .character-label {
      font-size: 13px;
      opacity: 0.85;
      margin-right: 4px;
    }

    .character-btn {
      width: 30px;
      height: 30px;
      border-radius: 999px;
      border: 2px solid transparent;
      cursor: pointer;
      padding: 0;
      background-clip: padding-box;
      box-shadow: 0 2px 6px rgba(0,0,0,0.4);
    }

    .character-btn.selected {
      border-color: #fde047;
      box-shadow: 0 0 0 2px rgba(250, 204, 21, 0.4), 0 3px 10px rgba(0,0,0,0.5);
    }
  </style>
</head>
<body>
  <div class="game-wrapper">
    <h1>Space Crossy</h1>
    <p class="info">
      Steuere deinen Astronauten durch den bunten Verkehr, sammle Münzen ein.<br />
      Steuerung: Tippen = vorwärts, Wischen links/rechts = zur Seite (Pfeiltasten gehen auch).
    </p>

    <div class="character-select">
      <span class="character-label">Figur wählen:</span>
      <button class="character-btn selected" data-id="red"
        style="background: radial-gradient(circle at 30% 20%, #fecaca 0, #b91c1c 35%, #7f1d1d 100%);"></button>
      <button class="character-btn" data-id="blue"
        style="background: radial-gradient(circle at 30% 20%, #bfdbfe 0, #1d4ed8 35%, #1e3a8a 100%);"></button>
      <button class="character-btn" data-id="yellow"
        style="background: radial-gradient(circle at 30% 20%, #fef9c3 0, #eab308 35%, #a16207 100%);"></button>
      <button class="character-btn" data-id="purple"
        style="background: radial-gradient(circle at 30% 20%, #e9d5ff 0, #7c3aed 35%, #4c1d95 100%);"></button>
    </div>

    <canvas id="game" width="400" height="600"></canvas>

    <div class="hud">
      <div>Score: <span id="score">0</span></div>
      <div>Best: <span id="best">0</span></div>
      <div>Münzen: <span id="coins">0</span></div>
    </div>

    <div class="button-row">
      <button id="restartBtn">Neustart</button>
    </div>
  </div>

  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const scoreEl = document.getElementById("score");
    const bestEl = document.getElementById("best");
    const coinsEl = document.getElementById("coins");
    const restartBtn = document.getElementById("restartBtn");
    const characterButtons = document.querySelectorAll(".character-btn");

    const width = canvas.width;
    const height = canvas.height;

    const laneHeight = 60;
    const laneCount = 8;
    const baseY = height - laneHeight * 1.5;

    let lastTime = 0;
    let gameOver = false;
    let score = 0;
    let best = Number(localStorage.getItem("spaceCrossyBest") || 0);
    let coinsCollected = 0;

    bestEl.textContent = best;
    coinsEl.textContent = coinsCollected;

    // Figuren
    const characters = {
      red:   { bodyColor: "#ef4444", visorColor: "#bfdbfe" },
      blue:  { bodyColor: "#3b82f6", visorColor: "#e0f2fe" },
      yellow:{ bodyColor: "#eab308", visorColor: "#fef9c3" },
      purple:{ bodyColor: "#8b5cf6", visorColor: "#e9d5ff" }
    };

    const player = {
      x: width / 2,
      y: baseY,
      w: 30,
      h: 38,
      lane: 0,
      speedX: laneHeight,
      bodyColor: characters.red.bodyColor,
      visorColor: characters.red.visorColor
    };

    const obstacles = [];
    const coins = [];

    const shipColors = [
      { body: "#22c55e", cockpit: "#a5b4fc" },
      { body: "#ec4899", cockpit: "#e0f2fe" },
      { body: "#f97316", cockpit: "#fed7aa" },
      { body: "#06b6d4", cockpit: "#bae6fd" }
    ];

    function createObstacles() {
      obstacles.length = 0;
      for (let i = 0; i < laneCount; i++) {
        const isRoad = i % 2 === 1;
        if (!isRoad) continue;

        const laneY = baseY - i * laneHeight;
        const direction = Math.random() < 0.5 ? 1 : -1;
        const speed = 60 + Math.random() * 80;
        const carCount = 2 + Math.floor(Math.random() * 2);

        for (let j = 0; j < carCount; j++) {
          const carWidth = 60;
          const carHeight = 30;
          const offset = (j * 160) + Math.random() * 60;
          const startX = direction === 1 ? -offset : width + offset;
          const color = shipColors[Math.floor(Math.random() * shipColors.length)];

          obstacles.push({
            x: startX,
            y: laneY - carHeight / 2,
            w: carWidth,
            h: carHeight,
            speed: speed * direction,
            bodyColor: color.body,
            cockpitColor: color.cockpit
          });
        }
      }
    }

    function createCoins() {
      coins.length = 0;
      coinsCollected = 0;
      coinsEl.textContent = coinsCollected;

      // Münzen nur auf "Gras"-Lanes (nicht Straße)
      for (let i = 1; i < laneCount; i++) {
        const isRoad = i % 2 === 1;
        if (isRoad) continue; // keine Münzen auf der Straße

        const y = baseY - i * laneHeight;
        const coinCount = 1 + Math.floor(Math.random() * 2); // 1–2 Münzen

        for (let j = 0; j < coinCount; j++) {
          const x = 60 + Math.random() * (width - 120);
          coins.push({
            x,
            y,
            r: 10,
            active: true
          });
        }
      }
    }

    function resetGame() {
      player.x = width / 2;
      player.y = baseY;
      player.lane = 0;
      score = 0;
      gameOver = false;
      scoreEl.te
