<template>
  <div class="poker-app">
    <!-- 大廳階段：座位選擇 -->
    <div v-if="gamePhase === 'lobby'" class="game-setup">
      <h1>德州撲克遊戲大廳</h1>
      
      <!-- 遊戲參數設定 -->
      <div class="game-config">
        <h2>遊戲設定</h2>
        <div class="config-row">
          <label>小盲注:</label>
          <input v-model.number="gameConfig.smallBlind" type="number" min="1" placeholder="5">
        </div>
        <div class="config-row">
          <label>大盲注:</label>
          <input v-model.number="gameConfig.bigBlind" type="number" min="1" placeholder="10">
        </div>
      </div>

      <!-- 座位區域 -->
      <div class="players-setup">
        <h2>選擇座位 ({{ playerCount }}/8人)</h2>
        
        <!-- 顯示撲克桌和座位 -->
        <div class="lobby-seats-container">
          <div v-for="seatNum in 8" :key="seatNum" :class="['lobby-seat-wrapper', `position-${seatNum - 1}`]">
            <div v-if="seats[seatNum - 1]" class="occupied-seat">
              <div class="player-info-lobby">
                <div class="player-name">{{ seats[seatNum - 1].playerName }}</div>
                <div class="player-chips">💰 {{ seats[seatNum - 1].chips }}</div>
              </div>
              <button v-if="seats[seatNum - 1].playerName === myPlayerName" 
                      @click="leaveSeat(seatNum - 1)" 
                      class="remove-btn">離開</button>
            </div>
            <button v-else @click="openJoinDialog(seatNum - 1)" class="add-player-btn">
              座位 {{ seatNum }}
            </button>
          </div>
        </div>
      </div>

      <!-- 加入座位對話框 -->
      <div v-if="showJoinDialog" class="join-dialog-overlay" @click="closeJoinDialog">
        <div class="join-dialog" @click.stop>
          <h3>加入座位 {{ selectedSeat + 1 }}</h3>
          <div class="dialog-input-group">
            <label>玩家名稱:</label>
            <input v-model="joinForm.playerName" placeholder="請輸入名稱" maxlength="10">
          </div>
          <div class="dialog-input-group">
            <label>籌碼:</label>
            <input v-model.number="joinForm.chips" type="number" placeholder="1000" min="1">
          </div>
          <div class="dialog-buttons">
            <button @click="joinSeat" class="confirm-btn">確認</button>
            <button @click="closeJoinDialog" class="cancel-btn">取消</button>
          </div>
        </div>
      </div>

      <!-- 開始遊戲按鈕 -->
      <div class="start-game-section">
        <button 
          @click="createGame" 
          :disabled="!canStartGame"
          class="start-game-btn"
        >
          創建遊戲
        </button>
        <p v-if="!canStartGame" class="validation-message">
          至少需要 2 名玩家才能開始遊戲
        </p>
      </div>
    </div>

    <!-- 遊戲進行階段 -->
    <div v-if="gamePhase === 'playing'" class="poker-table">
      <!-- 遊戲控制區 -->
      <div class="game-controls">
        <button @click="startHand">開始新手牌</button>
        <button @click="endGame">結束遊戲</button>
        <button @click="refreshState">刷新狀態</button>
        <button @click="openHistory">手牌歷史</button>
      </div>

      <!-- 遊戲狀態顯示 -->
      <div v-if="gameState" class="game-info">
        <div class="game-status">
          <span class="phase">階段: {{ translatePhase(gameState.currentPhase) }}</span>
          <span class="pot">底池: {{ gameState.pot?.total || 0 }}</span>
          <span class="current-player">當前玩家: {{ gameState.currentPlayer || '無' }}</span>
        </div>
        <div class="blinds-info">
          小盲/大盲: {{ gameState.smallBlindAmount }}/{{ gameState.bigBlindAmount }}
        </div>
      </div>

      <!-- 撲克桌 -->
      <div class="table-container">
        <!-- 公共牌區域 -->
        <div class="board-area">
          <div v-if="gameState?.board?.length" class="community-cards">
            <div class="cards">
              <span v-for="card in gameState.board" :key="card" class="card" :class="getCardColorClass(card)">
                {{ formatCard(card) }}
              </span>
            </div>
          </div>
          <div class="pot-display">
            底池: {{ gameState?.pot?.total || 0 }}
          </div>
        </div>

        <!-- 玩家座位 -->
        <div v-if="gameState?.players" class="players-circle">
          <div 
            v-for="(player, index) in gameState.players" 
            :key="player.name"
            :class="['player-seat', `seat-${index}`, {
              'current': player.name === gameState.currentPlayer,
              'folded': player.status === 'FOLDED',
              'all-in': player.status === 'ALL_IN'
            }]"
          >
            <!-- 玩家資訊 -->
            <div class="player-info">
              <div class="player-name">{{ player.name }}</div>
              <div class="player-chips">{{ player.chips }}</div>
              <div v-if="player.currentBet > 0" class="current-bet">{{ player.currentBet }}</div>
            </div>

            <!-- 位置標記 -->
            <div class="position-badges">
              <span v-if="player.button" class="badge dealer">D</span>
              <span v-if="player.smallBlind" class="badge sb">SB</span>
              <span v-if="player.bigBlind" class="badge bb">BB</span>
            </div>

            <!-- 手牌顯示 -->
            <div class="player-cards">
              <!-- Showdown 階段顯示所有玩家手牌 -->
              <div v-if="gameState.currentPhase === 'SHOWDOWN' && gameState.showdownHands && gameState.showdownHands[player.name]" class="hand-cards">
                <span v-for="card in gameState.showdownHands[player.name]" :key="card.rank + card.suit" class="card small" :class="getCardColorClassFromObject(card)">
                  {{ formatCardFromObject(card) }}
                </span>
              </div>
              <!-- 當前玩家可以查看自己的手牌 -->
              <div v-else-if="player.name === myPlayerName">
                <button v-if="!myHand" @click="viewMyHand" class="view-cards-btn">
                  查看手牌
                </button>
                <div v-if="myHand" class="hand-cards">
                  <span v-for="card in myHand" :key="card" class="card small" :class="getCardColorClass(card)">
                    {{ formatCard(card) }}
                  </span>
                </div>
              </div>
            </div>

            <!-- 行動按鈕 - 只有當前玩家且是自己時顯示 -->
            <div v-if="player.name === gameState.currentPlayer && player.name === myPlayerName" class="action-panel">
              <div class="quick-actions">
                <button v-if="gameState.legalActions?.includes('FOLD')"  @click="executeAction('FOLD')" class="action-btn fold">棄牌</button>
                <button v-if="gameState.legalActions?.includes('CHECK')"  @click="executeAction('CHECK')" class="action-btn check">過牌</button>
                <button  v-if="gameState.legalActions?.includes('CALL')"  @click="executeAction('CALL', gameState.currentHighestBet)" class="action-btn call">
                  跟注 ({{ gameState.currentHighestBet }})
                </button>
                <button v-if="gameState.legalActions?.includes('ALL_IN')" @click="executeAction('ALL_IN')" class="action-btn allin">全押</button>
              </div>
              
              <div class="bet-controls">
                <template v-if="gameState.legalActions?.includes('BET') || gameState.legalActions?.includes('RAISE')">
                <input 
                  v-model.number="betAmount" 
                  type="number" 
                  placeholder="金額" 
                  min="1"
                  :max="player.chips"
                  class="bet-input"
                >
                <button v-if="gameState.legalActions?.includes('BET')" @click="executeAction('BET', betAmount)" class="action-btn bet">下注</button>
                <button v-if="gameState.legalActions?.includes('RAISE')"  @click="executeAction('RAISE', betAmount)" class="action-btn raise">加注</button>
                </template>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 訊息顯示 -->
    <div v-if="message" class="message" :class="messageType">
      {{ message }}
    </div>

    <!-- 手牌歷史對話框 -->
