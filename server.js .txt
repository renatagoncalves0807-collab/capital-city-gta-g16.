const express = require('express');
const http = require('http');
const path = require('path');
const fs = require('fs');
const { Server } = require('socket.io');
const QRCode = require('qrcode');

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: '*' },
  pingTimeout: 30000,
  pingInterval: 10000
});

const PORT = process.env.PORT || 3000;
const PUBLIC_DIR = path.join(__dirname, 'public');
const ASSETS_DIR = path.join(PUBLIC_DIR, 'assets');

// Regras oficiais sincronizadas pelo servidor
const QUESTION_DURATION_SECONDS = 15;
const QUESTION_DURATION_MS = QUESTION_DURATION_SECONDS * 1000;
const BASE_POINTS = 1000;
const MAX_SPEED_BONUS = 500;

// As 10 imagens oficiais em public/assets/
const EXPECTED_ASSETS = [
  'gta-cover.jpg',
  'city-day.jpg',
  'downtown.jpg',
  'desert.jpg',
  'hills-car.jpg',
  'night-bike.webp',
  'rain.jpg',
  'rain-walk.jpg',
  'trevor.jpg',
  'trio.jpg'
];

// 7 Missões exclusivas de Capital Próprio (Disciplina: Auditoria | Português de Portugal)
// Nota: reveja cada enunciado face à matéria lecionada antes da apresentação final.
const MISSIONS = [
  {
    id: 1,
    title: 'Missão 1 — Reserva Legal',
    zoneName: 'Avenida Capital · Dia',
    theme: 'day',
    backdrop: '/assets/city-day.jpg',
    question: 'Qual a fração mínima anual dos lucros de exercício a afetar à reserva legal (CSC)?',
    options: [
      '5% dos lucros',
      '15% dos lucros',
      '25% dos lucros'
    ],
    correctIndex: 0,
    explanation: 'Os arts. 218.º e 295.º do CSC exigem pelo menos a vigésima parte (5%) dos lucros até perfazer o limite legal.'
  },
  {
    id: 2,
    title: 'Missão 2 — Participações Próprias',
    zoneName: 'Colinas de Capital · Pôr-do-sol',
    theme: 'hills',
    backdrop: '/assets/hills-car.jpg',
    question: 'Como se apresentam as ações ou quotas próprias no Balanço segundo o SNC?',
    options: [
      'No Ativo corrente',
      'A deduzir ao Capital Próprio',
      'No Passivo não corrente'
    ],
    correctIndex: 1,
    explanation: 'No SNC (conta 52), as ações ou quotas próprias são deduzidas diretamente ao Capital Próprio.'
  },
  {
    id: 3,
    title: 'Missão 3 — Prestações Suplementares',
    zoneName: 'Centro Financeiro',
    theme: 'downtown',
    backdrop: '/assets/downtown.jpg',
    question: 'Qual é uma regra legal obrigatória das prestações suplementares (CSC)?',
    options: [
      'Vencem juros anuais',
      'Podem ser bens imóveis',
      'São em dinheiro e sem juros'
    ],
    correctIndex: 2,
    explanation: 'Pelo art.º 210.º do CSC, têm sempre dinheiro por objeto, nunca vencem juros e integram o Capital Próprio (conta 53).'
  },
  {
    id: 4,
    title: 'Missão 4 — Suprimentos',
    zoneName: 'Baixa Urbana · Fim de Tarde',
    theme: 'sunset',
    backdrop: '/assets/rain-walk.jpg',
    question: 'Como são classificados contabilisticamente os suprimentos efetuados pelos sócios?',
    options: [
      'Passivo (financiamento)',
      'Reservas livres',
      'Capital subscrito'
    ],
    correctIndex: 0,
    explanation: 'Os suprimentos (art.º 243.º do CSC) são empréstimos dos sócios com caráter de permanência: constituem Passivo, não Capital Próprio.'
  },
  {
    id: 5,
    title: 'Missão 5 — Perda de Metade do Capital',
    zoneName: 'Autoestrada do Deserto',
    theme: 'desert',
    backdrop: '/assets/desert.jpg',
    question: 'Segundo o art.º 35.º do CSC, quando existe perda de metade do capital social?',
    options: [
      'Lucro cai 50% num ano',
      'Capital Próprio ≤ metade do capital',
      'Passivo excede o Ativo'
    ],
    correctIndex: 1,
    explanation: 'O art.º 35.º, n.º 2 do CSC considera perdida metade do capital quando o Capital Próprio é igual ou inferior a metade do capital social.'
  },
  {
    id: 6,
    title: 'Missão 6 — Prestações Acessórias',
    zoneName: 'Via Rápida · Noite',
    theme: 'night',
    backdrop: '/assets/night-bike.webp',
    question: 'O que distingue as prestações acessórias das prestações suplementares?',
    options: [
      'Podem ser onerosas ou em espécie',
      'São sempre gratuitas',
      'Dispensam previsão no contrato'
    ],
    correctIndex: 0,
    explanation: 'Ao contrário das suplementares, as prestações acessórias (arts. 209.º e 287.º do CSC) podem ser onerosas e em bens distintos de dinheiro.'
  },
  {
    id: 7,
    title: 'Missão 7 — Adiantamentos sobre Lucros',
    zoneName: 'Distrito Central · Chuva',
    theme: 'rain',
    backdrop: '/assets/rain.jpg',
    question: 'Para distribuir adiantamentos sobre lucros no exercício (SA), o CSC exige:',
    options: [
      'Apenas decisão da gestão',
      'Autorização bancária',
      'Contrato social e balanço certificado'
    ],
    correctIndex: 2,
    explanation: 'O art.º 297.º do CSC exige autorização no contrato de sociedade e balanço intercalar certificado pelo Revisor Oficial de Contas.'
  }
];

