/* =============================================
   DEAD GRID — script.js
   Full 2D Zombie Survival Game
   ============================================= */

// ─── CANVAS SETUP ──────────────────────────────
const canvas = document.getElementById('gameCanvas');
const ctx    = canvas.getContext('2d');

function resizeCanvas() {
  canvas.width  = window.innerWidth;
  canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ─────────────────────────────────
const TILE   = 40;
const MAP_W  = 60;   // tiles
const MAP_H  = 40;   // tiles

// ─── INPUT STATE ───────────────────────────────
const keys   = {};
const mouse  = { x: 0, y: 0, down: false };

window.addEventListener('keydown', e => {
  keys[e.code] = true;
  if (e.code === 'Space')  { e.preventDefault(); if (game.running) toggleShop(); }
  if (e.code === 'KeyB')   { if (game.running) toggleBuild(); }
  if (e.code === 'Escape') {
    if (game.shopOpen)  toggleShop();
    if (game.buildMode) toggleBuild();
  }
  if (e.code === 'KeyP') { if (game.running && !game.over) togglePause(); }
});
window.addEventListener('keyup',   e => keys[e.code] = false);
canvas.addEventListener('mousemove', e => { mouse.x = e.clientX; mouse.y = e.clientY; });
canvas.addEventListener('mousedown', e => {
  mouse.down = true;
  if (game.buildMode) { handleBuildPlace(); return; }
});
canvas.addEventListener('mouseup',   () => mouse.down = false);

// ─── CAMERA ────────────────────────────────────
const cam = { x: 0, y: 0 };
function worldToScreen(wx, wy) {
  return { x: wx - cam.x + canvas.width/2, y: wy - cam.y + canvas.height/2 };
}
function screenToWorld(sx, sy) {
  return { x: sx + cam.x - canvas.width/2, y: sy + cam.y - canvas.height/2 };
}

// ─── UTILITIES ─────────────────────────────────
function dist(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}
function rand(min, max) {
  return Math.random() * (max - min) + min;
}
function randInt(min, max) {
  return Math.floor(rand(min, max + 1));
}
function clamp(v, lo, hi) {
  return Math.max(lo, Math.min(hi, v));
}
function angle(from, to) {
  return Math.atan2(to.y - from.y, to.x - from.x);
}

// ─── DAMAGE NUMBERS ─────────────────────────────
function spawnDmgNum(wx, wy, val, color = '') {
  const sc = worldToScreen(wx, wy);
  const el = document.createElement('div');
  el.className = 'dmg-number' + (color ? ' ' + color : '');
  el.textContent = val;
  el.style.left = (sc.x + rand(-10, 10)) + 'px';
  el.style.top  = (sc.y - 20) + 'px';
  document.body.appendChild(el);
  setTimeout(() => el.remove(), 700);
}

// ─── WEAPONS CATALOG ───────────────────────────
const WEAPONS = {
  pistol: {
    id: 'pistol', name: 'Pistol', icon: '🔫',
    damage: 25, fireRate: 300, bulletSpeed: 700, spread: 0.05,
    bulletCount: 1, range: 600, price: 0, desc: 'Reliable sidearm'
  },
  shotgun: {
    id: 'shotgun', name: 'Shotgun', icon: '💥',
    damage: 18, fireRate: 700, bulletSpeed: 550, spread: 0.22,
    bulletCount: 6, range: 350, price: 300, desc: 'Close-range devastation'
  },
  sniper: {
    id: 'sniper', name: 'Sniper', icon: '🎯',
    damage: 120, fireRate: 1200, bulletSpeed: 1200, spread: 0.01,
    bulletCount: 1, range: 1200, price: 600, desc: 'Long-range precision'
  },
  smg: {
    id: 'smg', name: 'SMG', icon: '⚡',
    damage: 15, fireRate: 100, bulletSpeed: 620, spread: 0.12,
    bulletCount: 1, range: 450, price: 450, desc: 'High fire rate'
  }
};

// ─── MAP GENERATION ────────────────────────────
class GameMap {
  constructor() {
    this.w = MAP_W;
    this.h = MAP_H;
    this.tiles = [];      // 0=floor, 1=wall(obstacle)
    this.obstacles = [];  // {x,y,w,h} world-space
    this.generate();
  }
  generate() {
    // Fill floor
    this.tiles = Array.from({ length: this.h }, () => new Array(this.w).fill(0));
    // Place random obstacle clusters
    const count = randInt(18, 28);
    for (let i = 0; i < count; i++) {
      const tx = randInt(3, this.w - 6);
      const ty = randInt(3, this.h - 6);
      const tw = randInt(1, 4);
      const th = randInt(1, 3);
      // Keep center open (spawn zone)
      const cx = this.w / 2, cy = this.h / 2;
      if (Math.abs(tx - cx) < 5 && Math.abs(ty - cy) < 5) continue;
      for (let r = ty; r < ty + th; r++)
        for (let c = tx; c < tx + tw; c++)
          if (r >= 0 && r < this.h && c >= 0 && c < this.w)
            this.tiles[r][c] = 1;
    }
    // Build obstacle rects for collision
    this.buildObstacleRects();
  }
  buildObstacleRects() {
    this.obstacles = [];
    for (let r = 0; r < this.h; r++)
      for (let c = 0; c < this.w; c++)
        if (this.tiles[r][c] === 1)
          this.obstacles.push({ x: c*TILE, y: r*TILE, w: TILE, h: TILE });
  }
  draw() {
    const startCol = Math.floor((cam.x - canvas.width/2) / TILE) - 1;
    const endCol   = startCol + Math.ceil(canvas.width / TILE) + 2;
    const startRow = Math.floor((cam.y - canvas.height/2) / TILE) - 1;
    const endRow   = startRow + Math.ceil(canvas.height / TILE) + 2;

    for (let r = Math.max(0, startRow); r < Math.min(this.h, endRow); r++) {
      for (let c = Math.max(0, startCol); c < Math.min(this.w, endCol); c++) {
        const sc = worldToScreen(c * TILE, r * TILE);
        if (this.tiles[r][c] === 1) {
          ctx.fillStyle = '#1a1d24';
          ctx.fillRect(sc.x, sc.y, TILE + 1, TILE + 1);
          ctx.strokeStyle = '#222630';
          ctx.lineWidth = 1;
          ctx.strokeRect(sc.x, sc.y, TILE + 1, TILE + 1);
        } else {
          ctx.fillStyle = (r + c) % 2 === 0 ? '#0d0e12' : '#0f1014';
          ctx.fillRect(sc.x, sc.y, TILE + 1, TILE + 1);
        }
      }
    }
    // Map border
    const tl = worldToScreen(0, 0);
    ctx.strokeStyle = '#e63946';
    ctx.lineWidth = 3;
    ctx.strokeRect(tl.x, tl.y, this.w * TILE, this.h * TILE);
  }
  isSolid(wx, wy) {
    const c = Math.floor(wx / TILE);
    const r = Math.floor(wy / TILE);
    if (c < 0 || c >= this.w || r < 0 || r >= this.h) return true;
    return this.tiles[r][c] === 1;
  }
  // AABB vs map obstacles
  solidAABB(x, y, w, h) {
    const corners = [
      { x: x,     y: y },
      { x: x+w-1, y: y },
      { x: x,     y: y+h-1 },
      { x: x+w-1, y: y+h-1 }
    ];
    return corners.some(p => this.isSolid(p.x, p.y));
  }
}

// ─── STRUCTURES ────────────────────────────────
const STRUCTURE_DEFS = {
  woodWall:  { name: 'Wood Wall',  icon: '🪵', color: '#8B6914', cost: 50,  hp: 150, desc: 'Cheap, low HP' },
  metalWall: { name: 'Metal Wall', icon: '🔩', color: '#607080', cost: 150, hp: 500, desc: 'Sturdy defense' },
  spike:     { name: 'Spike Trap', icon: '⚔️', color: '#c0c0c0', cost: 80,  hp: 200, desc: 'Damages zombies' }
};

class Structure {
  constructor(tx, ty, type) {
    this.tx   = tx; this.ty = ty;
    this.x    = tx * TILE; this.y = ty * TILE;
    this.type = type;
    const def = STRUCTURE_DEFS[type];
    this.maxHp = def.hp;
    this.hp    = def.hp;
    this.color = def.color;
    this.alive = true;
    this.spikeTimer = 0;
  }
  takeDamage(dmg) {
    this.hp -= dmg;
    if (this.hp <= 0) this.alive = false;
  }
  draw() {
    const sc = worldToScreen(this.x, this.y);
    const ratio = this.hp / this.maxHp;
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    ctx.fillRect(sc.x + 2, sc.y + 2, TILE, TILE);
    // body
    ctx.fillStyle = this.color;
    ctx.globalAlpha = 0.7 + ratio * 0.3;
    ctx.fillRect(sc.x, sc.y, TILE, TILE);
    ctx.globalAlpha = 1;
    // hp bar
    ctx.fillStyle = '#111';
    ctx.fillRect(sc.x, sc.y - 6, TILE, 4);
    ctx.fillStyle = ratio > 0.5 ? '#2dc653' : ratio > 0.25 ? '#f4d03f' : '#e63946';
    ctx.fillRect(sc.x, sc.y - 6, TILE * ratio, 4);
    // icon
    ctx.font = '20px serif';
    ctx.textAlign = 'center';
    ctx.fillText(STRUCTURE_DEFS[this.type].icon, sc.x + TILE/2, sc.y + TILE/2 + 7);
  }
}

// ─── BULLET ────────────────────────────────────
class Bullet {
  constructor(x, y, vx, vy, damage, range, fromPlayer = true) {
    this.x = x; this.y = y;
    this.vx = vx; this.vy = vy;
    this.damage = damage;
    this.range  = range;
    this.fromPlayer = fromPlayer;
    this.travelDist = 0;
    this.alive = true;
    this.radius = fromPlayer ? 4 : 5;
    this.color  = fromPlayer ? '#f4d03f' : '#e63946';
  }
  update(dt) {
    const dx = this.vx * dt;
    const dy = this.vy * dt;
    this.x += dx;
    this.y += dy;
    this.travelDist += Math.hypot(dx, dy);
    if (this.travelDist > this.range) this.alive = false;
    if (gameMap.isSolid(this.x, this.y)) this.alive = false;
  }
  draw() {
    const sc = worldToScreen(this.x, this.y);
    ctx.beginPath();
    ctx.arc(sc.x, sc.y, this.radius, 0, Math.PI * 2);
    ctx.fillStyle = this.color;
    ctx.fill();
    // glow
    ctx.beginPath();
    ctx.arc(sc.x, sc.y, this.radius + 3, 0, Math.PI * 2);
    ctx.fillStyle = this.color + '44';
    ctx.fill();
  }
}

// ─── PLAYER ────────────────────────────────────
class Player {
  constructor() {
    this.x = (MAP_W / 2) * TILE;
    this.y = (MAP_H / 2) * TILE;
    this.radius = 14;
    this.speed  = 180;
    this.maxHp  = 100;
    this.hp     = 100;
    this.money  = 200;
    this.kills  = 0;
    this.alive  = true;

    // weapon state
    this.weapons  = ['pistol'];
    this.equipped = 'pistol';
    this.lastShot = 0;

    // upgrades
    this.dmgMult       = 1;
    this.fireRateMult  = 1;
    this.hpUpLevel     = 0;

    this.hitFlash = 0;
    this.angle    = 0;
  }

  get weapon() { return WEAPONS[this.equipped]; }

  upgrade(stat) {
    if (stat === 'damage')   { this.dmgMult += 0.25; }
    if (stat === 'firerate') { this.fireRateMult = Math.max(0.4, this.fireRateMult - 0.1); }
    if (stat === 'hp')       { this.maxHp += 30; this.hp = Math.min(this.hp + 30, this.maxHp); this.hpUpLevel++; }
  }

  shoot(now, bullets) {
    const w     = this.weapon;
    const delay = w.fireRate * this.fireRateMult;
    if (now - this.lastShot < delay) return;
    this.lastShot = now;
    const mw = screenToWorld(mouse.x, mouse.y);
    const a  = angle(this, mw);
    for (let i = 0; i < w.bulletCount; i++) {
      const spread = (Math.random() - 0.5) * w.spread * 2;
      const aa = a + spread;
      const dmg = w.damage * this.dmgMult;
      bullets.push(new Bullet(this.x, this.y, Math.cos(aa) * w.bulletSpeed, Math.sin(aa) * w.bulletSpeed, dmg, w.range));
    }
    // sound
    playSound('shoot', 0.15);
  }

  update(dt, bullets, now) {
    if (!this.alive) return;

    // movement
    let dx = 0, dy = 0;
    if (keys['KeyW'] || keys['ArrowUp'])    dy -= 1;
    if (keys['KeyS'] || keys['ArrowDown'])  dy += 1;
    if (keys['KeyA'] || keys['ArrowLeft'])  dx -= 1;
    if (keys['KeyD'] || keys['ArrowRight']) dx += 1;
    if (dx !== 0 && dy !== 0) { dx *= 0.707; dy *= 0.707; }

    const nx = this.x + dx * this.speed * dt;
    const ny = this.y + dy * this.speed * dt;
    const r  = this.radius;
    if (!gameMap.solidAABB(nx - r, this.y - r, r*2, r*2) && nx > r && nx < MAP_W*TILE - r)
      this.x = nx;
    if (!gameMap.solidAABB(this.x - r, ny - r, r*2, r*2) && ny > r && ny < MAP_H*TILE - r)
      this.y = ny;

    // aim
    this.angle = Math.atan2(mouse.y - canvas.height/2, mouse.x - canvas.width/2);

    // auto-shoot on mouse hold
    if (mouse.down && !game.shopOpen && !game.buildMode) this.shoot(now, bullets);

    if (this.hitFlash > 0) this.hitFlash -= dt;

    // death
    if (this.hp <= 0) { this.alive = false; game.triggerGameOver(); }
  }

  takeDamage(dmg) {
    this.hp -= dmg;
    this.hitFlash = 0.15;
    spawnDmgNum(this.x, this.y, Math.ceil(dmg), '');
    if (this.hp < 0) this.hp = 0;
  }

  draw() {
    const sc = worldToScreen(this.x, this.y);
    // hit flash
    if (this.hitFlash > 0) {
      ctx.beginPath();
      ctx.arc(sc.x, sc.y, this.radius + 8, 0, Math.PI*2);
      ctx.fillStyle = 'rgba(230,57,70,0.4)';
      ctx.fill();
    }
    // shadow
    ctx.beginPath();
    ctx.ellipse(sc.x, sc.y + 4, this.radius, this.radius * 0.5, 0, 0, Math.PI*2);
    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    ctx.fill();
    // body
    ctx.beginPath();
    ctx.arc(sc.x, sc.y, this.radius, 0, Math.PI*2);
    ctx.fillStyle = '#3a7bd5';
    ctx.fill();
    ctx.strokeStyle = '#6aa3ff';
    ctx.lineWidth = 2;
    ctx.stroke();
    // direction indicator
    ctx.beginPath();
    ctx.moveTo(sc.x, sc.y);
    ctx.lineTo(sc.x + Math.cos(this.angle) * (this.radius + 8), sc.y + Math.sin(this.angle) * (this.radius + 8));
    ctx.strokeStyle = '#f0f2f7';
    ctx.lineWidth = 2.5;
    ctx.stroke();
    // crosshair line to mouse
    ctx.beginPath();
    ctx.moveTo(sc.x, sc.y);
    ctx.lineTo(mouse.x, mouse.y);
    ctx.strokeStyle = 'rgba(255,255,255,0.08)';
    ctx.lineWidth = 1;
    ctx.setLineDash([4, 8]);
    ctx.stroke();
    ctx.setLineDash([]);
  }
}

// ─── ZOMBIE TYPES ──────────────────────────────
const ZOMBIE_DEFS = {
  normal:  { name: 'Normal',  color: '#3d7a3d', outline: '#5fb85f', hp: 80,  speed: 70,  dmg: 10, reward: 20, size: 13, range: 22, attackRate: 800 },
  runner:  { name: 'Runner',  color: '#7a6800', outline: '#c9a500', hp: 40,  speed: 160, dmg: 8,  reward: 25, size: 11, range: 20, attackRate: 500 },
  tank:    { name: 'Tank',    color: '#4a1a6a', outline: '#9a3fd1', hp: 350, speed: 40,  dmg: 20, reward: 50, size: 20, range: 28, attackRate: 1200 },
  spitter: { name: 'Spitter', color: '#1a5a3a', outline: '#35c47f', hp: 60,  speed: 55,  dmg: 12, reward: 35, size: 12, range: 300, attackRate: 2000 },
  boss:    { name: 'BOSS',    color: '#6a0000', outline: '#ff2222', hp: 1200,speed: 50,  dmg: 30, reward: 300,size: 28, range: 32, attackRate: 1000 }
};

class Zombie {
  constructor(x, y, type, waveScale) {
    this.x = x; this.y = y;
    this.type = type;
    const def = ZOMBIE_DEFS[type];
    this.name  = def.name;
    this.color = def.color;
    this.outline = def.outline;
    this.maxHp  = Math.floor(def.hp * waveScale);
    this.hp     = this.maxHp;
    this.speed  = def.speed * (0.9 + waveScale * 0.1);
    this.dmg    = def.dmg * Math.sqrt(waveScale);
    this.reward = def.reward;
    this.size   = def.size;
    this.range  = def.range;
    this.attackRate = def.attackRate;
    this.lastAttack = 0;
    this.alive  = true;
    this.hitFlash = 0;
    this.angle  = 0;
    this.attackRange = def.range;
  }

  takeDamage(dmg) {
    this.hp -= dmg;
    this.hitFlash = 0.1;
    if (this.hp <= 0) { this.alive = false; }
  }

  update(dt, player, structures, bullets, now) {
    if (!this.alive) return;
    if (this.hitFlash > 0) this.hitFlash -= dt;

    // Find nearest target: player or structure
    let target = player;
    let targetDist = dist(this, player);

    for (const s of structures) {
      if (!s.alive) continue;
      const d = dist(this, { x: s.x + TILE/2, y: s.y + TILE/2 });
      if (d < targetDist) {
        target = { x: s.x + TILE/2, y: s.y + TILE/2, _struct: s };
        targetDist = d;
      }
    }

    const a  = angle(this, target);
    this.angle = a;

    // Spitter: shoot at player from range
    if (this.type === 'spitter' && dist(this, player) < this.attackRange) {
      if (now - this.lastAttack > this.attackRate) {
        this.lastAttack = now;
        const aa = angle(this, player) + (Math.random() - 0.5) * 0.15;
        bullets.push(new Bullet(this.x, this.y, Math.cos(aa)*280, Math.sin(aa)*280, this.dmg, 500, false));
      }
      return; // don't move when shooting
    }

    // Move toward target
    const nx = this.x + Math.cos(a) * this.speed * dt;
    const ny = this.y + Math.sin(a) * this.speed * dt;

    if (!gameMap.solidAABB(nx - this.size, this.y - this.size, this.size*2, this.size*2))
      this.x = nx;
    if (!gameMap.solidAABB(this.x - this.size, ny - this.size, this.size*2, this.size*2))
      this.y = ny;

    // Clamp to map
    this.x = clamp(this.x, this.size, MAP_W*TILE - this.size);
    this.y = clamp(this.y, this.size, MAP_H*TILE - this.size);

    // Attack
    if (now - this.lastAttack > this.attackRate) {
      if (target._struct) {
        if (dist(this, target) < this.size + TILE/2 + 4) {
          target._struct.takeDamage(this.dmg * 2);
          this.lastAttack = now;
          playSound('hit', 0.06);
        }
      } else {
        if (dist(this, player) < this.size + player.radius + 2) {
          player.takeDamage(this.dmg);
          this.lastAttack = now;
          playSound('hurt', 0.2);
        }
      }
    }
  }

  draw(fogActive) {
    const sc = worldToScreen(this.x, this.y);
    if (fogActive) {
      const dp = dist({ x: sc.x, y: sc.y }, { x: canvas.width/2, y: canvas.height/2 });
      if (dp > 220) return; // hide in fog
    }
    // hit flash
    if (this.hitFlash > 0) {
      ctx.beginPath();
      ctx.arc(sc.x, sc.y, this.size + 5, 0, Math.PI*2);
      ctx.fillStyle = 'rgba(255,255,255,0.35)';
      ctx.fill();
    }
    // shadow
    ctx.beginPath();
    ctx.ellipse(sc.x, sc.y + 4, this.size, this.size * 0.45, 0, 0, Math.PI*2);
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fill();
    // body
    ctx.beginPath();
    ctx.arc(sc.x, sc.y, this.size, 0, Math.PI*2);
    ctx.fillStyle = this.color;
    ctx.fill();
    ctx.strokeStyle = this.outline;
    ctx.lineWidth = 2;
    ctx.stroke();
    // direction
    ctx.beginPath();
    ctx.moveTo(sc.x, sc.y);
    ctx.lineTo(sc.x + Math.cos(this.angle)*this.size, sc.y + Math.sin(this.angle)*this.size);
    ctx.strokeStyle = this.outline;
    ctx.lineWidth = 2;
    ctx.stroke();
    // hp bar
    const barW = this.size * 2.2;
    ctx.fillStyle = '#111';
    ctx.fillRect(sc.x - barW/2, sc.y - this.size - 10, barW, 4);
    const ratio = this.hp / this.maxHp;
    ctx.fillStyle = ratio > 0.5 ? '#2dc653' : ratio > 0.25 ? '#f4d03f' : '#e63946';
    ctx.fillRect(sc.x - barW/2, sc.y - this.size - 10, barW * ratio, 4);
    // boss label
    if (this.type === 'boss') {
      ctx.font = 'bold 10px "Share Tech Mono"';
      ctx.textAlign = 'center';
      ctx.fillStyle = '#ff2222';
      ctx.fillText('BOSS', sc.x, sc.y - this.size - 14);
    }
  }
}

// ─── WAVE MANAGER ──────────────────────────────
class WaveManager {
  constructor() {
    this.wave       = 0;
    this.active     = false;
    this.zombiesLeft= 0;
    this.spawnQueue = [];
    this.spawnTimer = 0;
    this.spawnDelay = 0.6;
    this.betweenWave= true;
    this.betweenTimer = 0;
    this.betweenDelay = 4.0;
    this.modifier   = null;
    this.skipPenalty= 1;
  }

  get scale() { return 1 + (this.wave - 1) * 0.18 * this.skipPenalty; }

  nextWave(skipped = false) {
    this.wave++;
    if (skipped) {
      this.skipPenalty = Math.min(3, this.skipPenalty + 0.3);
      player.money += 150;
      spawnDmgNum(player.x, player.y - 30, '+$150', 'yellow');
    }
    this.buildQueue();
    this.rollModifier();
    this.active = true;
    this.betweenWave = false;
    this.showWaveAnnounce();
    updateHUD();
  }

  buildQueue() {
    this.spawnQueue = [];
    const w = this.wave;
    const sc = this.scale;

    const count = Math.floor(6 + w * 2.2);
    for (let i = 0; i < count; i++) {
      let type = 'normal';
      const r = Math.random();
      if (w >= 5 && w % 5 === 0 && i === 0) type = 'boss';
      else if (w >= 3 && r < 0.15) type = 'runner';
      else if (w >= 4 && r < 0.25) type = 'tank';
      else if (w >= 6 && r < 0.35) type = 'spitter';
      // Override with modifier
      if (this.modifier === 'fast'  && type === 'normal') type = 'runner';
      if (this.modifier === 'tanks' && type === 'normal') type = 'tank';
      this.spawnQueue.push(type);
    }
    this.zombiesLeft = this.spawnQueue.length;
  }

  rollModifier() {
    const mods = [null, null, null, 'fast', 'tanks', 'fog'];
    this.modifier = mods[randInt(0, mods.length - 1)];
  }

  showWaveAnnounce() {
    const el = document.getElementById('waveAnnounce');
    el.classList.remove('hidden');
    let text = `WAVE ${this.wave}`;
    if (this.wave % 5 === 0) text = `⚠ BOSS WAVE ${this.wave} ⚠`;
    el.innerHTML = text;
    if (this.modifier) {
      const labels = { fast: '⚡ FAST ZOMBIES', tanks: '🛡 TANK HORDE', fog: '🌫 LOW VISIBILITY' };
      el.innerHTML += `<span class="modifier-tag">${labels[this.modifier] || ''}</span>`;
    }
    setTimeout(() => el.classList.add('hidden'), 2500);
  }

  spawnZombie() {
    if (!this.spawnQueue.length) return;
    const type = this.spawnQueue.shift();
    // Spawn from random edge
    const edge = randInt(0, 3);
    let x, y;
    if (edge === 0) { x = rand(0, MAP_W * TILE); y = -20; }
    else if (edge === 1) { x = MAP_W * TILE + 20; y = rand(0, MAP_H * TILE); }
    else if (edge === 2) { x = rand(0, MAP_W * TILE); y = MAP_H * TILE + 20; }
    else { x = -20; y = rand(0, MAP_H * TILE); }
    zombies.push(new Zombie(x, y, type, this.scale));
  }

  update(dt, now) {
    if (!this.active) {
      if (this.betweenWave) {
        this.betweenTimer += dt;
        if (this.betweenTimer >= this.betweenDelay) {
          this.betweenTimer = 0;
          this.nextWave();
        }
      }
      return;
    }

    // Spawn from queue
    if (this.spawnQueue.length > 0) {
      this.spawnTimer += dt;
      if (this.spawnTimer >= this.spawnDelay) {
        this.spawnTimer = 0;
        this.spawnZombie();
      }
    }

    // Check if all dead
    const alive = zombies.filter(z => z.alive);
    if (this.spawnQueue.length === 0 && alive.length === 0) {
      this.active = false;
      this.betweenWave = true;
      this.betweenTimer = 0;
      const bonus = 100 + this.wave * 20;
      player.money += bonus;
      spawnDmgNum(player.x, player.y - 40, `+$${bonus}`, 'green');
      playSound('wave_clear', 0.3);
      updateHUD();
    }
  }
}

// ─── SOUND ─────────────────────────────────────
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
function getAudioCtx() {
  if (!audioCtx) audioCtx = new AudioCtx();
  return audioCtx;
}
function playSound(type, vol = 0.3) {
  try {
    const ac = getAudioCtx();
    const o  = ac.createOscillator();
    const g  = ac.createGain();
    o.connect(g); g.connect(ac.destination);
    g.gain.value = vol;
    const t = ac.currentTime;
    if (type === 'shoot') {
      o.type = 'sawtooth'; o.frequency.setValueAtTime(280, t);
      o.frequency.exponentialRampToValueAtTime(80, t + 0.07);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.08);
      o.start(t); o.stop(t + 0.08);
    } else if (type === 'hit') {
      o.type = 'square'; o.frequency.setValueAtTime(120, t);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.06);
      o.start(t); o.stop(t + 0.06);
    } else if (type === 'hurt') {
      o.type = 'sawtooth'; o.frequency.setValueAtTime(200, t);
      o.frequency.exponentialRampToValueAtTime(60, t + 0.15);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.15);
      o.start(t); o.stop(t + 0.15);
    } else if (type === 'kill') {
      o.type = 'sine'; o.frequency.setValueAtTime(440, t);
      o.frequency.exponentialRampToValueAtTime(880, t + 0.06);
      g.gain.exponentialRampToValueAtTime(0.001, t + 0.1);
      o.start(t); o.stop(t + 0.1);
    } else if (type === 'wave_clear') {
      o.type = 'sine';
      const freqs = [523, 659, 784];
      freqs.forEach((f, i) => {
        const oo = ac.createOscillator();
        const gg = ac.createGain();
        oo.connect(gg); gg.connect(ac.destination);
        oo.frequency.value = f;
        gg.gain.setValueAtTime(vol * 0.5, t + i*0.12);
        gg.gain.exponentialRampToValueAtTime(0.001, t + i*0.12 + 0.25);
        oo.start(t + i*0.12); oo.stop(t + i*0.12 + 0.3);
      });
    }
  } catch(e) {}
}

