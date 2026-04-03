# Boss System Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current boss (octahedron + orbiting plates) with a particle-based "energy storm" boss featuring dual-mode combat (storm/converge cycle), boss HUD, and improved hit feedback.

**Architecture:** All code lives in the single `index.html` file. The new boss is encapsulated in a `createBoss(bossLevel)` factory function returning an object with `update(dt)`, `handleHit(hitPos)`, `getHitTargets()`, `getHudData()`, and `dispose()` methods. The main game loop calls these methods instead of the old scattered functions. A lightweight inline Simplex noise drives particle turbulence.

**Tech Stack:** THREE.js (v0.152.2), vanilla JS, inline in `index.html`

**Spec:** `docs/superpowers/specs/2026-04-03-boss-system-redesign.md`

---

## File Map

All changes are in a single file:

- **Modify:** `index.html`
  - **Delete:** Old boss functions (`spawnBoss` L1759-1840, `updateBoss` L1842-1901, `bossAttack` L1903-1925, `hitBoss` L1927-2049, `destroyBoss` L2051-2085)
  - **Add (insert before old boss section):** Simplex noise utility (~50 lines)
  - **Add (replace old boss section):** `createBoss()` factory + all phase update functions (~400 lines)
  - **Add (replace old boss section):** `checkBossCollisions()` function (~80 lines)
  - **Add (in CSS section):** Boss HUD styles (~30 lines)
  - **Add (in HTML section):** Boss HUD DOM elements
  - **Modify:** `animate()` L2408-2410 — use new boss API
  - **Modify:** `resetState()` L2700-2719 — add `piercingBonus`, tunnel radius reset
  - **Modify:** `constrainToCylinder()` L2311-2320 — use dynamic tunnel radius
  - **Modify:** `checkBallObstacleCollisions()` L2323-2344 — add piercing support
  - **Modify:** Global variables L606-608 — add `piercingBonus`, `tunnelRadius`

---

## Task 1: Add Simplex Noise Utility

**Files:**
- Modify: `index.html` — insert before old boss functions (before L1759)

This is a dependency for all particle motion. Add a minimal 3D simplex noise function.

- [ ] **Step 1: Add simplex noise implementation**

Insert this block right before the `// ─── BOSS ───` section (before `function spawnBoss`). This is a standard, minimal 3D simplex noise (~50 lines):

```javascript
  // ─── SIMPLEX NOISE (for boss particles) ───
  const SimplexNoise3D = (() => {
    const grad3 = [[1,1,0],[-1,1,0],[1,-1,0],[-1,-1,0],[1,0,1],[-1,0,1],[1,0,-1],[-1,0,-1],[0,1,1],[0,-1,1],[0,1,-1],[0,-1,-1]];
    const p = [];
    for (let i = 0; i < 256; i++) p[i] = (Math.random() * 256) | 0;
    const perm = new Array(512), permMod12 = new Array(512);
    for (let i = 0; i < 512; i++) { perm[i] = p[i & 255]; permMod12[i] = perm[i] % 12; }
    const F3 = 1/3, G3 = 1/6;
    return function(x, y, z) {
      const s = (x+y+z)*F3;
      const i = Math.floor(x+s), j = Math.floor(y+s), k = Math.floor(z+s);
      const t = (i+j+k)*G3;
      const X0 = i-t, Y0 = j-t, Z0 = k-t;
      const x0 = x-X0, y0 = y-Y0, z0 = z-Z0;
      let i1,j1,k1,i2,j2,k2;
      if(x0>=y0){if(y0>=z0){i1=1;j1=0;k1=0;i2=1;j2=1;k2=0;}else if(x0>=z0){i1=1;j1=0;k1=0;i2=1;j2=0;k2=1;}else{i1=0;j1=0;k1=1;i2=1;j2=0;k2=1;}}
      else{if(y0<z0){i1=0;j1=0;k1=1;i2=0;j2=1;k2=1;}else if(x0<z0){i1=0;j1=1;k1=0;i2=0;j2=1;k2=1;}else{i1=0;j1=1;k1=0;i2=1;j2=1;k2=0;}}
      const x1=x0-i1+G3,y1=y0-j1+G3,z1=z0-k1+G3;
      const x2=x0-i2+2*G3,y2=y0-j2+2*G3,z2=z0-k2+2*G3;
      const x3=x0-1+3*G3,y3=y0-1+3*G3,z3=z0-1+3*G3;
      const ii=i&255,jj=j&255,kk=k&255;
      const dot=(g,x,y,z)=>g[0]*x+g[1]*y+g[2]*z;
      let n0=0,n1=0,n2=0,n3=0;
      let t0=0.6-x0*x0-y0*y0-z0*z0; if(t0>0){t0*=t0;n0=t0*t0*dot(grad3[permMod12[ii+perm[jj+perm[kk]]]],x0,y0,z0);}
      let t1=0.6-x1*x1-y1*y1-z1*z1; if(t1>0){t1*=t1;n1=t1*t1*dot(grad3[permMod12[ii+i1+perm[jj+j1+perm[kk+k1]]]],x1,y1,z1);}
      let t2=0.6-x2*x2-y2*y2-z2*z2; if(t2>0){t2*=t2;n2=t2*t2*dot(grad3[permMod12[ii+i2+perm[jj+j2+perm[kk+k2]]]],x2,y2,z2);}
      let t3=0.6-x3*x3-y3*y3-z3*z3; if(t3>0){t3*=t3;n3=t3*t3*dot(grad3[permMod12[ii+1+perm[jj+1+perm[kk+1]]]],x3,y3,z3);}
      return 32*(n0+n1+n2+n3);
    };
  })();
```

- [ ] **Step 2: Verify noise works**

Open browser console, run: `SimplexNoise3D(1, 2, 3)` — should return a number between -1 and 1.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(boss): add inline simplex noise utility for particle system"
```

---

## Task 2: Add Global Variables and Boss HUD DOM/CSS

**Files:**
- Modify: `index.html` — global variables (L606-608), CSS section, HTML section

- [ ] **Step 1: Add new global variables**

Find the existing boss globals at line 606-608:

```javascript
  let boss = null, bossActive = false;
  let extraBalls = 0; // permanent reward: extra balls per shot
  let fireRateBonus = 0; // permanent reward: fire rate reduction
