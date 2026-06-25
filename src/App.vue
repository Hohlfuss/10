<script setup lang="ts">
import { ref, onUnmounted, onMounted } from "vue";
import HealthBar from "./components/HealthBar.vue";

interface Player {
  health: number;
  maxHealth: number;
  damage: number;
  speed: number;
  healAmount: number;
  healSpeed: number;
  gold: number;
  regentimer: ReturnType<typeof setInterval> | undefined;
  playerTimer: ReturnType<typeof setInterval> | undefined;
  currentScreen: string;
  isCombatActive: boolean;
  currentDifficulty: number;
  oakLogs: number;
  ironOre: number;
  ownsIronHatchet: boolean;
  ownsIronPickaxe: boolean;
  ownsIronSword: boolean;
}

interface Enemy {
  nameList: string[];
  name: string;
  health: number;
  maxHealth: number;
  damage: number;
  speed: number;
  enemyTimer: ReturnType<typeof setInterval> | undefined;
}

interface Woodcutting {
  woodPerFell: number;
  isChopping: boolean;
  woodcuttingTimer: ReturnType<typeof setInterval> | undefined;
  chopSpeed: number;
  oakProgress: number;
}

interface Mining {
  orePerRock: number;
  isMining: boolean;
  miningTimer: ReturnType<typeof setInterval> | undefined;
  miningSpeed: number;
  ironProgress: number;
}

const player = ref<Player>({
  health: 100,
  maxHealth: 100,
  damage: 10,
  speed: 1_000,
  healAmount: 5,
  healSpeed: 2_000,
  gold: 0,
  regentimer: undefined,
  playerTimer: undefined,
  currentScreen: "combat",
  isCombatActive: false,
  currentDifficulty: 1,
  oakLogs: 0,
  ironOre: 0,
  ownsIronHatchet: false,
  ownsIronPickaxe: false,
  ownsIronSword: false
})

const enemy = ref<Enemy>({
  nameList: ["Slime", "Goblin", "Rabid Bat", "Skeleton", "Forest Wolf", "Orc Untamed"],
  name: "Goblin",
  health: 100,
  maxHealth: 100,
  damage: 8,
  speed: 1_200,
  enemyTimer: undefined
})

const woodcutting = ref<Woodcutting>({
  woodPerFell: 1,
  isChopping: false,
  woodcuttingTimer: undefined,
  chopSpeed: 10_000,
  oakProgress: 0
})

const mining = ref<Mining>({
  orePerRock: 1,
  isMining: false,
  miningTimer: undefined,
  miningSpeed: 10_000,
  ironProgress: 0
})

const getRandomInt = (min: number, max: number) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

const goToMenu = () => {
  player.value.currentScreen = "menu";
};

const goToCombat = () => {
  player.value.currentScreen = "combat";
};

const goToWoodcutting = () => {
  player.value.currentScreen = "woodcutting";
};

const goToMining = () => {
  player.value.currentScreen = "mining";
}

const spawnNewEnemy = () => {
  enemy.value.name = enemy.value.nameList[Math.floor(Math.random() * enemy.value.nameList.length)];

  let randomHP = getRandomInt(60, 160);
  randomHP *= player.value.currentDifficulty;
  enemy.value.health = randomHP;
  enemy.value.maxHealth = randomHP;

  let enemyDamage = getRandomInt(5, 18);
  enemyDamage *= player.value.currentDifficulty;
  enemy.value.damage = enemyDamage;
  let enemySpeed = getRandomInt(800, 2_200);
  const speedScaleFactor = 0.95;
  let speedMultiplier = Math.pow(speedScaleFactor, player.value.currentDifficulty - 1);
  let calculatedSpeed = enemySpeed * speedMultiplier;
  enemy.value.speed = Math.floor(Math.max(200, calculatedSpeed));
};

const startPlayerLoop = () => {
  player.value.playerTimer = setInterval(() => {
    if (!player.value.isCombatActive) return;
  
    enemy.value.health -= player.value.damage;

    if (enemy.value.health <= 0) {
      handleEnemyDefeat();
    }
  }, player.value.speed);
};

