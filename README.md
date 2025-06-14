<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Mage Vinyl Dash</title>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      background: #87ceeb;
      overflow: hidden;
      font-family: Arial, sans-serif;
    }
    canvas {
      display: block;
      margin: 0 auto;
      background: linear-gradient(to top, #87ceeb, #b0e0e6);
    }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="600" height="600"></canvas>

<script>
// Sky gradient function (handles day, sunset, night, sunrise)
function getSkyGradient(frameCount) {
  const cycleTime = 60 * 60; // 60 seconds at 60fps
  const phase = (frameCount % cycleTime) / cycleTime;
  if (phase < 0.25) return ["#87ceeb", "#b0e0e6"];  // Day
  if (phase < 0.5)  return ["#fcb045", "#fd1d1d"];  // Sunset
  if (phase < 0.75) return ["#0b0c1a", "#1e1f38"];  // Night
  return ["#ffcf33", "#ffe9b3"];                   // Sunrise
}

// Full game logic initialization
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

const WIDTH = canvas.width;
const HEIGHT = canvas.height;
const GROUND_Y = HEIGHT - 50;

let score = 0;
let speed = 5;
let vinyls = [];
let obstacles = [];
let frame = 0;
let gameOver = false;
let running = false;

let barnX = WIDTH; // Initial position of the barn (right side)

// Player object initialization
const player = {
  x: 80,
  y: GROUND_Y - 60,
  width: 30,
  height: 60,
  vy: 0,
  gravity: 0.8,
  jumpStrength: -14,
  speedX: 0,
  ducking: false,
  onGround: true
};

// Reset the game function (initializes everything to the start)
function resetGame() {
  score = 0;
  speed = 5;
  vinyls = [];
  obstacles = [];
  frame = 0;
  gameOver = false;
  barnX = WIDTH; // Reset barn position
  running = true;
  gameLoop(); // Start the game loop immediately
}

// Start screen (add load screen image here, chat)
function drawStartScreen() {
  ctx.clearRect(0, 0, WIDTH, HEIGHT); // Clear canvas

  // NOTE: Replace this line with the Base64 string for your image
  const startImage = new Image();
  startImage.src = "mage_vinyl_dash_start_screen.png"; 
  // Paste your Base64 string here

  // Once the image is fully loaded, draw it on the canvas
  startImage.onload = function() {
    ctx.drawImage(startImage, 0, 0, WIDTH, HEIGHT); // Draw image on canvas
  };

  // Display text if the image is still loading
  if (!startImage.complete) {
    ctx.fillStyle = "white";
    ctx.font = "48px Arial";
    ctx.textAlign = "center";
    ctx.fillText("Press Space or Arrow Up to Start", WIDTH / 2, HEIGHT / 2);
  }
}

// Tree class to define tree properties
class Tree {
  constructor(x, height, speed) {
    this.x = x;
    this.y = GROUND_Y - height; // Align all trees to the same vertical plane
    this.height = height;
    this.width = 30; // Width of the tree trunk
    this.speed = speed; // Customizable speed
  }

  // Draw the tree
  draw() {
    // Tree trunk (brown)
    ctx.fillStyle = "#8b4513"; // Brown color for tree trunk
    ctx.fillRect(this.x - 10, this.y, 20, this.height); // Tree trunk

    // Tree foliage (green triangle)
    ctx.fillStyle = "#228B22"; // Green color for foliage
    ctx.beginPath();
    ctx.moveTo(this.x, this.y - this.height); // Top of the tree (triangle tip)
    ctx.lineTo(this.x - 30, this.y);  // Left bottom of the tree
    ctx.lineTo(this.x + 30, this.y);  // Right bottom of the tree
    ctx.closePath();
    ctx.fill();
  }

  // Update the tree position to move it horizontally
  update() {
    this.x -= this.speed; // Move the tree to the left
    if (this.x + this.width < 0) { // Recycle trees back to the right side
      this.x = canvas.width;
    }
  }
}

// Cat obstacle class
class Cat {
  constructor(x) {
    this.x = x;
    this.y = GROUND_Y - 30;
    this.width = 40;
    this.height = 20;
    this.speed = speed / 3; // Cats move slower than other obstacles
  }
}

// Dancing stick figure design and animation (adjusted position to move with barn)
function drawDancingStickFigure() {
  const danceFrame = Math.floor((frame / 15) % 3);
  
  // Adjust dx to move with the barn, and dy to keep it on the ground
  const dx = barnX + 40, dy = GROUND_Y - 125; // Make the stick figure move with the barn
  ctx.strokeStyle = "black";
  ctx.lineWidth = 2;

  // Body
  ctx.beginPath();
  ctx.moveTo(dx, dy);
  ctx.lineTo(dx, dy - 20);
  ctx.stroke();

  // Head (no brown circle anymore)
  ctx.beginPath();
  ctx.arc(dx, dy - 25, 4, 0, Math.PI * 2);
  ctx.stroke();

  // Arms (animated with dance frames)
  if (danceFrame === 0) {
    ctx.beginPath(); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx - 5, dy - 10); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx + 5, dy - 10); 
    ctx.stroke();
  } else if (danceFrame === 1) {
    ctx.beginPath(); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx - 6, dy - 15); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx + 6, dy - 15); 
    ctx.stroke();
  } else {
    ctx.beginPath(); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx - 5, dy - 20); 
    ctx.moveTo(dx, dy - 15); 
    ctx.lineTo(dx + 5, dy - 20); 
    ctx.stroke();
  }

  // Legs (add slight movement with animation frame)
  if (danceFrame === 0) {
    // Legs in default position
    ctx.beginPath();
    ctx.moveTo(dx, dy); // Left leg
    ctx.lineTo(dx - 5, dy + 10);
    ctx.moveTo(dx, dy); // Right leg
    ctx.lineTo(dx + 5, dy + 10);
    ctx.stroke();
  } else if (danceFrame === 1) {
    // Legs spread apart
    ctx.beginPath();
    ctx.moveTo(dx, dy); // Left leg
    ctx.lineTo(dx - 10, dy + 10);
    ctx.moveTo(dx, dy); // Right leg
    ctx.lineTo(dx + 10, dy + 10);
    ctx.stroke();
  } else {
    // Legs close together
    ctx.beginPath();
    ctx.moveTo(dx, dy); // Left leg
    ctx.lineTo(dx - 5, dy + 5);
    ctx.moveTo(dx, dy); // Right leg
    ctx.lineTo(dx + 5, dy + 5);
    ctx.stroke();
  }
}