```

Replace with:

```javascript
  let boss = null, bossActive = false;
  let extraBalls = 0; // permanent reward: extra balls per shot
  let fireRateBonus = 0; // permanent reward: fire rate reduction
  let piercingBonus = 0; // permanent reward: bullet piercing
  let tunnelRadius = 5; // dynamic tunnel radius (expands during boss)
  const TUNNEL_RADIUS_NORMAL = 5, TUNNEL_RADIUS_BOSS = 8;
```

- [ ] **Step 2: Update TUNNEL_RADIUS references to use dynamic tunnelRadius**

In `constrainToCylinder()` at line 2314, change:

```javascript
    const maxR = TUNNEL_RADIUS - BALL_RADIUS;
```

to:

```javascript
    const maxR = tunnelRadius - BALL_RADIUS;
```

In `spawnObstacle()` at line 1603, change:

```javascript
    const maxR = TUNNEL_RADIUS - size - 0.5;
```

to:

```javascript
    const maxR = tunnelRadius - size - 0.5;
```

In the powerup spawn function at line 1732, change:

```javascript
    const a = Math.random() * Math.PI * 2, r = Math.random() * (TUNNEL_RADIUS - 1.5);
```

to:

```javascript
    const a = Math.random() * Math.PI * 2, r = Math.random() * (tunnelRadius - 1.5);
```

- [ ] **Step 3: Add Boss HUD HTML**

Find the HUD section in the HTML. Add this after the existing HUD elements (after the `lives-panel` or similar container):

```html
    <div id="boss-hud" style="display:none;">
      <div id="boss-name"></div>
      <div id="boss-hp-bar-bg">
        <div id="boss-hp-bar-trail"></div>
        <div id="boss-hp-bar-fill"></div>
        <div id="boss-hp-frenzy-mark"></div>
      </div>
      <div id="boss-shield-bar-bg" style="display:none;">
        <div id="boss-shield-bar-fill"></div>
      </div>
      <div id="boss-phase-text"></div>
    </div>
```

- [ ] **Step 4: Add Boss HUD CSS**

Add these styles in the `<style>` section:

```css
    #boss-hud {
      position: fixed; top: 12px; left: 50%; transform: translateX(-50%);
      text-align: center; z-index: 100; transition: opacity 0.5s;
      pointer-events: none;
    }
    #boss-name {
      font-size: 11px; color: #aaeeff; letter-spacing: 2px;
      text-transform: uppercase; margin-bottom: 4px;
      text-shadow: 0 0 8px #00f0ff;
    }
    #boss-hp-bar-bg {
      width: 60vw; height: 8px; background: rgba(255,255,255,0.1);
      border-radius: 4px; position: relative; overflow: hidden;
      border: 1px solid rgba(0,240,255,0.3);
    }
    #boss-hp-bar-fill {
      position: absolute; top: 0; left: 0; height: 100%;
      width: 100%; border-radius: 4px;
      background: linear-gradient(90deg, #00f0ff, #8800ff);
      transition: width 0.15s ease-out;
    }
    #boss-hp-bar-trail {
      position: absolute; top: 0; left: 0; height: 100%;
      width: 100%; border-radius: 4px;
      background: rgba(255,255,255,0.3);
      transition: width 0.5s ease-out;
    }
    #boss-hp-frenzy-mark {
      position: absolute; top: -2px; bottom: -2px; width: 2px;
      left: 20%; background: rgba(255,0,85,0.7);
    }
    #boss-shield-bar-bg {
      width: 30vw; height: 4px; background: rgba(255,255,255,0.08);
      border-radius: 2px; position: relative; overflow: hidden;
      margin: 4px auto 0;
    }
    #boss-shield-bar-fill {
      position: absolute; top: 0; left: 0; height: 100%;
      width: 100%; border-radius: 2px;
      background: linear-gradient(90deg, #8800ff, #cc44ff);
      transition: width 0.15s ease-out;
    }
    #boss-phase-text {
      font-size: 10px; color: #886aaa; margin-top: 3px;
      letter-spacing: 1px;
    }
    #boss-hud.flash #boss-hp-bar-fill { background: white; }
