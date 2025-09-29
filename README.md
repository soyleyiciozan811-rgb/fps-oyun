[fps.html](https://github.com/user-attachments/files/22605437/fps.html)
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<title>FPS Demo - Kapı Düzgün</title>
<meta name="viewport" content="width=device-width,initial-scale=1" />
<style>
  html,body { height:100%; margin:0; background:#000; }
  canvas { display:block; display:block; }
  #crosshair {
    position:absolute; left:50%; top:50%;
    width:6px; height:6px; margin-left:-3px; margin-top:-3px;
    background:#ff4d4d; border-radius:50%; pointer-events:none; z-index:5;
  }
  #weapon2d {
    position:absolute; right:35%; bottom:8%; width:160px; height:80px;
    background-size:contain; background-repeat:no-repeat; pointer-events:none; z-index:5;
  }
  #scoreboard { position:absolute; left:12px; top:10px; color:white; font:18px/1 sans-serif; z-index:6; }
  #marketBtn, #settingsBtn {
    position:absolute; top:10px; font-size:26px; cursor:pointer;
    color:white; user-select:none; z-index:11; padding:6px; background:rgba(0,0,0,0.18); border-radius:6px;
  }
  #marketBtn { right:70px; }
  #settingsBtn { right:12px; }
  .panel { position:fixed; left:0; top:0; width:100%; height:100%; display:none; align-items:center; justify-content:center; z-index:20; }
  #settingsPanel { background:rgba(0,0,0,0.5); }   /* 50% */
  #marketPanel { background:rgba(0,0,0,0.25); }    /* 25% */
  .panel .card { background:rgba(255,255,255,0.04); padding:18px; border-radius:10px; color:white; text-align:center; min-width:300px; }
  .panel button { margin:8px; padding:10px 16px; font-size:16px; cursor:pointer; }
  .market-items { display:flex; gap:12px; justify-content:center; flex-wrap:wrap; margin-top:12px; }
  .market-items img { width:120px; border:2px solid rgba(255,255,255,0.25); cursor:pointer; border-radius:6px; }
  #message {
    position:absolute; left:50%; top:50%; transform:translate(-50%,-50%); color:white; font-size:28px;
    display:none; z-index:30; background:rgba(0,0,0,0.6); padding:12px 18px; border-radius:8px;
  }
  .hide-cursor { cursor:none; }
</style>
</head>
<body class="hide-cursor">
  <div id="crosshair"></div>
  <div id="weapon2d" style="background-image:url('img/pistol.png')"></div>
  <div id="scoreboard">Skor: 0</div>
  <div id="marketBtn" title="Market (E)">🛒</div>
  <div id="settingsBtn" title="Ayarlar (F/Esc)">⚙️</div>
  <div id="message">Yakında...</div>

  <!-- Settings panel -->
  <div id="settingsPanel" class="panel" aria-hidden="true">
    <div class="card">
      <h3>Ayarlar</h3>
      <div style="margin-top:8px;">
        <button id="btnView">Görüntü</button>
        <button id="btnCross">Nişangah</button>
        <button id="btnSettingsClose">Kapat</button>
      </div>
    </div>
  </div>

  <!-- Market panel -->
  <div id="marketPanel" class="panel" aria-hidden="true">
    <div class="card">
      <h3>Market - Silahlar</h3>
      <div style="font-size:13px; opacity:0.9;">Not: silah almak skoru sıfırlar.</div>
      <div class="market-items" id="marketItems">
        <img src="img/pistol.png" alt="Pistol" data-weapon="pistol" data-unlock="0" title="Tabanca - Başlangıç">
        <img src="img/silenced.png" alt="Silenced" data-weapon="silenced" data-unlock="25" title="Susturuculu Tabanca - unlock:25">
        <img src="img/uzi.png" alt="Uzi" data-weapon="uzi" data-unlock="50" title="Uzi - unlock:50">
        <img src="img/ak47.png" alt="AK47" data-weapon="ak47" data-unlock="100" title="AK47 - unlock:100">
      </div>
      <div style="margin-top:12px;">
        <button id="btnMarketClose">Kapat</button>
      </div>
    </div>
  </div>

