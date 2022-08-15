<script setup>
import { onMounted, ref } from 'vue';
import Board from './components/Board.vue';
import axios from 'axios';

const BASE_URL = 'http://localhost:8080';

const currentPlayer = ref(null);
const receivedInvite = ref(null);
const players = ref([]);
const gameState = ref({});

const ws = new WebSocket('ws://localhost:8081');

/**
 * Long pooling for fetch players
 */
const fetchPlayers = () => {
  setInterval(async () => {
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
  ws.send(JSON.stringify(request));
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
  ws.send(JSON.stringify(request));
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
  ws.send(JSON.stringify(request));
}

ws.onmessage = (e) => {
  console.log(e.data);
  const data = JSON.parse(e.data);

  switch (data.type) {
    case 'CreatePlayer': {
      currentPlayer.value = data.payload.player;
      break;
    }
    case 'InviteReceived': {
      receivedInvite.value = data.payload;
      break;
    }
    case 'InviteAccepted': {
      gameState.value = { gameSession: { id: data.payload.gameSessionId } }
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
  }
};

onMounted(() => fetchPlayers());
</script>

<template>
  <div class="bg-green-50 min-h-screen text-green-700">
    <div class="container">
      <div class="text-center py-3 mb-10">
        <h1 class="text-7xl text-center">Tic Tac Toe</h1>
      </div>

      <div>
        <div class="flex flex-nowrap h-[calc(100vh-12rem)]">
          <div class="w-full h-full">
            <div v-if="gameState.game && gameState.game.board">
              <Board :rows="gameState.game.board" @cellClick="onCellClick" />
            </div>
          </div>

          <div class="w-80 h-full bg-green-100 p-3 rounded-xl shadow">
            <div v-if="gameState.gameSession?.id" class="mb-2">
              <h3 class="text-xl text-center mb-2">Create Game</h3>
              <div class="flex justify-center gap-1">
                <div>
                  <button class="p-2 bg-white shadow rounded" @click="createGame(3)">3x3</button>
                </div>
                <div>
                  <button class="p-2 bg-white shadow rounded" @click="createGame(4)">4x4</button>
                </div>
                <div>
                  <button class="p-2 bg-white shadow rounded" @click="createGame(5)">5x5</button>
                </div>
              </div>
            </div>

            <div v-if="receivedInvite" class="mb-2">
              <h3 class="text-lg text-center mb-2">You received an invitation</h3>
              <div>
                <button class="p-2 bg-white shadow rounded" @click="confirmInvite">Confirm</button>
              </div>
            </div>

            <div v-if="!gameState.gameSession?.id">
              <h3 class="text-lg text-center">Create invite</h3>
              <div>
                <ul>
                  <li v-for="p in players" :key="p.id" v-on:click="invitePlayer(p.id)" class="hover:text-blue-700 cursor-pointer">
                    {{ p.id }}
                  </li>
                </ul>
              </div>
            </div>

            <div v-if="gameState.gameSession && gameState.gameSession.firstPlayer">
              <h3 class="text-xl text-center mb-2">Results</h3>
              <h4
                class="text-lg text-center mb-1"
                :class="currentPlayer.id === gameState.gameSession.firstPlayer.id ? 'text-red-600' : 'text-green-500'"
              >"X" - {{ gameState.gameSession.firstPlayer.winCount }}</h4>
              <h4
                class="text-lg text-center"
                :class="currentPlayer.id === gameState.gameSession.secondPlayer.id ? 'text-red-600' : 'text-green-500'"
              >"O" - {{ gameState.gameSession.secondPlayer.winCount }}</h4>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
