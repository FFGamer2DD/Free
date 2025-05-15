<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Flappy Bird</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            touch-action: manipulation;
        }

        body {
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #f0f0f0;
            font-family: Arial, sans-serif;
            overflow: hidden;
        }

        .game-container {
            position: relative;
            width: 100%;
            max-width: 400px;
            margin: 0 auto;
        }

        canvas {
            width: 100%;
            height: 600px;
            background-color: #87CEEB;
            display: block;
            border: 2px solid #333;
        }

        .game-over, .start-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(0, 0, 0, 0.7);
            color: white;
            z-index: 10;
            text-align: center;
        }

        .game-over {
            display: none;
        }

        .start-screen {
            display: flex;
        }

        button {
            padding: 12px 24px;
            font-size: 18px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 20px;
            min-width: 150px;
        }

        button:hover {
            background-color: #45a049;
        }

        h1, h2 {
            margin: 0 0 10px 0;
            padding: 0;
        }

        p {
            margin: 0 0 20px 0;
        }

        @media (max-width: 480px) {
            .game-container {
                width: 100vw;
                height: 100vh;
                max-width: none;
            }
            
            canvas {
                height: 100vh;
                border: none;
            }
            
            button {
                padding: 15px 30px;
                font-size: 20px;
            }
        }
    </style>