const startEnemyLoop = () => {
  if (enemy.value.enemyTimer) clearInterval(enemy.value.enemyTimer);

  enemy.value.enemyTimer = setInterval(() => {
    if (!player.value.isCombatActive || enemy.value.health <= 0) return;

    player.value.health -= enemy.value.damage;

    if (player.value.health <= 0) {
      player.value.health = 0;
      stopCombat();
    }
  }, enemy.value.speed);
};

const handleEnemyDefeat = () => {
  if (enemy.value.enemyTimer) clearInterval(enemy.value.enemyTimer);
  enemy.value.health = 0;
  let goldReward = getRandomInt(1, 10);
  goldReward *= player.value.currentDifficulty;
  player.value.gold += goldReward;

  spawnNewEnemy();

  startEnemyLoop();
};

const startCombat = () => {
  if (player.value.isCombatActive || player.value.health <= 0) return;
  player.value.isCombatActive = true;
  if (player.value.regentimer) clearInterval(player.value.regentimer);
  
  if (enemy.value.health <= 0) spawnNewEnemy();

  startPlayerLoop();
  startEnemyLoop();
};

const stopCombat = () => {
  player.value.isCombatActive = false;
  if (player.value.playerTimer) clearInterval(player.value.playerTimer);
  if (enemy.value.enemyTimer) clearInterval(enemy.value.enemyTimer);
  startRegen();
};

const startRegen = () => {
  if (player.value.regentimer) clearInterval(player.value.regentimer);

  player.value.regentimer = setInterval(() => {
    if (player.value.health < player.value.maxHealth) {
      player.value.health += player.value.healAmount;

      if (player.value.health > player.value.maxHealth) {
        player.value.health = player.value.maxHealth;
        clearInterval(player.value.regentimer);
      }
    } else {
      clearInterval(player.value.regentimer);
    }
  }, player.value.healSpeed);
};

const changeDifficulty = (amount: number) => {
  if (player.value.isCombatActive) return;

  const newDifficulty = player.value.currentDifficulty + amount;
  if (newDifficulty >= 1) {
    player.value.currentDifficulty = newDifficulty;

    spawnNewEnemy();
  }
};

const toggleChopping = () => {
  if (woodcutting.value.isChopping) {
    woodcutting.value.isChopping = false;
    if (woodcutting.value.woodcuttingTimer) clearInterval(woodcutting.value.woodcuttingTimer);
    woodcutting.value.oakProgress = 0;
  } else {
    woodcutting.value.isChopping = true;
    const tickRate = 50;

    woodcutting.value.woodcuttingTimer = setInterval(() => {
      woodcutting.value.oakProgress += (tickRate / woodcutting.value.chopSpeed) * 100;

      if (woodcutting.value.oakProgress >= 100) {
        woodcutting.value.oakProgress = 0;
        player.value.oakLogs += woodcutting.value.woodPerFell;
      }
    }, tickRate);
  }
};

const toggleMining = () => {
  if (mining.value.isMining) {
    mining.value.isMining = false;
    if (mining.value.miningTimer) clearInterval(mining.value.miningTimer);
    mining.value.ironProgress = 0;
  } else {
    mining.value.isMining = true;
    const tickRate = 50;

    mining.value.miningTimer = setInterval(() => {
      mining.value.ironProgress += (tickRate / mining.value.miningSpeed) * 100;

      if (mining.value.ironProgress >= 100) {
        mining.value.ironProgress = 0;
        player.value.ironOre += mining.value.orePerRock;
      }
    }, tickRate);
  }
}

const saveGame = () => {
  const gameData = {
    player: player.value,
    enemy: enemy.value,
    woodcutting: woodcutting.value,
    mining: mining.value
  };

  localStorage.setItem("gameSave", JSON.stringify(gameData));
  console.log("Game Saved!");
};

const loadGame = () => {
  const savedData = localStorage.getItem("gameSave");
  if (savedData) {
    const gameData = JSON.parse(savedData);
    player.value = { ...player.value, ...gameData.player };
    enemy.value = { ...enemy.value, ...gameData.enemy };
    woodcutting.value = { ...woodcutting.value, ...gameData.woodcutting };
    mining.value = { ...mining.value, ...gameData.mining };
    if (!player.value.currentDifficulty) player.value.currentDifficulty = 1;
    player.value.isCombatActive = false;
    woodcutting.value.isChopping = false;
    mining.value.isMining = false;
    console.log("Game loaded!");
  }
};

