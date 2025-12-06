<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>2D BlockWorld: Аккаунт и Камера</title>
    <style>
        body {
            margin: 0;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #34495e;
            font-family: 'Arial', sans-serif;
            color: #ecf0f1;
        }
        canvas {
            border: 5px solid #2c3e50;
            background-color: #7f8c8d; 
            margin-bottom: 10px;
        }
        
        /* КОНТЕЙНЕР ДЛЯ ВСЕХ МЕНЮ */
        #ui-container {
            position: absolute;
            width: 800px;
            height: 600px;
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 10;
            pointer-events: none;
        }
        .ui-screen {
            background: rgba(44, 62, 80, 0.95);
            border: 3px solid #f39c12;
            border-radius: 10px;
            padding: 30px;
            text-align: center;
            display: none;
            flex-direction: column;
            gap: 20px;
            pointer-events: auto;
        }
        .ui-screen h2 {
            margin-top: 0;
            color: #f39c12;
        }
        .ui-screen button {
            padding: 10px 20px;
            font-size: 20px;
            background-color: #2ecc71;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: background-color 0.2s;
        }
        .ui-screen button:hover {
            background-color: #27ae60;
        }

        /* Стиль для главного меню игрока (Пауза) */
        #player-menu-screen button {
            background-color: #3498db; 
        }
        #player-menu-screen button:hover {
            background-color: #2980b9; 
        }

        #account-screen input {
            padding: 10px;
            font-size: 16px;
            border: 1px solid #7f8c8d;
            border-radius: 5px;
            margin-top: 5px;
            width: 100%;
            box-sizing: border-box;
            color: #333;
        }
        
        /* GUI МОНЕТ */
        #money-gui {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(44, 62, 80, 0.8);
            padding: 10px 15px;
            border-radius: 5px;
            font-size: 20px;
            color: #f1c40f; 
            display: none; 
            pointer-events: none;
            border: 2px solid #f1c40f;
        }

        /* Игровой HUD (Кнопка меню) */
        #game-hud {
            position: absolute;
            top: 10px;
            left: 10px;
            background: rgba(44, 62, 80, 0.8);
            padding: 10px;
            border-radius: 5px;
            font-size: 16px;
            display: none;
            pointer-events: none;
        }
        #game-hud button {
             pointer-events: auto;
             background-color: #2c3e50; 
             font-size: 18px;
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas" width="800" height="600"></canvas>

    <div id="ui-container">
        
        <div id="account-screen" class="ui-screen">
            <h2>СОЗДАТЬ АККАУНТ</h2>
            <div style="text-align: left;">
                <label for="reg-name">Имя (Никнейм):</label>
                <input type="text" id="reg-name" value="Игрок_R6">
                
                <label style="margin-top: 10px; display: block;">Цвет Тела (Торс):</label>
                <input type="color" id="reg-color-body" value="#3498db">
            </div>
            <button id="btn-start-game">ИГРАТЬ</button>
        </div>

        <div id="player-menu-screen" class="ui-screen">
            <h2>МЕНЮ ИГРОКА</h2>
            <p>Аккаунт: <span id="menu-player-name"></span></p>
            <button onclick="world.unpauseGame()">Продолжить Игру</button>
            <button id="btn-add-friend">Добавить Друга (Имитация)</button>
            <button onclick="world.logOut()" style="background-color: #e67e22;">Выйти из Аккаунта</button>
            <button onclick="world.endGame()" style="background-color: #c0392b;">Выйти из Игры (Закрыть)</button>
        </div>
        
    </div>
    
    <div id="game-hud">
        <button id="btn-pause">МЕНЮ</button>
    </div>
    
    <div id="money-gui">
        Монеты: <span id="hud-money">0</span> 💰
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        // UI Elements
        const accountScreen = document.getElementById('account-screen');
        const playerMenuScreen = document.getElementById('player-menu-screen');
        const gameHUD = document.getElementById('game-hud');
        const moneyGUI = document.getElementById('money-gui');
        
        // Global Config
        const TILE_SIZE = 30; 
        const GAME_WIDTH = canvas.width;
        const GAME_HEIGHT = canvas.height;
        const GRAVITY = 0.5;
        const JUMP_POWER = -10;
        const CHEST_REWARD = 50;
        const CAMERA_Y_OFFSET = GAME_HEIGHT / 2; 

        let gameState = 'ACCOUNT'; // ACCOUNT, GAME, PAUSED
        let blocks = {};
        let cameraY = 0; 

        let playerData = {
            name: 'Игрок_R6',
            money: 0,
            colors: {
                head: '#f1c40f',
                body: '#3498db',
                limbs: '#e74c3c'
            }
        };
        
        const CHEST_BLOCK_COLOR = '#FFD700'; 
        let chestKey = ''; 


        // --- Класс Аватара (R6) ---
        class Avatar {
            constructor(x, y, data) {
                this.x = x;
                this.y = y;
                this.width = TILE_SIZE; 
                this.height = TILE_SIZE; 
                this.limbWidth = TILE_SIZE / 2; 
                this.limbHeight = TILE_SIZE; 
                this.vy = 0;
                this.isGrounded = false;
                this.speed = 4;
                
                this.name = data.name;
                this.money = data.money;
                this.colors = data.colors;
                
                this.animator = 0;
                this.animSpeed = 0.2; 
                this.maxAngle = 0.6; 
            }
            
            getTotalHeight() {
                return TILE_SIZE * 3; 
            }

            drawBlock(x, y, w, h, color) {
                ctx.fillStyle = color;
                ctx.fillRect(x, y, w, h);
                ctx.strokeStyle = 'black';
                ctx.strokeRect(x, y, w, h);
            }

            draw(ctx) {
                ctx.save();
                ctx.translate(0, cameraY); 
                
                let angle = 0;
                if (this.isGrounded && (world.keys['a'] || world.keys['d'])) {
                     this.animator += this.animSpeed;
                     angle = Math.sin(this.animator) * this.maxAngle;
                } else {
                    this.animator = 0;
                }

                const baseY = this.y; 
                const limbColor = this.colors.limbs;
                
                // 1. Имя
                ctx.fillStyle = 'white';
                ctx.font = '14px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(this.name, this.x + this.width / 2, baseY - TILE_SIZE - 10);
                
                // 2. Голова (Head)
                this.drawBlock(this.x, baseY - TILE_SIZE, this.width, TILE_SIZE, this.colors.head);

                // 3. Торс (Torso)
                this.drawBlock(this.x, baseY, this.width, this.height, this.colors.body);
                
                // 4. Правая рука (Right Arm) - Противофаза
                ctx.save();
                ctx.translate(this.x + this.width, baseY); 
                ctx.rotate(-angle); 
                this.drawBlock(0, 0, this.limbWidth, this.limbHeight, limbColor);
                ctx.restore();
                
                // 5. Левая рука (Left Arm) - Фаза
                ctx.save();
                ctx.translate(this.x - this.limbWidth, baseY);
                ctx.rotate(angle); 
                this.drawBlock(0, 0, this.limbWidth, this.limbHeight, limbColor);
                ctx.restore();
                
                // 6. Правая нога (Right Leg) - Фаза
                ctx.save();
                ctx.translate(this.x + this.width, baseY + this.height);
                ctx.rotate(angle); 
                this.drawBlock(0, 0, this.limbWidth, this.limbHeight, limbColor);
                ctx.restore();

                // 7. Левая нога (Left Leg) - Противофаза
                ctx.save();
                ctx.translate(this.x - this.limbWidth, baseY + this.height);
                ctx.rotate(-angle); 
                this.drawBlock(0, 0, this.limbWidth, this.limbHeight, limbColor);
                ctx.restore();
                
                // 8. Глаза
                ctx.fillStyle = 'black';
                ctx.fillRect(this.x + 5, baseY - TILE_SIZE + 10, 5, 5);
                ctx.fillRect(this.x + this.width - 10, baseY - TILE_SIZE + 10, 5, 5);
                
                ctx.restore(); 
            }

            update(keys, blocks) {
                this.applyGravity(blocks);
                this.handleMovement(keys, blocks);
            }
            
            checkBounds() {
                 if (this.x - this.limbWidth < 0) this.x = this.limbWidth; 
                 if (this.x + this.width + this.limbWidth > GAME_WIDTH) this.x = GAME_WIDTH - this.width - this.limbWidth;
            }

            handleMovement(keys, blocks) {
                if (keys['a']) this.x -= this.speed;
                if (keys['d']) this.x += this.speed;
                
                if (keys['w'] && this.isGrounded) {
                    this.vy = JUMP_POWER;
                    this.isGrounded = false;
                }
                
                this.checkBounds();
            }
            
            applyGravity(blocks) {
                this.vy += GRAVITY;
                this.y += this.vy;
                this.isGrounded = false;
                
                const totalHeight = this.getTotalHeight();
                const floorY = GAME_HEIGHT - totalHeight + TILE_SIZE; 
                
                if (this.y >= floorY) {
                    this.y = floorY;
                    this.vy = 0;
                    this.isGrounded = true;
                }
                
                this.checkBlockCollision(blocks, totalHeight);
            }
            
            checkBlockCollision(blocks, totalHeight) {
                 const playerRect = { 
                    x: this.x, 
                    y: this.y - TILE_SIZE, 
                    width: this.width, 
                    height: totalHeight 
                };
                
                for (const key in blocks) {
                    const [bx, by] = key.split(',').map(Number);
                    const blockRect = { x: bx, y: by, width: TILE_SIZE, height: TILE_SIZE };

                    if (this.intersects(playerRect, blockRect)) {
                        
                        if (this.vy > 0 && playerRect.y + playerRect.height - this.vy <= blockRect.y) {
                            this.y = blockRect.y - totalHeight + TILE_SIZE;
                            this.vy = 0;
                            this.isGrounded = true;
                            return;
                        } 
                        else if (this.vy < 0 && playerRect.y + TILE_SIZE <= blockRect.y + blockRect.height && playerRect.y + TILE_SIZE > blockRect.y + blockRect.height / 2) {
                            this.y = blockRect.y + TILE_SIZE + TILE_SIZE + 1; 
                            this.vy = 0;
                            return;
                        }
                        else {
                           if (playerRect.y + playerRect.height > blockRect.y + 5 && playerRect.y < blockRect.y + blockRect.height - 5) {
                                if (world.keys['d'] && playerRect.x + playerRect.width > blockRect.x && playerRect.x < blockRect.x) {
                                     this.x = blockRect.x - playerRect.width;
                                }
                                else if (world.keys['a'] && playerRect.x < blockRect.x + blockRect.width && playerRect.x + playerRect.width > blockRect.x + blockRect.width) {
                                     this.x = blockRect.x + blockRect.width;
                                }
                           }
                        }
                    }
                }
            }

            intersects(r1, r2) {
                return r1.x < r2.x + r2.width &&
                       r1.x + r1.width > r2.x &&
                       r1.y < r2.y + r2.height &&
                       r1.y + r1.height > r2.y;
            }
            
            setPlayerDetails(data) {
                this.name = data.name;
                this.colors.body = data.colors.body;
            }
        }

        // --- Главный класс Игры ---
        class BlockWorld {
            constructor() {
                this.player = new Avatar(GAME_WIDTH / 2, GAME_HEIGHT - TILE_SIZE * 3, playerData);
                this.keys = {};
                this.setupInitialBlocks();
                this.setupEventListeners();
                this.loadAccountScreen(); // СТАРТОВЫЙ ЭКРАН - АККАУНТ
                this.gameLoop();
            }
            
            setupInitialBlocks() {
                blocks = {}; 
                // 1. Пол
                for (let x = 0; x < GAME_WIDTH / TILE_SIZE; x++) {
                    blocks[`${x * TILE_SIZE},${GAME_HEIGHT - TILE_SIZE}`] = '#27ae60';
                }
                
                // 2. Паркур-Башня
                const BASE_X = 50;
                let y = GAME_HEIGHT - TILE_SIZE * 2;
                for (let i = 0; i < 15; i++) {
                    let x = BASE_X + (i % 2 === 0 ? 0 : 60); 
                    y -= TILE_SIZE * 1.5; 
                    blocks[`${x},${y}`] = '#95a5a6'; 
                    blocks[`${x + TILE_SIZE},${y}`] = '#95a5a6';
                }
                
                // 3. Сундук
                chestKey = `${BASE_X + 60},${y - TILE_SIZE}`; 
                blocks[chestKey] = CHEST_BLOCK_COLOR;
                blocks[`${BASE_X + 60},${GAME_HEIGHT - TILE_SIZE * 2}`] = '#95a5a6';
                blocks[`${0},${-200}`] = '#34495e';
            }

            setupEventListeners() {
                document.addEventListener('keydown', (e) => {
                    if (gameState === 'GAME') {
                        this.keys[e.key.toLowerCase()] = true;
                    }
                });
                document.addEventListener('keyup', (e) => {
                    if (gameState === 'GAME') {
                        this.keys[e.key.toLowerCase()] = false;
                    }
                    if (e.key === 'Escape' && gameState === 'GAME') {
                        this.pauseGame();
                    }
                });
                
                document.getElementById('btn-start-game').addEventListener('click', () => this.saveAndStartGame());
                document.getElementById('btn-pause').addEventListener('click', () => this.pauseGame());
                document.getElementById('btn-add-friend').addEventListener('click', () => this.addFriend());
            }
            
            // --- УПРАВЛЕНИЕ МЕНЮ ---

            loadAccountScreen() {
                gameState = 'ACCOUNT';
                this.hideAllScreens();
                accountScreen.style.display = 'flex';
            }

            pauseGame() {
                 if (gameState === 'GAME') {
                    gameState = 'PAUSED';
                    this.hideAllScreens();
                    document.getElementById('menu-player-name').textContent = playerData.name;
                    playerMenuScreen.style.display = 'flex';
                }
            }
            
            unpauseGame() {
                gameState = 'GAME';
                this.hideAllScreens();
            }

            // Новый пункт меню: Выйти из Аккаунта
            logOut() {
                this.loadAccountScreen(); // Перебрасываем на экран создания аккаунта
                this.keys = {};
                // Сбрасываем позицию игрока и монеты при выходе из аккаунта
                playerData.money = 0;
                this.updateMoneyGUI();
                this.player.x = GAME_WIDTH / 2;
                this.player.y = GAME_HEIGHT - this.player.getTotalHeight() + TILE_SIZE;
            }


            saveAndStartGame() {
                // Берем данные с экрана аккаунта
                playerData.name = document.getElementById('reg-name').value || 'Игрок';
                playerData.colors.body = document.getElementById('reg-color-body').value;
                this.startGame();
            }

            startGame() {
                gameState = 'GAME';
                this.hideAllScreens();
                gameHUD.style.display = 'block';
                moneyGUI.style.display = 'block';
                
                document.getElementById('hud-name').textContent = playerData.name;
                this.updateMoneyGUI();

                this.player.setPlayerDetails(playerData);
                const totalHeight = this.player.getTotalHeight();
                this.player.x = GAME_WIDTH / 2;
                this.player.y = GAME_HEIGHT - totalHeight + TILE_SIZE; 
                this.player.vy = 0;
            }
            
            endGame() {
                alert("Спасибо за игру! Окно будет закрыто.");
                // Имитация закрытия, возвращаемся на стартовый экран
                this.logOut(); 
            }

            hideAllScreens() {
                accountScreen.style.display = 'none';
                playerMenuScreen.style.display = 'none';
                gameHUD.style.display = 'none';
                moneyGUI.style.display = 'none';
            }

            addFriend() {
                 const friendName = prompt("Введите никнейм друга для отправки запроса:");
                 if (friendName) {
                     alert(`✅ Запрос дружбы отправлен игроку "${friendName}"! (Имитация)`);
                 }
            }
            
            // --- ЛОГИКА ИГРЫ ---
            
            checkChestCollision() {
                 if (blocks[chestKey] === CHEST_BLOCK_COLOR) { 
                    const [cx, cy] = chestKey.split(',').map(Number);
                    const chestRect = { x: cx, y: cy, width: TILE_SIZE, height: TILE_SIZE };
                    
                    const playerRect = { 
                        x: this.player.x, 
                        y: this.player.y - TILE_SIZE, 
                        width: this.player.width, 
                        height: this.player.getTotalHeight() 
                    };
                    
                    if (this.player.intersects(playerRect, chestRect)) {
                        delete blocks[chestKey];
                        playerData.money += CHEST_REWARD; 
                        this.updateMoneyGUI();
                        
                        alert(`💰 ${playerData.name}, вы получили ${CHEST_REWARD} монет! Башня перестроилась.`);
                        
                        this.setupInitialBlocks(); 
                    }
                }
            }

            updateMoneyGUI() {
                 document.getElementById('hud-money').textContent = playerData.money;
            }
            
            // --- КАМЕРА ---
            updateCamera() {
                const desiredY = this.player.y - CAMERA_Y_OFFSET + this.player.getTotalHeight() / 2;
                const smoothing = 0.05; 
                cameraY += (desiredY - cameraY) * smoothing;
                
                cameraY *= -1;
                
                const maxY = 0;
                if (cameraY > maxY) {
                    cameraY = maxY;
                }
            }

            // --- ОСНОВНОЙ ЦИКЛ ---

            update() {
                if (gameState === 'GAME') {
                    this.player.update(this.keys, blocks);
                    this.checkChestCollision();
                    this.updateCamera(); 
                }
            }

            draw() {
                ctx.clearRect(0, 0, GAME_WIDTH, GAME_HEIGHT);
                
                // РИСУЕМ БЛОКИ
                for (const key in blocks) {
                    const [x, y] = key.split(',').map(Number);
                    
                    ctx.save();
                    ctx.translate(0, cameraY);
                    
                    ctx.fillStyle = blocks[key];
                    ctx.fillRect(x, y, TILE_SIZE, TILE_SIZE);
                    ctx.strokeStyle = '#2c3e50';
                    ctx.strokeRect(x, y, TILE_SIZE, TILE_SIZE);
                    
                    ctx.restore();
                }

                if (gameState === 'GAME' || gameState === 'PAUSED') {
                    this.player.draw(ctx);
                }
            }
            
            gameLoop = () => {
                this.update();
                this.draw();
                requestAnimationFrame(this.gameLoop);
            }
        }

        // --- ЗАПУСК ---
        let world;
        window.onload = () => {
            world = new BlockWorld();
        };

    </script>
</body>
</html>