```

- [ ] **Step 5: Add Boss HUD update function**

Add this function near the other UI functions (after `spawnBonusPopup` around L1756):

```javascript
  // ─── BOSS HUD ───
  const bossHudEl = document.getElementById('boss-hud');
  const bossNameEl = document.getElementById('boss-name');
  const bossHpFill = document.getElementById('boss-hp-bar-fill');
  const bossHpTrail = document.getElementById('boss-hp-bar-trail');
  const bossShieldBg = document.getElementById('boss-shield-bar-bg');
  const bossShieldFill = document.getElementById('boss-shield-bar-fill');
  const bossPhaseText = document.getElementById('boss-phase-text');

  let bossHpTrailValue = 1; // tracks the delayed HP display

  function showBossHud(name) {
    bossNameEl.textContent = name;
    bossHpFill.style.width = '100%';
    bossHpTrail.style.width = '100%';
    bossHpTrailValue = 1;
    bossShieldBg.style.display = 'none';
    bossPhaseText.textContent = '';
    bossHudEl.style.display = 'block';
    bossHudEl.style.opacity = '0';
    requestAnimationFrame(() => bossHudEl.style.opacity = '1');
  }

  function hideBossHud() {
    bossHudEl.style.opacity = '0';
    setTimeout(() => { bossHudEl.style.display = 'none'; }, 500);
  }

  function updateBossHud(data) {
    if (!data) return;
    const hpPct = (data.hp / data.maxHp * 100).toFixed(1) + '%';
    bossHpFill.style.width = hpPct;

    // Delayed trail
    const targetTrail = data.hp / data.maxHp;
    if (bossHpTrailValue > targetTrail) {
      bossHpTrailValue = Math.max(targetTrail, bossHpTrailValue - 0.02);
    }
    bossHpTrail.style.width = (bossHpTrailValue * 100).toFixed(1) + '%';

    // HP color: cyan > purple > red
    const ratio = data.hp / data.maxHp;
    if (ratio > 0.5) {
      bossHpFill.style.background = 'linear-gradient(90deg, #00f0ff, #8800ff)';
    } else if (ratio > 0.2) {
      bossHpFill.style.background = 'linear-gradient(90deg, #8800ff, #cc44ff)';
    } else {
      bossHpFill.style.background = 'linear-gradient(90deg, #ff0055, #ff4488)';
    }

    // Shield bar (converge phase only)
    if (data.phase === 'converge' && data.shieldHp > 0) {
      bossShieldBg.style.display = 'block';
      bossShieldFill.style.width = (data.shieldHp / data.maxShieldHp * 100).toFixed(1) + '%';
    } else {
      bossShieldBg.style.display = 'none';
    }

    // Phase text
    const phaseNames = { entrance: '接近中', storm: '风暴态', converge: '聚合态', frenzy: '狂暴', dying: '' };
    bossPhaseText.textContent = phaseNames[data.phase] || '';
    if (data.phase === 'frenzy') bossPhaseText.style.color = '#ff0055';
    else bossPhaseText.style.color = '#886aaa';
  }

  function flashBossHud() {
    bossHudEl.classList.add('flash');
    setTimeout(() => bossHudEl.classList.remove('flash'), 80);
  }

  // ─── BOSS DAMAGE NUMBERS ───
  let bossDmgCombo = 0, bossDmgComboTimer = 0;
  function spawnDamageNumber(worldPos) {
    bossDmgCombo++;
    bossDmgComboTimer = 1.0;
    const el = document.createElement('div');
    el.textContent = '1';
    el.style.cssText = 'position:fixed;pointer-events:none;z-index:200;font-weight:bold;text-shadow:0 0 6px rgba(255,255,255,0.8);transition:all 0.8s ease-out;';
    if (bossDmgCombo > 3) {
      el.style.fontSize = '18px'; el.style.color = '#ffcc00';
    } else {
      el.style.fontSize = '14px'; el.style.color = '#ffffff';
    }
    // Project 3D position to screen
    const v = worldPos.clone().project(camera);
    const x = (v.x * 0.5 + 0.5) * window.innerWidth;
    const y = (-v.y * 0.5 + 0.5) * window.innerHeight;
    el.style.left = x + 'px'; el.style.top = y + 'px';
    document.body.appendChild(el);
    requestAnimationFrame(() => { el.style.top = (y - 40) + 'px'; el.style.opacity = '0'; });
    setTimeout(() => el.remove(), 800);
  }

  // Call in animate() to decay combo
  function updateDamageCombo(dt) {
    if (bossDmgComboTimer > 0) {
      bossDmgComboTimer -= dt;
      if (bossDmgComboTimer <= 0) bossDmgCombo = 0;
    }
  }
```

- [ ] **Step 6: Add piercingBonus to resetState()**

In `resetState()` at line 2709, change:

```javascript
    bossActive = false; extraBalls = 0; fireRateBonus = 0;
```

to:

```javascript
    bossActive = false; extraBalls = 0; fireRateBonus = 0; piercingBonus = 0;
    tunnelRadius = TUNNEL_RADIUS_NORMAL;
```

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(boss): add HUD DOM/CSS, dynamic tunnel radius, piercing global"
```

---

## Task 3: Implement createBoss() Factory — Particle System & Core

**Files:**
- Modify: `index.html` — replace `spawnBoss()` function (L1759-1840)

This task creates the visual representation: particle cloud + energy core.

- [ ] **Step 1: Delete old spawnBoss function**

Delete lines 1759-1840 (the entire `function spawnBoss(bossLevel) { ... }` block).

- [ ] **Step 2: Write createBoss() factory — particle system setup**

Insert the new factory function where `spawnBoss` was. This is the first half — visual construction and state:

```javascript
  // ─── BOSS SYSTEM ───
  function createBoss(bossLevel) {
    const maxHp = 30 + bossLevel * 15;
    const baseAttackInterval = Math.max(0.4, 0.8 - bossLevel * 0.05);
    const convergeSpeed = 8 + bossLevel * 1.5;

    const group = new THREE.Group();

    // ─ Particle cloud ─
    const PARTICLE_COUNT = 250;
    const positions = new Float32Array(PARTICLE_COUNT * 3);
    const colors = new Float32Array(PARTICLE_COUNT * 3);
    const sizes = new Float32Array(PARTICLE_COUNT);
    const particleBasePos = []; // store "home" offsets for each particle

    for (let i = 0; i < PARTICLE_COUNT; i++) {
      // Ellipsoid distribution
      const theta = Math.random() * Math.PI * 2;
      const phi = Math.acos(2 * Math.random() - 1);
      const r = 1.5 + Math.random() * 2.0;
      const x = r * Math.sin(phi) * Math.cos(theta);
      const y = r * Math.sin(phi) * Math.sin(theta);
      const z = r * Math.cos(phi) * 0.7; // flatten slightly
      particleBasePos.push(x, y, z);
      positions[i * 3] = x; positions[i * 3 + 1] = y; positions[i * 3 + 2] = z;
      // Cyan-purple gradient
      const hue = 0.5 + Math.random() * 0.2; // 0.5 (cyan) to 0.7 (purple)
      const col = new THREE.Color().setHSL(hue, 0.9, 0.6);
      colors[i * 3] = col.r; colors[i * 3 + 1] = col.g; colors[i * 3 + 2] = col.b;
      sizes[i] = 3.0 + Math.random() * 4.0;
    }

    const particleGeo = new THREE.BufferGeometry();
    particleGeo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    particleGeo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
    particleGeo.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1));

    const particleMat = new THREE.PointsMaterial({
      size: 0.35, vertexColors: true, transparent: true, opacity: 0.85,
      blending: THREE.AdditiveBlending, depthWrite: false,
      sizeAttenuation: true,
    });

    const particles = new THREE.Points(particleGeo, particleMat);
    group.add(particles);

    // ─ Energy core (weak point) ─
    const coreGeo = new THREE.SphereGeometry(0.6, 16, 12);
    const coreMat = new THREE.MeshBasicMaterial({
      color: 0x00f0ff, transparent: true, opacity: 0.9,
      blending: THREE.AdditiveBlending,
    });
    const core = new THREE.Mesh(coreGeo, coreMat);
    group.add(core);

    // ─ Core glow (larger, dimmer sphere for bloom pickup) ─
    const glowGeo = new THREE.SphereGeometry(1.2, 12, 8);
    const glowMat = new THREE.MeshBasicMaterial({
      color: 0x00f0ff, transparent: true, opacity: 0.25,
      blending: THREE.AdditiveBlending,
    });
    const coreGlow = new THREE.Mesh(glowGeo, glowMat);
    group.add(coreGlow);

    group.position.z = -70;
    scene.add(group);

    // ─ State ─
    const state = {
      group, particles, particleGeo, particleMat, core, coreMat, coreGlow, glowMat,
      particleBasePos,
      hp: maxHp, maxHp,
      level: bossLevel,
      phase: 'entrance', // entrance, storm, converge, frenzy, dying
      phaseTimer: 2.0, // entrance duration
      cycleCount: 0,
      attackTimer: 0,
      baseAttackInterval,
      convergeSpeed,
      particleShieldHp: 0,
      maxParticleShieldHp: 0,
      targetZ: -20,
      shake: 0,
      // Particle animation state
      particleSpread: 3.5, // current spread radius (lerps between storm/converge)
      targetSpread: 3.5,
      colorShift: 0, // 0 = cyan, 1 = red
      targetColorShift: 0,
      coreVisible: true,
      entranceProgress: 0,
    };
```