// Array to store trees and obstacles
let trees = [];
let smallTrees = [];
let cats = [];

// Create trees with random heights
for (let i = 0; i < 7; i++) {  // Adjust number of trees here
  let treeHeight = Math.random() * 100 + 50; // Random tree height between 50px and 150px
  let treeX = Math.random() * canvas.width; // Random x-position
  trees.push(new Tree(treeX, treeHeight, 1)); // Normal trees (move at default speed)
}

// Create smaller trees for depth (behind the barn)
for (let i = 0; i < 5; i++) {  // Adjust number of small trees here
  let treeHeight = Math.random() * 50 + 30; // Smaller trees
  let treeX = Math.random() * canvas.width; // Random x-position
  smallTrees.push(new Tree(treeX, treeHeight, 0.2)); // Smaller trees move at the same speed as the barn
}

// Drawing the background (sky, barn, and trees)
function drawBackground() {
  const [skyTop, skyBottom] = getSkyGradient(frame);
  const skyGrad = ctx.createLinearGradient(0, 0, 0, GROUND_Y);
  skyGrad.addColorStop(0, skyTop);
  skyGrad.addColorStop(1, skyBottom);
  ctx.fillStyle = skyGrad;
  ctx.fillRect(0, 0, WIDTH, GROUND_Y); // Draw sky gradient
  ctx.fillStyle = "#5d945d";
  ctx.fillRect(0, GROUND_Y, WIDTH, HEIGHT - GROUND_Y); // Ground

  // Draw the smaller trees behind the barn (create depth)
  for (let smallTree of smallTrees) {
    smallTree.update();
    smallTree.draw();
  }

  // Draw the trees in the background (in front of the barn)
  for (let tree of trees) {
    tree.update();
    tree.draw();
  }

  // Draw the moon/sun in the sky (positioned directly above the barn)
  ctx.fillStyle = frame % 2 === 0 ? "yellow" : "white"; // Yellow during day, white during night
  ctx.beginPath();
  ctx.arc(barnX + 60, GROUND_Y - 180, 15, 0, Math.PI * 2); // Moon/sun circle
  ctx.fill();

  // Draw the barn (it moves from right to left)
  ctx.fillStyle = "red";
  ctx.fillRect(barnX, GROUND_Y - 100, 120, 100); // Barn base
  ctx.beginPath();
  ctx.moveTo(barnX, GROUND_Y - 100); // Roof triangle
  ctx.lineTo(barnX + 60, GROUND_Y - 150);
  ctx.lineTo(barnX + 120, GROUND_Y - 100);
  ctx.closePath();
  ctx.fillStyle = "darkred";
  ctx.fill();
  ctx.strokeStyle = "white";
  ctx.lineWidth = 3;
  ctx.strokeRect(barnX + 45, GROUND_Y - 50, 30, 50); // Barn door

  // Draw the dancing stick figure next to the shed
  drawDancingStickFigure();
}