// ─── HUD UPDATE ────────────────────────────────
function updateHUD() {
  if (!player) return;
  const ratio = player.hp / player.maxHp;
  document.getElementById('hpBar').style.width  = (ratio * 100) + '%';
  document.getElementById('hpBar').style.background =
    ratio > 0.5 ? '#2dc653' : ratio > 0.25 ? '#f4d03f' : '#e63946';
  document.getElementById('hpText').textContent = Math.ceil(player.hp);
  document.getElementById('waveNum').textContent = waveManager.wave;
  document.getElementById('moneyDisplay').textContent = player.money;
  document.getElementById('weaponName').textContent = WEAPONS[player.equipped].name;
  document.getElementById('killCount').textContent = player.kills;
}

// ─── SHOP ──────────────────────────────────────
function buildShopUI() {
  const wEl = document.getElementById('weaponItems');
  const uEl = document.getElementById('upgradeItems');
  wEl.innerHTML = '';
  uEl.innerHTML = '';
  document.getElementById('shopMoney').textContent = '$' + player.money;

  // Weapons
  const weaponList = [
    { id: 'pistol',  ...WEAPONS.pistol  },
    { id: 'shotgun', ...WEAPONS.shotgun },
    { id: 'sniper',  ...WEAPONS.sniper  },
    { id: 'smg',     ...WEAPONS.smg     }
  ];
  for (const w of weaponList) {
    const owned    = player.weapons.includes(w.id);
    const equipped = player.equipped === w.id;
    const afford   = player.money >= w.price;
    const div = document.createElement('div');
    div.className = 'shop-item' +
      (owned && equipped ? ' equipped' : owned ? ' owned' : !afford ? ' cant-afford' : '');
    div.innerHTML = `
      <span class="item-icon">${w.icon}</span>
      <div class="item-info">
        <div class="item-name">${w.name}</div>
        <div class="item-desc">${w.desc}</div>
      </div>
      <span class="item-price">${owned ? (equipped ? '<span class="item-badge">EQUIPPED</span>' : '<span class="item-badge" style="background:#3a7bd5">EQUIP</span>') : '$'+w.price}</span>
    `;
    div.addEventListener('click', () => {
      if (owned) { player.equipped = w.id; buildShopUI(); updateHUD(); return; }
      if (!afford) return;
      player.money -= w.price;
      player.weapons.push(w.id);
      player.equipped = w.id;
      playSound('kill', 0.2);
      buildShopUI();
      updateHUD();
    });
    wEl.appendChild(div);
  }

  // Upgrades
  const upgrades = [
    { id: 'damage',   name: 'Damage Up',    icon: '💢', price: 200, desc: `+25% dmg (lv${Math.round((player.dmgMult-1)/0.25)})` },
    { id: 'firerate', name: 'Fire Rate Up', icon: '⚡', price: 250, desc: `+10% fire rate` },
    { id: 'hp',       name: 'Max HP Up',    icon: '❤️', price: 180, desc: `+30 max HP (lv${player.hpUpLevel})` }
  ];
  for (const u of upgrades) {
    const afford = player.money >= u.price;
    const div = document.createElement('div');
    div.className = 'shop-item' + (!afford ? ' cant-afford' : '');
    div.innerHTML = `
      <span class="item-icon">${u.icon}</span>
      <div class="item-info">
        <div class="item-name">${u.name}</div>
        <div class="item-desc">${u.desc}</div>
      </div>
      <span class="item-price">$${u.price}</span>
    `;
    div.addEventListener('click', () => {
      if (!afford) return;
      player.money -= u.price;
      player.upgrade(u.id);
      spawnDmgNum(player.x, player.y - 30, '⬆ ' + u.name, 'green');
      playSound('kill', 0.2);
      buildShopUI();
      updateHUD();
    });
    uEl.appendChild(div);
  }
}