(continued in next step)

- [ ] **Step 3: Write createBoss() — update methods**

Continue the factory function with the update dispatch and particle animation:

```javascript
    // ─ Particle animation ─
    function animateParticles(dt, time) {
      const pos = particleGeo.attributes.position.array;
      const col = particleGeo.attributes.color.array;
      const spread = state.particleSpread;
      const shift = state.colorShift;
      const hpRatio = state.hp / state.maxHp;
      // Instability increases as HP drops
      const instability = 1 + (1 - hpRatio) * 2.0;

      for (let i = 0; i < PARTICLE_COUNT; i++) {
        const bx = particleBasePos[i * 3];
        const by = particleBasePos[i * 3 + 1];
        const bz = particleBasePos[i * 3 + 2];
        // Noise-driven turbulence
        const ns = 0.4;
        const nx = SimplexNoise3D(bx * ns + time * 0.5, by * ns, bz * ns) * instability;
        const ny = SimplexNoise3D(bx * ns, by * ns + time * 0.5, bz * ns) * instability;
        const nz = SimplexNoise3D(bx * ns, by * ns, bz * ns + time * 0.5) * instability;
        // Scale by current spread
        const scale = spread / 3.5; // normalize to base spread
        pos[i * 3]     = bx * scale + nx;
        pos[i * 3 + 1] = by * scale + ny;
        pos[i * 3 + 2] = bz * scale + nz;
        // Color: lerp from cyan-purple to red based on colorShift
        const baseHue = 0.5 + (particleBasePos[i * 3] * 0.05); // slight variation per particle
        const hue = baseHue * (1 - shift) + (0.95 + particleBasePos[i * 3] * 0.02) * shift;
        const c = new THREE.Color().setHSL(hue % 1, 0.9, 0.5 + Math.sin(time * 3 + i) * 0.1);
        col[i * 3] = c.r; col[i * 3 + 1] = c.g; col[i * 3 + 2] = c.b;
      }
      particleGeo.attributes.position.needsUpdate = true;
      particleGeo.attributes.color.needsUpdate = true;

      // Leaking particles when damaged
      if (hpRatio < 0.6 && Math.random() < (1 - hpRatio) * 0.3) {
        const idx = (Math.random() * PARTICLE_COUNT) | 0;
        pos[idx * 3] += (Math.random() - 0.5) * 4;
        pos[idx * 3 + 1] += (Math.random() - 0.5) * 4;
        pos[idx * 3 + 2] += (Math.random() - 0.5) * 4;
      }
    }

    // ─ Core animation ─
    function animateCore(dt, time) {
      // Breathing pulse
      const breathe = 0.8 + Math.sin(time * 3) * 0.2;
      core.scale.setScalar(breathe);
      coreGlow.scale.setScalar(breathe * 1.8);
      // Flash recovery
      if (coreMat.color.r > 0.1 && state._coreFlash) {
        state._coreFlash -= dt * 5;
        if (state._coreFlash <= 0) {
          state._coreFlash = 0;
          const hue = state.colorShift > 0.5 ? 0.95 : 0.5;
          coreMat.color.setHSL(hue, 0.9, 0.7);
          glowMat.color.copy(coreMat.color);
        }
      }
    }
```

- [ ] **Step 4: Write createBoss() — phase update functions**

Continue with the phase-specific update logic:

```javascript
    // ─ Phase: Entrance ─
    function updateEntrance(dt) {
      state.phaseTimer -= dt;
      state.entranceProgress = 1 - (state.phaseTimer / 2.0);
      // Expand tunnel
      tunnelRadius = TUNNEL_RADIUS_NORMAL + (TUNNEL_RADIUS_BOSS - TUNNEL_RADIUS_NORMAL) * Math.min(1, state.entranceProgress);
      // Move into position
      const t = Math.min(1, state.entranceProgress);
      group.position.z = -70 + (70 + state.targetZ) * t * t; // ease-in
      // Screen effects ramp up
      shakeIntensity = Math.min(shakeIntensity + dt * 0.15, 0.3);
      if (state.phaseTimer <= 0) {
        group.position.z = state.targetZ;
        tunnelRadius = TUNNEL_RADIUS_BOSS;
        state.phase = 'storm';
        state.phaseTimer = 7; // storm duration
        state.cycleCount = 1;
        state.attackTimer = state.baseAttackInterval;
        spawnBonusPopup('⚠ BOSS 出现 ⚠');
      }
    }

    // ─ Phase: Storm (ranged barrage) ─
    function updateStorm(dt, time) {
      state.phaseTimer -= dt;
      // Spread particles out
      state.targetSpread = 3.5;
      state.targetColorShift = 0;
      state.coreVisible = true;
      // Gentle movement
      group.position.z = state.targetZ + Math.sin(time * 1.5) * 1.0;
      group.position.x = Math.cos(time * 0.8) * 2.5;
      group.position.y = Math.sin(time * 0.9) * 2.0;
      // Fire projectiles
      state.attackTimer -= dt;
      if (state.attackTimer <= 0) {
        const interval = Math.max(0.3, state.baseAttackInterval - state.cycleCount * 0.15);
        state.attackTimer = interval;
        fireStormAttack();
      }
      // Transition to converge
      if (state.phaseTimer <= 0) {
        beginConverge();
      }
    }

    // ─ Phase: Converge (charging approach) ─
    function updateConverge(dt, time) {
      state.targetSpread = 1.0;
      state.targetColorShift = 1;
      state.coreVisible = false; // hidden behind particles until shield broken
      // Boost bloom + chromatic aberration
      if (glitchPass) glitchPass.uniforms.u_aberration.value = Math.max(glitchPass.uniforms.u_aberration.value, 0.08);
      if (bloomPass) bloomPass.strength = Math.max(bloomPass.strength, 1.8);
      // Move toward player
      const speed = state.convergeSpeed + state.cycleCount * 2;
      let moveSpeed = speed;
      // Slow down from hits (temporary)
      if (state._slowTimer > 0) {
        state._slowTimer -= dt;
        moveSpeed *= 0.5;
      }
      group.position.z += moveSpeed * dt;
      // Drop debris projectiles (low freq, large)
      state.attackTimer -= dt;
      if (state.attackTimer <= 0) {
        state.attackTimer = 1.2;
        fireConvergeDebris();
      }
      // Check if shield broken -> expose core briefly
      if (state.particleShieldHp <= 0 && !state._shieldBroken) {
        state._shieldBroken = true;
        state._coreExposeTimer = 2.0;
        state.coreVisible = true;
        state.targetSpread = 3.0; // burst open
        spawnBonusPopup('核心暴露！');
        shakeIntensity = Math.min(shakeIntensity + 0.4, 0.8);
        glitchIntensity = Math.min(glitchIntensity + 0.5, 1.0);
      }
      if (state._shieldBroken && state._coreExposeTimer > 0) {
        state._coreExposeTimer -= dt;
        if (state._coreExposeTimer <= 0) {
          returnToStorm();
          return;
        }
      }
      // Reached danger zone
      if (group.position.z >= -5) {
        // Punish player
        for (let i = 0; i < 2; i++) takeDamage();
        // Force boss back
        group.position.z = state.targetZ;
        createGPUExplosion(group.position.clone(), new THREE.Color(0xff0055), 60, new THREE.Vector3(0,0,1));
        shakeIntensity = Math.min(shakeIntensity + 0.6, 1.0);
        returnToStorm();
      }
    }

    function returnToStorm() {
      state.phase = 'storm';
      state.phaseTimer = 7;
      state.cycleCount++;
      state.attackTimer = state.baseAttackInterval;
      state._shieldBroken = false;
      // Reset shield HP for new cycle
      state.particleShieldHp = getShieldHp();
      state.maxParticleShieldHp = state.particleShieldHp;
      // Retreat to far position
      group.position.z = state.targetZ;
      // Check frenzy trigger: cycle count >= 5
      if (state.cycleCount >= 5) {
        beginFrenzy();
      }
    }

    function beginConverge() {
      state.phase = 'converge';
      state.particleShieldHp = getShieldHp();
      state.maxParticleShieldHp = state.particleShieldHp;
      state.attackTimer = 1.2;
      state._shieldBroken = false;
      state._slowTimer = 0;
      // Visual signal: particles contract + color shift
      spawnBonusPopup('⚠ 聚合 ⚠');
    }

    function getShieldHp() {
      const base = [0, 8, 12, 15, 18, 18];
      const idx = Math.min(state.cycleCount, 5);
      return (base[idx] || 8) + bossLevel * 3;
    }

    // ─ Phase: Frenzy ─
    function beginFrenzy() {
      state.phase = 'frenzy';
      state.targetSpread = 2.5;
      state.targetColorShift = 0.8;
      state.coreVisible = true;
      state.attackTimer = 0.4;
      spawnBonusPopup('⚠ 狂暴！ ⚠');
      shakeIntensity = Math.min(shakeIntensity + 0.6, 1.0);
      glitchIntensity = 1.0;
    }

    function updateFrenzy(dt, time) {
      state.targetColorShift = 0.5 + Math.sin(time * 8) * 0.5; // red-white flicker
      state.coreVisible = true;
      // Slow approach
      if (group.position.z < -5) {
        group.position.z += 4 * dt;
      }
      group.position.x = Math.cos(time * 1.5) * 2.0;
      group.position.y = Math.sin(time * 1.3) * 1.5;
      // Continuous barrage
      state.attackTimer -= dt;
      if (state.attackTimer <= 0) {
        state.attackTimer = 0.4;
        fireStormAttack();
      }
    }

    // ─ Phase: Dying ─
    function updateDying(dt, time) {
      state.phaseTimer -= dt;
      const t = 1 - (state.phaseTimer / 2.0);
      if (t < 0.5) {
        // Contract
        state.targetSpread = 3.5 * (1 - t * 2);
        particleMat.opacity = 0.85;
      } else {
        // Explode outward
        state.targetSpread = 3.5 + (t - 0.5) * 20;
        particleMat.opacity = Math.max(0, 1 - (t - 0.5) * 2);
        coreMat.opacity = Math.max(0, 1 - (t - 0.5) * 3);
      }
      state.shake = 0.5 + t * 1.5;
      if (state.phaseTimer <= 0) {
        disposeInternal();
      }
    }
```

- [ ] **Step 5: Write createBoss() — attack functions**

