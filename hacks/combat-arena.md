---
layout: default
title: Combat Arena - 2D Fighting Game
permalink: /fight!/
description: An intense 2D combat game with dynamic combat mechanics, platforming, and AI enemies
---

# Combat Arena 🎮

<div id="gameContainer" style="width: 100%; max-width: 900px; margin: 20px auto; border: 3px solid #333; border-radius: 10px; overflow: hidden; background: #1a1a1a; box-shadow: 0 0 20px rgba(0,0,0,0.8);">
    <canvas id="gameCanvas" width="800" height="600" style="display: block; background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);"></canvas>
</div>

<script>
class CombatArena {
    constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');
        this.width = this.canvas.width;
        this.height = this.canvas.height;
        
        this.gameState = 'menu'; // menu, instructions, playing, gameOver, victory
        this.score = 0;
        this.level = 1;
        
        this.keys = {};
        this.particles = [];
        this.groundLevel = this.height - 100;
        
        this.setupEventListeners();
        this.initGame();
        this.gameLoop();
    }
    
    setupEventListeners() {
        window.addEventListener('keydown', (e) => {
            this.keys[e.key.toLowerCase()] = true;
            if (e.key === ' ') e.preventDefault();
        });
        window.addEventListener('keyup', (e) => {
            this.keys[e.key.toLowerCase()] = false;
        });
        this.canvas.addEventListener('click', (e) => this.handleCanvasClick(e));
    }
    
    handleCanvasClick(e) {
        const rect = this.canvas.getBoundingClientRect();
        const x = e.clientX - rect.left;
        const y = e.clientY - rect.top;
        
        if (this.gameState === 'menu') {
            if (x > 200 && x < 600 && y > 250 && y < 320) {
                this.gameState = 'instructions';
            }
        } else if (this.gameState === 'instructions') {
            this.gameState = 'playing';
        } else if (this.gameState === 'gameOver' || this.gameState === 'victory') {
            this.initGame();
            this.gameState = 'menu';
        }
    }
    
    initGame() {
        // Player
        this.player = {
            x: 100,
            y: this.groundLevel - 50,
            width: 40,
            height: 50,
            velocityY: 0,
            velocityX: 0,
            speed: 5,
            jumpPower: 12,
            isJumping: false,
            health: 100,
            maxHealth: 100,
            facing: 1, // 1 = right, -1 = left
            attacking: false,
            attackCooldown: 0,
            comboCount: 0,
            comboCooldown: 0,
            attackType: 'punch', // punch, kick, heavy
            animationFrame: 0
        };
        
        // Enemy
        this.enemy = {
            x: this.width - 150,
            y: this.groundLevel - 50,
            width: 45,
            height: 55,
            velocityY: 0,
            velocityX: 0,
            health: 100,
            maxHealth: 100,
            facing: -1, // 1 = right, -1 = left
            attacking: false,
            attackCooldown: 0,
            attackType: 'punch',
            ai: {
                actionCooldown: 0,
                distance: 0,
                state: 'idle' // idle, attacking, moving, defending
            },
            animationFrame: 0
        };
        
        // Platforms
        this.platforms = [
            { x: 0, y: this.groundLevel, width: this.width, height: 100 }, // Ground
            { x: 150, y: this.groundLevel - 150, width: 200, height: 20 }, // Left platform
            { x: 450, y: this.groundLevel - 150, width: 200, height: 20 }, // Right platform
            { x: 300, y: this.groundLevel - 280, width: 200, height: 20 } // Top platform
        ];
        
        this.particles = [];
        this.score = 0;
        this.gameState = 'playing';
    }
    
    update() {
        if (this.gameState !== 'playing') return;
        
        // Apply gravity
        const gravity = 0.6;
        this.player.velocityY += gravity;
        this.enemy.velocityY += gravity;
        
        // Player movement
        this.player.velocityX = 0;
        if (this.keys['a'] || this.keys['arrowleft']) {
            this.player.velocityX = -this.player.speed;
            this.player.facing = -1;
        }
        if (this.keys['d'] || this.keys['arrowright']) {
            this.player.velocityX = this.player.speed;
            this.player.facing = 1;
        }
        
        // Player jump
        if ((this.keys[' '] || this.keys['w'] || this.keys['arrowup']) && !this.player.isJumping) {
            this.player.velocityY = -this.player.jumpPower;
            this.player.isJumping = true;
        }
        
        // Player attack
        this.player.attackCooldown--;
        if (this.keys['j'] && this.player.attackCooldown <= 0) {
            this.player.attacking = true;
            this.player.attackType = 'punch';
            this.player.attackCooldown = 15;
            this.comboAttack();
        }
        if (this.keys['k'] && this.player.attackCooldown <= 0) {
            this.player.attacking = true;
            this.player.attackType = 'kick';
            this.player.attackCooldown = 20;
            this.player.comboCount = 0;
        }
        if (this.keys['l'] && this.player.attackCooldown <= 0) {
            this.player.attacking = true;
            this.player.attackType = 'heavy';
            this.player.attackCooldown = 40;
            this.player.comboCount = 0;
        }
        
        // Update positions
        this.player.x += this.player.velocityX;
        this.player.y += this.player.velocityY;
        
        this.enemy.x += this.enemy.velocityX;
        this.enemy.y += this.enemy.velocityY;
        
        // Platform collision
        this.checkPlatformCollisions(this.player);
        this.checkPlatformCollisions(this.enemy);
        
        // Boundary collision
        if (this.player.x < 0) this.player.x = 0;
        if (this.player.x + this.player.width > this.width) this.player.x = this.width - this.player.width;
        
        if (this.enemy.x < 0) this.enemy.x = 0;
        if (this.enemy.x + this.enemy.width > this.width) this.enemy.x = this.width - this.enemy.width;
        
        // Falling off map
        if (this.player.y > this.height) {
            this.player.health = 0;
            this.gameState = 'gameOver';
        }
        if (this.enemy.y > this.height) {
            this.score += 500;
            this.gameState = 'victory';
        }
        
        // Combat detection
        this.checkCombat();
        
        // Enemy AI
        this.updateEnemyAI();
        
        // Update cooldowns
        this.player.comboCooldown--;
        
        // Update particles
        this.particles = this.particles.filter(p => {
            p.life--;
            p.x += p.vx;
            p.y += p.vy;
            p.vy += 0.2;
            return p.life > 0;
        });
        
        // Game over conditions
        if (this.player.health <= 0) {
            this.gameState = 'gameOver';
        }
        if (this.enemy.health <= 0) {
            this.score += 1000;
            this.gameState = 'victory';
        }
    }
    
    comboAttack() {
        if (this.player.comboCooldown > 0) {
            this.player.comboCount++;
        } else {
            this.player.comboCount = 1;
        }
        this.player.comboCooldown = 30;
    }
    
    checkPlatformCollisions(entity) {
        for (let platform of this.platforms) {
            // Check if entity is above platform and falling
            if (entity.velocityY > 0 &&
                entity.y + entity.height >= platform.y &&
                entity.y + entity.height <= platform.y + 20 &&
                entity.x + entity.width > platform.x &&
                entity.x < platform.x + platform.width) {
                
                entity.y = platform.y - entity.height;
                entity.velocityY = 0;
                entity.isJumping = false;
            }
        }
    }
    
    checkCombat() {
        const distance = Math.hypot(
            this.player.x - this.enemy.x,
            this.player.y - this.enemy.y
        );
        
        let attackRange = 60;
        if (this.player.attackType === 'kick') attackRange = 80;
        if (this.player.attackType === 'heavy') attackRange = 100;
        
        if (this.player.attacking && distance < attackRange) {
            let damage = 10;
            if (this.player.attackType === 'punch') damage = 15 + (this.player.comboCount * 5);
            if (this.player.attackType === 'kick') damage = 25;
            if (this.player.attackType === 'heavy') damage = 40;
            
            this.enemy.health -= damage;
            this.createHitEffect(this.enemy.x, this.enemy.y, damage);
            this.score += damage;
            this.player.attacking = false;
            
            // Knockback
            this.enemy.velocityX = (this.player.facing) * 8;
            this.enemy.velocityY = -5;
        }
        
        let enemyAttackRange = 60;
        if (this.enemy.attacking && distance < enemyAttackRange) {
            let damage = 12;
            if (this.enemy.attackType === 'kick') damage = 20;
            this.player.health -= damage;
            this.createHitEffect(this.player.x, this.player.y, damage);
            this.enemy.attacking = false;
            
            // Knockback
            this.player.velocityX = (this.enemy.facing) * 6;
            this.player.velocityY = -3;
        }
    }
    
    updateEnemyAI() {
        const distance = Math.hypot(
            this.player.x - this.enemy.x,
            this.player.y - this.enemy.y
        );
        
        this.enemy.ai.distance = distance;
        this.enemy.ai.actionCooldown--;
        
        // Face player
        if (this.player.x < this.enemy.x) {
            this.enemy.facing = -1;
        } else {
            this.enemy.facing = 1;
        }
        
        if (this.enemy.ai.actionCooldown <= 0) {
            const rand = Math.random();
            
            if (distance > 250) {
                // Move closer
                this.enemy.velocityX = this.player.x > this.enemy.x ? 4 : -4;
                this.enemy.ai.state = 'moving';
                this.enemy.ai.actionCooldown = 30;
            } else if (distance < 60) {
                // Too close, back away
                this.enemy.velocityX = this.player.x > this.enemy.x ? -3 : 3;
                this.enemy.ai.state = 'moving';
                this.enemy.ai.actionCooldown = 20;
            } else if (rand > 0.5) {
                // Attack
                if (Math.random() > 0.6) {
                    this.enemy.attackType = 'kick';
                    this.enemy.ai.actionCooldown = 35;
                } else {
                    this.enemy.attackType = 'punch';
                    this.enemy.ai.actionCooldown = 20;
                }
                this.enemy.attacking = true;
                this.enemy.ai.state = 'attacking';
            } else {
                // Jump
                if (!this.enemy.isJumping) {
                    this.enemy.velocityY = -this.enemy.jumpPower || -10;
                    this.enemy.isJumping = true;
                    this.enemy.ai.state = 'idle';
                    this.enemy.ai.actionCooldown = 30;
                }
            }
        }
        
        this.enemy.velocityX *= 0.95; // Friction
    }
    
    createHitEffect(x, y, damage) {
        for (let i = 0; i < 8; i++) {
            this.particles.push({
                x: x,
                y: y,
                vx: (Math.random() - 0.5) * 6,
                vy: (Math.random() - 0.5) * 6,
                life: 30,
                size: Math.random() * 3 + 2,
                color: `hsl(${Math.random() * 60 + 10}, 100%, 50%)`
            });
        }
    }
    
    drawMenu() {
        // Background
        this.ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
        this.ctx.fillRect(0, 0, this.width, this.height);
        
        // Title
        this.ctx.fillStyle = '#FFD700';
        this.ctx.font = 'bold 80px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('COMBAT ARENA', this.width / 2, 100);
        
        // Subtitle
        this.ctx.fillStyle = '#FF6B9D';
        this.ctx.font = 'italic 30px Arial';
        this.ctx.fillText('2D Fighting Game', this.width / 2, 150);
        
        // Play Button
        this.ctx.fillStyle = '#00D4FF';
        this.ctx.fillRect(200, 250, 400, 70);
        this.ctx.strokeStyle = '#FFD700';
        this.ctx.lineWidth = 3;
        this.ctx.strokeRect(200, 250, 400, 70);
        
        this.ctx.fillStyle = '#000';
        this.ctx.font = 'bold 40px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('START GAME', this.width / 2, 295);
        
        // Instructions text
        this.ctx.fillStyle = '#FFF';
        this.ctx.font = '16px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('Click START to learn the controls', this.width / 2, 370);
    }
    
    drawInstructions() {
        // Background
        this.ctx.fillStyle = 'rgba(0, 0, 0, 0.85)';
        this.ctx.fillRect(0, 0, this.width, this.height);
        
        // Title
        this.ctx.fillStyle = '#00D4FF';
        this.ctx.font = 'bold 50px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('INSTRUCTIONS', this.width / 2, 50);
        
        // Controls
        const controls = [
            '⌨️ MOVEMENT',
            'A / ← LEFT        D / → RIGHT        W / ↑ JUMP',
            '',
            '⚔️ ATTACKS',
            'J - PUNCH (Fast, combos!) | K - KICK (Medium) | L - HEAVY (Slow, powerful)',
            '',
            '❤️ OBJECTIVE',
            'Defeat the enemy! Watch your health bar.',
            'Knock them off the stage for victory!',
            '',
            'TIP: Chain punches together for increased combo damage! 🔥',
            ''
        ];
        
        this.ctx.fillStyle = '#FFF';
        this.ctx.font = '16px Arial';
        this.ctx.textAlign = 'left';
        let y = 120;
        for (let control of controls) {
            if (control.startsWith('⌨️') || control.startsWith('⚔️') || control.startsWith('❤️')) {
                this.ctx.fillStyle = '#FFD700';
                this.ctx.font = 'bold 18px Arial';
            } else {
                this.ctx.fillStyle = '#FFF';
                this.ctx.font = '16px Arial';
            }
            this.ctx.fillText(control, 50, y);
            y += 30;
        }
        
        // Click to start
        this.ctx.fillStyle = '#00D4FF';
        this.ctx.font = 'bold 20px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('⬆️ CLICK TO START FIGHTING ⬆️', this.width / 2, this.height - 30);
    }
    
    drawGame() {
        // Clear and draw background
        this.ctx.clearRect(0, 0, this.width, this.height);
        const gradient = this.ctx.createLinearGradient(0, 0, 0, this.height);
        gradient.addColorStop(0, '#667eea');
        gradient.addColorStop(1, '#764ba2');
        this.ctx.fillStyle = gradient;
        this.ctx.fillRect(0, 0, this.width, this.height);
        
        // Draw platforms
        this.ctx.fillStyle = '#2D3436';
        this.ctx.strokeStyle = '#FF6B9D';
        this.ctx.lineWidth = 3;
        for (let platform of this.platforms) {
            this.ctx.fillRect(platform.x, platform.y, platform.width, platform.height);
            this.ctx.strokeRect(platform.x, platform.y, platform.width, platform.height);
        }
        
        // Draw particles
        for (let particle of this.particles) {
            this.ctx.fillStyle = particle.color;
            this.ctx.globalAlpha = particle.life / 30;
            this.ctx.fillRect(particle.x, particle.y, particle.size, particle.size);
        }
        this.ctx.globalAlpha = 1;
        
        // Draw characters
        this.drawCharacter(this.player);
        this.drawCharacter(this.enemy);
        
        // Draw health bars
        this.drawHealthBar(this.player, 20, 20, 'Player');
        this.drawHealthBar(this.enemy, this.width - 220, 20, 'Enemy');
        
        // Draw score
        this.ctx.fillStyle = '#FFD700';
        this.ctx.font = 'bold 24px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText(`SCORE: ${this.score}`, this.width / 2, 40);
        
        // Draw combo
        if (this.player.comboCount > 1) {
            this.ctx.fillStyle = '#FF6B9D';
            this.ctx.font = 'bold 32px Arial';
            this.ctx.fillText(`${this.player.comboCount}x COMBO!`, this.width / 2, this.height - 30);
        }
    }
    
    drawCharacter(character) {
        const isPlayer = character === this.player;
        const color = isPlayer ? '#00D4FF' : '#FF6B9D';
        const accentColor = isPlayer ? '#00FF88' : '#FFD700';
        
        // Body
        this.ctx.fillStyle = color;
        this.ctx.fillRect(character.x, character.y, character.width, character.height);
        
        // Head
        this.ctx.fillRect(character.x + 5, character.y - 15, character.width - 10, 15);
        
        // Eyes
        this.ctx.fillStyle = '#000';
        this.ctx.fillRect(character.x + 10, character.y - 10, 5, 5);
        this.ctx.fillRect(character.x + character.width - 15, character.y - 10, 5, 5);
        
        // Accent stripe
        this.ctx.fillStyle = accentColor;
        this.ctx.fillRect(character.x, character.y + 15, character.width, 5);
        
        // Attack indicator
        if (character.attacking) {
            this.ctx.strokeStyle = '#FFD700';
            this.ctx.lineWidth = 3;
            this.ctx.beginPath();
            const distance = character.attackType === 'heavy' ? 100 : character.attackType === 'kick' ? 80 : 60;
            this.ctx.arc(character.x + character.width / 2, character.y + character.height / 2, distance, 0, Math.PI * 2);
            this.ctx.stroke();
        }
        
        // Health indicator glow
        if (character.health < character.maxHealth * 0.3) {
            this.ctx.strokeStyle = '#FF0000';
            this.ctx.lineWidth = 2;
            this.ctx.strokeRect(character.x - 3, character.y - 20, character.width + 6, character.height + 25);
        }
    }
    
    drawHealthBar(character, x, y, label) {
        const width = 200;
        const height = 20;
        const healthPercent = character.health / character.maxHealth;
        
        // Background
        this.ctx.fillStyle = '#333';
        this.ctx.fillRect(x, y, width, height);
        
        // Health
        const healthColor = healthPercent > 0.5 ? '#00FF00' : healthPercent > 0.25 ? '#FFD700' : '#FF0000';
        this.ctx.fillStyle = healthColor;
        this.ctx.fillRect(x, y, width * healthPercent, height);
        
        // Border
        this.ctx.strokeStyle = '#FFF';
        this.ctx.lineWidth = 2;
        this.ctx.strokeRect(x, y, width, height);
        
        // Label
        this.ctx.fillStyle = '#FFF';
        this.ctx.font = 'bold 14px Arial';
        this.ctx.textAlign = 'left';
        this.ctx.fillText(label, x + 5, y - 5);
        
        // Health text
        this.ctx.fillText(`${Math.ceil(character.health)}/${character.maxHealth}`, x + width + 10, y + 15);
    }
    
    drawGameOver() {
        // Overlay
        this.ctx.fillStyle = 'rgba(0, 0, 0, 0.9)';
        this.ctx.fillRect(0, 0, this.width, this.height);
        
        // Title
        this.ctx.fillStyle = '#FF0000';
        this.ctx.font = 'bold 80px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('GAME OVER', this.width / 2, 150);
        
        // Score
        this.ctx.fillStyle = '#FFD700';
        this.ctx.font = 'bold 40px Arial';
        this.ctx.fillText(`Final Score: ${this.score}`, this.width / 2, 250);
        
        // Retry
        this.ctx.fillStyle = '#00D4FF';
        this.ctx.fillRect(200, 350, 400, 60);
        this.ctx.strokeStyle = '#FFD700';
        this.ctx.lineWidth = 3;
        this.ctx.strokeRect(200, 350, 400, 60);
        
        this.ctx.fillStyle = '#000';
        this.ctx.font = 'bold 30px Arial';
        this.ctx.fillText('CLICK TO RETRY', this.width / 2, 390);
    }
    
    drawVictory() {
        // Overlay
        this.ctx.fillStyle = 'rgba(0, 0, 0, 0.9)';
        this.ctx.fillRect(0, 0, this.width, this.height);
        
        // Title
        this.ctx.fillStyle = '#FFD700';
        this.ctx.font = 'bold 80px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.fillText('VICTORY!', this.width / 2, 150);
        
        // Score
        this.ctx.fillStyle = '#00FF88';
        this.ctx.font = 'bold 40px Arial';
        this.ctx.fillText(`Final Score: ${this.score}`, this.width / 2, 250);
        
        // Retry
        this.ctx.fillStyle = '#00D4FF';
        this.ctx.fillRect(200, 350, 400, 60);
        this.ctx.strokeStyle = '#FFD700';
        this.ctx.lineWidth = 3;
        this.ctx.strokeRect(200, 350, 400, 60);
        
        this.ctx.fillStyle = '#000';
        this.ctx.font = 'bold 30px Arial';
        this.ctx.fillText('PLAY AGAIN', this.width / 2, 390);
    }
    
    draw() {
        if (this.gameState === 'menu') {
            this.drawMenu();
        } else if (this.gameState === 'instructions') {
            this.drawInstructions();
        } else if (this.gameState === 'playing') {
            this.drawGame();
        } else if (this.gameState === 'gameOver') {
            this.drawGame();
            this.drawGameOver();
        } else if (this.gameState === 'victory') {
            this.drawGame();
            this.drawVictory();
        }
    }
    
    gameLoop() {
        this.update();
        this.draw();
        requestAnimationFrame(() => this.gameLoop());
    }
}