function toggleShop() {
  game.shopOpen = !game.shopOpen;
  const el = document.getElementById('shopOverlay');
  if (game.shopOpen) { buildShopUI(); el.classList.remove('hidden'); }
  else el.classList.add('hidden');
}

// ─── BUILD ─────────────────────────────────────
const BUILD_TYPES = ['woodWall', 'metalWall', 'spike'];
let selectedBuild = 'woodWall';

function buildBuildMenu() {
  const el = document.getElementById('buildItems');
  el.innerHTML = '';
  for (const type of BUILD_TYPES) {
    const def = STRUCTURE_DEFS[type];
    const div = document.createElement('div');
    div.className = 'build-item' + (selectedBuild === type ? ' selected' : '');
    div.innerHTML = `
      <span class="b-icon">${def.icon}</span>
      <span class="b-name">${def.name}</span>
      <span class="b-cost">$${def.cost}</span>
    `;
    div.addEventListener('click', () => { selectedBuild = type; buildBuildMenu(); });
    el.appendChild(div);
  }
}

function toggleBuild() {
  game.buildMode = !game.buildMode;
  const el = document.getElementById('buildMenu');
  if (game.buildMode) { buildBuildMenu(); el.classList.remove('hidden'); }
  else el.classList.add('hidden');
}

