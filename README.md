<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no, viewport-fit=cover" />
  <title>TON Rush — MVP (poster style, fixed)</title>
  <style>
    html,body { height:100%; margin:0; background:#0b1025; }
    #game { width:100vw; height:100dvh; }
    canvas { image-rendering: pixelated; image-rendering: crisp-edges; }
    .hud{position:fixed;left:0;right:0;top:0;display:flex;justify-content:space-between;padding:6px 8px;font:12px/1 monospace;color:#b7c6ff;pointer-events:none}
    .btnCol{position:fixed;right:8px;bottom:12px;display:flex;flex-direction:column;gap:8px}
    .btnCol .btn{background:#0a0e20;color:#e6ecff;border:1px solid #2a3b7a;border-radius:10px;padding:10px 12px;font:12px/1 monospace;box-shadow:0 0 8px rgba(36,160,255,.3)}
    .btnCol .btn:disabled{opacity:.35}
    .btnShield{position:fixed;left:8px;bottom:12px;background:#0a0e20;color:#e6ecff;border:1px solid #2a3b7a;border-radius:10px;padding:10px 12px;font:12px/1 monospace;box-shadow:0 0 8px rgba(36,160,255,.3)}
  </style>
  <script src="https://cdn.jsdelivr.net/npm/phaser@3.80.0/dist/phaser.min.js"></script>
</head>
<body>
  <div class="hud">
    <div>SPD:<span id="spd">0</span></div>
    <div>BOOST:<span id="boost">0%</span></div>
    <div>ITEMS: 🚀<span id="r">1</span> 🧨<span id="m">2</span> 🛡️<span id="s">1</span></div>
  </div>
  <div class="btnCol">
    <button class="btn" id="btnRocket">🚀 Rocket</button>
    <button class="btn" id="btnMine">🧨 Mine</button>
  </div>
  <button class="btnShield" id="btnShield">🛡️ Shield</button>
  <div id="game"></div>

<script>
/* ======================== PALETTE / CONFIG ======================== */
const WIDTH = 360, HEIGHT = 640;

const COLORS = {
  bgTop:    0x0b1025,
  bgBot:    0x12183a,
  road:     0x12224a,   // контрастнее к фону
  railGlow: 0x1dd1ff,
  railCore: 0x2fe1ff,
  center:   0x34a2ff,
  skylineA: 0x120d2b,
  skylineB: 0x1a1340,
  uiText:   0xb7c6ff,
  // obstacles
  cubeFill:     0xff8a3a,
  cubeEdge:     0xffe2b0,
  pyramidFill:  0xff5a3a,
  hedgehogFill: 0x86ff3a,
  hedgehogEdge: 0x26c6ff
};

const config = {
  type: Phaser.AUTO,
  parent: 'game',
  width: WIDTH,
  height: HEIGHT,
  pixelArt: true,
  roundPixels: true,
  backgroundColor: '#0b1025',
  scale: { mode: Phaser.Scale.FIT, autoCenter: Phaser.Scale.CENTER_BOTH },
  scene: [Boot, Race]
};

/* ======================== BOOT: MAKE TEXTURES ======================== */
class Boot extends Phaser.Scene{
  constructor(){ super('Boot'); }
  preload(){}
  create(){
    this.makeDither();
    this.makeCity();
    this.makeBikes();
    this.makeObstacles();
    this.scene.start('Race');
  }

  makeDither(){
    const g=this.make.graphics({x:0,y:0,add:false});
    g.fillStyle(0x0d1230,1);
    for(let y=0;y<16;y++) for(let x=0;x<32;x++) if((x+y*3)%9===0) g.fillRect(x,y,1,1);
    const rt=this.add.renderTexture(0,0,32,16); rt.draw(g,0,0); rt.saveTexture('dither'); rt.destroy(); g.destroy();
  }

  makeCity(){
    const band=(w,h,seed,key,color)=>{
      const g=this.make.graphics({x:0,y:0,add:false});
      g.fillStyle(color,1);
      let x=0; const rnd=new Phaser.Math.RandomDataGenerator([seed]);
      while(x<w){
        const bw=rnd.between(6,18), bh=rnd.between(h*0.35,h*0.95);
        g.fillRect(x,h-bh,bw,bh);
        // окна
        g.fillStyle(0x203069,1);
        for(let yy=h-bh+4; yy<h-3; yy+= rnd.between(6,9)) g.fillRect(x+2, yy, bw-4, 1);
        x += bw + rnd.between(2,6);
      }
      const rt=this.add.renderTexture(0,0,w,h); rt.draw(g,0,0); rt.saveTexture(key); rt.destroy(); g.destroy();
    };
    band(WIDTH, 90,  1337, 'city_far',  COLORS.skylineA);
    band(WIDTH, 120, 4242, 'city_near', COLORS.skylineB);
  }

  makeBikes(){
    const mk = (key, outline, glow)=>{
      const w=18,h=20;
      const g=this.make.graphics({x:0,y:0,add:false});
      // силуэт
      g.fillStyle(0x0a0a12,1);
      g.fillRect(7,2,4,4); g.fillRect(5,6,8,8); g.fillRect(6,14,6,4);
      // контур
      g.lineStyle(2, outline, 1); g.strokeRect(5.5,5.5,9,9);
      // «свечение» дорожки
      g.lineStyle(2, glow, 1); g.beginPath(); g.moveTo(4,16); g.lineTo(14,16); g.strokePath();
      const rt=this.add.renderTexture(0,0,w,h); rt.draw(g,0,0); rt.saveTexture(key); rt.destroy(); g.destroy();
    };
    mk('bike_player', 0x9a4dff, 0x25eaff); // фиолет+бирюза
    mk('bike_rival1', 0xff5a3a, 0xffb03a); // красный+оранж
    mk('bike_rival2', 0x86ff3a, 0x26c6ff); // лайм+голубой
  }

  makeObstacles(){
    // куб
    let g=this.make.graphics({x:0,y:0,add:false});
    g.fillStyle(COLORS.cubeFill,1); g.fillRect(0,0,18,14);
    g.lineStyle(2, COLORS.cubeEdge, .9); g.strokeRect(1,1,16,12);
    let rt=this.add.renderTexture(0,0,18,14); rt.draw(g,0,0); rt.saveTexture('obs_cube'); rt.destroy(); g.destroy();

    // высокий куб
    g=this.make.graphics({x:0,y:0,add:false});
    g.fillStyle(COLORS.cubeFill,1); g.fillRect(0,0,18,18);
    g.lineStyle(2, COLORS.cubeEdge, .9); g.strokeRect(1,1,16,16);
    rt=this.add.renderTexture(0,0,18,18); rt.draw(g,0,0); rt.saveTexture('obs_cube_tall'); rt.destroy(); g.destroy();

    // пирамида
    g=this.make.graphics({x:0,y:0,add:false});
    g.fillStyle(COLORS.pyramidFill,1);
    for(let y=0;y<14;y++){ const w=2+y; const x=9 - Math.floor(w/2); g.fillRect(x,13-y,w,1); }
    rt=this.add.renderTexture(0,0,18,14); rt.draw(g,0,0); rt.saveTexture('obs_pyramid'); rt.destroy(); g.destroy();

    // еж
    g=this.make.graphics({x:0,y:0,add:false});
    g.fillStyle(COLORS.hedgehogFill,1);
    g.fillRect(8,0,2,14); g.fillRect(0,6,18,2);
    g.fillRect(3,3,12,8);
    g.lineStyle(2, COLORS.hedgehogEdge, .9); g.strokeRect(1,1,16,12);
    rt=this.add.renderTexture(0,0,18,14); rt.draw(g,0,0); rt.saveTexture('obs_hedgehog'); rt.destroy(); g.destroy();
  }
}

/* ======================== RACE ======================== */
// слоты/перспектива
const LANES = 3;
const HALF_COUNT = LANES*2;
const TWEEN_MS = 120;
const V_CRUISE = 100, ACCEL_A=180, BRAKE_DECEL=220;

const HORIZON_Y   = 140;
const ROAD_BOTTOM = 560;
const ROAD_TOP_W  = 120;
const ROAD_BOT_W  = 280;
const ROAD_CENTER = WIDTH/2;

function roadHalfWidthAtY(y){
  const t = Phaser.Math.Clamp((y - HORIZON_Y) / (ROAD_BOTTOM - HORIZON_Y), 0, 1);
  return Phaser.Math.Linear(ROAD_TOP_W/2, ROAD_BOT_W/2, t);
}
function roadLeftAtY(y){  return ROAD_CENTER - roadHalfWidthAtY(y); }
function roadRightAtY(y){ return ROAD_CENTER + roadHalfWidthAtY(y); }
function slotToXAtY(slot, y){
  const left  = roadLeftAtY(y);
  const right = roadRightAtY(y);
  const step = (right - left) / 6;
  return left + step*(slot + 0.5);
}

class Race extends Phaser.Scene{
  constructor(){ super('Race'); }
  init(){
    this.speed=0; this.isBraking=false;
    this.playerSlot=3; // центр правая половина (0..5)
    this.playerY = ROAD_BOTTOM - 80;
    this.obstacles=[]; this.segTimer=0;
    this.items={ rocket:1, mine:2, shield:1 };
    this.trail=[];
  }

  create(){
    // градиент неба
    const sky=this.add.graphics().setDepth(-1000);
    const top=Phaser.Display.Color.IntegerToColor(COLORS.bgTop);
    const bot=Phaser.Display.Color.IntegerToColor(COLORS.bgBot);
    for(let i=0;i<HEIGHT;i++){
      const t=i/HEIGHT;
      const c=Phaser.Display.Color.Interpolate.ColorWithColor(top,bot,1,t);
      sky.fillStyle(Phaser.Display.Color.GetColor(c.r,c.g,c.b),1).fillRect(0,i,WIDTH,1);
    }

    // город
    this.cityFar  = this.add.tileSprite(WIDTH/2, HORIZON_Y+8,  WIDTH, 90,  'city_far').setOrigin(0.5,1).setAlpha(.65).setDepth(-950);
    this.cityNear = this.add.tileSprite(WIDTH/2, HORIZON_Y+24, WIDTH, 120, 'city_near').setOrigin(0.5,1).setAlpha(.95).setDepth(-940);

    // дорога
    this.roadG  = this.add.graphics().setDepth(-900);
    this.railsG = this.add.graphics().setDepth(-880);
    this.centerG= this.add.graphics().setDepth(-879);
    this.drawRoad();

    // игрок и след
    this.player=this.add.sprite(slotToXAtY(this.playerSlot, this.playerY), this.playerY, 'bike_player')
      .setOrigin(0.5,0.5).setScale(1.2).setDepth(20);

    // смещение на 1/2 по тапам
    this.input.on('pointerdown', (p)=>{
      const el=p.event.target; if(el && (el.id==='btnRocket'||el.id==='btnMine'||el.id==='btnShield')) return;
      const isLeft = p.x < this.player.x; const delta = isLeft? -1: 1;
      const next = Phaser.Math.Clamp(this.playerSlot + delta, 0, HALF_COUNT-1);
      if(next!==this.playerSlot){
        this.tweens.add({targets:this.player,x:slotToXAtY(next,this.playerY),duration:TWEEN_MS,ease:'Sine.inOut'});
        this.playerSlot=next;
      }
    });

    // HUD refs
    this.$spd=document.getElementById('spd');
    this.$boost=document.getElementById('boost');

    // генератор
    this.obsGroup=this.add.group();
  }

  // ИСПРАВЛЁННАЯ ФУНКЦИЯ ОТРИСОВКИ ДОРОГИ + РЕЛЬС
  drawRoad(){
    this.roadG.clear();
    this.railsG.clear();

    // полотно дороги
    const g = this.roadG;
    g.fillStyle(COLORS.road, 1);
    g.beginPath();
    g.moveTo(roadLeftAtY(HORIZON_Y), HORIZON_Y);
    g.lineTo(roadRightAtY(HORIZON_Y), HORIZON_Y);
    g.lineTo(roadRightAtY(ROAD_BOTTOM), ROAD_BOTTOM);
    g.lineTo(roadLeftAtY(ROAD_BOTTOM), ROAD_BOTTOM);
    g.closePath();
    g.fillPath();

    // рельсы со свечением рисуем на railsG
    const r = this.railsG;
    const drawLine = (target, w, col, a, x0,y0,x1,y1)=>{
      target.lineStyle(w, col, a);
      target.beginPath(); target.moveTo(x0,y0); target.lineTo(x1,y1); target.strokePath();
    };

    const lTop  = roadLeftAtY(HORIZON_Y),  lBot  = roadLeftAtY(ROAD_BOTTOM);
    const rTop  = roadRightAtY(HORIZON_Y), rBot  = roadRightAtY(ROAD_BOTTOM);

    // внешнее свечение
    drawLine(r, 6, COLORS.railGlow, 0.28, lTop, HORIZON_Y, lBot, ROAD_BOTTOM);
    drawLine(r, 6, COLORS.railGlow, 0.28, rTop, HORIZON_Y, rBot, ROAD_BOTTOM);
    // внутреннее свечение
    drawLine(r, 3, COLORS.railGlow, 0.65, lTop, HORIZON_Y, lBot, ROAD_BOTTOM);
    drawLine(r, 3, COLORS.railGlow, 0.65, rTop, HORIZON_Y, rBot, ROAD_BOTTOM);
    // «ядро»
    drawLine(r, 1, COLORS.railCore, 1.00, lTop, HORIZON_Y, lBot, ROAD_BOTTOM);
    drawLine(r, 1, COLORS.railCore, 1.00, rTop, HORIZON_Y, rBot, ROAD_BOTTOM);
  }

  spawnSegment(){
    // 1..3 препятствия, 4 типа, каждая занимает 1/2 полосы
    const count = Phaser.Math.Between(1,3);
    const lanes = Phaser.Utils.Array.Shuffle([0,1,2]).slice(0,count);
    lanes.forEach(ln=>{
      const half = Phaser.Math.Between(0,1);
      const type = Phaser.Math.RND.weightedPick(['cube','cube','doubleCube','pyramid','hedgehog']);
      const slot = ln*2 + half;
      const y = -20;
      const x = slotToXAtY(slot,y);
      const key = type==='cube'?'obs_cube': type==='doubleCube'?'obs_cube_tall': type==='pyramid'?'obs_pyramid':'obs_hedgehog';
      const go = this.add.sprite(x,y,key).setOrigin(0.5,1).setDepth(8);
      this.obstacles.push({slot, go, type, alive:true});
    });
  }

  update(time, delta){
    const dt=delta/1000;

    // параллакс города
    this.cityFar.tilePositionX  += 10*dt;
    this.cityNear.tilePositionX += 16*dt;

    // центр-линия (пунктир «бежит»)
    this.centerG.clear();
    this.centerG.lineStyle(2, COLORS.center, 1); // ярче
    const dash=18, gap=12;
    let y = HORIZON_Y+6, phase = (performance.now()/80)%(dash+gap);
    y += (dash+gap) - phase;
    for(; y<ROAD_BOTTOM; y+=dash+gap){
      const x = ROAD_CENTER;
      this.centerG.beginPath(); this.centerG.moveTo(x,y);
      this.centerG.lineTo(x, Math.min(y+dash, ROAD_BOTTOM)); this.centerG.strokePath();
    }

    // простая скорость (без торможения пока)
    if(this.speed < V_CRUISE) this.speed = Math.min(V_CRUISE, this.speed + ACCEL_A*dt*0.5);

    // генерация сегментов
    this.segTimer -= delta;
    if(this.segTimer<=0){ this.spawnSegment(); this.segTimer=650; }

    // скролл препятствий и пересчёт их X по перспективе
    const scroll = (60 + this.speed*0.6)*dt;
    for(const o of this.obstacles){
      if(!o.alive) continue;
      o.go.y += scroll;
      o.go.x  = slotToXAtY(o.slot, o.go.y);   // ключ: X зависит от Y
      if(o.go.y > HEIGHT+40){ o.alive=false; o.go.destroy(); }
    }

    // след шин
    this.trail.push({ x:this.player.x, y:this.player.y+10, life:1 });
    if(this.trail.length>60) this.trail.shift();
    this.drawTrail(dt);

    // HUD
    document.getElementById('spd').textContent = Math.round(this.speed);
    // буст-шкала (заглушка)
    this._boost = (this._boost||0) + 20*dt; if(this._boost>100) this._boost=0;
    document.getElementById('boost').textContent = Math.round(this._boost)+'%';
  }

  drawTrail(dt){
    if(!this.trailG){ this.trailG=this.add.graphics().setDepth(5); }
    const g=this.trailG; g.clear();
    for(const t of this.trail){
      t.life -= dt*0.8; if(t.life<0) t.life=0;
      const a = t.life*0.5;
      g.lineStyle(2, 0x203a6a, a);
      g.beginPath(); g.moveTo(t.x-4, t.y); g.lineTo(t.x-1, t.y+10); g.strokePath();
      g.beginPath(); g.moveTo(t.x+4, t.y); g.lineTo(t.x+1, t.y+10); g.strokePath();
    }
  }
}

new Phaser.Game(config);

/* ======================== UI buttons (заглушки) ======================== */
document.getElementById('btnRocket').onclick = ()=> console.log('Rocket: TODO');
document.getElementById('btnMine').onclick   = ()=> console.log('Mine: TODO');
document.getElementById('btnShield').onclick = ()=> console.log('Shield: TODO');
</script>
</body>
</html>