// Drawing the player (blue circle head, witch hat, blue dress, cloak)
function drawPlayer() {
  ctx.save();
  ctx.translate(player.x, player.y);

  // Ducking: Adjust mage size and position when ducking
  if (player.ducking) {
    ctx.scale(1, 0.5); // Reduce height and scale the mage's position
  }

  // Head (Blue Circle)
  ctx.fillStyle = "#0000FF"; // Blue head
  ctx.beginPath();
  ctx.arc(15, 15, 15, 0, Math.PI * 2); // Circle for the head
  ctx.fill();

  // Witch Hat (Solid Blue)
  ctx.fillStyle = "#1e3b72"; // Darker blue hat
  ctx.beginPath();
  ctx.moveTo(0, 15); // Starting point (bottom of head)
  ctx.lineTo(30, 15); // Bottom-right of the hat
  ctx.lineTo(15, -25); // Top of the triangle
  ctx.closePath();
  ctx.fill();

  // Body (Blue Dress)
  ctx.fillStyle = "#4169e1"; // Blue dress
  ctx.fillRect(0, 30, 30, 60); // Body rectangle

  // Cloak (Flowing shape)
  ctx.fillStyle = "#1e3b72"; // Dark blue cloak
  ctx.beginPath();
  ctx.moveTo(15, 30); // Centered on the body
  ctx.lineTo(-15, 70); // Left side of the cloak
  ctx.lineTo(45, 70);   // Right side of the cloak
  ctx.lineTo(15, 30);   // Back to the center
  ctx.closePath();
  ctx.fill();

  ctx.restore();
}