function handleBuildPlace() {
  if (!game.buildMode || game.shopOpen) return;
  const wPos = screenToWorld(mouse.x, mouse.y);
  const tx = Math.floor(wPos.x / TILE);
  const ty = Math.floor(wPos.y / TILE);
  if (tx < 0 || ty < 0 || tx >= MAP_W || ty >= MAP_H) return;
  if (gameMap.tiles[ty][tx] === 1) return;
  // Check if already a structure there
  if (structures.some(s => s.tx === tx && s.ty === ty && s.alive)) return;
  const def = STRUCTURE_DEFS[selectedBuild];
  if (player.money < def.cost) { spawnDmgNum(player.x, player.y - 20, 'No $', 'red'); return; }
  player.money -= def.cost;
  structures.push(new Structure(tx, ty, selectedBuild));
  // Mark tile solid for bullets (not for movement — player can't walk through anyway)
  updateHUD();
}

// ─── BUILD PREVIEW ─────────────────────────────
function drawBuildPreview() {
  if (!game.buildMode) return;
  const wPos = screenToWorld(mouse.x, mouse.y);
  const tx = Math.floor(wPos.x / TILE);
  const ty = Math.floor(wPos.y / TILE);
  const sc = worldToScreen(tx * TILE, ty * TILE);
  const def = STRUCTURE_DEFS[selectedBuild];
  const canPlace = tx >= 0 && ty >= 0 && tx < MAP_W && ty < MAP_H
    && gameMap.tiles[ty][tx] !== 1
    && !structures.some(s => s.tx === tx && s.ty === ty && s.alive)
    && player.money >= def.cost;
  ctx.globalAlpha = 0.5;
  ctx.fillStyle = canPlace ? def.color : '#e63946';
  ctx.fillRect(sc.x, sc.y, TILE, TILE);
  ctx.strokeStyle = canPlace ? '#ffffff' : '#ff0000';
  ctx.lineWidth = 2;
  ctx.strokeRect(sc.x, sc.y, TILE, TILE);
  ctx.globalAlpha = 1;
}

