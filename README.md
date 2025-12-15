# SecretSanta
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Roulette Spinner — Conditional Spin</title>
<style>
html, body {
  margin:0; 
  padding:0; 
  font-family: Arial, Helvetica, sans-serif;
  /* Removed height:100% and overflow:hidden to allow scrolling */
  /*background: #000;*/
}

/* Background */
.background-container {
  position: fixed; 
  inset:0;
  background: url('https://cdn.stocksnap.io/img-thumbs/960w/wooden-wall_XQV3531O4I.jpg') no-repeat center center/cover;
  z-index:-2;
}
.background-overlay {
  position: fixed; 
  inset:0;
  background: rgba(0,0,0,0.4);
  z-index:-1;
}

/* Title Box */
#titleBox {
  text-align: center;
  font-size: 32px;
  font-weight: bold;
  color: #ffffff;
  margin:20px auto;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
}

/* Spinner Container */
.spinner-container {
  position: relative;
  width: 400px; 
  height: 400px;
  margin: 30px auto;
}

canvas {
  width: 100%; 
  height: 100%;
  border-radius: 50%;
  transform-origin: 50% 50%;
  background: transparent;
}

.pointer {
  position: absolute;
  left: 50%; 
  top: 50%;
  transform: translate(-50%, -190px);
  width: 0; 
  height: 0;
  border-left: 15px solid transparent;
  border-right: 15px solid transparent;
  border-bottom: 35px solid #e53935;
  z-index: 10;
}

/* Buttons */
button {
  display: block;
  margin: 12px auto 0;
  padding: 12px 24px;
  font-size: 18px;
  cursor: pointer;
}

#tryAgainBtn, #resetBtn, #acceptBtn {
  display: none;
}

/* Prize Popup */
.prize-popup {
  width: 320px;
  margin: 18px auto;
  padding: 18px;
  background: #ffd700;
  border-radius: 12px;
  text-align: center;
  display: none;
  box-shadow: 0 6px 20px rgba(0,0,0,0.25);
}

.prize-name { font-size: 20px; font-weight: 700; }
.prize-description { margin-top: 8px; font-size: 15px; }
.prize-popup img { margin-top: 12px; max-width: 100%; border-radius: 8px; }

@media (max-width: 520px){
  .spinner-container { width: 300px; height: 300px; }
  .pointer { transform: translate(-50%, -145px); }
  .prize-popup { width: 90%; }
}
</style>
</head>
<body>

<!-- Title -->
<div id="titleBox">Roulette Time, Spin the wheel two times to determine your presents for 2025!</div>

<div class="background-container"></div>
<div class="background-overlay"></div>

<div class="spinner-container">
  <canvas id="spinner"></canvas>
  <div class="pointer"></div>
</div>

<button id="spinBtn">SPIN</button>
<button id="tryAgainBtn">2nd TURN</button>
<button id="acceptBtn">ACCEPT</button>
<button id="resetBtn">GAME OVER</button>

<div class="prize-popup" id="prizePopup">
  <div class="prize-name"></div>
  <div class="prize-description"></div>
  <img class="prize-image" src="" alt="">
</div>

<script>
/* ---------- Original Prize List ---------- */
const ORIGINAL_PRIZES = [
  { name:"Prize A", description:"Congratulations you have made the nice list", image:"https://i.etsystatic.com/26917812/r/il/93ea63/4393521497/il_fullxfull.4393521497_d13e.jpg" },
  { name:"Prize B", description:"Winner Winner", image:"https://i.etsystatic.com/26917812/r/il/93ea63/4393521497/il_fullxfull.4393521497_d13e.jpg" },
  { name:"Prize C", description:"Oh No! - Been a naughty boy this year?", image:"https://i.etsystatic.com/23425151/r/il/dd9580/2600800801/il_1080xN.2600800801_l4rn.jpg" },
  { name:"Prize D", description:"Congratulations you have made the nice list", image:"https://i.etsystatic.com/26917812/r/il/93ea63/4393521497/il_fullxfull.4393521497_d13e.jpg" },
  { name:"Prize E", description:"Bugger - You didnt quite make the nice list", image:"https://i.etsystatic.com/23425151/r/il/dd9580/2600800801/il_1080xN.2600800801_l4rn.jpg" },
  { name:"Prize F", description:"Whoops - this could be a dud spin", image:"https://i.etsystatic.com/23425151/r/il/dd9580/2600800801/il_1080xN.2600800801_l4rn.jpg" }
];

let prizes = JSON.parse(JSON.stringify(ORIGINAL_PRIZES));
const wedgeColors = ["#90EE90","#FF7F7F","#90EE90","#FF7F7F","#90EE90","#FF7F7F"];

const canvas = document.getElementById("spinner");
const ctx = canvas.getContext("2d");
const container = document.querySelector(".spinner-container");
const popup = document.getElementById("prizePopup");
const prizeNameEl = popup.querySelector(".prize-name");
const prizeDescEl = popup.querySelector(".prize-description");
const prizeImgEl = popup.querySelector(".prize-image");

const spinBtn = document.getElementById("spinBtn");
const tryAgainBtn = document.getElementById("tryAgainBtn");
const acceptBtn = document.getElementById("acceptBtn");
const resetBtn = document.getElementById("resetBtn");