// Drawing Vinyls (Objects the player collects)
function drawVinyls() {
  vinyls.forEach(v => {
    ctx.fillStyle = v.color;
    ctx.beginPath();
    ctx.arc(v.x, v.y, v.r, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = "black";
    ctx.beginPath();
    ctx.arc(v.x, v.y, v.r / 3, 0, Math.PI * 2);
    ctx.fill();
  });
}

// Drawing obstacles (Dishes, Bats, and Cats)
function drawObstacles() {
  obstacles.forEach(o => {
    if (o.type === "dish") { // Dish obstacle
      ctx.fillStyle = "white";
      ctx.beginPath();
      ctx.arc(o.x + o.w / 2, GROUND_Y - 15, 15, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = "green";
      for (let i = 0; i < 4; i++) {
        ctx.beginPath();
        ctx.arc(o.x + o.w / 2 + Math.random() * 10 - 5, GROUND_Y - 15 + Math.random() * 10 - 5, 3, 0, Math.PI * 2);
        ctx.fill();
      }
    } else if (o.type === "bat") { // Bat obstacle
      // Call the bat drawing function with flashing wings
      drawBat(o, frame); // Pass frame to control wing flashing
    } else if (o.type === "cat") { // Cat obstacle
      o.update();
      o.draw();
    }
  });
}

// Bat rendering logic with flashing wings
function drawBat(o, frame) {
  ctx.fillStyle = "black";

  // Bat body (rectangle)
  ctx.fillRect(o.x, o.y, o.w, o.h);

  // Flashing wings (triangle effect)
  ctx.fillStyle = `rgba(0, 0, 0, ${Math.abs(Math.sin(frame / 10))})`; // Flash effect
  ctx.beginPath();
  ctx.moveTo(o.x, o.y + o.h / 2); // Middle of the body
  ctx.lineTo(o.x - 20, o.y - 10); // Left wing tip
  ctx.lineTo(o.x + o.w + 20, o.y - 10); // Right wing tip
  ctx.closePath();
  ctx.fill();
}

function updateGame() {
  player.vy += player.gravity;
  player.y += player.vy;
  const targetY = GROUND_Y - (player.ducking ? 30 : 60);
  if (player.y > targetY) {
    player.y = targetY;
    player.vy = 0;
    player.onGround = true;
  } else {
    player.onGround = false;
  }

  // Prevent ducking and jumping at the same time
  if (player.ducking && player.vy < 0) {
    player.vy = 0; // Stop jumping if ducking
  }

  player.x += player.speedX; // Left/Right movement

  vinyls.forEach(v => v.x -= speed);
  obstacles.forEach(o => o.x -= speed);

  vinyls = vinyls.filter(v => v.x + v.r > 0);
  obstacles = obstacles.filter(o => o.x + o.w > 0);

  const px = player.x;
  const py = player.y;
  const pw = player.width;
  const ph = player.ducking ? 30 : player.height;

  for (let o of obstacles) {
    if (px < o.x + o.w && px + pw > o.x && py < o.y + o.h && py + ph > o.y) {
      if (o.type === "cat") {
        // Cat obstacle can only be jumped over
        if (player.vy > 0) {  // Only hit when player is falling or not jumping
          gameOver = true;
          running = false;
        }
      } else {
        gameOver = true;
        running = false;
      }
    }
  }

  for (let i = vinyls.length - 1; i >= 0; i--) {
    const v = vinyls[i];
    const dx = v.x - (px + pw / 2);
    const dy = v.y - (py + ph / 2);
    if (dx * dx + dy * dy < (v.r + 10) * (v.r + 10)) {
      vinyls.splice(i, 1);
      score++;
      speed += 0.3;
    }
  }

  // Update the barn's position
  barnX -= 0.2; // Slow down barn movement

  if (barnX + 120 <= 0) { // If the barn is completely off the left side
    barnX = WIDTH; // Loop it back to the right side
  }

  // Speed up trees along with the game speed
  trees.forEach(tree => tree.speed = 1 * (speed / 5));
  smallTrees.forEach(tree => tree.speed = 1 * (speed / 5));

  frame++;
  if (frame % 90 === 0) {
    const y = Math.random() < 0.4 ? GROUND_Y - 120 : GROUND_Y - 30;
    const color = Math.random() < 0.5 ? "orange" : "purple";
    vinyls.push({ x: WIDTH + 30, y, r: 15, color });
  }
  if (frame % 130 === 0) {
    const type = Math.random() < 0.5 ? "dish" : (Math.random() < 0.5 ? "bat" : "cat");
    const low = Math.random() < 0.5;
    const y = type === "dish" ? GROUND_Y - 30 : (low ? GROUND_Y - 30 : GROUND_Y - 60);
    const h = 20;
    if (type === "cat") {
      obstacles.push(new Cat(WIDTH + 30)); // Add cat obstacle
    } else {
      obstacles.push({ type, x: WIDTH + 30, y, w: 30, h });
    }
  }
}

function gameLoop() {
  ctx.clearRect(0, 0, WIDTH, HEIGHT);
  drawBackground();
  drawVinyls();
  drawPlayer();
  drawObstacles();
  ctx.fillStyle = "black";
  ctx.font = "20px Arial";
  ctx.fillText("Score: " + score, WIDTH - 100, 25); // Fix score cut-off on the left side

  updateGame();
  if (!gameOver) requestAnimationFrame(gameLoop);
  else {
    ctx.fillStyle = "rgba(0,0,0,0.5)";
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
    ctx.fillStyle = "white";
    ctx.font = "40px Arial";
    ctx.textAlign = "center";
    ctx.fillText("Game Over", WIDTH / 2, HEIGHT / 2 - 20);
    ctx.font = "20px Arial";
    ctx.fillText("Final Score: " + score, WIDTH / 2, HEIGHT / 2 + 10);
    ctx.fillText("Press R to Restart", WIDTH / 2, HEIGHT / 2 + 40);
  }
}

document.addEventListener("keydown", e => {
  if (!running && !gameOver && (e.code === "Space" || e.code === "ArrowUp")) {
    resetGame();
  }
  if (gameOver && e.code === "KeyR") {
    resetGame();
  }
  if ((e.code === "Space" || e.code === "ArrowUp") && player.onGround) {
    player.vy = player.jumpStrength;
    player.onGround = false;
  }
  if (e.code === "ArrowDown") player.ducking = true;
  if (e.code === "ArrowLeft") player.speedX = -5; // Move left
  if (e.code === "ArrowRight") player.speedX = 5; // Move right
});
document.addEventListener("keyup", e => {
  if (e.code === "ArrowDown") player.ducking = false;
  if (e.code === "ArrowLeft" || e.code === "ArrowRight") player.speedX = 0; // Stop moving sideways
});

drawStartScreen();

</script>
</body>
</html>