// ─── FOG EFFECT ────────────────────────────────
function drawFog() {
  if (!waveManager.modifier || waveManager.modifier !== 'fog') return;
  const sc = worldToScreen(player.x, player.y);
  const gradient = ctx.createRadialGradient(sc.x, sc.y, 60, sc.x, sc.y, 280);
  gradient.addColorStop(0, 'rgba(0,0,0,0)');
  gradient.addColorStop(1, 'rgba(0,0,0,0.92)');
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── PAUSE ─────────────────────────────────────
function togglePause() {
  game.paused = !game.paused;
  const el = document.getElementById('pauseScreen');
  if (game.paused) el.classList.remove('hidden');
  else el.classList.add('hidden');
}

// ─── GAME STATE ────────────────────────────────
const game = {
  running:   false,
  paused:    false,
  over:      false,
  shopOpen:  false,
  buildMode: false,
  lastTime:  0,
  triggerGameOver() {
    this.over    = true;
    this.running = false;
    document.getElementById('hud').classList.add('hidden');
    const stats = document.getElementById('finalStats');
    stats.innerHTML = `
      <div><span class="stat-val">${waveManager.wave}</span>WAVE</div>
      <div><span class="stat-val">${player.kills}</span>KILLS</div>
      <div><span class="stat-val">$${player.money}</span>MONEY</div>
    `;
    document.getElementById('gameOverScreen').classList.remove('hidden');
  }
};

// ─── GLOBAL ENTITIES ───────────────────────────
let gameMap, player, waveManager, zombies, bullets, structures;

function initGame() {
  gameMap     = new GameMap();
  player      = new Player();
  waveManager = new WaveManager();
  zombies     = [];
  bullets     = [];
  structures  = [];

  // Close overlays
  document.getElementById('gameOverScreen').classList.add('hidden');
  document.getElementById('pauseScreen').classList.add('hidden');
  document.getElementById('shopOverlay').classList.add('hidden');
  document.getElementById('buildMenu').classList.add('hidden');
  document.getElementById('waveAnnounce').classList.add('hidden');
  document.getElementById('hud').classList.remove('hidden');

  game.running   = true;
  game.paused    = false;
  game.over      = false;
  game.shopOpen  = false;
  game.buildMode = false;
  game.lastTime  = performance.now();

  // Start first wave countdown
  waveManager.betweenWave  = true;
  waveManager.betweenDelay = 3.0;

  updateHUD();
  requestAnimationFrame(gameLoop);
}

// ─── MAIN LOOP ─────────────────────────────────
function gameLoop(now) {
  if (!game.running) return;
  if (game.over) return;

  const dt = Math.min((now - game.lastTime) / 1000, 0.05);
  game.lastTime = now;

  if (!game.paused && !game.shopOpen) {
    // Update
    player.update(dt, bullets, now);
    cam.x = player.x;
    cam.y = player.y;

    // Bullets
    for (const b of bullets) b.update(dt);

    // Bullet vs zombie / structure collision
    for (const b of bullets) {
      if (!b.alive) continue;
      if (b.fromPlayer) {
        for (const z of zombies) {
          if (!z.alive) continue;
          if (dist(b, z) < z.size + b.radius) {
            b.alive = false;
            z.takeDamage(b.damage);
            spawnDmgNum(z.x, z.y, Math.ceil(b.damage));
            if (!z.alive) {
              player.money += z.reward;
              player.kills++;
              spawnDmgNum(z.x, z.y + 15, '+$' + z.reward, 'yellow');
              playSound('kill', 0.15);
              updateHUD();
            }
            break;
          }
        }
      } else {
        // Enemy bullet vs player
        if (dist(b, player) < player.radius + b.radius) {
          b.alive = false;
          player.takeDamage(b.damage);
          updateHUD();
        }
      }
    }

    // Spike trap damage
    const now2 = now;
    for (const s of structures) {
      if (!s.alive || s.type !== 'spike') continue;
      s.spikeTimer += dt;
      if (s.spikeTimer > 0.5) {
        s.spikeTimer = 0;
        for (const z of zombies) {
          if (!z.alive) continue;
          if (dist(z, { x: s.x + TILE/2, y: s.y + TILE/2 }) < TILE/2 + z.size) {
            z.takeDamage(15);
            if (!z.alive) { player.money += z.reward; player.kills++; updateHUD(); }
          }
        }
      }
    }

    // Zombies
    for (const z of zombies) z.update(dt, player, structures.filter(s => s.alive), bullets, now2);

    // Wave
    waveManager.update(dt, now);

    // Cleanup dead
    bullets.splice(0, bullets.length, ...bullets.filter(b => b.alive));
    zombies.splice(0, zombies.length, ...zombies.filter(z => z.alive || Date.now() - z._dieTime < 300));
    structures.splice(0, structures.length, ...structures.filter(s => s.alive));
  }

  // ── RENDER ──
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = '#0a0a0a';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  gameMap.draw();

  // Structures
  for (const s of structures) s.draw();

  // Zombies
  const fogActive = waveManager.modifier === 'fog';
  for (const z of zombies) z.draw(fogActive);

  // Bullets
  for (const b of bullets) b.draw();

  // Player
  player.draw();

  // Build preview
  drawBuildPreview();

  // Fog overlay
  drawFog();

  // Between-wave countdown
  if (waveManager.betweenWave && !game.shopOpen) {
    const timeLeft = Math.ceil(waveManager.betweenDelay - waveManager.betweenTimer);
    ctx.save();
    ctx.font = '14px "Share Tech Mono"';
    ctx.textAlign = 'center';
    ctx.fillStyle = 'rgba(200,205,216,0.6)';
    ctx.fillText(`Next wave in ${timeLeft}s`, canvas.width/2, canvas.height - 60);
    ctx.restore();
  }

  requestAnimationFrame(gameLoop);
}

// ─── UI WIRING ─────────────────────────────────
document.getElementById('startBtn').addEventListener('click', () => {
  document.getElementById('startScreen').classList.add('hidden');
  initGame();
});

document.getElementById('restartBtn').addEventListener('click', () => {
  document.getElementById('gameOverScreen').classList.add('hidden');
  initGame();
});

document.getElementById('restartFromPause').addEventListener('click', () => {
  document.getElementById('pauseScreen').classList.add('hidden');
  initGame();
});

document.getElementById('resumeBtn').addEventListener('click', togglePause);
document.getElementById('pauseBtn').addEventListener('click', togglePause);
document.getElementById('shopBtn').addEventListener('click', toggleShop);
document.getElementById('buildBtn').addEventListener('click', toggleBuild);
document.getElementById('closeShop').addEventListener('click', toggleShop);
document.getElementById('closeBuild').addEventListener('click', toggleBuild);

document.getElementById('skipWaveBtn').addEventListener('click', () => {
  if (!waveManager.betweenWave && waveManager.active) {
    // Kill remaining zombies
    zombies.forEach(z => z.alive = false);
    waveManager.spawnQueue = [];
    waveManager.active = false;
    waveManager.betweenWave = false;
    waveManager.nextWave(true);
  } else if (waveManager.betweenWave) {
    waveManager.betweenTimer = waveManager.betweenDelay; // force start
    waveManager.update(0, performance.now());
    // Also give bonus
    player.money += 150;
    spawnDmgNum(player.x, player.y - 30, '+$150', 'yellow');
    updateHUD();
  }
});