<div v-if="showHistoryDialog" class="history-overlay" @click="showHistoryDialog = false">
  <div class="history-dialog" @click.stop>
    <h3>手牌歷史記錄</h3>
    
    <!-- 歷史列表 -->
    <div v-if="!selectedHistory" class="history-list">
      <div v-if="handHistories.length === 0" class="empty-message">
        暫無歷史記錄
      </div>
      <div v-else>
        <div v-for="history in handHistories" :key="history.handNumber" 
             class="history-item"
             @click="viewHistoryDetail(history.handNumber)">
          <div class="history-header">
            <span class="hand-number">手牌 #{{ history.handNumber }}</span>
            <span class="hand-time">{{ new Date(history.timestamp).toLocaleTimeString() }}</span>
          </div>
          <div class="history-info">
            <span>底池: {{ history.potAmount }}</span>
            <span>贏家: {{ history.winners.map(w => w.playerName).join(', ') }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 手牌詳情 -->
    <div v-else class="history-detail">
      <button @click="closeHistoryDetail" class="back-btn">← 返回列表</button>
      
      <h4>手牌 #{{ selectedHistory.handNumber }}</h4>
      
      <div class="detail-section">
        <h5>公共牌</h5>
        <div class="cards">
          <span v-for="card in selectedHistory.board" :key="card" class="card small" :class="getCardColorClass(card)">
            {{ formatCard(card) }}
          </span>
        </div>
      </div>
      
      <div class="detail-section">
        <h5>底池: {{ selectedHistory.potAmount }}</h5>
      </div>
      
      <div class="detail-section">
        <h5>贏家</h5>
        <div v-for="winner in selectedHistory.winners" :key="winner.playerName">
          {{ winner.playerName }} 贏得 {{ winner.amountWon }}
        </div>
      </div>
      
      <div class="detail-section">
        <h5>玩家手牌</h5>
        <div v-for="player in selectedHistory.players" :key="player.name" class="player-detail">
          <strong>{{ player.name }}</strong>
          <div class="cards">
            <span v-for="card in player.hand" :key="card" class="card small" :class="getCardColorClass(card)">
              {{ formatCard(card) }}
            </span>
          </div>
          <span>籌碼: {{ player.chips }} | 動作: {{ translateAction(player.lastAction) }}</span>
        </div>
      </div>
    </div>
    
    <button @click="showHistoryDialog = false" class="close-btn">關閉</button>
  </div>
</div>
  </div>
</template>

<script>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'
import axios from 'axios'

export default {
  name: 'PokerApp',
  setup() {
    // 常量配置
    const isDev = import.meta.env.DEV

    const API_BASE = isDev 
  ? 'http://localhost:8080/api/game'
  : 'https://campusplus.xyz/api/game'

const WS_URL = isDev
  ? 'ws://localhost:8080/ws/game'
  : 'wss://campusplus.xyz/ws/game'
    

  console.log('當前環境:', isDev ? '開發' : '生產')
console.log('API 地址:', API_BASE)
console.log('WebSocket 地址:', WS_URL)
    // WebSocket
    let ws = null
    
    // 遊戲階段
    const gamePhase = ref('lobby') // 'lobby' | 'playing'
    
    // 玩家個人信息
    const myPlayerName = ref('')
    const myToken = ref('')
    const mySeatNumber = ref(null)
    
    // 大廳數據
    const seats = ref({}) // { 0: SeatInfo, 1: SeatInfo, ... }
    const playerCount = ref(0)
    const gameConfig = reactive({
      smallBlind: 5,
      bigBlind: 10
    })
    
    // 加入座位對話框
    const showJoinDialog = ref(false)
    const selectedSeat = ref(null)
    const joinForm = reactive({
      playerName: '',
      chips: 1000
    })
    
    // 遊戲數據
    const gameState = ref(null)
    const myHand = ref(null)
    const betAmount = ref(0)
    
    // 訊息提示
    const message = ref('')
    const messageType = ref('info')

    //牌局紀錄
    const handHistories = ref([])
const showHistoryDialog = ref(false)
const selectedHistory = ref(null)


    // 計算屬性
    const canStartGame = computed(() => {
      return playerCount.value >= 2 && 
             gameConfig.smallBlind > 0 && 
             gameConfig.bigBlind > gameConfig.smallBlind
    })

    const canStartNewHand = computed(() => {
      return gameState.value && 
             (gameState.value.currentPhase === 'SHOWDOWN' || !gameState.value.currentPlayer)
    })

     const getCardColorClass = (cardStr) => {
      if (!cardStr) return ''
      const [rank, suit] = cardStr.split('_')
      return getColorClassBySuit(suit)
    }

    const getCardColorClassFromObject = (card) => {
      if (!card) return ''
      return getColorClassBySuit(card.suit)
    }

    const getColorClassBySuit = (suit) => {
      const redSuits = ['HEARTS', 'DIAMONDS']
      return redSuits.includes(suit) ? 'card-red' : 'card-black'
    }

    // 獲取手牌歷史列表
const fetchHandHistory = async () => {
  try {
    const response = await axios.get(`${API_BASE}/history`)
    if (response.data.success) {
      handHistories.value = response.data.data
    }
  } catch (error) {
    showMessage('無法獲取手牌歷史', error)
  }
}

// 查看特定手牌詳情
const viewHistoryDetail = async (handNumber) => {
  try {
    const response = await axios.get(`${API_BASE}/history/${handNumber}`)
    if (response.data.success) {
      selectedHistory.value = response.data.data
      showMessage(`查看手牌 #${handNumber}`, 'info')
    }
  } catch (error) {
    showMessage('無法獲取手牌詳情', error)
  }
}

// 打開歷史記錄對話框
const openHistory = async () => {
  await fetchHandHistory()
  showHistoryDialog.value = true
}

// 關閉詳情
const closeHistoryDetail = () => {
  selectedHistory.value = null
}


    // WebSocket 連接
    const connectWebSocket = () => {
      ws = new WebSocket(WS_URL)
      
      ws.onopen = () => {
        console.log('WebSocket 連接成功')
      }
      
      ws.onmessage = (event) => {
        const msg = JSON.parse(event.data)
        handleWebSocketMessage(msg)
      }
      
      ws.onerror = (error) => {
        console.error('WebSocket 錯誤:', error)
      }
      
      ws.onclose = () => {
        console.log('WebSocket 連接關閉')
        // 5秒後重連
        setTimeout(connectWebSocket, 5000)
      }
    }

    const handleWebSocketMessage = (msg) => {
      console.log('收到 WebSocket 訊息:', msg)
      
      switch(msg.type) {
        case 'SEATS_UPDATE':
          updateSeats(msg.data)
          break
        case 'GAME_STATE_UPDATE':
          gameState.value = msg.data
          if (gamePhase.value === 'lobby' && msg.data) {
    gamePhase.value = 'playing'
  }

          // Showdown 階段自動清除本地手牌，因為會顯示 showdownHands
          if (msg.data.currentPhase === 'SHOWDOWN') {
            myHand.value = null
          }
          break
        case 'PLAYER_ACTION':
          console.log('玩家行動:', msg.data)
          break
        case 'PHASE_CHANGE':
          console.log('階段變化:', msg.data)
          break
      }
    }

    // 工具方法
    const showMessage = (msg, type = 'info') => {
      message.value = msg
      messageType.value = type
      setTimeout(() => {
        message.value = ''
      }, 3000)
    }

    // 座位管理
    const fetchSeats = async () => {
      try {
        const response = await axios.get(`${API_BASE}/seats`)
        if (response.data.success) {
          updateSeats(response.data.data)
        }
      } catch (error) {
        console.error('獲取座位失敗:', error)
      }
    }

    const updateSeats = (data) => {
      seats.value = data.seats || {}
      playerCount.value = data.playerCount || 0
    }

    const openJoinDialog = (seatNum) => {
      selectedSeat.value = seatNum
      showJoinDialog.value = true
    }

    const closeJoinDialog = () => {
      showJoinDialog.value = false
      selectedSeat.value = null
      joinForm.playerName = ''
      joinForm.chips = 1000
    }

    const joinSeat = async () => {
      if (!joinForm.playerName.trim() || joinForm.chips <= 0) {
        showMessage('請輸入有效的名稱和籌碼', 'error')
        return
      }

      try {
        const response = await axios.post(`${API_BASE}/join-seat`, {
          seatNumber: selectedSeat.value,
          playerName: joinForm.playerName.trim(),
          chips: joinForm.chips
        })
        
        if (response.data.success) {
          const data = response.data.data
          myPlayerName.value = data.seatInfo.playerName
          myToken.value = data.token
          mySeatNumber.value = selectedSeat.value
          
          showMessage('入座成功', 'success')
          closeJoinDialog()
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '加入座位失敗', 'error')
      }
    }

    const leaveSeat = async (seatNum) => {
      try {
        const response = await axios.post(`${API_BASE}/leave-seat`, {
          seatNumber: seatNum
        })
        
        if (response.data.success) {
          if (seatNum === mySeatNumber.value) {
            myPlayerName.value = ''
            myToken.value = ''
            mySeatNumber.value = null
          }
          showMessage('離座成功', 'success')
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '離座失敗', 'error')
      }
    }

    // 遊戲管理
    const createGame = async () => {
      if (!canStartGame.value) return

      try {
        const response = await axios.post(`${API_BASE}/create`, {
          smallBlind: gameConfig.smallBlind,
          bigBlind: gameConfig.bigBlind
        })
        
        if (response.data.success) {
          gamePhase.value = 'playing'
          showMessage('遊戲創建成功', 'success')
          await refreshState()
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '創建遊戲失敗', 'error')
      }
    }

    const startHand = async () => {
      try {
        const response = await axios.post(`${API_BASE}/start-hand`)
        if (response.data.success) {
          myHand.value = null // 清除舊手牌
          showMessage('新手牌開始', 'success')
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '開始手牌失敗', 'error')
      }
    }

    const endGame = async () => {
      try {
        const response = await axios.post(`${API_BASE}/end`)
        if (response.data.success) {
          gamePhase.value = 'lobby'
          gameState.value = null
          myHand.value = null
          showMessage('遊戲已結束', 'success')
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '結束遊戲失敗', 'error')
      }
    }

    const refreshState = async () => {
      try {
        const response = await axios.get(`${API_BASE}/state`)
        if (response.data.success) {
          gameState.value = response.data.data
        }
      } catch (error) {
        console.error('無法獲取遊戲狀態:', error)
      }
    }

    // 玩家操作
    const viewMyHand = async () => {
      if (!myToken.value) {
        showMessage('請先加入座位', 'error')
        return
      }

      try {
        const response = await axios.get(`${API_BASE}/hand`, {
          headers: {
            'Authorization': `Bearer ${myToken.value}`
          }
        })
        
        if (response.data.success) {
          myHand.value = response.data.data.cards
        }
      } catch (error) {
        showMessage(error.response?.data?.message || '無法獲取手牌', 'error')
      }
    }

    const executeAction = async (actionType, amount = null) => {
  if (!myToken.value) {
    showMessage('請先加入座位', 'error')
    return
  }

  try {
    const actionData = {
      action: actionType
    }
    
    // 只有需要金額的動作才傳 amount
    if (actionType === 'BET' || actionType === 'RAISE' || actionType === 'CALL') {
      actionData.amount = amount || 0
    }

    console.log('發送的動作數據:', actionData) // Debug
    
    const response = await axios.post(`${API_BASE}/action`, actionData, {
      headers: {
        'Authorization': `Bearer ${myToken.value}`
      }
    })
    
    if (response.data.success) {
      showMessage(`執行 ${translateAction(actionType)}`, 'success')
      myHand.value = null
    }
  } catch (error) {
    showMessage(error.response?.data?.message || '行動失敗', 'error')
    
    // Debug 信息
    console.log('當前玩家名稱:', gameState.value?.currentPlayer)
    console.log('我的玩家名稱:', myPlayerName.value)
    console.log('是否相等:', gameState.value?.currentPlayer === myPlayerName.value)
  }
}

    // 格式化方法
    const formatCard = (cardStr) => {
      if (!cardStr) return ''
      const [rank, suit] = cardStr.split('_')
      return formatCardParts(rank, suit)
    }

    const formatCardFromObject = (card) => {
      if (!card) return ''
      return formatCardParts(card.rank, card.suit)
    }

    const formatCardParts = (rank, suit) => {
      const suitSymbols = {
        'SPADES': '♠',
        'HEARTS': '♥',
        'DIAMONDS': '♦',
        'CLUBS': '♣'
      }
      const rankSymbols = {
        'ACE': 'A', 'KING': 'K', 'QUEEN': 'Q', 'JACK': 'J', 'TEN': '10',
        'NINE': '9', 'EIGHT': '8', 'SEVEN': '7', 'SIX': '6', 'FIVE': '5',
        'FOUR': '4', 'THREE': '3', 'TWO': '2'
      }
      return `${rankSymbols[rank] || rank}${suitSymbols[suit] || suit}`
    }

    const translatePhase = (phase) => {
      const phases = {
        'PREFLOP': '翻牌前',
        'FLOP': '翻牌',
        'TURN': '轉牌',
        'RIVER': '河牌',
        'SHOWDOWN': '攤牌'
      }
      return phases[phase] || phase
    }

    const translateAction = (action) => {
      const actions = {
        'FOLD': '棄牌',
        'CHECK': '過牌',
        'CALL': '跟注',
        'BET': '下注',
        'RAISE': '加注',
        'ALL_IN': '全押'
      }
      return actions[action] || action
    }

    // 生命週期
    onMounted(() => {
      connectWebSocket()
      fetchSeats()
    })

    onUnmounted(() => {
      if (ws) {
        ws.close()
      }
    })

    return {
      // 狀態
      gamePhase,
      myPlayerName,
      seats,
      playerCount,
      gameConfig,
      showJoinDialog,
      selectedSeat,
      joinForm,
      gameState,
      myHand,
      betAmount,
      message,
      messageType,
      
      // 計算屬性
      canStartGame,
      canStartNewHand,
      
      // 方法
      openJoinDialog,
      closeJoinDialog,
      joinSeat,
      leaveSeat,
      createGame,
      startHand,
      endGame,
      refreshState,
      viewMyHand,
      executeAction,
      formatCard,
      formatCardFromObject,
      translatePhase,
      translateAction,
      getCardColorClass,        
      getCardColorClassFromObject,
       handHistories,
  showHistoryDialog,
  selectedHistory,
  fetchHandHistory,
  viewHistoryDetail,
  openHistory,
  closeHistoryDetail,
      
    }
  }
}
</script>

<style scoped>
.poker-app {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  font-family: 'Arial', sans-serif;
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  color: white;
  box-sizing: border-box;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 大廳階段樣式 */
.game-setup {
  width: min(95vw, 1400px);
  height: 95vh; /* 改為固定高度 */
  background: rgba(255, 255, 255, 0.1);
  padding: 20px; /* 減少 padding */
  border-radius: 15px;
  backdrop-filter: blur(10px);
  overflow: hidden; /* 隱藏滾輪 */
  display: flex;
  flex-direction: column;
}

.game-setup h1 {
  text-align: center;
  margin-bottom: 20px; /* 減少間距 */
  font-size: 2rem; /* 稍微縮小字體 */
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  flex-shrink: 0; /* 防止標題被壓縮 */
}

.game-config {
  margin-bottom: 15px; /* 從 20px 減少到 15px */
  padding: 10px 15px; /* 從 15px 減少到 10px 15px */
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px; /* 從 10px 減少到 8px */
  flex-shrink: 0;
}

.game-config h2 {
  text-align: center;
  margin-bottom: 10px; /* 從 15px 減少到 10px */
  color: #ffc107;
  font-size: 1.3rem; /* 新增：縮小字體 */
}

.config-row {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 10px 0; /* 從 15px 減少到 10px */
  gap: 12px; /* 從 15px 減少到 12px */
}

.config-row label {
  min-width: 80px; /* 從 100px 減少到 80px */
  font-weight: bold;
  font-size: 14px; /* 從 16px 減少到 14px */
}

.config-row input {
  padding: 8px 12px; /* 從 10px 15px 減少到 8px 12px */
  border-radius: 4px; /* 從 5px 減少到 4px */
  border: none;
  width: 100px; /* 從 120px 減少到 100px */
  font-size: 14px; /* 從 16px 減少到 14px */
  text-align: center;
}

.players-setup {
  flex: 1; /* 占用剩餘空間 */
  display: flex;
  flex-direction: column;
  min-height: 0; /* 允許內容縮小 */
}

.players-setup h2 {
  text-align: center;
  margin-bottom: 20px; /* 減少間距 */
  color: #ffc107;
  flex-shrink: 0;
}

/* 大廳座位佈局 - 調整為響應式 */
.lobby-seats-container {
  position: relative;
  width: min(90%, 700px); /* 響應式寬度 */
  height: min(60vh, 400px); /* 響應式高度 */
  margin: 0 auto;
  background: radial-gradient(ellipse, #0f4c3a 0%, #0a3d2e 70%);
  border-radius: 50%;
  border: 8px solid #8b4513;
  flex-shrink: 0;
}

.lobby-seat-wrapper {
  position: absolute;
  width: 150px;
  transform: translate(-50%, -50%);
}

.position-0 { top: 10%; left: 50%; }
.position-1 { top: 25%; left: 85%; }
.position-2 { top: 60%; left: 88%; }
.position-3 { top: 90%; left: 50%; }
.position-4 { top: 60%; left: 12%; }
.position-5 { top: 25%; left: 15%; }
.position-6 { top: 15%; left: 25%; }
.position-7 { top: 15%; left: 75%; }

.occupied-seat {
  background: rgba(0, 0, 0, 0.8);
  border: 2px solid #28a745;
  border-radius: 10px;
  padding: 15px;
  text-align: center;
}

.player-info-lobby {
  margin-bottom: 10px;
}

.player-info-lobby .player-name {
  font-weight: bold;
  font-size: 16px;
  margin-bottom: 5px;
}

.player-info-lobby .player-chips {
  color: #ffc107;
  font-size: 14px;
}

.add-player-btn {
  width: 100%;
  padding: 20px;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  border: 2px dashed rgba(255, 255, 255, 0.5);
  border-radius: 10px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s;
}

.add-player-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  border-color: #28a745;
}

.remove-btn {
  width: 100%;
  padding: 8px;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
}

.remove-btn:hover {
  background: #c82333;
}

/* 加入對話框 */
.join-dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.join-dialog {
  background: #2a5298;
  padding: 30px;
  border-radius: 15px;
  min-width: 400px;
}

.join-dialog h3 {
  text-align: center;
  margin-bottom: 20px;
  color: #ffc107;
}

.dialog-input-group {
  margin: 15px 0;
}

.dialog-input-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: bold;
}

.dialog-input-group input {
  width: 100%;
  padding: 10px;
  border-radius: 5px;
  border: none;
  font-size: 16px;
}

.dialog-buttons {
  display: flex;
  gap: 15px;
  margin-top: 25px;
}

.confirm-btn, .cancel-btn {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
}

.confirm-btn {
  background: #28a745;
  color: white;
}

.confirm-btn:hover {
  background: #218838;
}

.cancel-btn {
  background: #6c757d;
  color: white;
}

.cancel-btn:hover {
  background: #5a6268;
}

.start-game-section {
  text-align: center;
  padding-top: 15px; /* 減少間距 */
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  flex-shrink: 0; /* 防止被壓縮 */
  margin-top: auto; /* 推到底部 */
}

.start-game-btn {
  padding: 18px 50px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 20px;
  font-weight: bold;
  cursor: pointer;
  transition: background 0.3s;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.start-game-btn:hover:not(:disabled) {
  background: #0056b3;
}

.start-game-btn:disabled {
  background: #6c757d;
  cursor: not-allowed;
}

.validation-message {
  margin-top: 10px; /* 減少間距 */
  color: #ffc107;
  font-size: 14px; /* 稍微縮小字體 */
}

/* 遊戲階段樣式 */
.poker-table {
  width: 100vw;
  height: 100vh;
  position: relative;
}

.game-controls {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  display: flex;
  gap: 10px;
  padding: 15px 25px;
  background: rgba(0, 0, 0, 0.8);
  border-radius: 25px;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.game-controls button {
  margin: 0 8px;
  padding: 10px 20px;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s;
}

.game-controls button:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
}

.game-controls button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.game-info {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  padding: 15px 25px;
  background: rgba(0, 0, 0, 0.8);
  border-radius: 25px;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  min-width: 500px;
  text-align: center;
}

.game-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  gap: 30px;
}

.game-status span {
  font-size: 16px;
  font-weight: bold;
}

.blinds-info {
  font-size: 14px;
  color: #ffc107;
}

.table-container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: min(85vw, 1000px);
  height: min(85vh, 700px);
  background: radial-gradient(ellipse, #0f4c3a 0%, #0a3d2e 70%);
  border-radius: 50%;
  border: 8px solid #8b4513;
  overflow: visible;
}

.board-area {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}

.community-cards .cards {
  display: flex;
  gap: 12px;
  justify-content: center;
  margin-bottom: 25px;
  flex-wrap: wrap;
}

.card {
  background: white;
  border: 2px solid #333;
  border-radius: 8px;
  padding: 10px 14px;
  font-weight: bold;
  font-size: 24px;
  min-width: 60px;
  text-align: center;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  transition: transform 0.2s;
}

.card:hover {
  transform: translateY(-5px);
}

.card.small {
  padding: 4px 8px;
  font-size: 14px;
  min-width: 35px;
}

.card.card-red {
  color: #dc3545; /* 紅色：紅桃和方塊 */
}

.card.card-black {
  color: #212529; /* 黑色：黑桃和梅花 */
}

.pot-display {
  background: rgba(0, 0, 0, 0.9);
  padding: 12px 25px;
  border-radius: 20px;
  font-size: 20px;
  font-weight: bold;
  color: #ffc107;
  border: 2px solid #ffc107;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.players-circle {
  position: relative;
  width: 100%;
  height: 100%;
}

.player-seat {
  position: absolute;
  width: 180px;
  padding: 15px;
  background: rgba(0, 0, 0, 0.85);
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 12px;
  text-align: center;
  transform: translate(-50%, -50%);
  backdrop-filter: blur(8px);
  transition: all 0.3s ease;
}

.player-seat.current {
  border-color: #28a745;
  box-shadow: 0 0 25px rgba(40, 167, 69, 0.8);
  transform: translate(-50%, -50%) scale(1.05);
}

.player-seat.folded {
  opacity: 0.4;
  background: rgba(139, 69, 19, 0.7);
}

.player-seat.all-in {
  border-color: #ffc107;
  box-shadow: 0 0 25px rgba(255, 193, 7, 0.8);
}

.seat-0 { top: 10%; left: 50%; }
.seat-1 { top: 25%; left: 85%; }
.seat-2 { top: 60%; left: 88%; }
.seat-3 { top: 90%; left: 50%; }
.seat-4 { top: 60%; left: 12%; }
.seat-5 { top: 25%; left: 15%; }
.seat-6 { top: 15%; left: 25%; }
.seat-7 { top: 15%; left: 75%; }

.player-info {
  margin-bottom: 12px;
}

.player-name {
  font-weight: bold;
  font-size: 16px;
  margin-bottom: 6px;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.player-chips {
  color: #ffc107;
  font-weight: bold;
  font-size: 14px;
}

.current-bet {
  color: #28a745;
  font-size: 14px;
  margin-top: 6px;
  font-weight: bold;
}

.position-badges {
  margin: 8px 0;
}

.badge {
  display: inline-block;
  padding: 3px 8px;
  border-radius: 50%;
  font-size: 11px;
  font-weight: bold;
  margin: 0 3px;
  min-width: 24px;
  height: 24px;
  line-height: 18px;
}

.badge.dealer { background: #007bff; color: white; }
.badge.sb { background: #28a745; color: white; }
.badge.bb { background: #dc3545; color: white; }

.view-cards-btn {
  padding: 6px 12px;
  background: #17a2b8;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 12px;
  margin-bottom: 8px;
  transition: background 0.2s;
}

.view-cards-btn:hover {
  background: #138496;
}

.hand-cards {
  display: flex;
  gap: 6px;
  justify-content: center;
  margin-bottom: 8px;
}

.action-panel {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 2px solid rgba(255, 255, 255, 0.3);
}

.quick-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  justify-content: center;
  margin-bottom: 10px;
}

.action-btn {
  padding: 6px 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 11px;
  font-weight: bold;
  min-width: 55px;
  transition: all 0.2s;
}

.action-btn:hover {
  transform: scale(1.1);
}

.action-btn.fold { background: #dc3545; color: white; }
.action-btn.check { background: #6c757d; color: white; }
.action-btn.call { background: #28a745; color: white; }
.action-btn.bet { background: #007bff; color: white; }
.action-btn.raise { background: #fd7e14; color: white; }
.action-btn.allin { background: #6f42c1; color: white; }

.bet-controls {
  display: flex;
  gap: 6px;
  justify-content: center;
  align-items: center;
}

.bet-input {
  width: 70px;
  padding: 6px 8px;
  border: none;
  border-radius: 4px;
  text-align: center;
  font-size: 12px;
}

.message {
  position: fixed;
  top: 100px;
  right: 20px;
  padding: 15px 20px;
  border-radius: 8px;
  font-weight: bold;
  z-index: 1000;
  max-width: 300px;
  font-size: 14px;
  backdrop-filter: blur(10px);
  animation: slideIn 0.3s ease-out;
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.message.success {
  background: rgba(212, 237, 218, 0.95);
  color: #155724;
  border: 1px solid #c3e6cb;
}

.message.error {
  background: rgba(248, 215, 218, 0.95);
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.message.info {
  background: rgba(209, 236, 241, 0.95);
  color: #0c5460;
  border: 1px solid #bee5eb;
}

/* 手牌歷史對話框 */
.history-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.history-dialog {
  background: #2a5298;
  padding: 30px;
  border-radius: 15px;
  min-width: 600px;
  max-width: 800px;
  max-height: 80vh;
  overflow-y: auto;
}

.history-dialog h3 {
  text-align: center;
  margin-bottom: 20px;
  color: #ffc107;
}

.history-list {
  margin-bottom: 20px;
}

.empty-message {
  text-align: center;
  padding: 20px;
  color: rgba(255, 255, 255, 0.6);
}

.history-item {
  background: rgba(255, 255, 255, 0.1);
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.3s;
}

.history-item:hover {
  background: rgba(255, 255, 255, 0.2);
}

.history-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.hand-number {
  font-weight: bold;
  color: #ffc107;
}

.hand-time {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.7);
}

.history-info {
  display: flex;
  gap: 20px;
  font-size: 14px;
}

.history-detail {
  color: white;
}

.detail-section {
  margin: 20px 0;
  padding: 15px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
}

.detail-section h5 {
  margin-bottom: 10px;
  color: #ffc107;
}

.player-detail {
  margin: 10px 0;
  padding: 10px;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 5px;
}

.back-btn {
  margin-bottom: 15px;
  padding: 8px 15px;
  background: #6c757d;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.back-btn:hover {
  background: #5a6268;
}

.close-btn {
  width: 100%;
  padding: 12px;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  margin-top: 20px;
}

.close-btn:hover {
  background: #c82333;
}

/* 響應式設計 */
@media (max-width: 1200px) {
  .lobby-seats-container {
    width: min(85%, 600px);
    height: min(55vh, 350px);
  }
}

@media (max-width: 768px) {
  .game-setup {
    padding: 15px;
    height: 98vh;
  }
  
  .game-setup h1 {
    font-size: 1.8rem;
    margin-bottom: 15px;
  }
  
  .lobby-seats-container {
    width: 95%;
    height: min(50vh, 300px);
  }
  
  .game-config {
    padding: 12px;
    margin-bottom: 15px;
  }
}
</style>