let autoSaveTimer: ReturnType<typeof setInterval> | undefined;

onUnmounted(() => {
  stopCombat();
  clearInterval(player.value.regentimer);
  if (woodcutting.value.woodcuttingTimer) clearInterval(woodcutting.value.woodcuttingTimer);
  if (mining.value.miningTimer) clearInterval(mining.value.miningTimer);

  if (autoSaveTimer) clearInterval(autoSaveTimer);
  
  saveGame();
});

onMounted(() => {
  loadGame();

  autoSaveTimer = setInterval(() => {
    saveGame();
  }, 10_000);
});
</script>

<template>
  <div v-if="player.currentScreen === 'menu'" class="menu-screen">
    
    <div class="currency-hud">
      <span>💰 {{ player.gold }}</span>
      <span>🪵 {{ player.oakLogs }}</span>
      <span>🪨 {{ player.ironOre }}</span>
    </div>
    
    <div class="menu-content">
      <h1>Idle Battler</h1>
      
      <div class="menu-buttons-container">
        <button class="menu-btn play-btn" @click="goToCombat">Back to Combat</button>
        <button class="menu-btn" @click="goToWoodcutting">Woodcutting</button>
        <button class="menu-btn" @click="goToMining">Mining</button>
        <button class="menu-btn">Settings (Coming Soon)</button>
        <button class="menu-btn">Upgrades (Coming Soon)</button>
        <button class="menu-btn" @click="saveGame">Manual Save</button>
      </div>
    </div>
  </div>

  <div v-else-if="player.currentScreen === 'combat'" class="game-container">

    <div class="currency-hud">
      <span>💰 {{ player.gold }}</span>
      <span>🪵 {{ player.oakLogs }}</span>
      <span>🪨 {{ player.ironOre }}</span>
    </div>
    
    <button class="nav-to-menu-btn" @click="goToMenu">
      ☰ Main Menu
    </button>

    <header class="player-section">
      <div class="info-row">
        <div class="name-label">Player</div>
        <div class="combat-stats player-stats">
          ⚔️ {{ player.damage }} | ⚡ {{ (player.speed / 1000).toFixed(1) }}s
        </div>
        <div class="spacer"></div> 
      </div>
      <HealthBar :current="player.health" :max="player.maxHealth" color="#42b883" />
    </header>

    <main class="action-section">
      <button 
        v-if="!player.isCombatActive" 
        @click="startCombat" 
        class="game-btn attack"
        :disabled="player.health <= 0"
      >
        Start Combat
      </button>

      <button 
        v-else 
        @click="stopCombat" 
        class="game-btn stop"
      >
        Stop Combat
      </button>
    </main>

    <footer class="enemy-section">
      <div class="compact-diff-selector">
        <button 
          class="tiny-btn" 
          @click="changeDifficulty(-1)" 
          :disabled="player.currentDifficulty <= 1 || player.isCombatActive"
        >
          &lt;
        </button>
        
        <span class="tiny-label">Enemy Tier {{ player.currentDifficulty }}</span>
        
        <button 
          class="tiny-btn" 
          @click="changeDifficulty(1)" 
          :disabled="player.isCombatActive"
        >
          &gt;
        </button>
      </div>

      <div class="info-row">
        <div class="name-label">{{ enemy.name }}</div>
        <div class="combat-stats enemy-stats">
          ⚔️ {{ enemy.damage }} | ⚡ {{ (enemy.speed / 1000).toFixed(1) }}s
        </div>
        <div class="spacer"></div>
      </div>
      <HealthBar :current="enemy.health" :max="enemy.maxHealth" color="#ff4a4a" />
    </footer>

  </div>

  <div v-else-if="player.currentScreen === 'woodcutting'" class="game-container">
    
    <div class="currency-hud">
      <span>💰 {{ player.gold }}</span>
      <span>🪵 {{ player.oakLogs }}</span>
      <span>🪨 {{ player.ironOre }}</span>
    </div>
    
    <button class="nav-to-menu-btn" @click="goToMenu">
      ☰ Main Menu
    </button>

    <header class="player-section" style="text-align: center;">
      <h2 style="margin: 0; color: #8b5a2b;">Oak Tree</h2>
      <p style="margin: 5px 0 0 0; color: #aaa;">Yield: {{ woodcutting.woodPerFell }} OakLogs / tree</p>
    </header>

    <main class="action-section">
      <div class="wood-progress-container" @click="toggleChopping">
        <div class="wood-progress-fill" :style="{ width: woodcutting.oakProgress + '%' }"></div>
        <div class="wood-progress-text">
          <span v-if="!woodcutting.isChopping">Click to Start Chopping</span>
          <span v-else>Chopping Oak... ({{ Math.floor(woodcutting.oakProgress) }}%)</span>
        </div>
      </div>
    </main>
    
    <footer class="enemy-section"></footer>
  </div>

  <div v-else-if="player.currentScreen === 'mining'" class="game-container">
    
    <div class="currency-hud">
      <span>💰 {{ player.gold }}</span>
      <span>🪵 {{ player.oakLogs }}</span>
      <span>🪨 {{ player.ironOre }}</span>
    </div>
    
    <button class="nav-to-menu-btn" @click="goToMenu">
      ☰ Main Menu
    </button>

    <header class="player-section" style="text-align: center;">
      <h2 style="margin: 0; color: #a9a9a9;">Iron ore</h2>
      <p style="margin: 5px 0 0 0; color: #aaa;">Yield: {{ mining.orePerRock }} Iron ore / rock</p>
    </header>

    <main class="action-section">
      
      <div class="mining-progress-container" @click="toggleMining">
        <div class="mining-progress-fill" :style="{ width: mining.ironProgress + '%' }"></div>
        <div class="mining-progress-text">
          <span v-if="!mining.isMining">Click to Start Mining</span>
          <span v-else>Mining Iron... ({{ Math.floor(mining.ironProgress) }}%)</span>
        </div>
      </div>

    </main>
    
    <footer class="enemy-section"></footer>
  </div>
  
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100vw;
  background-color: rgb(66, 66, 66);
  color: #ffffff;
  font-family: "Arial", sans-serif;
  overflow: hidden;
  box-sizing: border-box;
}