function normalizeFilename(name) {
  return name.toLowerCase().replace(/[^a-z0-9.]/g, '');
}

function resolveAssetFile(requestedName) {
  if (!fs.existsSync(ASSETS_DIR)) return null;
  const exactPath = path.join(ASSETS_DIR, requestedName);
  if (fs.existsSync(exactPath) && fs.statSync(exactPath).isFile()) {
    return exactPath;
  }
  const files = fs.readdirSync(ASSETS_DIR);
  const targetNorm = normalizeFilename(requestedName);
  const match = files.find((f) => normalizeFilename(f) === targetNorm);
  return match ? path.join(ASSETS_DIR, match) : null;
}

// Serve /assets/* de forma tolerante a diferenças de maiúsculas ou hífens (ex: night-bike.webp vs nightbike.webp)
app.get('/assets/:filename', (req, res, next) => {
  const safeName = path.basename(req.params.filename);
  const resolved = resolveAssetFile(safeName);
  if (resolved) {
    res.setHeader('Cache-Control', 'public, max-age=3600');
    return res.sendFile(resolved);
  }
  return next();
});

// Diagnóstico público para validar as 10 imagens no Render após deploy
app.get('/api/assets-check', (req, res) => {
  const details = EXPECTED_ASSETS.map((name) => {
    const resolved = resolveAssetFile(name);
    return {
      expected: name,
      url: `/assets/${name}`,
      found: Boolean(resolved),
      actualFile: resolved ? path.basename(resolved) : null
    };
  });
  const missing = details.filter((item) => !item.found).map((item) => item.expected);
  res.json({
    ok: missing.length === 0,
    totalExpected: EXPECTED_ASSETS.length,
    totalFound: details.length - missing.length,
    missing,
    details
  });
});

app.use(express.static(PUBLIC_DIR));

app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    game: 'Capital City',
    group: 'Grupo 16',
    missions: MISSIONS.length,
    secondsPerMission: QUESTION_DURATION_SECONDS
  });
});

// Gestão de salas multijogador
const rooms = new Map();

function generateRoomCode() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  let code = '';
  do {
    code = '';
    for (let i = 0; i < 4; i += 1) {
      code += chars[Math.floor(Math.random() * chars.length)];
    }
  } while (rooms.has(code));
  return code;
}

function getSortedLeaderboard(room) {
  const list = Array.from(room.players.values());
  list.sort((a, b) => {
    if (b.score !== a.score) return b.score - a.score;
    if (b.correctCount !== a.correctCount) return b.correctCount - a.correctCount;
    if (a.totalResponseMs !== b.totalResponseMs) return a.totalResponseMs - b.totalResponseMs;
    return a.name.localeCompare(b.name, 'pt');
  });

  return list.map((p, index) => ({
    rank: index + 1,
    id: p.id,
    name: p.name,
    score: p.score,
    correctCount: p.correctCount,
    connected: p.connected
  }));
}