```javascript
    // ─ Attacks ─
    function fireStormAttack() {
      const count = 2 + state.cycleCount;
      const patterns = ['fan', 'spiral', 'random'];
      const pattern = patterns[state._attackCount % patterns.length];
      state._attackCount = (state._attackCount || 0) + 1;

      for (let i = 0; i < count; i++) {
        let angle, r;
        if (pattern === 'fan') {
          const spread = Math.PI * 0.6;
          angle = -spread / 2 + (spread / (count - 1 || 1)) * i;
          r = 1.5 + Math.random() * 1.5;
        } else if (pattern === 'spiral') {
          angle = (Math.PI * 2 / count) * i + globalTime * 2;
          r = 2.0 + Math.random() * 1.0;
        } else {
          angle = Math.random() * Math.PI * 2;
          r = 1.0 + Math.random() * 2.5;
        }
        const px = Math.cos(angle) * r + group.position.x;
        const py = Math.sin(angle) * r + group.position.y;
        const pz = group.position.z - 1;
        const speed = OBSTACLE_SPEED * 1.3;
        const color = new THREE.Color().setHSL(0.55, 0.8, 0.5);
        const emissive = new THREE.Color().setHSL(0.55, 0.9, 0.4);
        const entry = acquireObstacle(px, py, pz, 0.5, 1, color, emissive, speed);
        if (entry) {
          obstacles.push({ mesh: entry.group, body: entry.body, speed, size: 0.5, hp: 1, maxHp: 1, color: color.clone(), coreMat: entry.coreMat, shellMat: entry.shellMat, _poolEntry: entry });
        }
      }
    }

    function fireConvergeDebris() {
      const count = 1 + Math.floor(state.cycleCount / 2);
      for (let i = 0; i < count; i++) {
        const angle = Math.random() * Math.PI * 2;
        const r = 1.0 + Math.random() * 1.5;
        const px = Math.cos(angle) * r + group.position.x;
        const py = Math.sin(angle) * r + group.position.y;
        const pz = group.position.z + 1;
        const speed = OBSTACLE_SPEED * 0.8;
        const color = new THREE.Color().setHSL(0.85, 0.8, 0.5);
        const emissive = new THREE.Color().setHSL(0.85, 0.9, 0.4);
        const entry = acquireObstacle(px, py, pz, 0.8, 1, color, emissive, speed);
        if (entry) {
          obstacles.push({ mesh: entry.group, body: entry.body, speed, size: 0.8, hp: 1, maxHp: 1, color: color.clone(), coreMat: entry.coreMat, shellMat: entry.shellMat, _poolEntry: entry });
        }
      }
    }
```

- [ ] **Step 6: Write createBoss() — hit handling and public interface**

```javascript
    // ─ Hit handling ─
    state._coreFlash = 0;
    state._slowTimer = 0;
    state._attackCount = 0;
    state._shieldBroken = false;
    state._coreExposeTimer = 0;

    function handleHit(hitPos) {
      if (state.phase === 'entrance' || state.phase === 'dying') return { damaged: false };

      const bossPos = group.position;
      const dx = hitPos.x - bossPos.x, dy = hitPos.y - bossPos.y, dz = hitPos.z - bossPos.z;
      const dist = Math.sqrt(dx * dx + dy * dy + dz * dz);

      // In converge phase with active shield: hit shield
      if (state.phase === 'converge' && state.particleShieldHp > 0) {
        if (dist < 4.0) {
          state.particleShieldHp--;
          state._slowTimer = 0.3; // slow boss temporarily
          // Scatter particles at hit point
          createGPUExplosion(hitPos.clone(), new THREE.Color(0x8800ff), 15, new THREE.Vector3(0, 0, 1));
          return { damaged: true, amount: 0, type: 'shield' };
        }
        return { damaged: false };
      }

      // Core hit (storm, frenzy, or exposed in converge)
      if (state.coreVisible && dist < 2.5) {
        state.hp--;
        state.shake = 0.5;
        state._coreFlash = 1;
        coreMat.color.set(0xffffff);
        glowMat.color.set(0xffffff);
        flashBossHud();
        // Particle burst + damage number
        createGPUExplosion(hitPos.clone(), new THREE.Color(0x00f0ff), 30, new THREE.Vector3(0, 0, 1));
        spawnDamageNumber(hitPos.clone());
        shakeIntensity = Math.min(shakeIntensity + 0.2, 0.6);
        hitStopTimer = 0.03;
        playHit();

        // Check frenzy trigger
        if (state.hp > 0 && state.hp <= state.maxHp * 0.2 && state.phase !== 'frenzy') {
          beginFrenzy();
        }
        // Check death
        if (state.hp <= 0) {
          state.phase = 'dying';
          state.phaseTimer = 2.0;
          hitStopTimer = 0.1;
          shakeIntensity = 1.0;
          glitchIntensity = 1.0;
          // Staggered explosions
          for (let j = 0; j < 6; j++) {
            setTimeout(() => {
              if (group.parent) {
                const p = group.position.clone().add(new THREE.Vector3((Math.random()-0.5)*5, (Math.random()-0.5)*5, (Math.random()-0.5)*5));
                createGPUExplosion(p, new THREE.Color(0xff0055), 100, new THREE.Vector3(0,0,1));
              }
            }, j * 200);
          }
        }
        return { damaged: true, amount: 1, type: 'core' };
      }

      return { damaged: false };
    }

    function getHitTargets() {
      const targets = [];
      const bossPos = group.position;
      if (state.phase === 'converge' && state.particleShieldHp > 0) {
        targets.push({ pos: bossPos.clone(), radius: 4.0, type: 'shield' });
      }
      if (state.coreVisible) {
        targets.push({ pos: bossPos.clone(), radius: 2.5, type: 'core' });
      }
      return targets;
    }

    function getHudData() {
      return {
        hp: state.hp, maxHp: state.maxHp,
        phase: state.phase,
        name: '维度风暴 Lv.' + bossLevel,
        shieldHp: state.particleShieldHp,
        maxShieldHp: state.maxParticleShieldHp,
      };
    }
```

- [ ] **Step 7: Write createBoss() — main update, dispose, and return**

