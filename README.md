<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>2D Runner — Гном</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background: #1a1a2e;
            font-family: Arial, sans-serif;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
            width: 100%;
            max-width: 800px;
            height: auto;
        }
        canvas {
            display: block;
            width: 100%;
            height: auto;
            background: #0f3460;
            border: 3px solid #537791;
        }
        #ui {
            position: absolute;
            top: 12px;
            left: 12px;
            color: white;
            font-size: 18px;
            z-index: 10;
        }
        #gameOver, #levelUp {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(15, 23, 42, 0.95);
            color: white;
            padding: 25px 50px;
            border-radius: 12px;
            text-align: center;
            display: none;
            z-index: 20;
            border: 2px solid #3b82f6;
        }
        button {
            margin-top: 18px;
            padding: 10px 24px;
            font-size: 17px;
            background: #10b981;
            color: white;
            border: none;
            border-radius: 6px;
            cursor: pointer;
        }
        #jumpBtn {
            position: absolute;
            bottom: 20px;
            right: 20px;
            width: 80px;
            height: 80px;
            background: rgba(59, 130, 246, 0.7);
            border: 3px solid white;
            border-radius: 50%;
            display: none;
            justify-content: center;
            align-items: center;
            font-size: 28px;
            color: white;
            user-select: none;
            -webkit-tap-highlight-color: transparent;
            cursor: pointer;
            z-index: 10;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <div id="ui">
            Уровень: <span id="level">1</span> | 
            Счёт: <span id="score">0</span> | 
            Жизни: <span id="lives">3</span>
        </div>

        <div id="gameOver">
            <h2>Игра окончена!</h2>
            <p>Финальный счёт: <span id="finalScore">0</span></p>
            <button onclick="restartGame()">Играть снова</button>
        </div>

        <div id="levelUp">
            <h2>Уровень <span id="nextLevel">2</span>!</h2>
            <p>Скорость ↑, препятствия сложнее!</p>
            <button onclick="continueGame()">Продолжить</button>
        </div>

        <canvas id="gameCanvas"></canvas>
        <div id="jumpBtn">↑</div>
    </div>

    <script>
        const container = document.getElementById('gameContainer');
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        function resizeCanvas() {
            const displayWidth = container.clientWidth;
            const displayHeight = Math.round(displayWidth * 0.5);
            canvas.width = 800;
            canvas.height = 400;
            canvas.style.width = `${displayWidth}px`;
            canvas.style.height = `${displayHeight}px`;
        }

        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // === ГНОМ: 48x64 пикселей, масштаб x2 ===
        const PLAYER_WIDTH = 48;
        const PLAYER_HEIGHT = 64;
        const SCALE = 2;

        const player = {
            x: 100,
            y: 400 - 20 - PLAYER_HEIGHT * SCALE,
            width: PLAYER_WIDTH * SCALE,
            height: PLAYER_HEIGHT * SCALE,
            velY: 0,
            jumpForce: -14,
            gravity: 0.75,
            onGround: true,
            frame: 0,
            lastFrameTime: 0,
            isJumping: false,
            takingDamage: false,
            damageTimer: 0
        };

        // Цвета (тёплая палитра)
        const C = {
            hat: '#5D4037',
            hatShadow: '#4E342E',
            hair: '#A1887F',
            skin: '#EFEBE9',
            blush: '#FFAB91',
            eye: '#3E2723',
            beard: '#ECEFF1',
            beardMid: '#CFD8DC',
            beardShadow: '#B0BEC5',
            vest: '#6D4C41',
            vestShadow: '#5D4037',
            shirt: '#D7CCC8',
            shirtStripe: '#8D6E63',
            button: '#FFD54F',
            belt: '#B71C1C',
            pants: '#607D8B',
            pantsShadow: '#455A64',
            boots: '#4E342E',
            sole: '#263238'
        };

        // Вспомогательная функция: рисует пиксель (логический → физический)
        function drawPixel(logicalX, logicalY, color) {
            ctx.fillStyle = color;
            ctx.fillRect(
                player.x + logicalX * SCALE,
                player.y + logicalY * SCALE,
                SCALE, SCALE
            );
        }

        // Анимация БЕГА (4 кадра)
        const runFrames = [
            // Кадр 0: правая нога вперёд
            () => {
                for (let y = 0; y < 64; y++) {
                    for (let x = 0; x < 48; x++) {
                        let color = C.hat; // фон — шляпа (перекроется)
                        if (y < 8) {
                            // Шляпа
                            if ((y === 0 && x >= 18 && x <= 29) ||
                                (y === 1 && x >= 16 && x <= 31) ||
                                (y === 2 && x >= 14 && x <= 33) ||
                                (y === 3 && x >= 12 && x <= 35) ||
                                (y === 4 && x >= 10 && x <= 37) ||
                                (y === 5 && x >= 8 && x <= 39) ||
                                (y === 6 && x >= 6 && x <= 41) ||
                                (y === 7 && x >= 4 && x <= 43)) {
                                color = (y === 5 && x >= 20 && x <= 27) ? '#8D6E63' : C.hat;
                            } else color = 'transparent';
                        } else if (y >= 8 && y <= 17) {
                            // Лицо и борода
                            if (x >= 14 && x <= 33) {
                                if (y <= 11) {
                                    // Лицо
                                    color = C.skin;
                                    if (y === 10 && x >= 18 && x <= 19) color = C.eye;
                                    if (y === 10 && x >= 28 && x <= 29) color = C.eye;
                                    if (y === 11 && x >= 22 && x <= 25) color = C.blush;
                                } else {
                                    // Борода
                                    if (x >= 16 && x <= 31) {
                                        if (y === 12) color = C.beard;
                                        else if (y === 13) color = (x % 3 === 0) ? C.beardShadow : C.beardMid;
                                        else color = C.beardShadow;
                                    } else color = 'transparent';
                                }
                            } else color = 'transparent';
                        } else if (y >= 18 && y <= 33) {
                            // Туловище
                            if (x >= 12 && x <= 35) {
                                if (y <= 25) {
                                    // Жилет
                                    color = C.vest;
                                    if (x >= 22 && x <= 25 && y >= 20 && y <= 24) color = C.button;
                                } else {
                                    // Пояс
                                    color = (y <= 27) ? C.belt : C.pants;
                                }
                            } else color = 'transparent';
                        } else if (y >= 34 && y <= 55) {
                            // Ноги (бег: правая вперёд)
                            if ((x >= 18 && x <= 25 && y >= 34 && y <= 45) || // левая назад
                                (x >= 22 && x <= 29 && y >= 40 && y <= 55)) { // правая вперёд
                                color = C.pants;
                                if (y > 45) color = C.boots;
                            } else color = 'transparent';
                        } else if (y >= 56 && y <= 63) {
                            // Ступни
                            if (x >= 20 && x <= 27) color = C.sole;
                            else color = 'transparent';
                        } else {
                            color = 'transparent';
                        }
                        if (color !== 'transparent') drawPixel(x, y, color);
                    }
                }
            },
            // Кадр 1: ноги вместе
            () => {
                for (let y = 0; y < 64; y++) {
                    for (let x = 0; x < 48; x++) {
                        let color = 'transparent';
                        if (y < 8) {
                            if (x >= 10 && x <= 37 && y >= 4) color = C.hat;
                            else if (x >= 18 && x <= 29 && y === 5) color = '#8D6E63';
                        } else if (y >= 8 && y <= 17) {
                            if (x >= 14 && x <= 33) {
                                if (y <= 11) {
                                    color = C.skin;
                                    if ((y === 10 && x === 18) || (y === 10 && x === 29)) color = C.eye;
                                    if (y === 11 && x >= 22 && x <= 25) color = C.blush;
                                } else {
                                    color = (x >= 16 && x <= 31) ? C.beard : 'transparent';
                                }
                            }
                        } else if (y >= 18 && y <= 33) {
                            if (x >= 12 && x <= 35) {
                                color = (y <= 25) ? C.vest : (y <= 27 ? C.belt : C.pants);
                                if (x >= 22 && x <= 25 && y >= 20 && y <= 24) color = C.button;
                            }
                        } else if (y >= 34 && y <= 55) {
                            if (x >= 20 && x <= 27) color = (y > 45) ? C.boots : C.pants;
                        } else if (y >= 56 && y <= 63) {
                            if (x >= 20 && x <= 27) color = C.sole;
                        }
                        if (color !== 'transparent') drawPixel(x, y, color);
                    }
                }
            },
            // Кадр 2: левая нога вперёд
            () => {
                for (let y = 0; y < 64; y++) {
                    for (let x = 0; x < 48; x++) {
                        let color = 'transparent';
                        if (y < 8) {
                            if (x >= 10 && x <= 37 && y >= 4) color = C.hat;
                            else if (x >= 18 && x <= 29 && y === 5) color = '#8D6E63';
                        } else if (y >= 8 && y <= 17) {
                            if (x >= 14 && x <= 33) {
                                if (y <= 11) {
                                    color = C.skin;
                                    if ((y === 10 && x === 18) || (y === 10 && x === 29)) color = C.eye;
                                    if (y === 11 && x >= 22 && x <= 25) color = C.blush;
                                } else {
                                    color = (x >= 16 && x <= 31) ? C.beard : 'transparent';
                                }
                            }
                        } else if (y >= 18 && y <= 33) {
                            if (x >= 12 && x <= 35) {
                                color = (y <= 25) ? C.vest : (y <= 27 ? C.belt : C.pants);
                                if (x >= 22 && x <= 25 && y >= 20 && y <= 24) color = C.button;
                            }
                        } else if (y >= 34 && y <= 55) {
                            if ((x >= 18 && x <= 25 && y >= 40 && y <= 55) || // левая вперёд
                                (x >= 22 && x <= 29 && y >= 34 && y <= 45)) { // правая назад
                                color = (y > 45) ? C.boots : C.pants;
                            }
                        } else if (y >= 56 && y <= 63) {
                            if (x >= 20 && x <= 27) color = C.sole;
                        }
                        if (color !== 'transparent') drawPixel(x, y, color);
                    }
                }
            },
            // Кадр 3: ноги вместе (зеркало кадра 1)
            () => {
                // Повторим кадр 1 для симметрии
                for (let y = 0; y < 64; y++) {
                    for (let x = 0; x < 48; x++) {
                        let color = 'transparent';
                        if (y < 8) {
                            if (x >= 10 && x <= 37 && y >= 4) color = C.hat;
                            else if (x >= 18 && x <= 29 && y === 5) color = '#8D6E63';
                        } else if (y >= 8 && y <= 17) {
                            if (x >= 14 && x <= 33) {
                                if (y <= 11) {
                                    color = C.skin;
                                    if ((y === 10 && x === 18) || (y === 10 && x === 29)) color = C.eye;
                                    if (y === 11 && x >= 22 && x <= 25) color = C.blush;
                                } else {
                                    color = (x >= 16 && x <= 31) ? C.beard : 'transparent';
                                }
                            }
                        } else if (y >= 18 && y <= 33) {
                            if (x >= 12 && x <= 35) {
                                color = (y <= 25) ? C.vest : (y <= 27 ? C.belt : C.pants);
                                if (x >= 22 && x <= 25 && y >= 20 && y <= 24) color = C.button;
                            }
                        } else if (y >= 34 && y <= 55) {
                            if (x >= 20 && x <= 27) color = (y > 45) ? C.boots : C.pants;
                        } else if (y >= 56 && y <= 63) {
                            if (x >= 20 && x <= 27) color = C.sole;
                        }
                        if (color !== 'transparent') drawPixel(x, y, color);
                    }
                }
            }
        ];

        // Анимация ПРЫЖКА
        function drawJumpFrame() {
            for (let y = 0; y < 64; y++) {
                for (let x = 0; x < 48; x++) {
                    let color = 'transparent';
                    if (y < 8) {
                        if (x >= 10 && x <= 37 && y >= 4) color = C.hat;
                        else if (x >= 18 && x <= 29 && y === 5) color = '#8D6E63';
                    } else if (y >= 8 && y <= 17) {
                        if (x >= 14 && x <= 33) {
                            if (y <= 11) {
                                color = C.skin;
                                if ((y === 10 && x === 18) || (y === 10 && x === 29)) color = C.eye;
                                if (y === 11 && x >= 22 && x <= 25) color = C.blush;
                            } else {
                                color = (x >= 16 && x <= 31) ? C.beard : 'transparent';
                            }
                        }
                    } else if (y >= 18 && y <= 33) {
                        if (x >= 12 && x <= 35) {
                            color = (y <= 25) ? C.vest : (y <= 27 ? C.belt : C.pants);
                            if (x >= 22 && x <= 25 && y >= 20 && y <= 24) color = C.button;
                        }
                    } else if (y >= 34 && y <= 55) {
                        if (x >= 20 && x <= 27) color = (y > 45) ? C.boots : C.pants;
                    } else if (y >= 56 && y <= 63) {
                        if (x >= 20 && x <= 27) color = C.sole;
                    }
                    if (color !== 'transparent') drawPixel(x, y, color);
                }
            }
        }

        // Анимация УРОНА
        function drawDamageFrame() {
            for (let y = 0; y < 64; y++) {
                for (let x = 0; x < 48; x++) {
                    let color = 'transparent';
                    if (y < 8) {
                        if (x >= 10 && x <= 37 && y >= 4) color = C.hat;
                    } else if (y >= 8 && y <= 17) {
                        if (x >= 14 && x <= 33) {
                            if (y <= 11) {
                                color = C.skin;
                                // Глаза испуганы
                                if (y === 9 && x >= 17 && x <= 20) color = C.eye;
                                if (y === 9 && x >= 27 && x <= 30) color = C.eye;
                                if (y === 10 && x >= 18 && x <= 19) color = '#FFFFFF'; // белки больше
                                if (y === 10 && x >= 28 && x <= 29) color = '#FFFFFF';
                                if (y === 11 && x >= 23 && x <= 24) color = '#F44336'; // открытый рот
                            } else {
                                color = (x >= 16 && x <= 31) ? C.beard : 'transparent';
                            }
                        }
                    } else if (y >= 18 && y <= 33) {
                        if (x >= 12 && x <= 35) {
                            color = (y <= 25) ? C.vest : (y <= 27 ? C.belt : C.pants);
                        }
                    } else if (y >= 34 && y <= 55) {
                        if (x >= 20 && x <= 27) color = (y > 45) ? C.boots : C.pants;
                    } else if (y >= 56 && y <= 63) {
                        if (x >= 20 && x <= 27) color = C.sole;
                    }
                    if (color !== 'transparent') drawPixel(x, y, color);
                }
            }
        }

        // === ОСТАЛЬНОЙ КОД ИГРЫ (без изменений) ===

        const isMobile = /Android|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
        const jumpBtn = document.getElementById('jumpBtn');
        if (isMobile) {
            jumpBtn.style.display = 'flex';
            const jump = () => {
                if (player.onGround) {
                    player.velY = player.jumpForce;
                    player.onGround = false;
                    player.isJumping = true;
                }
            };
            jumpBtn.addEventListener('touchstart', e => { e.preventDefault(); jump(); }, { passive: false });
            jumpBtn.addEventListener('mousedown', e => { e.preventDefault(); jump(); });
        }

        document.addEventListener('keydown', e => {
            if ((e.code === 'ArrowUp' || e.code === 'Space') && player.onGround) {
                player.velY = player.jumpForce;
                player.onGround = false;
                player.isJumping = true;
            }
        });

        let obstacles = [];
        let score = 0;
        let level = 1;
        let lives = 3;
        let gameOver = false;
        let gamePaused = false;
        let nextObstacleTime = 0;
        let isLevelingUp = false;

        const levelConfig = [
            { speed: 5, types: ['ground', 'triple', 'ground+ground'] },
            { speed: 6, types: ['ground', 'slider', 'ground+ground', 'double', 'triple'] },
            { speed: 7, types: ['ground', 'pendulum', 'air', 'double', 'ground+air', 'ground+ground', 'triple'] },
            { speed: 8, types: ['double', 'air', 'slider', 'ground+air', 'ground+ground'] },
            { speed: 9, types: ['triple', 'air', 'ground+air', 'slider'] },
            { speed: 10, types: ['ground+air', 'pendulum'] },
            { speed: 11, types: ['ground+air', 'slider'] },
            { speed: 12, types: ['pendulum', 'ground+air'] },
            { speed: 13, types: ['slider', 'pendulum', 'ground+air'] },
            { speed: 14, types: ['slider', 'pendulum', 'ground+air'] }
        ];

        function createObstacle(type) {
            const groundY = 380;
            switch (type) {
                case 'ground':
                    return {
                        x: 800,
                        y: groundY - (20 + Math.random() * 40),
                        width: 30 + Math.random() * 20,
                        height: 20 + Math.random() * 40,
                        type: 'ground',
                        passed: false,
                        color: '#E91E63'
                    };
                case 'slider':
                    const startX = player.x + 400 + Math.random() * 200;
                    return {
                        x: startX,
                        y: groundY - 55,
                        width: 50,
                        height: 35,
                        type: 'slider',
                        passed: false,
                        color: '#009688',
                        speed: 4 + level * 0.3
                    };
                case 'pendulum':
                    return {
                        x: 800,
                        y: groundY - 120,
                        width: 40,
                        height: 40,
                        type: 'pendulum',
                        passed: false,
                        color: '#673AB7',
                        angle: 0,
                        angularVel: 0.1
                    };
                case 'air':
                    return {
                        x: 800,
                        y: 150 + Math.random() * 200,
                        width: 30,
                        height: 30,
                        type: 'air',
                        passed: false,
                        color: '#f00cbe',
                        vx: -5 - Math.random() * 1.5
                    };
                case 'double':
                    return {
                        x: 800,
                        y: groundY - 40,
                        width: 30,
                        height: 40,
                        type: 'double',
                        parts: [
                            { dx: 0, dy: 0 },
                            { dx: 40, dy: -30 }
                        ],
                        passed: false,
                        color: '#FF9800'
                    };
                case 'triple':
                    return {
                        x: 800,
                        y: groundY - 30,
                        width: 10,
                        height: 30,
                        type: 'triple',
                        parts: [
                            { dx: 0, dy: 0 },
                            { dx: 30, dy: -10 },
                            { dx: 60, dy: -20 }
                        ],
                        passed: false,
                        color: '#9C27B0'
                    };
                case 'ground+ground':
                    const obs1 = createObstacle('ground');
                    obs1.x = 800;
                    const gap = 150 + Math.random() * 80;
                    const obs2 = createObstacle('ground');
                    obs2.x = 800 + obs1.width + gap;
                    return { type: 'group', items: [obs1, obs2] };
                case 'ground+air':
                    const g = createObstacle('ground');
                    g.x = 800;
                    const a = createObstacle('air');
                    a.x = 800 + g.width * 0.3;
                    a.y = g.y - 80 - Math.random() * 30;
                    a.vx = -1.3;
                    return { type: 'group', items: [g, a] };
                default:
                    return createObstacle('ground');
            }
        }

        function updateObstacles() {
            if (gameOver || gamePaused) return;
            const cfg = levelConfig[level - 1] || levelConfig[levelConfig.length - 1];
            const speed = cfg.speed;
            const now = Date.now();
            if (now > nextObstacleTime) {
                const type = cfg.types[Math.floor(Math.random() * cfg.types.length)];
                const obs = createObstacle(type);
                if (obs.type === 'group') {
                    obstacles.push(...obs.items);
                } else {
                    obstacles.push(obs);
                }
                const baseInterval = Math.max(800, 2200 - (level - 1) * 180);
                nextObstacleTime = now + baseInterval * (0.7 + Math.random() * 0.5);
            }

            for (let i = obstacles.length - 1; i >= 0; i--) {
                const obs = obstacles[i];
                if (!obs) continue;
                obs.x -= speed * 0.9;
                if (obs.type === 'slider') obs.x -= obs.speed;
                if (obs.type === 'pendulum') {
                    obs.angle += obs.angularVel;
                    obs.y = (400 - 120) + Math.sin(obs.angle) * 85;
                }
                if (obs.type === 'air') obs.x += (obs.vx || -1);

                if (!obs.passed && obs.x + obs.width < player.x - 20) {
                    obs.passed = true;
                    score += 10 * level;
                    document.getElementById('score').textContent = score;
                    const scoreForNextLevel = 100 * level * (level + 1);
                    if (!isLevelingUp && !gamePaused && score >= scoreForNextLevel && level < 10) {
                        levelUp();
                    }
                }

                if (obs.x + obs.width < -150) {
                    obstacles.splice(i, 1);
                    continue;
                }

                let collided = false;
                if (obs.type === 'double' || obs.type === 'triple') {
                    for (const part of obs.parts) {
                        const px = obs.x + part.dx;
                        const py = obs.y + part.dy;
                        if (
                            player.x < px + obs.width &&
                            player.x + player.width > px &&
                            player.y < py + obs.height &&
                            player.y + player.height > py
                        ) {
                            collided = true;
                            break;
                        }
                    }
                } else {
                    if (
                        player.x < obs.x + obs.width &&
                        player.x + player.width > obs.x &&
                        player.y < obs.y + obs.height &&
                        player.y + player.height > obs.y
                    ) {
                        collided = true;
                    }
                }

                if (collided) {
                    takeDamage();
                    obstacles.splice(i, 1);
                    continue;
                }
            }
        }

        function takeDamage() {
            if (gameOver || gamePaused || isLevelingUp) return;
            lives--;
            document.getElementById('lives').textContent = lives;
            player.takingDamage = true;
            player.damageTimer = 30; // 0.5 сек при 60 FPS
            if (lives <= 0) endGame();
        }

        function levelUp() {
            if (gameOver || gamePaused || isLevelingUp) return;
            isLevelingUp = true;
            level++;
            obstacles = [];
            nextObstacleTime = Date.now() + 2500;
            document.getElementById('level').textContent = level;
            document.getElementById('nextLevel').textContent = level;
            gamePaused = true;
            document.getElementById('levelUp').style.display = 'block';
            setTimeout(() => { isLevelingUp = false; }, 3000);
            if (level >= 10) {
                setTimeout(() => {
                    if (!gameOver && gamePaused) {
                        document.getElementById('levelUp').style.display = 'none';
                        document.getElementById('gameOver').style.display = 'block';
                        document.getElementById('finalScore').textContent = score;
                        document.querySelector('#gameOver h2').textContent = '🏆 Победа!';
                        gameOver = true;
                    }
                }, 1800);
            }
        }

        function continueGame() {
            if (gameOver || !gamePaused) return;
            document.getElementById('levelUp').style.display = 'none';
            setTimeout(() => {
                gamePaused = false;
                isLevelingUp = false;
                player.onGround = (player.y >= 380 - player.height / SCALE * SCALE);
            }, 120);
        }

        function endGame() {
            if (gameOver) return;
            gameOver = true;
            document.getElementById('finalScore').textContent = score;
            document.getElementById('gameOver').style.display = 'block';
        }

        function restartGame() {
            player.y = 400 - 20 - PLAYER_HEIGHT * SCALE;
            player.velY = 0;
            player.onGround = true;
            player.isJumping = false;
            player.takingDamage = false;
            player.damageTimer = 0;
            obstacles = [];
            score = 0;
            level = 1;
            lives = 3;
            gameOver = false;
            gamePaused = false;
            isLevelingUp = false;
            nextObstacleTime = Date.now() + 2000;
            document.getElementById('score').textContent = score;
            document.getElementById('level').textContent = level;
            document.getElementById('lives').textContent = lives;
            document.getElementById('gameOver').style.display = 'none';
            document.getElementById('levelUp').style.display = 'none';
        }

        function draw() {
            ctx.fillStyle = '#0f3460';
            ctx.fillRect(0, 0, 800, 400);
            ctx.fillStyle = '#444';
            ctx.fillRect(0, 380, 800, 20);

            // Рисуем гнома
            if (player.takingDamage && player.damageTimer > 0) {
                drawDamageFrame();
                player.damageTimer--;
            } else if (!player.onGround) {
                drawJumpFrame();
            } else {
                // Анимация бега
                const now = performance.now();
                if (now - player.lastFrameTime > 150) { // смена кадра каждые 150 мс
                    player.frame = (player.frame + 1) % 4;
                    player.lastFrameTime = now;
                }
                runFrames[player.frame]();
            }

            // Препятствия
            obstacles.forEach(obs => {
                if (!obs) return;
                ctx.fillStyle = obs.color;
                if (obs.type === 'double' || obs.type === 'triple') {
                    obs.parts.forEach(part => {
                        ctx.fillRect(obs.x + part.dx, obs.y + part.dy, obs.width, obs.height);
                    });
                } else {
                    ctx.fillRect(obs.x, obs.y, obs.width, obs.height);
                }
            });
        }

        function updatePlayer() {
            if (gameOver || gamePaused) return;
            player.velY += player.gravity;
            player.y += player.velY;
            const groundY = 380 - PLAYER_HEIGHT * SCALE;
            if (player.y >= groundY) {
                player.y = groundY;
                player.velY = 0;
                player.onGround = true;
                player.isJumping = false;
            } else {
                player.onGround = false;
            }
        }

        let lastTime = 0;
        function gameLoop(timestamp) {
            const delta = timestamp - lastTime;
            lastTime = timestamp;
            if (!gameOver && !gamePaused) {
                updatePlayer();
                updateObstacles();
                draw();
            }
            requestAnimationFrame(gameLoop);
        }

        restartGame();
        requestAnimationFrame(gameLoop);
    </script>
</body>
</html>
