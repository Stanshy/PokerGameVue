<template>
  <div class="poker-app">
    <!-- 頂部控制欄 -->
    <div class="top-bar">
      <div class="game-settings">
        <div class="setting-item">
          <label>小盲:</label>
          <input v-model.number="smallBlind" type="number" min="1" :disabled="gameStarted">
        </div>
        <div class="setting-item">
          <label>大盲:</label>
          <input v-model.number="bigBlind" type="number" min="1" :disabled="gameStarted">
        </div>
      </div>
      
      <div class="game-info">
        <span class="player-count">{{ seatsData.playerCount || 0 }} 位玩家</span>
        <button 
          v-if="!gameStarted && seatsData.canStartGame" 
          @click="createGame"
          class="btn-start"
        >
          開始遊戲
        </button>
        <button 
          v-if="gameStarted && canStartNewHand" 
          @click="startNewHand"
          class="btn-start"
        >
          開始新手牌
        </button>
      </div>
    </div>

    <!-- 遊戲桌 -->
    <div class="poker-table">
      <!-- 中央區域 -->
      <div class="table-center">
        <div v-if="gameStarted" class="pot-display">
          <div class="pot-amount">${{ gameState.pot?.total || 0 }}</div>
          <div class="pot-label">底池</div>
        </div>
        
        <!-- 公共牌 -->
        <div v-if="gameStarted && gameState.board" class="community-cards">
          <div v-for="(card, idx) in gameState.board" :key="idx" class="card">
            <div class="card-rank" :class="getCardColor(card.suit)">{{ formatRank(card.rank) }}</div>
            <div class="card-suit" :class="getCardColor(card.suit)">{{ formatSuit(card.suit) }}</div>
          </div>
        </div>

        <!-- 當前階段 -->
        <div v-if="gameStarted" class="phase-display">
          {{ formatPhase(gameState.currentPhase) }}
        </div>
      </div>

      <!-- 8個座位 -->
      <div 
        v-for="seatNum in 8" 
        :key="seatNum - 1"
        :class="['seat', `seat-${seatNum - 1}`, getSeatClass(seatNum - 1)]"
      >
        <!-- 空座位 -->
        <div v-if="!getSeat(seatNum - 1) && !gameStarted" class="empty-seat" @click="openJoinModal(seatNum - 1)">
          <div class="seat-number">座位 {{ seatNum - 1 }}</div>
          <div class="join-prompt">點擊入座</div>
        </div>

        <!-- 已入座（遊戲開始前） -->
        <div v-else-if="getSeat(seatNum - 1) && !gameStarted" class="seated-player">
          <div class="player-name">{{ getSeat(seatNum - 1).playerName }}</div>
          <div class="player-chips">${{ getSeat(seatNum - 1).chips }}</div>
          <button 
            v-if="getSeat(seatNum - 1).playerName === myPlayerName"
            @click="leaveSeat(seatNum - 1)"
            class="btn-leave"
          >
            離座
          </button>
        </div>

        <!-- 遊戲中的玩家 -->
        <div v-else-if="getPlayer(seatNum - 1) && gameStarted" class="game-player">
          <div class="player-info">
            <div class="player-name">{{ getPlayer(seatNum - 1).name }}</div>
            <div class="player-chips">${{ getPlayer(seatNum - 1).chips }}</div>
            <div v-if="getPlayer(seatNum - 1).currentBet > 0" class="current-bet">
              下注: ${{ getPlayer(seatNum - 1).currentBet }}
            </div>
          </div>

          <!-- 位置標記 -->
          <div class="badges">
            <span v-if="getPlayer(seatNum - 1).button" class="badge dealer">D</span>
            <span v-if="getPlayer(seatNum - 1).smallBlind" class="badge sb">SB</span>
            <span v-if="getPlayer(seatNum - 1).bigBlind" class="badge bb">BB</span>
          </div>

          <!-- 手牌 -->
          <div class="player-cards">
            <!-- 自己的手牌 -->
            <div v-if="getPlayer(seatNum - 1).name === myPlayerName && myHand" class="hand-cards">
              <div v-for="(card, idx) in myHand" :key="idx" class="card">
                <div class="card-rank" :class="getCardColor(card.suit)">{{ formatRank(card.rank) }}</div>
                <div class="card-suit" :class="getCardColor(card.suit)">{{ formatSuit(card.suit) }}</div>
              </div>
            </div>
            <!-- 攤牌時顯示所有人手牌 -->
            <div v-else-if="gameState.currentPhase === 'SHOWDOWN' && gameState.showdownHands && gameState.showdownHands[getPlayer(seatNum - 1).name]" class="hand-cards">
              <div v-for="(card, idx) in gameState.showdownHands[getPlayer(seatNum - 1).name]" :key="idx" class="card">
                <div class="card-rank" :class="getCardColor(card.suit)">{{ formatRank(card.rank) }}</div>
                <div class="card-suit" :class="getCardColor(card.suit)">{{ formatSuit(card.suit) }}</div>
              </div>
            </div>
            <!-- 其他玩家的牌背 -->
            <div v-else-if="getPlayer(seatNum - 1).status !== 'FOLDED'" class="card-back">
              <div class="card small">🂠</div>
              <div class="card small">🂠</div>
            </div>
          </div>

          <!-- 動作按鈕（輪到自己） -->
          <div v-if="getPlayer(seatNum - 1).name === myPlayerName && gameState.currentPlayer === myPlayerName" class="action-panel">
            <button @click="playerAction('FOLD')" class="btn-action fold">棄牌</button>
            <button @click="playerAction('CHECK')" class="btn-action check">過牌</button>
            <button @click="playerAction('CALL', gameState.currentHighestBet)" class="btn-action call">
              跟注 ${{ gameState.currentHighestBet }}
            </button>
            <div class="bet-group">
              <input v-model.number="betAmount" type="number" :min="gameState.currentHighestBet" class="bet-input">
              <button @click="playerAction('RAISE', betAmount)" class="btn-action raise">加注</button>
            </div>
            <button @click="playerAction('ALL_IN', getPlayer(seatNum - 1).chips)" class="btn-action allin">All-In</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 入座表單 -->
    <div v-if="showJoinModal" class="modal-overlay" @click.self="showJoinModal = false">
      <div class="modal">
        <h3>加入座位 {{ selectedSeat }}</h3>
        <div class="form-group">
          <label>玩家名稱</label>
          <input v-model="joinForm.playerName" type="text" placeholder="輸入名稱" maxlength="20">
        </div>
        <div class="form-group">
          <label>籌碼</label>
          <input v-model.number="joinForm.chips" type="number" min="100" max="100000">
        </div>
        <div class="modal-actions">
          <button @click="showJoinModal = false" class="btn-cancel">取消</button>
          <button @click="joinSeat" class="btn-confirm">確認</button>
        </div>
      </div>
    </div>

    <!-- 訊息提示 -->
    <div v-if="message" class="message" :class="messageType">{{ message }}</div>
  </div>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from 'vue'

