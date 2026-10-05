const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreDisplay = document.getElementById('score');

const field = {
  width: 820,
  height: 480,
  margin: 30,
  goalHeight: 170,
  goalDepth: 58,
};

const keys = {};
const input = {
  forward: false,
  backward: false,
  left: false,
  right: false,
  boost: false,
  jump: false,
};

const state = {
  scoreLeft: 0,
  scoreRight: 0,
  lastTime: 0,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function makeCar(x, y, angle, color, isPlayer) {
  return {
    x,
    y,
    vx: 0,
    vy: 0,
    angle,
    radius: 22,
    width: 34,
    length: 56,
    color,
    isPlayer,
    boost: 100,
    jump: 0,
    grounded: true,
    jumpVelocity: 0,
    aiTimer: 0,
  };
}

const player = makeCar(field.width * 0.28, field.height / 2, 0, '#4ad5ff', true);
const bot = makeCar(field.width * 0.72, field.height / 2, Math.PI, '#ffb04d', false);
const ball = {
  x: field.width / 2,
  y: field.height / 2,
  vx: 0,
  vy: 0,
  radius: 16,
  z: 0,
  vz: 0,
};

const boostPads = [
  { x: field.width * 0.2, y: field.height * 0.3, r: 20, strength: 240 },
  { x: field.width * 0.2, y: field.height * 0.7, r: 20, strength: 240 },
  { x: field.width * 0.8, y: field.height * 0.3, r: 20, strength: 240 },
  { x: field.width * 0.8, y: field.height * 0.7, r: 20, strength: 240 },
];

function resetRound() {
  ball.x = field.width / 2;
  ball.y = field.height / 2;
  ball.vx = 0;
  ball.vy = 0;
  ball.z = 0;
  ball.vz = 0;

  player.x = field.width * 0.28;
  player.y = field.height / 2;
  player.vx = 0;
  player.vy = 0;
  player.angle = 0;
  player.boost = 100;
  player.jump = 0;
  player.grounded = true;
  player.jumpVelocity = 0;

  bot.x = field.width * 0.72;
  bot.y = field.height / 2;
  bot.vx = 0;
  bot.vy = 0;
  bot.angle = Math.PI;
  bot.boost = 100;
  bot.jump = 0;
  bot.grounded = true;
  bot.jumpVelocity = 0;
}

function keyFor(event) {
  if (event.key === ' ') return 'space';
  if (event.key === 'Shift') return 'shift';
  return event.key.toLowerCase();
}

window.addEventListener('keydown', (event) => {
  const key = keyFor(event);
  keys[key] = true;

  if (['w', 'a', 's', 'd', 'arrowup', 'arrowdown', 'arrowleft', 'arrowright', ' ', 'shift'].includes(key)) {
    event.preventDefault();
  }

  if (key === 'w' || key === 'arrowup') input.forward = true;
  if (key === 's' || key === 'arrowdown') input.backward = true;
  if (key === 'a' || key === 'arrowleft') input.left = true;
  if (key === 'd' || key === 'arrowright') input.right = true;
  if (key === 'shift') input.boost = true;
  if (key === 'space') input.jump = true;
});

window.addEventListener('keyup', (event) => {
  const key = keyFor(event);
  keys[key] = false;

  if (key === 'w' || key === 'arrowup') input.forward = false;
  if (key === 's' || key === 'arrowdown') input.backward = false;
  if (key === 'a' || key === 'arrowleft') input.left = false;
  if (key === 'd' || key === 'arrowright') input.right = false;
  if (key === 'shift') input.boost = false;
  if (key === 'space') input.jump = false;
});

function handleInput(car, dt) {
  const turnDirection = (input.right ? 1 : 0) - (input.left ? 1 : 0);
  const throttle = (input.forward ? 1 : 0) - (input.backward ? 1 : 0);

  if (car.isPlayer) {
    if (throttle !== 0) {
      const ax = Math.cos(car.angle) * throttle * 360;
      const ay = Math.sin(car.angle) * throttle * 360;
      car.vx += ax * dt;
      car.vy += ay * dt;
    }

    if (turnDirection !== 0) {
      car.angle += turnDirection * 2.5 * dt * (0.6 + clamp(Math.hypot(car.vx, car.vy) / 250, 0, 1));
    }

    if (input.boost && car.boost > 0) {
      const bonus = 260;
      car.vx += Math.cos(car.angle) * bonus * dt;
      car.vy += Math.sin(car.angle) * bonus * dt;
      car.boost = clamp(car.boost - 42 * dt, 0, 100);
    } else {
      car.boost = clamp(car.boost + 18 * dt, 0, 100);
    }

    if (input.jump && car.grounded) {
      car.jumpVelocity = 240;
      car.grounded = false;
    }
  }
}

function updateBot(car, dt) {
  const targetX = ball.x;
  const targetY = ball.y;
  const angleToBall = Math.atan2(targetY - car.y, targetX - car.x);
  const angleDelta = Math.atan2(Math.sin(angleToBall - car.angle), Math.cos(angleToBall - car.angle));

  const steer = clamp(angleDelta, -1, 1);
  car.angle += steer * 2.2 * dt;

  const distToBall = Math.hypot(targetX - car.x, targetY - car.y);
  const throttle = distToBall > 45 ? 1 : 0.2;

  if (throttle !== 0) {
    car.vx += Math.cos(car.angle) * throttle * 330 * dt;
    car.vy += Math.sin(car.angle) * throttle * 330 * dt;
  }

  if (distToBall < 90 && Math.abs(angleDelta) < 0.7) {
    car.vx += Math.cos(car.angle) * 260 * dt;
    car.vy += Math.sin(car.angle) * 260 * dt;
  }

  if (car.boost > 15 && distToBall > 180) {
    car.vx += Math.cos(car.angle) * 220 * dt;
    car.vy += Math.sin(car.angle) * 220 * dt;
    car.boost = clamp(car.boost - 36 * dt, 0, 100);
  } else {
    car.boost = clamp(car.boost + 18 * dt, 0, 100);
  }

  if (distToBall < 40 && Math.random() < 0.04 && car.grounded) {
    car.jumpVelocity = 230;
    car.grounded = false;
  }
}

function updateCar(car, dt) {
  if (car.isPlayer) {
    handleInput(car, dt);
  } else {
    updateBot(car, dt);
  }

  const speed = Math.hypot(car.vx, car.vy);
  if (speed > 0) {
    const damp = car.isPlayer ? 0.985 : 0.98;
    car.vx *= damp;
    car.vy *= damp;
  }

  if (!car.grounded) {
    car.jump += car.jumpVelocity * dt;
    car.jumpVelocity -= 700 * dt;
    if (car.jump <= 0) {
      car.jump = 0;
      car.jumpVelocity = 0;
      car.grounded = true;
    }
  }

  car.x += car.vx * dt;
  car.y += car.vy * dt;

  const maxX = field.width - field.margin;
  const maxY = field.height - field.margin;
  car.x = clamp(car.x, field.margin, maxX);
  car.y = clamp(car.y, field.margin, maxY);

  const floorPad = 12;
  const maxSpeed = 260;
  if (speed > maxSpeed) {
    const scale = maxSpeed / speed;
    car.vx *= scale;
    car.vy *= scale;
  }

  if (car.x <= floorPad) car.x = floorPad;
  if (car.y <= floorPad) car.y = floorPad;
  if (car.x >= field.width - floorPad) car.x = field.width - floorPad;
  if (car.y >= field.height - floorPad) car.y = field.height - floorPad;
}

function boostPadEffect(car, dt) {
  for (const pad of boostPads) {
    const dx = car.x - pad.x;
    const dy = car.y - pad.y;
    if (Math.hypot(dx, dy) < pad.r + car.radius + 6) {
      car.vx += (car.x - pad.x) * 0.02 * pad.strength * dt;
      car.vy += (car.y - pad.y) * 0.02 * pad.strength * dt;
      car.vx += Math.cos(car.angle) * 160 * dt;
      car.vy += Math.sin(car.angle) * 160 * dt;
    }
  }
}

function applyBallCollision(car) {
  const dx = ball.x - car.x;
  const dy = ball.y - car.y;
  const dist = Math.hypot(dx, dy) || 0.0001;
  const minimum = ball.radius + car.radius + 8;

  if (dist < minimum) {
    const nx = dx / dist;
    const ny = dy / dist;
    const overlap = minimum - dist;

    ball.x += nx * overlap;
    ball.y += ny * overlap;

    const impactSpeed = car.vx * nx + car.vy * ny;
    const push = 160 + Math.max(0, impactSpeed) * 0.8;
    ball.vx += nx * push;
    ball.vy += ny * push;

    ball.vx += car.vx * 0.12;
    ball.vy += car.vy * 0.12;
  }
}

function updateBall(dt) {
  ball.x += ball.vx * dt;
  ball.y += ball.vy * dt;
  ball.vx *= 0.992;
  ball.vy *= 0.992;

  if (ball.y - ball.radius < 0) {
    ball.y = ball.radius;
    ball.vy *= -0.82;
  }
  if (ball.y + ball.radius > field.height) {
    ball.y = field.height - ball.radius;
    ball.vy *= -0.82;
  }

  const goalTop = field.height / 2 - field.goalHeight / 2;
  const goalBottom = field.height / 2 + field.goalHeight / 2;

  if (ball.x - ball.radius < 0) {
    if (ball.y > goalTop && ball.y < goalBottom) {
      state.scoreRight += 1;
      scoreDisplay.textContent = `${state.scoreLeft} : ${state.scoreRight}`;
      resetRound();
      return;
    }
    ball.x = ball.radius;
    ball.vx *= -0.82;
  }

  if (ball.x + ball.radius > field.width) {
    if (ball.y > goalTop && ball.y < goalBottom) {
      state.scoreLeft += 1;
      scoreDisplay.textContent = `${state.scoreLeft} : ${state.scoreRight}`;
      resetRound();
      return;
    }
    ball.x = field.width - ball.radius;
    ball.vx *= -0.82;
  }

  const ballSpeed = Math.hypot(ball.vx, ball.vy);
  const maxBallSpeed = 780;
  if (ballSpeed > maxBallSpeed) {
    const scale = maxBallSpeed / ballSpeed;
    ball.vx *= scale;
    ball.vy *= scale;
  }

  applyBallCollision(player);
  applyBallCollision(bot);
}

function drawField() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#0e813d';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = 'rgba(255,255,255,0.06)';
  ctx.fillRect(field.margin, field.margin, field.width, field.height);

  ctx.strokeStyle = 'rgba(255,255,255,0.7)';
  ctx.lineWidth = 3;
  ctx.strokeRect(field.margin, field.margin, field.width, field.height);

  ctx.beginPath();
  ctx.moveTo(field.width / 2 + field.margin, field.margin);
  ctx.lineTo(field.width / 2 + field.margin, field.height + field.margin);
  ctx.moveTo(field.margin, field.height / 2 + field.margin);
  ctx.lineTo(field.width + field.margin, field.height / 2 + field.margin);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(field.width / 2 + field.margin, field.height / 2 + field.margin, 80, 0, Math.PI * 2);
  ctx.stroke();

  const goalTop = field.height / 2 - field.goalHeight / 2;
  const goalBottom = field.height / 2 + field.goalHeight / 2;

  ctx.strokeStyle = 'rgba(255,255,255,0.4)';
  ctx.strokeRect(0, goalTop, field.goalDepth, field.goalHeight);
  ctx.strokeRect(field.width + field.margin + 10, goalTop, field.goalDepth, field.goalHeight);

  ctx.fillStyle = 'rgba(255,255,255,0.18)';
  ctx.fillRect(0, goalTop, field.goalDepth, field.goalHeight);
  ctx.fillRect(field.width + field.margin + 10, goalTop, field.goalDepth, field.goalHeight);

  ctx.strokeStyle = 'rgba(255,255,255,0.18)';
  ctx.strokeRect(field.margin, field.margin, field.width, field.height);

  for (const pad of boostPads) {
    const glow = ctx.createRadialGradient(pad.x, pad.y, 4, pad.x, pad.y, 28);
    glow.addColorStop(0, 'rgba(255,255,125,1)');
    glow.addColorStop(0.4, 'rgba(255,255,110,0.9)');
    glow.addColorStop(1, 'rgba(255,255,110,0)');
    ctx.fillStyle = glow;
    ctx.beginPath();
    ctx.arc(pad.x, pad.y, 28, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = '#f2f1ae';
    ctx.beginPath();
    ctx.arc(pad.x, pad.y, 10, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawCar(car) {
  const lift = car.jump * 0.3;
  ctx.save();
  ctx.translate(car.x, car.y - lift);
  ctx.rotate(car.angle);

  ctx.fillStyle = 'rgba(0,0,0,0.2)';
  ctx.fillRect(-20, -18, 40, 36);

  ctx.fillStyle = car.color;
  ctx.fillRect(-car.width / 2, -car.length / 2, car.width, car.length);

  ctx.fillStyle = 'rgba(255,255,255,0.25)';
  ctx.fillRect(-car.width / 2 + 8, -car.length / 2 + 8, car.width - 16, car.length * 0.3);

  ctx.fillStyle = '#1f2c3d';
  ctx.fillRect(-car.width / 2 - 6, -car.length / 2 + 6, 6, 12);
  ctx.fillRect(car.width / 2, -car.length / 2 + 6, 6, 12);
  ctx.fillRect(-car.width / 2 - 6, car.length / 2 - 18, 6, 12);
  ctx.fillRect(car.width / 2, car.length / 2 - 18, 6, 12);

  ctx.restore();
}

function drawBall() {
  const shadowOffset = 6 + ball.z * 0.2;
  ctx.fillStyle = 'rgba(0,0,0,0.25)';
  ctx.beginPath();
  ctx.ellipse(ball.x, ball.y + shadowOffset, ball.radius + 3, ball.radius + 2, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#f9fafb';
  ctx.beginPath();
  ctx.arc(ball.x, ball.y - ball.z * 0.4, ball.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.lineWidth = 2;
  ctx.strokeStyle = '#d8e3f0';
  ctx.stroke();
}

function render() {
  drawField();
  drawCar(player);
  drawCar(bot);
  drawBall();
}

function update(dt) {
  updateCar(player, dt);
  updateCar(bot, dt);
  boostPadEffect(player, dt);
  boostPadEffect(bot, dt);
  updateBall(dt);

  scoreDisplay.textContent = `${state.scoreLeft} : ${state.scoreRight}`;
}

function gameLoop(timestamp) {
  const dt = Math.min((timestamp - state.lastTime) / 1000 || 0.016, 0.03);
  state.lastTime = timestamp;

  update(dt);
  render();

  requestAnimationFrame(gameLoop);
}

resetRound();
scoreDisplay.textContent = `${state.scoreLeft} : ${state.scoreRight}`;
requestAnimationFrame(gameLoop);