let currentRotation = 0;
let spinCount = 0; // 0 = first spin, 1 = second spin
let firstPrize = null;

/* ---------- Responsive Canvas ---------- */
function resizeCanvas(){
  const rect = container.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;
  canvas.width = rect.width * dpr;
  canvas.height = rect.height * dpr;
  canvas.style.width = rect.width + "px";
  canvas.style.height = rect.height + "px";
  ctx.setTransform(dpr,0,0,dpr,0,0);
  drawWheel();
}
window.addEventListener("resize", resizeCanvas);
resizeCanvas();

/* ---------- Draw Wheel ---------- */
function drawWheel(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  const cssW = canvas.width / (window.devicePixelRatio||1);
  const cssH = canvas.height / (window.devicePixelRatio||1);
  const center = { x: cssW/2, y: cssH/2 };
  const radius = Math.min(cssW, cssH)/2 - 6;
  const numWedges = prizes.length;
  const wedgeAngle = (2*Math.PI)/numWedges;

  for(let i=0;i<numWedges;i++){
    const start = i*wedgeAngle;
    const end = start + wedgeAngle;

    ctx.beginPath();
    ctx.moveTo(center.x, center.y);
    ctx.arc(center.x, center.y, radius, start, end);
    ctx.closePath();
    ctx.fillStyle = wedgeColors[i % wedgeColors.length];
    ctx.fill();

    ctx.strokeStyle = "#000";
    ctx.lineWidth = 2;
    ctx.stroke();

    ctx.save();
    ctx.translate(center.x, center.y);
    ctx.rotate(start + wedgeAngle/2);
    ctx.textAlign = "right";
    ctx.fillStyle = "#000";
    ctx.font = `${Math.max(12, Math.floor(radius/6))}px Arial`;
    ctx.fillText(prizes[i].name, radius - 18, 6);
    ctx.restore();
  }
}

/* ---------- Spin Function ---------- */
function spin(){
  popup.style.display = "none";
  spinBtn.disabled = true;

  const numWedgesNow = prizes.length;
  const wedgeAngleNow = (2*Math.PI)/numWedgesNow;
  const chosenIndex = Math.floor(Math.random()*numWedgesNow);

  const extraSpins = Math.floor(Math.random()*3) + 4;
  const wedgeCenter = chosenIndex * wedgeAngleNow + wedgeAngleNow/2;
  const targetRotation = extraSpins*2*Math.PI + (-Math.PI/2 - wedgeCenter);

  const startRotation = currentRotation;
  const endRotation = currentRotation + targetRotation;

  const duration = 4200;
  const startTime = performance.now();

  function animate(now){
    const t = Math.min(1, (now-startTime)/duration);
    const eased = easeOutCubic(t);
    const rot = startRotation + (endRotation - startRotation)*eased;
    canvas.style.transform = `rotate(${rot}rad)`;

    if(t<1){
      requestAnimationFrame(animate);
    } else {
      currentRotation = endRotation % (2*Math.PI);
      const selectedPrize = prizes[chosenIndex];
      showPrize(selectedPrize);

      if(spinCount === 0){
        firstPrize = selectedPrize;
        tryAgainBtn.style.display = "block";
        acceptBtn.style.display = "none";
        spinBtn.style.display = "none";
      } else {
        resetBtn.style.display = "block";
        spinBtn.style.display = "none";
      }

      spinCount++;
      spinBtn.disabled = false;
    }
  }
  requestAnimationFrame(animate);
}

/* ---------- Show Prize ---------- */
function showPrize(prize){
  prizeNameEl.textContent = prize.name;
  prizeDescEl.textContent = prize.description;
  prizeImgEl.src = prize.image;
  prizeImgEl.style.display = "block";
  popup.style.display = "block";
}

/* ---------- Buttons ---------- */
tryAgainBtn.addEventListener("click", () => {
  popup.style.display = "none";
  tryAgainBtn.style.display = "none";
  acceptBtn.style.display = "none";

  const index = prizes.findIndex(p => p.name === firstPrize.name);
  if(index >= 0) prizes.splice(index, 1);

  currentRotation = 0;
  canvas.style.transform = `rotate(0rad)`;
  drawWheel();

  spinBtn.style.display = "block";
});

acceptBtn.addEventListener("click", () => {
  popup.style.display = "block";
  tryAgainBtn.style.display = "none";
  acceptBtn.style.display = "none";
  resetBtn.style.display = "block";
  spinBtn.style.display = "none";
});

resetBtn.addEventListener("click", () => {
  prizes = JSON.parse(JSON.stringify(ORIGINAL_PRIZES));
  spinCount = 0;
  firstPrize = null;
  popup.style.display = "none";
  tryAgainBtn.style.display = "none";
  acceptBtn.style.display = "none";
  resetBtn.style.display = "none";
  currentRotation = 0;
  canvas.style.transform = "rotate(0rad)";
  drawWheel();

  spinBtn.style.display = "block";
});

spinBtn.addEventListener("click", spin);

/* ---------- Easing ---------- */
function easeOutCubic(t){ return (--t)*t*t + 1; }

</script>
</body>
</html>