<script src="https://cdn.jsdelivr.net/npm/three/build/three.min.js"></script>
<script>
/* ================= SETUP ================= */
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

const mapSize = 50;
let score = 0;
const scoreboard = document.getElementById('scoreboard');
const messageEl = document.getElementById('message');

const loader = new THREE.TextureLoader();

/* ---- Floor ---- */
const floor = new THREE.Mesh(new THREE.PlaneGeometry(mapSize,mapSize), new THREE.MeshBasicMaterial({color:0x228B22}));
floor.rotation.x = -Math.PI/2; scene.add(floor);

/* ---- 3D trees (plane textured) ---- */
loader.load('img/tree.png', tex => {
  for(let i=0;i<20;i++){
    const plane = new THREE.Mesh(new THREE.PlaneGeometry(1.6,2.4), new THREE.MeshBasicMaterial({map:tex, transparent:true}));
    plane.position.set(Math.random()*mapSize-mapSize/2,1.2,Math.random()*mapSize-mapSize/2);
    // rotate random a bit for variety
    plane.rotation.y = Math.random()*Math.PI*2;
    scene.add(plane);
  }
});

/* ---- Door (plane with texture) ---- */
let doorMesh = null;
loader.load('img/door.png', tex=>{
  const p = new THREE.Mesh(new THREE.PlaneGeometry(2,4), new THREE.MeshBasicMaterial({map:tex, transparent:true}));
  p.position.set(0,2,mapSize/2 - 6);
  p.name = 'door';
  doorMesh = p;
  scene.add(p);
});

/* ================= CONTROLS ================= */
let keys = {};
window.addEventListener('keydown', e => {
  keys[e.key.toLowerCase()] = true;
  if(e.key.toLowerCase() === 'e') toggleMarket();
  if(e.key === 'f' || e.key === 'F' || e.key === 'Escape') toggleSettings();
});
window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

let yaw = 0, pitch = 0;
function requestPointerLock(){ if(document.body.requestPointerLock) document.body.requestPointerLock(); }
function exitPointerLock(){ if(document.exitPointerLock) document.exitPointerLock(); }

document.addEventListener('mousemove', e=>{
  if(document.pointerLockElement === document.body && !settingsOpen && !marketOpen){
    yaw -= e.movementX * 0.002;
    pitch += e.movementY * 0.002; // corrected direction (mouse up -> look up)
    pitch = Math.max(-Math.PI/2, Math.min(Math.PI/2, pitch));
  }
});

// click to grab pointer when panels closed
document.body.addEventListener('click', ()=>{
  if(!settingsOpen && !marketOpen){
    requestPointerLock();
    document.body.style.cursor = 'none';
    document.body.classList.add('hide-cursor');
  }
});

/* ================= ENEMIES (humans with limbs) ================= */
function createHuman(x,z){
  const human = new THREE.Group();
  human.userData = { bodyHits: 0, alive: true };

  const bodyMat = new THREE.MeshBasicMaterial({ color: 0xff4444 });
  const limbMat = new THREE.MeshBasicMaterial({ color: 0xff4444 });
  const headMat = new THREE.MeshBasicMaterial({ color: 0xfff07a });

  const body = new THREE.Mesh(new THREE.BoxGeometry(0.5,1.2,0.3), bodyMat); body.position.y = 0.9; body.name = 'body'; human.add(body);
  const head = new THREE.Mesh(new THREE.SphereGeometry(0.25,16,16), headMat); head.position.y = 1.8; head.name = 'head'; human.add(head);

  const leftArm = new THREE.Mesh(new THREE.BoxGeometry(0.18,0.8,0.18), limbMat); leftArm.position.set(-0.45,1.05,0); leftArm.name='body'; human.add(leftArm);
  const rightArm = new THREE.Mesh(new THREE.BoxGeometry(0.18,0.8,0.18), limbMat); rightArm.position.set(0.45,1.05,0); rightArm.name='body'; human.add(rightArm);

  const leftLeg = new THREE.Mesh(new THREE.BoxGeometry(0.18,0.9,0.18), new THREE.MeshBasicMaterial({color:0x2b6bff})); leftLeg.position.set(-0.15,0.1,0); leftLeg.name='body'; human.add(leftLeg);
  const rightLeg = new THREE.Mesh(new THREE.BoxGeometry(0.18,0.9,0.18), new THREE.MeshBasicMaterial({color:0x2b6bff})); rightLeg.position.set(0.15,0.1,0); rightLeg.name='body'; human.add(rightLeg);

  human.position.set(x,0,z);
  scene.add(human);
  return human;
}