function buildRankingPayload(room, playerId) {
  const full = getSortedLeaderboard(room);
  const top4 = full.slice(0, 4);
  const me = playerId ? full.find((entry) => entry.id === playerId) : null;
  const myRank = me ? me.rank : null;
  const inTop4 = Boolean(myRank && myRank <= 4);

  let positionText = '';
  if (myRank) {
    positionText = inTop4
      ? `Estás no Top 4 — ${myRank}.º lugar!`
      : `Estás em ${myRank}.º lugar`;
  }

  return {
    top4,
    totalPlayers: full.length,
    myRank,
    myScore: me ? me.score : 0,
    inTop4,
    positionText,
    fullRanking: full
  };
}

function getRoomPlayersSummary(room) {
  return Array.from(room.players.values()).map((p) => ({
    id: p.id,
    name: p.name,
    score: p.score,
    connected: p.connected,
    answeredCurrent: p.answeredCurrent
  }));
}

function clearMissionTimers(room) {
  if (room.missionTimeout) {
    clearTimeout(room.missionTimeout);
    room.missionTimeout = null;
  }
  if (room.tickInterval) {
    clearInterval(room.tickInterval);
    room.tickInterval = null;
  }
}

function emitLobbyUpdate(room) {
  io.to(room.code).emit('lobby_update', {
    roomCode: room.code,
    players: getRoomPlayersSummary(room),
    totalMissions: MISSIONS.length,
    secondsPerMission: QUESTION_DURATION_SECONDS
  });
}

function emitAnswerProgress(room) {
  const players = Array.from(room.players.values());
  const answeredCount = players.filter((p) => p.answeredCurrent).length;
  io.to(room.code).emit('answer_progress', {
    answeredCount,
    totalPlayers: players.length
  });
}

function finishCurrentMission(room) {
  if (room.status !== 'question') return;
  clearMissionTimers(room);
  room.status = 'mission_results';

  const mission = MISSIONS[room.currentMissionIndex];

  for (const player of room.players.values()) {
    if (!player.answeredCurrent) {
      player.answeredCurrent = true;
      player.totalResponseMs += QUESTION_DURATION_MS;
      player.lastResult = {
        correct: false,
        timedOut: true,
        statusText: 'MISSÃO FALHADA!',
        subText: 'Tempo esgotado (15s).',
        pointsEarned: 0,
        selectedIndex: null
      };

      if (player.socketId) {
        io.to(player.socketId).emit('answer_result', {
          missionNumber: room.currentMissionIndex + 1,
          totalMissions: MISSIONS.length,
          ...player.lastResult,
          totalScore: player.score,
          ranking: buildRankingPayload(room, player.id)
        });
      }
    }
  }

  const basePayload = {
    missionNumber: room.currentMissionIndex + 1,
    totalMissions: MISSIONS.length,
    isLastMission: room.currentMissionIndex === MISSIONS.length - 1,
    question: mission.question,
    options: mission.options,
    correctIndex: mission.correctIndex,
    correctLetter: ['A', 'B', 'C'][mission.correctIndex],
    correctText: mission.options[mission.correctIndex],
    explanation: mission.explanation
  };

  if (room.hostSocketId) {
    io.to(room.hostSocketId).emit('mission_summary', {
      ...basePayload,
      ranking: buildRankingPayload(room, null)
    });
  }

  for (const player of room.players.values()) {
    if (player.socketId) {
      io.to(player.socketId).emit('mission_summary', {
        ...basePayload,
        playerResult: player.lastResult,
        ranking: buildRankingPayload(room, player.id)
      });
    }
  }
}

function startMission(room, missionIndex) {
  if (missionIndex < 0 || missionIndex >= MISSIONS.length) return;
  clearMissionTimers(room);

  room.currentMissionIndex = missionIndex;
  room.status = 'question';
  room.missionStartTime = Date.now();
  room.missionEndTime = room.missionStartTime + QUESTION_DURATION_MS;

  for (const player of room.players.values()) {
    player.answeredCurrent = false;
    player.lastResult = null;
  }

  const mission = MISSIONS[missionIndex];
  io.to(room.code).emit('mission_started', {
    missionNumber: missionIndex + 1,
    totalMissions: MISSIONS.length,
    title: mission.title,
    zoneName: mission.zoneName,
    theme: mission.theme,
    backdrop: mission.backdrop,
    question: mission.question,
    options: mission.options,
    durationSeconds: QUESTION_DURATION_SECONDS,
    durationMs: QUESTION_DURATION_MS,
    remainingMs: QUESTION_DURATION_MS
  });

  emitAnswerProgress(room);

  room.tickInterval = setInterval(() => {
    const remainingMs = Math.max(0, room.missionEndTime - Date.now());
    io.to(room.code).emit('timer_sync', {
      remainingMs,
      durationMs: QUESTION_DURATION_MS
    });
  }, 500);

  room.missionTimeout = setTimeout(() => {
    finishCurrentMission(room);
  }, QUESTION_DURATION_MS);
}

