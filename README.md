<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <title>Space Crossy – Among-Us-Figur</title>
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
  </style>
</head>
<body>
  <div class="game-wrapper">
    <h1>Space Crossy</h1>
    <p class="info">
      Tippen = vorwärts • Wischen links/rechts = zur Seite • Pfeiltasten/WASD gehen auch.
    </p>

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

    // Fixe Among-Us-Figur (rot)
    const player = {
      x: width / 2,
      y: baseY,
      w: 30,
      h: 38,
      lane: 0,
      speedX: laneHeight,
      bodyColor: "#ef4444",
      visorColor: "#bfdbfe"
    };

    const obstacles = [];
    const coins = [];

    // Bunte Fahrzeuge/Raumschiffe
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
        const direction = Math.random() < 0