const enemies = [];
for(let i=0;i<10;i++){
  enemies.push(createHuman(Math.random()*mapSize-mapSize/2, Math.random()*mapSize-mapSize/2));
}

function teleportHuman(h){
  const angle = Math.random()*Math.PI*2;
  const dist = Math.random()*10;
  const nx = THREE.MathUtils.clamp(camera.position.x + Math.cos(angle)*dist, -mapSize/2+1, mapSize/2-1);
  const nz = THREE.MathUtils.clamp(camera.position.z + Math.sin(angle)*dist, -mapSize/2+1, mapSize/2-1);
  h.position.set(nx, 0, nz);
  h.userData.bodyHits = 0;
  h.traverse(obj => {
    if(obj.isMesh){
      if(obj.name === 'head') obj.material.color.set(0xfff07a);
      else obj.material.color.set(0xff4444);
    }
  });
}

/* ================= WEAPONS ================= */
const WEAPONS = {
  pistol: { id:'pistol', label:'Tabanca', unlock:0, bodyHits:3, headHits:1, skin:'img/pistol.png', suppressed:false },
  silenced: { id:'silenced', label:'Susturuculu', unlock:25, bodyHits:3, headHits:1, skin:'img/silenced.png', suppressed:true },
  uzi: { id:'uzi', label:'Uzi', unlock:50, bodyHits:2, headHits:1, skin:'img/uzi.png', suppressed:false },
  ak47: { id:'ak47', label:'AK-47', unlock:100, bodyHits:1, headHits:1, skin:'img/ak47.png', suppressed:false }
};
let currentWeapon = WEAPONS.pistol;
const weapon2d = document.getElementById('weapon2d');
weapon2d.style.backgroundImage = `url('${currentWeapon.skin}')`;

/* helper to find human group */
function getHumanGroup(obj){
  let cur = obj;
  while(cur){
    if(cur.userData && typeof cur.userData.bodyHits !== 'undefined') return cur;
    cur = cur.parent;
  }
  return null;
}

/* ================= SHOOTING & RAYCAST ================= */
function updateScoreDisplay(){ scoreboard.textContent = `Skor: ${score}`; }
function handleHit(intersect){
  const o = intersect.object;
  // door check (doorMesh exists)
  if(doorMesh && (o === doorMesh || o.parent === doorMesh || o.name === 'door')) {
    messageEl.textContent = 'Yakında...';
    messageEl.style.display = 'block';
    setTimeout(()=>{ messageEl.style.display = 'none'; messageEl.textContent = 'Yakında...'; }, 1400);
    return;
  }

  const parent = getHumanGroup(o);
  if(!parent || !parent.userData.alive) return;

  // head
  if(o.name === 'head'){
    score++;
    teleportHuman(parent);
    updateScoreDisplay();
    return;
  }

  // body/limb
  parent.userData.bodyHits++;
  if(o.material) o.material.color.set(0xff5555);
  if(parent.userData.bodyHits >= currentWeapon.bodyHits){
    score++;
    teleportHuman(parent);
    updateScoreDisplay();
  }
}

document.addEventListener('click', ()=>{
  if(settingsOpen || marketOpen) return;
  const raycaster = new THREE.Raycaster();
  raycaster.setFromCamera(new THREE.Vector2(0,0), camera);
  const targets = [...enemies];
  if(doorMesh) targets.push(doorMesh);
  const intersects = raycaster.intersectObjects(targets, true);
  if(intersects.length) handleHit(intersects[0]);
});

/* ================= PANELS (settings & market) ================= */
const settingsPanel = document.getElementById('settingsPanel');
const marketPanel = document.getElementById('marketPanel');
const marketBtn = document.getElementById('marketBtn');
const settingsBtn = document.getElementById('settingsBtn');
const btnSettingsClose = document.getElementById('btnSettingsClose');
const btnMarketClose = document.getElementById('btnMarketClose');

