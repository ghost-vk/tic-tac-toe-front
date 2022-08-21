<script setup>
import { onMounted, ref } from 'vue';
import Board from './components/Board.vue';
import Button from './components/Button.vue';
import axios from 'axios';

const BASE_URL = 'http://localhost:8080';
const currentPlayer = ref(null);
const receivedInvite = ref(null);
const players = ref([]);
const gameState = ref({});
const ws = ref(null);

const connect = () => {
  if (ws.value instanceof WebSocket) {
    console.info('Already connected');
    return;
  }
  ws.value = new WebSocket('ws://localhost:8081');
  ws.value.onmessage = (e) => onWebSocketMessage(e);
}

const disconnect = () => {
  if (!(ws.value instanceof WebSocket)) {
    console.info('Connection is not established');
    return;
  }
  ws.value.close();
  currentPlayer.value = null;
  gameState.value = null;
  ws.value = null;
}

/**
 * Long pooling for fetch players
 */
const fetchPlayers = () => {
  setInterval(async () => {
    if (!currentPlayer.value) return;
    const { data } = await axios.get(`${BASE_URL}/players`);
    if (Array.isArray(data.players)) {
      players.value = data.players.filter((p) => p.id !== currentPlayer.value.id);
    }
  }, 1500);
};

const invitePlayer = (playerId) => {
  return axios.post(`${BASE_URL}/invite`, {
    invite: {
      actorId: currentPlayer.value.id,
      invitedPlayerId: playerId,
    }
  });
}

const confirmInvite = () => {
  if (!receivedInvite.value.inviteId) return;
  const request = {
    action: 'AcceptInvite',
    payload: { inviteId: receivedInvite.value.inviteId }
  }
  ws.value.send(JSON.stringify(request));
  receivedInvite.value = null;
}

const createGame = (boardSize) => {
  if (!gameState.value.gameSession?.id) return;
  const request = {
    action: 'CreateGame',
    payload: {
      gameSessionId: gameState.value.gameSession.id,
      boardSize,
    }
  };
  ws.value.send(JSON.stringify(request));
}

const onCellClick = (coord) => {
  if (!gameState.value.gameSession?.id) return;
  const request = {
    action: 'MakeStep',
    payload: {
      x: coord.x,
      y: coord.y,
      gameSessionId: gameState.value.gameSession.id,
      actor: { id: currentPlayer.value.id },
    }
  }
  ws.value.send(JSON.stringify(request));
}

const onWebSocketMessage = (e) => {
  console.log(e.data);
  const data = JSON.parse(e.data);

  switch (data.type) {
    case 'CreatePlayer': {
      currentPlayer.value = data.payload.player;
      break;
    }
    case 'InviteAccepted': {
      gameState.value = { gameSession: { id: data.payload.gameSessionId } }
      break;
    }
    case 'InviteReceived': {
      receivedInvite.value = data.payload;
      break;
    }
    case 'GameCreated': {
      gameState.value = data.payload;
      break;
    }
    case 'GameStep': {
      gameState.value = data.payload;
      break;
    }
    case 'StepTimeout': {
      gameState.value = data.payload;
      break;
    }
    case 'OpponentLeftTheGameSession': {
      alert('Ops! Your opponent has left the game.');
      gameState.value = null;
      break;
    }
    case 'FailCreateGame': {
      alert(data.payload.error);
      break;
    }
    case 'After3SerialWins': {
      alert(data.payload.message);
      gameState.value = null;
      ws.value = null;
      break;
    }
    case 'After10TotalWins': {
      alert(data.payload.message);
      gameState.value = null;
      ws.value = null;
      break;
    }
  }
}

onMounted(() => fetchPlayers());
</script>

<template>
  <div class="bg-green-50 min-h-screen text-green-700">
    <div class="container">
      <div class="text-center py-3 mb-10">
        <h1 class="text-7xl text-center">Tic Tac Toe</h1>
      </div>

      <div>
        <div class="flex flex-nowrap">
          <div class="w-full">
            <div v-if="gameState && gameState.game && gameState.game.board">
              <Board :rows="gameState.game.board" @cellClick="onCellClick" />
            </div>
          </div>

          <div class="w-80 h-full bg-green-100 p-3 rounded-xl shadow">
            <div class="flex justify-center mb-4">
              <Button
                v-if="ws === null"
                @click="connect"
                text="Connect"
              />
              <Button
                v-else
                @click="disconnect"
                text="Disconnect"
              />
            </div>

            <div
              v-if="gameState && gameState.gameSession && gameState.gameSession.firstPlayer"
              class="mb-2"
            >
              <div class="w-full h-px bg-green-300 mb-2"></div>
              <h3 class="text-xl text-center mb-2">Results</h3>
              <h4
                class="text-lg text-center mb-1"
                :class="currentPlayer.id !== gameState.gameSession.firstPlayer.id ? 'text-red-600' : 'text-green-500'"
              >"X" - {{ gameState.gameSession.firstPlayer.winCount }}</h4>
              <h4
                class="text-lg text-center"
                :class="currentPlayer.id !== gameState.gameSession.secondPlayer.id ? 'text-red-600' : 'text-green-500'"
              >"O" - {{ gameState.gameSession.secondPlayer.winCount }}</h4>
            </div>

            <div v-if="gameState && gameState.gameSession?.id" class="mb-2">
              <div class="w-full h-px bg-green-300 mb-2"></div>
              <h3 class="text-xl text-center mb-2">Create Game</h3>
              <div class="flex justify-center gap-1">
                <div>
                  <Button
                    @click="createGame(3)"
                    text="3x3"
                  />
                </div>
                <div>
                  <Button
                    @click="createGame(4)"
                    text="4x4"
                  />
                </div>
                <div>
                  <Button
                    @click="createGame(5)"
                    text="5x5"
                  />
                </div>
              </div>
            </div>

            <div v-if="receivedInvite" class="mb-3">
              <div class="w-full h-px bg-green-300 mb-2"></div>
              <h3 class="text-lg text-center mb-2">You received an invitation</h3>
              <div class="flex justify-center">
                <Button
                  @click="confirmInvite"
                  text="Confirm"
                />
              </div>
            </div>

            <div
              v-if="!receivedInvite && ws"
              class="mb-2"
            >
              <div class="w-full h-px bg-green-300 mb-2"></div>
              <h3 class="text-lg text-center">Create invite</h3>
              <div>
                <ul>
                  <li
                    v-for="p in players"
                    :key="p.id"
                    v-on:click="invitePlayer(p.id)"
                    class="hover:text-blue-700 cursor-pointer"
                  >
                    {{ p.id }}
                  </li>
                </ul>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