export default {
  setup() {
    const API_BASE = 'http://localhost:8080/api/game'
    const WS_URL = 'ws://localhost:8080/ws/game'

    // 狀態
    const seats = ref({})
    const seatsData = ref({ playerCount: 0, canStartGame: false })
    const gameState = ref({})
    const gameStarted = ref(false)
    const myPlayerName = ref(sessionStorage.getItem('myPlayerName') || '')
    const myToken = ref(sessionStorage.getItem('myToken') || '')
    const myHand = ref(null)
    const smallBlind = ref(10)
    const bigBlind = ref(20)
    const betAmount = ref(0)
    const showJoinModal = ref(false)
    const selectedSeat = ref(0)
    const joinForm = ref({ playerName: '', chips: 1000 })
    const message = ref('')
    const messageType = ref('info')
    const ws = ref(null)
    const seatMapping = ref({}) // 儲存座位與玩家的對應關係

    // 計算屬性
    const canStartNewHand = computed(() => {
      return gameState.value.currentPhase === 'SHOWDOWN'
    })

    // WebSocket
    const connectWebSocket = () => {
      ws.value = new WebSocket(WS_URL)
      
      ws.value.onopen = () => console.log('WebSocket 連接成功')
      
      ws.value.onmessage = (event) => {
        const msg = JSON.parse(event.data)
        
        if (msg.type === 'SEATS_UPDATE') {
          seats.value = msg.data.seats || {}
          seatsData.value = msg.data
        }
        
        if (msg.type === 'GAME_STATE_UPDATE') {
          gameState.value = msg.data
          gameStarted.value = true
          if (myToken.value && myPlayerName.value) {
            loadMyHand()
          }
        }
      }
      
      ws.value.onerror = (error) => console.error('WebSocket 錯誤:', error)
      ws.value.onclose = () => console.log('WebSocket 已關閉')
    }

    // API 方法
    const loadSeats = async () => {
      const res = await fetch(`${API_BASE}/seats`)
      const data = await res.json()
      if (data.success) {
        seats.value = data.data.seats || {}
        seatsData.value = data.data
      }
    }

    const loadGameState = async () => {
      try {
        const res = await fetch(`${API_BASE}/state`)
        const data = await res.json()
        if (data.success) {
          gameState.value = data.data
          gameStarted.value = true
        }
      } catch (err) {
        console.error('載入遊戲狀態失敗:', err)
      }
    }

    const loadMyHand = async () => {
      if (!myToken.value) return
      try {
        const res = await fetch(`${API_BASE}/hand`, {
          headers: { 'Authorization': `Bearer ${myToken.value}` }
        })
        const data = await res.json()
        if (data.success) {
          myHand.value = data.data.cards
        }
      } catch (err) {
        console.error('載入手牌失敗:', err)
      }
    }

    const openJoinModal = (seatNum) => {
      selectedSeat.value = seatNum
      showJoinModal.value = true
    }

    const joinSeat = async () => {
      const res = await fetch(`${API_BASE}/join-seat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          seatNumber: selectedSeat.value,
          playerName: joinForm.value.playerName,
          chips: joinForm.value.chips
        })
      })
      const data = await res.json()
      if (data.success) {
        myPlayerName.value = joinForm.value.playerName
        myToken.value = data.data.token
        sessionStorage.setItem('myPlayerName', myPlayerName.value)
        sessionStorage.setItem('myToken', myToken.value)
        showJoinModal.value = false
        showMessage('入座成功', 'success')
      } else {
        showMessage(data.message || '入座失敗', 'error')
      }
    }

    const leaveSeat = async (seatNum) => {
      const res = await fetch(`${API_BASE}/leave-seat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ seatNumber: seatNum })
      })
      const data = await res.json()
      if (data.success) {
        sessionStorage.removeItem('myPlayerName')
        sessionStorage.removeItem('myToken')
        myPlayerName.value = ''
        myToken.value = ''
        showMessage('離座成功', 'success')
      }
    }

    const createGame = async () => {
  // 創建遊戲前先保存座位映射
  seatMapping.value = {}
  Object.entries(seats.value).forEach(([num, seat]) => {
    seatMapping.value[num] = seat.playerName
  })
  
  const res = await fetch(`${API_BASE}/create`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ smallBlind: smallBlind.value, bigBlind: bigBlind.value })
  })
  const data = await res.json()
  if (data.success) {
    showMessage('遊戲創建成功', 'success')
    gameStarted.value = true
  }
}

    const startNewHand = async () => {
      const res = await fetch(`${API_BASE}/start-hand`, { method: 'POST' })
      const data = await res.json()
      if (data.success) {
        showMessage('新手牌開始', 'success')
        myHand.value = null
      }
    }

    const playerAction = async (actionType, amount = 0) => {
      const res = await fetch(`${API_BASE}/action`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${myToken.value}`
        },
        body: JSON.stringify({ action: actionType, amount })
      })
      const data = await res.json()
      if (data.success) {
        showMessage(`${actionType} 成功`, 'success')
      } else {
        showMessage(data.message || '動作失敗', 'error')
      }
    }

    // 工具方法
    const getSeat = (seatNum) => seats.value[seatNum]
    
    const getPlayer = (seatNum) => {
  if (!gameState.value.players) return null
  const playerName = seatMapping.value[seatNum]
  if (!playerName) return null
  return gameState.value.players.find(p => p.name === playerName)
}

    const getSeatClass = (seatNum) => {
      const player = getPlayer(seatNum)
      if (!player) return ''
      return {
        'current-turn': player.name === gameState.value.currentPlayer,
        'folded': player.status === 'FOLDED'
      }
    }

    const formatRank = (rank) => {
      const ranks = { ACE: 'A', KING: 'K', QUEEN: 'Q', JACK: 'J', TEN: '10',
        NINE: '9', EIGHT: '8', SEVEN: '7', SIX: '6', FIVE: '5', FOUR: '4', THREE: '3', TWO: '2' }
      return ranks[rank] || rank
    }

    const formatSuit = (suit) => {
      const suits = { SPADES: '♠', HEARTS: '♥', DIAMONDS: '♦', CLUBS: '♣' }
      return suits[suit] || suit
    }

    const getCardColor = (suit) => {
      return (suit === 'HEARTS' || suit === 'DIAMONDS') ? 'red' : 'black'
    }

    const formatPhase = (phase) => {
      const phases = { PREFLOP: '翻牌前', FLOP: '翻牌圈', TURN: '轉牌圈', RIVER: '河牌圈', SHOWDOWN: '攤牌' }
      return phases[phase] || phase
    }

    const showMessage = (msg, type = 'info') => {
      message.value = msg
      messageType.value = type
      setTimeout(() => { message.value = '' }, 3000)
    }

    // 生命週期
    onMounted(() => {
      loadSeats()
      loadGameState()
      connectWebSocket()
    })

    onUnmounted(() => {
      if (ws.value) ws.value.close()
    })

    return {
      seats, seatsData, gameState, gameStarted, myPlayerName, myHand,
      smallBlind, bigBlind, betAmount, showJoinModal, selectedSeat, joinForm,
      message, messageType, canStartNewHand,
      openJoinModal, joinSeat, leaveSeat, createGame, startNewHand, playerAction,
      getSeat, getPlayer, getSeatClass, formatRank, formatSuit, getCardColor, formatPhase
    }
  }
}
</script>

<style scoped>
.poker-app {
  min-height: 100vh;
  background: linear-gradient(135deg, #0a0e27 0%, #1a2332 100%);
  color: white;
  font-family: 'Arial', sans-serif;
}

.top-bar {
  background: rgba(0, 0, 0, 0.8);
  padding: 15px 30px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.game-settings {
  display: flex;
  gap: 20px;
}

.setting-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.setting-item input {
  width: 80px;
  padding: 5px 10px;
  border-radius: 4px;
  border: none;
}

.game-info {
  display: flex;
  align-items: center;
  gap: 15px;
}

.player-count {
  font-size: 16px;
  color: #ffd700;
}

.btn-start {
  padding: 10px 25px;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
}

.btn-start:hover {
  background: #218838;
}

.poker-table {
  position: relative;
  width: 900px;
  height: 600px;
  margin: 40px auto;
  background: radial-gradient(ellipse at center, #0d4d0d 0%, #062806 100%);
  border: 12px solid #8b4513;
  border-radius: 50%;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.8);
}

.table-center {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  min-width: 350px;
}

.pot-display {
  margin-bottom: 20px;
}

.pot-amount {
  font-size: 32px;
  font-weight: bold;
  color: #ffd700;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
}

.pot-label {
  color: #ccc;
  font-size: 14px;
}

.community-cards {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin: 20px 0;
}

.card {
  width: 70px;
  height: 100px;
  background: white;
  border: 2px solid #333;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  padding: 8px 4px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.card.small {
  width: 50px;
  height: 70px;
  font-size: 12px;
}

.card-rank {
  font-size: 24px;
  font-weight: bold;
}

.card-suit {
  font-size: 28px;
}

.red { color: #dc3545; }
.black { color: #000; }

.phase-display {
  font-size: 18px;
  color: #28a745;
  font-weight: bold;
  margin-top: 15px;
}

.seat {
  position: absolute;
  width: 200px;
  min-height: 120px;
  background: rgba(0, 0, 0, 0.9);
  border: 2px solid #444;
  border-radius: 12px;
  padding: 15px;
  transform: translate(-50%, -50%);
}

.seat.current-turn {
  border-color: #28a745;
  box-shadow: 0 0 20px rgba(40, 167, 69, 0.8);
}

.seat.folded {
  opacity: 0.4;
}

.seat-0 { bottom: -80px; left: 50%; }
.seat-1 { bottom: 20%; right: -80px; }
.seat-2 { top: 50%; right: -80px; transform: translate(0, -50%); }
.seat-3 { top: 20%; right: -80px; }
.seat-4 { top: -80px; right: 30%; }
.seat-5 { top: -80px; left: 30%; }
.seat-6 { top: 20%; left: -80px; }
.seat-7 { top: 50%; left: -80px; transform: translate(0, -50%); }

.empty-seat {
  text-align: center;
  cursor: pointer;
  padding: 20px;
}

.empty-seat:hover {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
}

.seat-number {
  font-size: 14px;
  color: #888;
  margin-bottom: 8px;
}

.join-prompt {
  color: #28a745;
  font-size: 16px;
}

.seated-player, .game-player {
  text-align: center;
}

.player-name {
  font-weight: bold;
  font-size: 16px;
  margin-bottom: 5px;
}

.player-chips {
  color: #ffd700;
  font-size: 14px;
}

.current-bet {
  color: #28a745;
  font-size: 12px;
  margin-top: 5px;
}

.badges {
  margin: 8px 0;
}

.badge {
  display: inline-block;
  padding: 2px 6px;
  border-radius: 50%;
  font-size: 10px;
  margin: 0 2px;
  color: white;
}

.badge.dealer { background: #007bff; }
.badge.sb { background: #28a745; }
.badge.bb { background: #dc3545; }

.btn-leave {
  padding: 5px 15px;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  margin-top: 8px;
}

.player-cards {
  margin: 10px 0;
}

.hand-cards {
  display: flex;
  justify-content: center;
  gap: 8px;
}

.card-back {
  display: flex;
  gap: 5px;
  justify-content: center;
}

.action-panel {
  margin-top: 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  justify-content: center;
}

.btn-action {
  padding: 6px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  color: white;
  font-weight: bold;
}

.btn-action.fold { background: #6c757d; }
.btn-action.check { background: #17a2b8; }
.btn-action.call { background: #28a745; }
.btn-action.raise { background: #fd7e14; }
.btn-action.allin { background: #dc3545; }

.bet-group {
  display: flex;
  gap: 5px;
}

.bet-input {
  width: 60px;
  padding: 4px;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  color: black;
  padding: 30px;
  border-radius: 10px;
  width: 400px;
}

.modal h3 {
  margin-top: 0;
}

.form-group {
  margin: 15px 0;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.form-group input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

.btn-cancel, .btn-confirm {
  padding: 8px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-cancel {
  background: #6c757d;
  color: white;
}

.btn-confirm {
  background: #007bff;
  color: white;
}

.message {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 15px 25px;
  border-radius: 8px;
  z-index: 2000;
  font-weight: bold;
}

.message.success {
  background: #d4edda;
  color: #155724;
}

.message.error {
  background: #f8d7da;
  color: #721c24;
}

.message.info {
  background: #d1ecf1;
  color: #0c5460;
}
</style>