function finishGame(room) {
  clearMissionTimers(room);
  room.status = 'final';

  if (room.hostSocketId) {
    io.to(room.hostSocketId).emit('game_finished', {
      totalMissions: MISSIONS.length,
      ranking: buildRankingPayload(room, null)
    });
  }

  for (const player of room.players.values()) {
    if (player.socketId) {
      io.to(player.socketId).emit('game_finished', {
        totalMissions: MISSIONS.length,
        ranking: buildRankingPayload(room, player.id)
      });
    }
  }
}

io.on('connection', (socket) => {
  socket.on('create_room', async (payload = {}, callback) => {
    try {
      const code = generateRoomCode();
      const origin = typeof payload.origin === 'string' && payload.origin.startsWith('http')
        ? payload.origin.replace(/\/+$/, '')
        : `http://localhost:${PORT}`;

      const joinUrl = `${origin}/?room=${code}`;
      const qrDataUrl = await QRCode.toDataURL(joinUrl, {
        width: 320,
        margin: 2,
        color: {
          dark: '#111318',
          light: '#F7F3E9'
        }
      });

      const room = {
        code,
        hostSocketId: socket.id,
        status: 'lobby',
        currentMissionIndex: -1,
        missionStartTime: 0,
        missionEndTime: 0,
        missionTimeout: null,
        tickInterval: null,
        players: new Map()
      };

      rooms.set(code, room);
      socket.join(code);
      socket.data.roomCode = code;
      socket.data.role = 'host';

      callback?.({
        ok: true,
        roomCode: code,
        joinUrl,
        qrDataUrl,
        totalMissions: MISSIONS.length,
        secondsPerMission: QUESTION_DURATION_SECONDS
      });
      emitLobbyUpdate(room);
    } catch (err) {
      callback?.({ ok: false, error: 'Não foi possível criar a sala. Tenta novamente.' });
    }
  });

  socket.on('join_room', (payload = {}, callback) => {
    const rawCode = String(payload.roomCode || '').trim().toUpperCase();
    const rawName = String(payload.name || '').trim().slice(0, 22);
    const clientId = String(payload.playerId || '').trim() || `p_${socket.id}`;

    if (!rawCode) return callback?.({ ok: false, error: 'Introduz o código da sala.' });
    if (!rawName) return callback?.({ ok: false, error: 'Introduz o teu nome de jogador.' });

    const room = rooms.get(rawCode);
    if (!room) return callback?.({ ok: false, error: 'Sala não encontrada. Verifica o código.' });

    let player = room.players.get(clientId);
    if (!player) {
      player = {
        id: clientId,
        socketId: socket.id,
        name: rawName,
        score: 0,
        correctCount: 0,
        totalResponseMs: 0,
        answeredCurrent: false,
        lastResult: null,
        connected: true
      };
      room.players.set(clientId, player);
    } else {
      player.socketId = socket.id;
      player.name = rawName;
      player.connected = true;
    }

    socket.join(room.code);
    socket.data.roomCode = room.code;
    socket.data.role = 'player';
    socket.data.playerId = clientId;

    const currentMission = room.currentMissionIndex >= 0 ? MISSIONS[room.currentMissionIndex] : null;
    const remainingMs = room.status === 'question'
      ? Math.max(0, room.missionEndTime - Date.now())
      : 0;

    callback?.({
      ok: true,
      roomCode: room.code,
      playerId: player.id,
      name: player.name,
      status: room.status,
      totalMissions: MISSIONS.length,
      secondsPerMission: QUESTION_DURATION_SECONDS,
      currentMission: room.status === 'question' && currentMission
        ? {
            missionNumber: room.currentMissionIndex + 1,
            totalMissions: MISSIONS.length,
            title: currentMission.title,
            zoneName: currentMission.zoneName,
            theme: currentMission.theme,
            backdrop: currentMission.backdrop,
            question: currentMission.question,
            options: currentMission.options,
            durationSeconds: QUESTION_DURATION_SECONDS,
            durationMs: QUESTION_DURATION_MS,
            remainingMs,
            alreadyAnswered: player.answeredCurrent
          }
        : null,
      ranking: buildRankingPayload(room, player.id)
    });

    emitLobbyUpdate(room);
    if (room.status === 'question') {
      emitAnswerProgress(room);
    }
  });

  socket.on('host_start_game', (_, callback) => {
    const room = rooms.get(socket.data.roomCode);
    if (!room || room.hostSocketId !== socket.id) {
      return callback?.({ ok: false, error: 'Apenas o apresentador pode iniciar as missões.' });
    }
    startMission(room, 0);
    callback?.({ ok: true });
  });

  socket.on('host_next_mission', (_, callback) => {
    const room = rooms.get(socket.data.roomCode);
    if (!room || room.hostSocketId !== socket.id) {
      return callback?.({ ok: false, error: 'Apenas o apresentador pode avançar.' });
    }

    if (room.status === 'question') {
      finishCurrentMission(room);
      return callback?.({ ok: true, action: 'closed_mission' });
    }

    const nextIndex = room.currentMissionIndex + 1;
    if (nextIndex < MISSIONS.length) {
      startMission(room, nextIndex);
      return callback?.({ ok: true, action: 'next_mission' });
    }

    finishGame(room);
    return callback?.({ ok: true, action: 'finished' });
  });

  socket.on('submit_answer', (payload = {}, callback) => {
    const room = rooms.get(socket.data.roomCode);
    if (!room || room.status !== 'question') {
      return callback?.({ ok: false, error: 'A missão já encerrou.' });
    }

    const player = room.players.get(socket.data.playerId);
    if (!player) return callback?.({ ok: false, error: 'Jogador não registado.' });
    if (player.answeredCurrent) return callback?.({ ok: false, error: 'Resposta já confirmada.' });

    const now = Date.now();
    const remainingMs = Math.max(0, room.missionEndTime - now);
    if (remainingMs <= 0) return callback?.({ ok: false, error: 'Tempo esgotado.' });

    const laneIndex = Number(payload.laneIndex);
    if (![0, 1, 2].includes(laneIndex)) return callback?.({ ok: false, error: 'Faixa inválida.' });

    const mission = MISSIONS[room.currentMissionIndex];
    const elapsedMs = Math.min(QUESTION_DURATION_MS, Math.max(0, now - room.missionStartTime));
    const isCorrect = laneIndex === mission.correctIndex;

    let pointsEarned = 0;
    if (isCorrect) {
      const speedRatio = remainingMs / QUESTION_DURATION_MS;
      pointsEarned = BASE_POINTS + Math.round(MAX_SPEED_BONUS * speedRatio);
      player.score += pointsEarned;
      player.correctCount += 1;
    }

    player.totalResponseMs += elapsedMs;
    player.answeredCurrent = true;
    player.lastResult = {
      correct: isCorrect,
      timedOut: false,
      statusText: isCorrect ? 'MISSÃO CONCLUÍDA!' : 'MISSÃO FALHADA!',
      subText: isCorrect
        ? `Ultrapassagem concluída na Faixa ${['A', 'B', 'C'][laneIndex]}!`
        : `Saída de trajetória na Faixa ${['A', 'B', 'C'][laneIndex]}.`,
      pointsEarned,
      selectedIndex: laneIndex
    };

    const responsePayload = {
      missionNumber: room.currentMissionIndex + 1,
      totalMissions: MISSIONS.length,
      ...player.lastResult,
      totalScore: player.score,
      ranking: buildRankingPayload(room, player.id)
    };

    socket.emit('answer_result', responsePayload);
    callback?.({ ok: true, ...responsePayload });
    emitAnswerProgress(room);

    const activePlayers = Array.from(room.players.values()).filter((p) => p.connected);
    const allAnswered = activePlayers.length > 0 && activePlayers.every((p) => p.answeredCurrent);
    if (allAnswered) {
      clearMissionTimers(room);
      room.missionTimeout = setTimeout(() => {
        finishCurrentMission(room);
      }, 1500);
    }
  });

  socket.on('disconnect', () => {
    const code = socket.data.roomCode;
    if (!code || !rooms.has(code)) return;
    const room = rooms.get(code);

    if (socket.data.role === 'player' && socket.data.playerId) {
      const player = room.players.get(socket.data.playerId);
      if (player) {
        player.connected = false;
        player.socketId = null;
      }
      emitLobbyUpdate(room);
    }

    if (socket.data.role === 'host' && room.hostSocketId === socket.id) {
      room.hostSocketId = null;
      const anyConnected = Array.from(room.players.values()).some((p) => p.connected);
      if (!anyConnected) {
        clearMissionTimers(room);
        rooms.delete(code);
      }
    }
  });
});

server.listen(PORT, () => {
  console.log(`Capital City | Grupo 16 ativo na porta ${PORT}`);
});