```javascript
    // ─ Main update ─
    function update(dt) {
      const time = globalTime;

      // Lerp particle spread & color
      state.particleSpread += (state.targetSpread - state.particleSpread) * Math.min(1, dt * 4);
      state.colorShift += (state.targetColorShift - state.colorShift) * Math.min(1, dt * 4);

      // Shake
      if (state.shake > 0) {
        group.position.x += (Math.random() - 0.5) * state.shake;
        group.position.y += (Math.random() - 0.5) * state.shake;
        state.shake *= Math.pow(0.01, dt);
        if (state.shake < 0.01) state.shake = 0;
      }

      // Phase dispatch
      if (state.phase === 'entrance') updateEntrance(dt);
      else if (state.phase === 'storm') updateStorm(dt, time);
      else if (state.phase === 'converge') updateConverge(dt, time);
      else if (state.phase === 'frenzy') updateFrenzy(dt, time);
      else if (state.phase === 'dying') updateDying(dt, time);

      // Always animate visuals
      animateParticles(dt, time);
      animateCore(dt, time);
    }

    // ─ Dispose ─
    function disposeInternal() {
      scene.remove(group);
      particleGeo.dispose();
      particleMat.dispose();
      coreGeo.dispose();
      coreMat.dispose();
      glowGeo.dispose();
      glowMat.dispose();

      // Rewards
      const bossScore = 8000 * bossLevel;
      score += bossScore; scoreEl.textContent = score;
      spawnScorePopup(bossScore);

      // Final explosion — gold
      createGPUExplosion(group.position.clone(), new THREE.Color(0xffcc00), 150, new THREE.Vector3(0,0,1));

      // Permanent buff (3-way rotation)
      const buffType = bossLevel % 3;
      if (buffType === 1) {
        extraBalls++;
        spawnBonusPopup('永久强化: 每发 +1 球');
      } else if (buffType === 2) {
        fireRateBonus += 0.02;
        spawnBonusPopup('永久强化: 射速提升');
      } else {
        piercingBonus++;
        spawnBonusPopup('永久强化: 穿透 +1');
      }

      // Screen effects
      glitchIntensity = 1.0;
      fovPulse = 1.0;
      shakeIntensity = 0.8;
      tunnelUniforms.u_hueShift.value += 0.15;

      // Tunnel shrink back
      const shrinkStart = tunnelRadius;
      const shrinkDuration = 1.0;
      let shrinkTime = 0;
      const shrinkInterval = setInterval(() => {
        shrinkTime += 0.016;
        const t = Math.min(1, shrinkTime / shrinkDuration);
        tunnelRadius = shrinkStart + (TUNNEL_RADIUS_NORMAL - shrinkStart) * t;
        if (t >= 1) { tunnelRadius = TUNNEL_RADIUS_NORMAL; clearInterval(shrinkInterval); }
      }, 16);

      boss = null;
      bossActive = false;
      hideBossHud();

      waveBreakTimer = WAVE_BREAK_DURATION;
      lives = MAX_LIVES; updateLivesUI();
    }

    function dispose() {
      if (group.parent) scene.remove(group);
      particleGeo.dispose(); particleMat.dispose();
      coreGeo.dispose(); coreMat.dispose();
      glowGeo.dispose(); glowMat.dispose();
    }

    // ─ Public interface ─
    return {
      get hp() { return state.hp; },
      get maxHp() { return state.maxHp; },
      get phase() { return state.phase; },
      get group() { return group; },
      update,
      handleHit,
      getHitTargets,
      getHudData,
      dispose,
    };
  }
```

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(boss): implement createBoss factory with particle system and dual-mode combat"
```

---

## Task 4: Replace Old Boss Functions with New Collision System

**Files:**
- Modify: `index.html` — delete old functions, add new collision, update game loop

- [ ] **Step 1: Delete old boss functions**

Delete these entire functions:
- `updateBoss()` (L1842-1901 — now inside boss.update)
- `bossAttack()` (L1903-1925 — now inside createBoss)
- `hitBoss()` (L1927-2049 — replaced by checkBossCollisions)
- `destroyBoss()` (L2051-2085 — now inside createBoss.disposeInternal)

- [ ] **Step 2: Add checkBossCollisions()**

Insert where the old `hitBoss()` was:

```javascript
  function checkBossCollisions(b) {
    if (!b || b.phase === 'entrance' || b.phase === 'dying') return;
    for (let i = balls.length - 1; i >= 0; i--) {
      const ball = balls[i];
      const bp = ball.body.position;
      const result = b.handleHit(new THREE.Vector3(bp.x, bp.y, bp.z));
      if (result.damaged) {
        releaseBullet(ball);
        balls.splice(i, 1);
      }
    }
  }
```

- [ ] **Step 3: Update game loop — boss section**

In `animate()`, find the boss active block (around L2408):

```javascript
    if (bossActive) {
      updateBoss(dt);
      hitBoss();
    }
```

Replace with:

```javascript
    if (bossActive && boss) {
      boss.update(dt);
      checkBossCollisions(boss);
      updateBossHud(boss.getHudData());
      updateDamageCombo(dt);
    }
```

- [ ] **Step 4: Update boss spawn trigger**

In the wave break section (around L2418), find:

```javascript
          const bossLevel = wave / 10;
          spawnBoss(bossLevel);
          wavePanelEl.textContent = 'BOSS ' + bossLevel;
```

Replace with:

```javascript
          const bossLevel = wave / 10;
          boss = createBoss(bossLevel);
          bossActive = true;
          showBossHud('维度风暴 Lv.' + bossLevel);
          announceWave('⚠ 警告: 维度风暴 接近 ⚠');
          wavePanelEl.textContent = 'BOSS ' + bossLevel;
```

- [ ] **Step 5: Update resetState() boss cleanup**

In `resetState()`, find:

```javascript
    if (boss) { scene.remove(boss.group); boss = null; }