.player-section, .enemy-section {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.player-section {
  background-color: #222;
  border-bottom: 2px solid #333;
}

.enemy-section {
  background-color: #222;
  border-top: 2px solid #333;
  margin-top: auto;
}

.action-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 16px;
  padding: 20px;
}

.game-btn {
  padding: 16px 32px;
  font-size: 1.1rem;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  width: 80%;
  max-width: 300px;
}

.attack { background-color: #ff4a4a; color: white; }
.hurt { background-color: #ffd200; color: #1a1a1a; }

.attack { background-color: #ff4a4a; color: white; }
.stop { background-color: #555555; color: white; border: 2px solid #888; }
/* Adding a visual cue for when the button is disabled */
.game-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.label-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
}

.stats-bug {
  font-size: 0.85rem;
  color: #aaa;
  font-family: monospace;
}

/* --- SCREEN SWAP STYLES --- */

.nav-to-menu-btn {
  position: absolute;
  top: 5px;
  right: 5px;
  padding: 4px 6px;
  background-color: #444;
  color: white;
  border: 2px solid #666;
  border-radius: 6px;
  font-weight: bold;
  font-size: 0.8rem;
  cursor: pointer;
}

/* Make the menu screen act exactly like the game container (full screen) */
.menu-screen {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  width: 100vw;
  background-color: #1a1a1a;
  color: #ffffff;
}

.menu-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 30px;
  width: 80%;
  max-width: 400px;
}

.menu-content h1 {
  font-size: 2.5rem;
  margin: 0;
  text-transform: uppercase;
  letter-spacing: 3px;
  color: #42b883; /* Vue Green */
}

.menu-buttons-container {
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: 80%;
}

.menu-btn {
  padding: 8px;
  background-color: #333;
  color: #ccc;
  border: 2px solid #555;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}

.menu-btn:hover {
  background-color: #444;
}

.play-btn {
  background-color: #42b883;
  color: #1a1a1a;
  border-color: #42b883;
}

.play-btn:hover {
  background-color: #33a06f;
}

/* --- TOP BAR LAYOUT --- */
.info-row {
  display: grid;
  /* Creates 3 columns: Left (1 fraction), Center (auto-sized to fit text), Right (1 fraction) */
  grid-template-columns: 1fr auto 1fr; 
  align-items: center;
  margin-bottom: 6px;
}

.name-label {
  font-size: 0.8rem;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 1px;
  text-align: left; /* Pins the name to the left */
}

/* The spacer takes up the right side so the center stays perfectly centered */
.spacer {
  width: 100%;
}

/* --- MIDDLE STAT BADGES --- */
.combat-stats {
  background-color: #2a2a2a;
  padding: 4px 12px; /* Made slightly smaller so it fits cleanly next to the name */
  border-radius: 20px;
  font-family: monospace;
  font-size: 0.9rem;
  font-weight: bold;
  border: 1px solid #555;
  box-shadow: 0 2px 4px rgba(0,0,0,0.3);
  text-align: center;
  white-space: nowrap; /* Prevents the stats from breaking onto two lines on very narrow phones */
}

.player-stats {
  color: #42b883;
  border-color: #42b883;
}

.enemy-stats {
  color: #ff4a4a;
  border-color: #ff4a4a;
}

.currency-hud {
  position: absolute;
  top: 2px;
  left: 6px;
  padding: 4px 6px;
  background-color: #222;
  color: #ffd700; /* Classic RPG Gold */
  border: 2px solid #daa520;
  border-radius: 6px;
  font-weight: bold;
  font-size: 0.7rem;
  z-index: 10;
  box-shadow: 0 2px 4px rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  gap: 2px;
}

/* --- COMPACT DIFFICULTY SELECTOR --- */
.compact-diff-selector {
  display: flex;
  justify-content: flex-start; /* Aligns it nicely to the left over the name */
  align-items: center;
  gap: 12px;
  margin-bottom: 2px; /* Keeps it tight against the info row */
}

.tiny-label {
  font-size: 0.75rem; /* Very small text */
  color: #888; /* Faded grey so it isn't distracting */
  text-transform: uppercase;
  letter-spacing: 1px;
  font-weight: bold;
}

.tiny-btn {
  background-color: transparent;
  color: #888;
  border: 1px solid #444;
  border-radius: 4px;
  padding: 2px 8px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.tiny-btn:hover:not(:disabled) {
  background-color: #444;
  color: white;
  border-color: #666;
}

.tiny-btn:disabled {
  opacity: 0.2;
  cursor: not-allowed;
}

/* --- WOODCUTTING PROGRESS BAR --- */
.wood-progress-container {
  width: 90%;
  max-width: 450px;
  height: 80px; /* Huge and easy to tap */
  background-color: #222;
  border: 4px solid #555;
  border-radius: 12px;
  position: relative;
  overflow: hidden;
  cursor: pointer;
  box-shadow: 0 8px 16px rgba(0,0,0,0.6);
  transition: transform 0.1s;
}

/* Add a tiny "click" effect when tapping the bar */
.wood-progress-container:active {
  transform: scale(0.98);
}

.wood-progress-fill {
  height: 100%;
  background-color: #2e8b57; /* Nice earthy Sea Green */
  width: 0%; /* This gets overwritten by the Vue :style tag */
  transition: width 0.05s linear; /* Makes the 50ms ticks look buttery smooth */
}

.wood-progress-text {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1.3rem;
  font-weight: bold;
  color: white;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
  pointer-events: none; /* Ensures clicking the text still clicks the container behind it */
}

#player {
  margin-top: 50%;
}

/* --- MINING PROGRESS BAR --- */
.mining-progress-container {
  width: 90%;
  max-width: 450px;
  height: 80px; 
  background-color: #1c1c1c;
  border: 4px solid #333;
  border-radius: 12px;
  position: relative;
  overflow: hidden;
  cursor: pointer;
  box-shadow: 0 8px 16px rgba(0,0,0,0.6);
  transition: transform 0.1s;
}

.mining-progress-container:active {
  transform: scale(0.98);
}

.mining-progress-fill {
  height: 100%;
  background-color: #696969; /* Dim Gray for stone */
  width: 0%; 
  transition: width 0.05s linear; 
}

.mining-progress-text {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1.3rem;
  font-weight: bold;
  color: white;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
  pointer-events: none; 
}
</style>
