<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Galaxy Shooter - Bắn Gà Vũ Trụ</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: linear-gradient(180deg, #000428, #004e92); overflow: hidden;
      font-family: 'Arial', sans-serif; color: #fff; display: flex; justify-content: center; align-items: center; height: 100vh;
    }
    canvas {
      border: 3px solid #00ffff; border-radius: 10px; box-shadow: 0 0 30px #00ffff;
      background: radial-gradient(circle, #001122, #000);
    }
    .ui {
      position: absolute; top: 20px; left: 20px; z-index: 10; font-size: 20px; text-shadow: 0 0 10px #00ffff;
    }
    .game-over {
      position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
      text-align: center; color: #ff0040; font-size: 40px; font-weight: bold; display: none; z-index: 20;
    }
    .restart {
      margin-top: 20px; padding: 15px 30px; background: #00ffff; color: #000;
      border: none; border-radius: 10px; font-size: 18px; cursor: pointer; box-shadow: 0 0 20px #00ffff;
    }
    .restart:hover { background: #00cccc; }
    .wave { position: absolute; top: 20px; right: 20px; font-size: 18px; color: #ffff00; text-shadow: 0 0 10px #ffff00; }
  </style>
</head>
<body>
  <div class="ui">
    Score: <span id="score">0</span><br>
    Lives: <span id="lives">3</span>
  </div>
  <div class="wave">Wave: <span id="wave">1</span></div>
  <canvas id="gameCanvas" width="400" height="600"></canvas>

  <div class="game-over" id="gameOver">
    <div>GAME OVER!</div>
    <div style="font-size:25px; margin:10px 0;">Score: <span id="finalScore">0</span></div>
    <button class="restart" onclick="restartGame()">Chơi Lại</button>
  </div>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('score');
    const livesEl = document.getElementById('lives');
    const waveEl = document.getElementById('wave');
    const finalScoreEl = document.getElementById('finalScore');
    const gameOverScreen = document.getElementById('gameOver');

    let player = { x: 180, y: 500, width: 40, height: 40, speed: 5 };
    let bullets = [];
    let enemies = [];
    let particles = [];
    let powerups = [];
    let score = 0;
    let lives = 3;
    let wave = 1;
    let gameActive = true;
    let keys = {};

    // Âm thanh (base64 đơn giản)
    function playSound(type) {
      // Giả lập âm thanh bằng console (thay bằng Audio nếu cần)
      console.log(`Sound: ${type}`);
    }

    function createEnemy() {
      const side = Math.random() < 0.5 ? 'left' : 'right';
      enemies.push({
        x: side === 'left' ? -50 : canvas.width + 50,
        y: Math.random() * 200 + 50,
        width: 40, height: 40,
        speedX: (side === 'left' ? 2 + wave : -(2 + wave)),
        speedY: 1 + wave * 0.5,
        type: Math.random() < 0.7 ? 'chicken' : 'egg'
      });
    }

    function createBullet(x, y) {
      bullets.push({ x, y, width: 5, height: 10, speed: 8 });
      playSound('shoot');
    }

    function createPowerup(x, y) {
      powerups.push({ x, y, type: Math.random() < 0.5 ? 'shield' : 'rapid', width: 20, height: 20, speed: 2 });
    }

    function drawPlayer() {
      ctx.fillStyle = '#00ff00';
      ctx.shadowBlur = 15; ctx.shadowColor = '#00ff00';
      ctx.fillRect(player.x, player.y, player.width, player.height);
      ctx.fillStyle = '#fff';
      ctx.fillRect(player.x + 10, player.y + 5, 5, 5); // Đèn
      ctx.fillRect(player.x + 25, player.y + 5, 5, 5);
      ctx.shadowBlur = 0;
    }

    function drawEnemies() {
      enemies.forEach((e, i) => {
        ctx.fillStyle = e.type === 'chicken' ? '#ff6600' : '#ffff00';
        ctx.shadowBlur = 10; ctx.shadowColor = e.type === 'chicken' ? '#ff6600' : '#ffff00';
        ctx.fillRect(e.x, e.y, e.width, e.height);
        e.x += e.speedX; e.y += e.speedY;

        // Gà bắn trứng
        if (e.type === 'chicken' && Math.random() < 0.01 + wave * 0.005) {
          createBullet(e.x + 15, e.y + 40); // Trứng bắn xuống
        }

        if (e.x < -50 || e.x > canvas.width + 50 || e.y > canvas.height + 50) {
          enemies.splice(i, 1);
        }
      });
      ctx.shadowBlur = 0;
    }

    function drawBullets() {
      bullets.forEach((b, i) => {
        ctx.fillStyle = '#00ffff';
        ctx.fillRect(b.x, b.y, b.width, b.height);
        b.y -= b.speed;
        if (b.y < -10) bullets.splice(i, 1);
      });
    }

    function drawParticles() {
      particles.forEach((p, i) => {
        ctx.fillStyle = p.color;
        ctx.fillRect(p.x, p.y, p.size, p.size);
        p.x += p.vx; p.y += p.vy; p.life--;
        p.size *= 0.98;
        if (p.life <= 0) particles.splice(i, 1);
      });
    }

    function drawPowerups() {
      powerups.forEach((p, i) => {
        ctx.fillStyle = p.type === 'shield' ? '#00ff00' : '#ffff00';
        ctx.fillRect(p.x, p.y, p.width, p.height);
        p.y += p.speed;
        if (p.y > canvas.height) powerups.splice(i, 1);
      });
    }

    function checkCollision() {
      // Bullet vs Enemy
      bullets.forEach((b, bi) => {
        enemies.forEach((e, ei) => {
          if (b.x < e.x + e.width && b.x + b.width > e.x && b.y < e.y + e.height && b.y + b.height > e.y) {
            createParticles(e.x + 20, e.y + 20, '#ff6600');
            bullets.splice(bi, 1);
            enemies.splice(ei, 1);
            score += 10;
            scoreEl.textContent = score;
            playSound('hit');
            if (Math.random() < 0.2) createPowerup(e.x + 20, e.y + 20);
          }
        });
      });

      // Enemy vs Player
      enemies.forEach((e, i) => {
        if (player.x < e.x + e.width && player.x + player.width > e.x && player.y < e.y + e.height && player.y + player.height > e.y) {
          createParticles(player.x + 20, player.y + 20, '#ff0000');
          enemies.splice(i, 1);
          lives--;
          livesEl.textContent = lives;
          playSound('hurt');
          if (lives <= 0) {
            gameActive = false;
            finalScoreEl.textContent = score;
            gameOverScreen.style.display = 'block';
          }
        }
      });

      // Player vs Powerup
      powerups.forEach((p, i) => {
        if (player.x < p.x + p.width && player.x + player.width > p.x && player.y < p.y + p.height && player.y + player.height > p.y) {
          powerups.splice(i, 1);
          if (p.type === 'shield') lives = Math.min(5, lives + 1);
          else bullets.forEach(b => b.speed *= 1.5); // Rapid fire tạm thời
          playSound('powerup');
        }
      });
    }

    function createParticles(x, y, color) {
      for (let i = 0; i < 15; i++) {
        particles.push({
          x, y,
          vx: (Math.random() - 0.5) * 10,
          vy: (Math.random() - 0.5) * 10,
          life: 30,
          color,
          size: Math.random() * 4 + 2
        });
      }
    }

    function gameLoop() {
      if (!gameActive) return;

      ctx.clearRect(0, 0, canvas.width, canvas.height);
      drawPlayer();
      drawEnemies();
      drawBullets();
      drawParticles();
      drawPowerups();
      checkCollision();

      // Spawn enemies
      if (enemies.length < 5 + wave) createEnemy();

      // Wave progression
      if (enemies.length === 0 && Math.random() < 0.01) {
        wave++;
        waveEl.textContent = wave;
        // Boss: Gà lớn
        if (wave % 5 === 0) {
          enemies.push({ x: 180, y: -100, width: 80, height: 80, speedX: 0, speedY: 1, type: 'boss' });
        }
      }

      // Bắn tự động
      if (Date.now() % 200 < 16) createBullet(player.x + 18, player.y);

      requestAnimationFrame(gameLoop);
    }

    // Điều khiển
    document.addEventListener('keydown', e => {
      keys[e.key] = true;
      if (e.key === ' ') createBullet(player.x + 18, player.y);
    });
    document.addEventListener('keyup', e => keys[e.key] = false);

    function updatePlayer() {
      if (keys['ArrowLeft'] && player.x > 0) player.x -= player.speed;
      if (keys['ArrowRight'] && player.x < canvas.width - player.width) player.x += player.speed;
      if (keys['ArrowUp'] && player.y > 0) player.y -= player.speed;
      if (keys['ArrowDown'] && player.y < canvas.height - player.height) player.y += player.speed;
    }

    setInterval(updatePlayer, 16);

    // Touch
    let touchStart = { x: 0, y: 0 };
    canvas.addEventListener('touchstart', e => {
      touchStart = { x: e.touches[0].clientX - canvas.offsetLeft, y: e.touches[0].clientY - canvas.offsetTop };
    });
    canvas.addEventListener('touchmove', e => {
      e.preventDefault();
      const touch = { x: e.touches[0].clientX - canvas.offsetLeft, y: e.touches[0].clientY - canvas.offsetTop };
      player.x = Math.max(0, Math.min(canvas.width - player.width, player.x + (touch.x - touchStart.x) / 5));
      player.y = Math.max(0, Math.min(canvas.height - player.height, player.y + (touch.y - touchStart.y) / 5));
      touchStart = touch;
    });

    function restartGame() {
      enemies = []; bullets = []; particles = []; powerups = []; score = 0; lives = 3; wave = 1; gameActive = true;
      player.x = 180; player.y = 500;
      scoreEl.textContent = '0'; livesEl.textContent = '3'; waveEl.textContent = '1';
      gameOverScreen.style.display = 'none';
      gameLoop();
    }

    gameLoop();
  </script>
</body>
</html>