let settingsOpen = false, marketOpen = false;
function openSettings(){ settingsOpen = true; settingsPanel.style.display = 'flex'; exitPointerLock(); document.body.style.cursor='auto'; document.body.classList.remove('hide-cursor'); }
function closeSettings(){ settingsOpen = false; settingsPanel.style.display = 'none'; document.body.style.cursor='none'; document.body.classList.add('hide-cursor'); }
function toggleSettings(){ settingsOpen ? closeSettings() : openSettings(); }

function openMarket(){ marketOpen = true; marketPanel.style.display = 'flex'; exitPointerLock(); document.body.style.cursor='auto'; document.body.classList.remove('hide-cursor'); refreshMarketUI(); }
function closeMarket(){ marketOpen = false; marketPanel.style.display = 'none'; document.body.style.cursor='none'; document.body.classList.add('hide-cursor'); }
function toggleMarket(){ marketOpen ? closeMarket() : openMarket(); }

settingsBtn.addEventListener('click', toggleSettings);
marketBtn.addEventListener('click', toggleMarket);
btnSettingsClose && btnSettingsClose.addEventListener('click', closeSettings);
btnMarketClose && btnMarketClose.addEventListener('click', closeMarket);

/* Market UI */
function refreshMarketUI(){
  const items = document.querySelectorAll('#marketItems img');
  items.forEach(img => {
    const weaponId = img.dataset.weapon;
    const unlock = parseInt(img.dataset.unlock,10) || 0;
    if(score >= unlock) {
      img.style.filter = 'none';
      img.title = `${WEAPONS[weaponId].label} - Al (gereksinim: ${unlock})`;
    } else {
      img.style.filter = 'grayscale(80%) brightness(0.7)';
      img.title = `${WEAPONS[weaponId].label} - Kilitli (gereksinim: ${unlock})`;
    }
  });
}

document.getElementById('marketItems').addEventListener('click', e=>{
  const img = e.target.closest('img');
  if(!img) return;
  const weaponId = img.dataset.weapon;
  const unlock = parseInt(img.dataset.unlock,10) || 0;
  if(score >= unlock){
    // equip
    currentWeapon = WEAPONS[weaponId];
    weapon2d.style.backgroundImage = `url('${currentWeapon.skin}')`;
    // reset score on buy
    score = 0;
    updateScoreDisplay();
    closeMarket();
    messageEl.textContent = `${currentWeapon.label} alındı! Skor sıfırlandı.`;
    messageEl.style.display = 'block';
    setTimeout(()=>{ messageEl.style.display = 'none'; messageEl.textContent = 'Yakında...'; }, 1400);
  } else {
    messageEl.textContent = `Kilitli: ${unlock} kill gerekir`;
    messageEl.style.display = 'block';
    setTimeout(()=>{ messageEl.style.display = 'none'; messageEl.textContent = 'Yakında...'; }, 1400);
  }
});

/* ================= ANIMATION / GAME LOOP ================= */
camera.position.set(0,2,5);
function animate(){
  requestAnimationFrame(animate);
  const speed = 0.12;
  if(!settingsOpen && !marketOpen){
    const forward = new THREE.Vector3(-Math.sin(yaw),0,-Math.cos(yaw));
    const right = new THREE.Vector3(Math.cos(yaw),0,-Math.sin(yaw));
    let np = camera.position.clone();
    if(keys['w']) np.add(forward.clone().multiplyScalar(speed));
    if(keys['s']) np.add(forward.clone().multiplyScalar(-speed));
    if(keys['a']) np.add(right.clone().multiplyScalar(-speed));
    if(keys['d']) np.add(right.clone().multiplyScalar(speed));
    np.x = THREE.MathUtils.clamp(np.x, -mapSize/2 + 0.6, mapSize/2 - 0.6);
    np.z = THREE.MathUtils.clamp(np.z, -mapSize/2 + 0.6, mapSize/2 - 0.6);
    camera.position.copy(np);
  }
  camera.rotation.set(pitch, yaw, 0);
  renderer.render(scene, camera);
}
animate();

/* responsive */
window.addEventListener('resize', ()=>{
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});

/* initial UI state */
updateScoreDisplay();
refreshMarketUI();

</script>
</body>
</html>