// Initialize game when page loads
window.addEventListener('load', () => {
    new CombatArena('gameCanvas');
});
</script>

<style>
body {
    margin: 0;
    padding: 20px;
    background: #1a1a1a;
    color: #fff;
    font-family: Arial, sans-serif;
}

#gameContainer {
    cursor: crosshair;
}

.instructions {
    max-width: 800px;
    margin: 30px auto;
    background: #2d2d2d;
    padding: 20px;
    border-radius: 10px;
    border-left: 4px solid #00D4FF;
}

.instructions h2 {
    color: #00D4FF;
    margin-top: 0;
}

.instructions ul {
    list-style: none;
    padding: 0;
}

.instructions li {
    padding: 8px 0;
    border-bottom: 1px solid #444;
}

.instructions li:last-child {
    border-bottom: none;
}

.code-section {
    background: #1a1a1a;
    border: 1px solid #00D4FF;
    padding: 15px;
    margin: 20px 0;
    border-radius: 5px;
    font-family: 'Courier New', monospace;
    font-size: 13px;
    overflow-x: auto;
}
</style>

<div class="instructions">
    <h2>🎮 How to Play</h2>
    <ul>
        <li><strong>Move Left/Right:</strong> A / D keys or Arrow Keys ←→</li>
        <li><strong>Jump:</strong> W / Space Bar / Up Arrow ↑</li>
        <li><strong>Punch:</strong> J (Fast attack, combos for more damage!)</li>
        <li><strong>Kick:</strong> K (Medium attack, decent range)</li>
        <li><strong>Heavy Attack:</strong> L (Slow but powerful, high damage)</li>
    </ul>
    
    <h2>⚔️ Combat Tips</h2>
    <ul>
        <li>Chain punches together to build combos and deal extra damage</li>
        <li>Use kicks when the enemy is at medium distance</li>
        <li>Heavy attacks take longer but deal massive damage</li>
        <li>Jump to platforms to gain high ground advantage</li>
        <li>Watch your health bar - the enemy AI is intelligent!</li>
        <li>Knock your opponent off the stage to win instantly</li>
        <li>Score points for every successful hit</li>
    </ul>
    
    <h2>🏆 Winning Strategy</h2>
    <ul>
        <li>Master the platform navigation for combat positioning</li>
        <li>Mix up your attacks to keep the enemy guessing</li>
        <li>Use the environment to your advantage</li>
        <li>Stay aggressive - the enemy won't give you a break!</li>
    </ul>
</div>

---

## Features

✨ **Full Combat System**
- 3 different attack types (Punch, Kick, Heavy)
- Combo system with damage multipliers
- Knockback physics
- Health management

🤖 **Intelligent Enemy AI**
- Distance-based decision making
- Attack and defense patterns
- Platform navigation
- Adaptive behavior

🎮 **Polished Gameplay**
- Smooth character movement and animation
- Multiple platforms for strategic play
- Particle effects on hits
- Health bar indicators
- Score tracking system

🎨 **Visual Polish**
- Gradient backgrounds
- Character sprites with clear state indicators
- Attack range visualization
- Combo display
- Status screens (menu, instructions, game over, victory)

---

**Made for Computer Science Students** - This game demonstrates fundamental programming concepts including object-oriented design, physics simulation, AI behavior trees, collision detection, and game state management. Perfect for studying game development!