</head>
<body>
    <div class="game-container">
        <canvas id="gameCanvas"></canvas>
        <div id="gameOver" class="game-over">
            <h2>Game Over</h2>
            <p>Score: <span id="finalScore">0</span></p>
            <button id="restartButton">Play Again</button>
        </div>
        <div id="startScreen" class="start-screen">
            <h1>Flappy Bird</h1>
            <p>Press SPACE, UP ARROW or TAP to jump</p>
            <button id="startButton">Start Game</button>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const canvas = document.getElementById('gameCanvas');
            const ctx = canvas.getContext('2d');
            const startScreen = document.getElementById('startScreen');
            const gameOverScreen = document.getElementById('gameOver');
            const finalScoreElement = document.getElementById('finalScore');
            const startButton = document.getElementById('startButton');
            const restartButton = document.getElementById('restartButton');

            // Set canvas size based on device
            function resizeCanvas() {
                const width = Math.min(400, window.innerWidth);
                const height = Math.min(600, window.innerHeight);
                canvas.width = width;
                canvas.height = height;
            }
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);

            // Game assets
            const assets = {
                bird: new Image(),
                pipeTop: new Image(),
                pipeBottom: new Image(),
                background: new Image()
            };

            assets.bird.src = 'https://iili.io/3geC3ep.png';
            assets.pipeTop.src = 'https://iili.io/3gOOEZl.png';
            assets.pipeBottom.src = 'https://iili.io/3gOOEZl.png';
            assets.background.src = 'https://iili.io/3gNnF6J.png';

            // Game variables
            let bird = {
                x: 50,
                y: canvas.height / 2,
                width: 40,
                height: 30,
                velocity: 0,
                gravity: 0.5,
                jump: -10
            };

            let pipes = [];
            let score = 0;
            let gameRunning = false;
            let animationId;
            let pipeGap = 150;
            let pipeFrequency = 1500; // milliseconds
            let lastPipeTime = 0;
            let backgroundOffset = 0;
            let assetsLoaded = 0;

            // Check if all assets are loaded
            function checkAssetsLoaded() {
                assetsLoaded++;
                if (assetsLoaded === Object.keys(assets).length) {
                    startButton.disabled = false;
                }
            }

            // Set up asset loading
            for (const key in assets) {
                assets[key].onload = checkAssetsLoaded;
                assets[key].onerror = () => {
                    console.error(`Failed to load asset: ${key}`);
                    checkAssetsLoaded(); // Continue even if some assets fail to load
                };
            }

            // Event listeners
            function jump() {
                if (!gameRunning && startScreen.style.display === 'none') {
                    startGame();
                    return;
                }
                if (gameRunning) {
                    bird.velocity = bird.jump;
                }
            }

            canvas.addEventListener('click', jump);
            canvas.addEventListener('touchstart', (e) => {
                e.preventDefault();
                jump();
            });

            document.addEventListener('keydown', (e) => {
                if (e.code === 'Space' || e.key === ' ' || e.key === 'ArrowUp') {
                    e.preventDefault();
                    jump();
                }
            });

            startButton.addEventListener('click', startGame);
            restartButton.addEventListener('click', startGame);

            function startGame() {
                // Reset game state
                bird.y = canvas.height / 2;
                bird.velocity = 0;
                pipes = [];
                score = 0;
                gameRunning = true;
                lastPipeTime = 0;
                backgroundOffset = 0;

                // Hide screens
                startScreen.style.display = 'none';
                gameOverScreen.style.display = 'none';

                // Start game loop
                if (animationId) {
                    cancelAnimationFrame(animationId);
                }
                gameLoop();
            }

            function gameLoop(timestamp = 0) {
                if (!gameRunning) return;

                // Clear canvas
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                // Draw background
                drawBackground();

                // Update bird
                updateBird();

                // Generate pipes
                if (timestamp - lastPipeTime > pipeFrequency) {
                    createPipe();
                    lastPipeTime = timestamp;
                }

                // Update and draw pipes
                updatePipes();

                // Draw bird
                drawBird();

                // Draw score
                drawScore();

                // Check collisions
                if (checkCollisions()) {
                    gameOver();
                    return;
                }

                animationId = requestAnimationFrame(gameLoop);
            }

            function drawBackground() {
                // Draw the background image with parallax effect
                backgroundOffset -= 0.5;
                const bgWidth = assets.background.width * (canvas.height / assets.background.height);
                const bgX = backgroundOffset % bgWidth;
                
                ctx.drawImage(assets.background, bgX, 0, bgWidth, canvas.height);
                if (bgX + bgWidth < canvas.width) {
                    ctx.drawImage(assets.background, bgX + bgWidth, 0, bgWidth, canvas.height);
                }
            }

            function updateBird() {
                bird.velocity += bird.gravity;
                bird.y += bird.velocity;

                // Prevent bird from going above canvas
                if (bird.y < 0) {
                    bird.y = 0;
                    bird.velocity = 0;
                }

                // Check if bird hits the ground
                if (bird.y + bird.height > canvas.height) {
                    bird.y = canvas.height - bird.height;
                    gameOver();
                }
            }

            function drawBird() {
                if (assets.bird.complete) {
                    // Draw bird image with rotation based on velocity
                    ctx.save();
                    ctx.translate(bird.x + bird.width / 2, bird.y + bird.height / 2);
                    const rotation = Math.min(Math.max(bird.velocity * 3, -30), 30);
                    ctx.rotate(rotation * Math.PI / 180);
                    ctx.drawImage(assets.bird, -bird.width / 2, -bird.height / 2, bird.width, bird.height);
                    ctx.restore();
                } else {
                    // Fallback if image not loaded
                    ctx.fillStyle = '#FFD700';
                    ctx.fillRect(bird.x, bird.y, bird.width, bird.height);
                }
            }

            function createPipe() {
                const minHeight = 80;
                const maxHeight = canvas.height - pipeGap - minHeight;
                const height = Math.floor(Math.random() * (maxHeight - minHeight + 1)) + minHeight;
                
                pipes.push({
                    x: canvas.width,
                    height: height,
                    width: 80,
                    passed: false
                });
            }

            function updatePipes() {
                for (let i = 0; i < pipes.length; i++) {
                    const pipe = pipes[i];
                    
                    // Move pipe
                    pipe.x -= 2;
                    
                    // Draw top pipe
                    if (assets.pipeTop.complete) {
                        ctx.drawImage(
                            assets.pipeTop, 
                            pipe.x, 
                            pipe.height - assets.pipeTop.height, 
                            pipe.width, 
                            assets.pipeTop.height
                        );
                    } else {
                        ctx.fillStyle = '#4CAF50';
                        ctx.fillRect(pipe.x, 0, pipe.width, pipe.height);
                    }
                    
                    // Draw bottom pipe
                    const bottomPipeY = pipe.height + pipeGap;
                    if (assets.pipeBottom.complete) {
                        ctx.drawImage(
                            assets.pipeBottom, 
                            pipe.x, 
                            bottomPipeY, 
                            pipe.width, 
                            canvas.height - bottomPipeY
                        );
                    } else {
                        ctx.fillStyle = '#4CAF50';
                        ctx.fillRect(pipe.x, bottomPipeY, pipe.width, canvas.height - bottomPipeY);
                    }
                    
                    // Check if bird passed the pipe
                    if (!pipe.passed && bird.x > pipe.x + pipe.width) {
                        pipe.passed = true;
                        score++;
                    }
                    
                    // Remove pipes that are off screen
                    if (pipe.x + pipe.width < 0) {
                        pipes.splice(i, 1);
                        i--;
                    }
                }
            }

            function drawScore() {
                ctx.fillStyle = '#FFF';
                ctx.strokeStyle = '#000';
                ctx.lineWidth = 2;
                ctx.font = 'bold 30px Arial';
                ctx.textAlign = 'center';
                
                // Draw text with outline
                ctx.strokeText(score, canvas.width / 2, 50);
                ctx.fillText(score, canvas.width / 2, 50);
            }

            function checkCollisions() {
                for (const pipe of pipes) {
                    // Check collision with top pipe
                    if (bird.x + bird.width > pipe.x && 
                        bird.x < pipe.x + pipe.width && 
                        bird.y < pipe.height) {
                        return true;
                    }
                    
                    // Check collision with bottom pipe
                    if (bird.x + bird.width > pipe.x && 
                        bird.x < pipe.x + pipe.width && 
                        bird.y + bird.height > pipe.height + pipeGap) {
                        return true;
                    }
                }
                return false;
            }

            function gameOver() {
                gameRunning = false;
                cancelAnimationFrame(animationId);
                finalScoreElement.textContent = score;
                gameOverScreen.style.display = 'flex';
            }

            // Disable start button until assets are loaded
            startButton.disabled = true;
            restartButton.disabled = true;
            
            // Enable buttons when assets are loaded
            const checkButtons = setInterval(() => {
                if (assetsLoaded === Object.keys(assets).length) {
                    startButton.disabled = false;
                    restartButton.disabled = false;
                    clearInterval(checkButtons);
                }
            }, 100);
        });
    </script>
</body>
</html>