```

Replace with:

```javascript
    if (boss) { boss.dispose(); boss = null; }
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(boss): wire up new boss system, remove old boss functions"
```

---

## Task 5: Add Bullet Piercing Mechanic

**Files:**
- Modify: `index.html` — `checkBallObstacleCollisions()` (L2323-2344)

- [ ] **Step 1: Modify checkBallObstacleCollisions to support piercing**

Find the current function (L2323):

```javascript
  function checkBallObstacleCollisions() {
    for (let bi = balls.length - 1; bi >= 0; bi--) {
      const bp = balls[bi].body.position;
      for (let oi = obstacles.length - 1; oi >= 0; oi--) {
        const obs = obstacles[oi], op = obs.body.position;
        const dx = bp.x-op.x, dy = bp.y-op.y, dz = bp.z-op.z;
        const dist = Math.sqrt(dx*dx+dy*dy+dz*dz);
        const hitDist = BALL_RADIUS + obs.size * 0.55;
        if (dist < hitDist && dist > 0.001) {
          const nx=dx/dist, ny=dy/dist, nz=dz/dist;
          const v = balls[bi].body.velocity;
          const dot = v.x*nx+v.y*ny+v.z*nz;
          if (dot < 0) { v.x -= 2*dot*nx; v.y -= 2*dot*ny; v.z -= 2*dot*nz; }
          const overlap = hitDist - dist;
          bp.x += nx*overlap; bp.y += ny*overlap; bp.z += nz*overlap;
          // Pass ball velocity direction for directional explosion
          const bv = balls[bi].body.velocity;
          const ballDir = new THREE.Vector3(bv.x, bv.y, bv.z).normalize();
          hitObstacle(oi, ballDir); break;
        }
      }
    }
  }
```

Replace with:

```javascript
  function checkBallObstacleCollisions() {
    for (let bi = balls.length - 1; bi >= 0; bi--) {
      const ball = balls[bi];
      const bp = ball.body.position;
      let pierceRemaining = ball._pierceLeft !== undefined ? ball._pierceLeft : piercingBonus;
      let destroyed = false;
      for (let oi = obstacles.length - 1; oi >= 0; oi--) {
        const obs = obstacles[oi], op = obs.body.position;
        const dx = bp.x-op.x, dy = bp.y-op.y, dz = bp.z-op.z;
        const dist = Math.sqrt(dx*dx+dy*dy+dz*dz);
        const hitDist = BALL_RADIUS + obs.size * 0.55;
        if (dist < hitDist && dist > 0.001) {
          const nx=dx/dist, ny=dy/dist, nz=dz/dist;
          const overlap = hitDist - dist;
          bp.x += nx*overlap; bp.y += ny*overlap; bp.z += nz*overlap;
          const bv = ball.body.velocity;
          const ballDir = new THREE.Vector3(bv.x, bv.y, bv.z).normalize();
          hitObstacle(oi, ballDir);
          if (pierceRemaining > 0) {
            pierceRemaining--;
            ball._pierceLeft = pierceRemaining;
            // Don't bounce — keep going straight
            continue;
          } else {
            // Bounce as before
            const v = ball.body.velocity;
            const dot = v.x*nx+v.y*ny+v.z*nz;
            if (dot < 0) { v.x -= 2*dot*nx; v.y -= 2*dot*ny; v.z -= 2*dot*nz; }
            destroyed = false;
            break;
          }
        }
      }
    }
  }
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat(boss): add bullet piercing mechanic for permanent reward"
```

---

## Task 6: Update Tunnel Geometry to Use Dynamic Radius

**Files:**
- Modify: `index.html` — `createTunnel()` and tunnel shader

The tunnel is a static `CylinderGeometry`. To make radius dynamic, we need to pass `tunnelRadius` as a shader uniform so the vertex shader can scale it.

- [ ] **Step 1: Add tunnelRadius uniform**

In `createTunnel()`, add to `tunnelUniforms` (around L1454):

```javascript
      u_tunnelRadius: { value: TUNNEL_RADIUS_NORMAL },
      u_baseTunnelRadius: { value: TUNNEL_RADIUS_NORMAL },
```

- [ ] **Step 2: Modify tunnel vertex shader**

Find the `TUNNEL_VERT` shader string. Add the radius scaling. The tunnel geometry is built with `TUNNEL_RADIUS = 5`, so we scale vertex positions by the ratio `u_tunnelRadius / u_baseTunnelRadius`:

Find the vertex shader variable declaration section and add:

```glsl
uniform float u_tunnelRadius;
uniform float u_baseTunnelRadius;
```

In the vertex shader's main(), before `gl_Position`, add this line to scale the X/Y position (the radial component):

```glsl
vec3 scaled = position;
float radiusScale = u_tunnelRadius / u_baseTunnelRadius;
scaled.xy *= radiusScale;
```

Then use `scaled` instead of `position` in the `gl_Position` calculation.

- [ ] **Step 3: Update tunnel uniform each frame**

In the `animate()` function, find where tunnel uniforms are updated (search for `tunnelUniforms.u_time.value`). Add nearby:

```javascript
    tunnelUniforms.u_tunnelRadius.value = tunnelRadius;
```

- [ ] **Step 4: Update constrainToCylinder to match**

Already done in Task 2 Step 2 — `constrainToCylinder` uses `tunnelRadius` variable.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(boss): dynamic tunnel radius via shader uniform for boss expansion"
```

---

## Task 7: Integration Testing & Polish

**Files:**
- Modify: `index.html` — minor fixes and tuning

- [ ] **Step 1: Quick-test boss spawn**

Open the game in browser. To test quickly, temporarily change the boss spawn condition from `wave % 10 === 0` to `wave % 2 === 0` (around L2418). Play through wave 2 and verify:

1. Tunnel expands smoothly during entrance
2. Particle cloud appears and animates
3. Boss HUD slides in with name and full health bar

- [ ] **Step 2: Test storm phase**

During storm phase, verify:
- Projectiles fire in alternating patterns (fan, spiral, random)
- Player can shoot and hit the core (HP decreases, HUD updates)
- Hit feedback: core flashes white, particles burst, screen shake

- [ ] **Step 3: Test converge phase**

After 7 seconds, verify:
- Boss contracts and charges forward (color shifts to red)
- Shield bar appears in HUD
- Hitting the shield slows the boss
- Breaking shield triggers "核心暴露！" message
- If boss reaches z=-5, player loses 2 lives and boss retreats

- [ ] **Step 4: Test frenzy and death**

Reduce boss HP in console (`boss.hp = 5`) to test:
- Frenzy triggers at 20% HP (red-white flicker, both threats active)
- Death animation: contract → explode → gold particles
- Tunnel shrinks back to normal
- Rewards granted (score, permanent buff)
- HUD fades out

- [ ] **Step 5: Revert test condition**

Change boss spawn back to `wave % 10 === 0`.

- [ ] **Step 6: Final commit**

```bash
git add index.html
git commit -m "feat(boss): complete boss system redesign with particle storm and dual-mode combat"
```
