<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import Peer from 'peerjs';
import { marked } from 'marked';
import { v4 as uuidv4 } from 'uuid';

interface Conversation {
  id: string;
  title: string;
  peerId: string;
  messages: { sender: string; content: string }[];
}

const peerId = ref('');
const remotePeerId = ref('');
const message = ref('');
const conversations = ref<Conversation[]>([]);
const activeConversationId = ref<string | null>(null);
const peer = ref<Peer | null>(null);
const connections = ref<{ [key: string]: Peer.DataConnection }>({});
const errorMessage = ref('');

const activeConversation = computed(() => 
  conversations.value.find(c => c.id === activeConversationId.value) || null
);

onMounted(() => {
  peer.value = new Peer();
  peer.value.on('open', (id) => {
    peerId.value = id;
  });
  peer.value.on('connection', (conn) => {
    setupConnection(conn);
  });
});

function connect() {
  if (peer.value && remotePeerId.value) {
    if (remotePeerId.value === peerId.value) {
      errorMessage.value = "You can't connect to yourself!";
      setTimeout(() => { errorMessage.value = ''; }, 3000);
      return;
    }
    const conn = peer.value.connect(remotePeerId.value);
    setupConnection(conn);
  }
}

function setupConnection(conn: Peer.DataConnection) {
  connections.value[conn.peer] = conn;
  const newConversation: Conversation = { 
    id: uuidv4(), 
    title: `Chat with ${conn.peer.substring(0, 6)}...`, 
    peerId: conn.peer, 
    messages: [] 
  };
  conversations.value.push(newConversation);
  activeConversationId.value = newConversation.id;

  conn.on('data', (data: string) => {
    const conversation = conversations.value.find(c => c.peerId === conn.peer);
    if (conversation) {
      conversation.messages.push({ sender: 'Remote', content: data });
    }
  });
}

function sendMessage() {
  if (activeConversationId.value && connections.value[activeConversation.value!.peerId] && message.value) {
    const conn = connections.value[activeConversation.value!.peerId];
    conn.send(message.value);
    activeConversation.value!.messages.push({ sender: 'You', content: message.value });
    message.value = '';
  }
}

function renderMarkdown(content: string) {
  return marked(content);
}

function newConversation() {
  remotePeerId.value = '';
  activeConversationId.value = null;
}

function deleteConversation(id: string) {
  const index = conversations.value.findIndex(c => c.id === id);
  if (index !== -1) {
    conversations.value.splice(index, 1);
    if (activeConversationId.value === id) {
      activeConversationId.value = conversations.value.length > 0 ? conversations.value[0].id : null;
    }
  }
}

function renameConversation(id: string, newTitle: string) {
  const conversation = conversations.value.find(c => c.id === id);
  if (conversation) {
    conversation.title = newTitle;
  }
}
</script>

<template>
  <div class="chat-app">
    <div class="app-header">
      <h1 class="app-title">Unison</h1>
      <div class="peer-id">Your Peer ID: {{ peerId }}</div>
    </div>
    <div class="chat-grid">
      <div v-if="!activeConversationId" class="connect-form chat-box">
        <h2>Start a New Chat</h2>
        <input v-model="remotePeerId" placeholder="Enter remote Peer ID" />
        <button @click="connect" class="connect-btn">
          <font-awesome-icon icon="link" /> Connect
        </button>
        <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
      </div>
      <div v-for="conv in conversations" :key="conv.id" class="chat-box" :class="{ active: activeConversationId === conv.id }">
        <div class="chat-header">
          <input v-model="conv.title" @blur="renameConversation(conv.id, conv.title)" class="chat-title" />
          <button @click="deleteConversation(conv.id)" class="delete-btn">
            <font-awesome-icon icon="trash" />
          </button>
        </div>
        <div class="messages" @click="activeConversationId = conv.id">
          <div v-for="(msg, index) in conv.messages" :key="index" class="message" :class="msg.sender.toLowerCase()">
            <strong>{{ msg.sender }}:</strong>
            <div v-html="renderMarkdown(msg.content)"></div>
          </div>
        </div>
        <div v-if="activeConversationId === conv.id" class="message-input">
          <textarea v-model="message" placeholder="Type your message (Markdown supported)"></textarea>
          <button @click="sendMessage" class="send-btn">
            <font-awesome-icon icon="paper-plane" />
          </button>
        </div>
      </div>
    </div>
    <button @click="newConversation" class="new-chat-btn">
      <font-awesome-icon icon="plus" /> New Chat
    </button>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;700&display=swap');

.chat-app {
  max-width: 1200px;
  margin: 2rem auto;
  font-family: 'JetBrains Mono', monospace;
  color: #e0e0e0;
  position: relative;
  z-index: 1;
}

.chat-app::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: 
    linear-gradient(to bottom, #0a0a0a, #0a1a0a),
    repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0, 255, 0, 0.03) 2px, rgba(0, 255, 0, 0.03) 4px),
    repeating-linear-gradient(90deg, transparent, transparent 2px, rgba(0, 255, 0, 0.03) 2px, rgba(0, 255, 0, 0.03) 4px);
  z-index: -1;
}

.app-header {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.app-title {
  font-size: 2.5rem;
  color: #2ecc71;
  margin: 0;
}

.peer-id {
  background: rgba(46, 204, 113, 0.1);
  color: #2ecc71;
  padding: 10px;
  border-radius: 8px;
  font-size: 0.9rem;
}

.chat-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.chat-box {
  background: rgba(18, 18, 18, 0.8);
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1), 0 0 0 1px rgba(46, 204, 113, 0.1);
  transition: all 0.3s ease;
  position: relative;
}

.chat-box::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: 
    linear-gradient(135deg, rgba(46, 204, 113, 0.1) 0%, transparent 100%),
    linear-gradient(225deg, rgba(46, 204, 113, 0.1) 0%, transparent 100%);
  opacity: 0.5;
  z-index: 0;
  pointer-events: none;
}

.chat-box.active {
  box-shadow: 0 0 0 2px rgba(46, 204, 113, 0.5), 0 8px 16px rgba(0, 0, 0, 0.2);
  transform: translateY(-4px);
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background: rgba(0, 0, 0, 0.2);
  border-bottom: 1px solid rgba(46, 204, 113, 0.2);
}

.chat-title {
  background: transparent;
  border: none;
  color: #2ecc71;
  font-size: 1rem;
  font-weight: bold;
  flex-grow: 1;
  font-family: 'JetBrains Mono', monospace;
}

.delete-btn {
  background: transparent;
  border: none;
  color: #e0e0e0;
  cursor: pointer;
  transition: color 0.3s ease;
}

.delete-btn:hover {
  color: #ff6b6b;
}

.messages {
  flex-grow: 1;
  padding: 15px;
  overflow-y: auto;
  max-height: 300px;
}

.message {
  margin-bottom: 10px;
  padding: 8px 12px;
  border-radius: 8px;
  max-width: 80%;
  animation: fadeIn 0.3s ease;
  transition: all 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.message.you {
  background-color: rgba(46, 204, 113, 0.1);
  border: 1px solid rgba(46, 204, 113, 0.3);
  align-self: flex-end;
  margin-left: auto;
}

.message.remote {
  background-color: rgba(52, 152, 219, 0.1);
  border: 1px solid rgba(52, 152, 219, 0.3);
  align-self: flex-start;
}

.message-input {
  display: flex;
  padding: 10px;
  background: rgba(0, 0, 0, 0.2);
  border-top: 1px solid rgba(46, 204, 113, 0.2);
}

.message-input textarea {
  flex-grow: 1;
  padding: 8px;
  border: none;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.05);
  color: #e0e0e0;
  resize: none;
  font-family: 'JetBrains Mono', monospace;
  transition: all 0.3s ease;
}

.message-input textarea:focus {
  background-color: rgba(255, 255, 255, 0.1);
  outline: none;
  box-shadow: 0 0 0 2px rgba(46, 204, 113, 0.3);
}

.connect-form {
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.connect-form h2 {
  margin-top: 0;
  color: #2ecc71;
}

.connect-form input {
  padding: 10px;
  border: none;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.05);
  color: #e0e0e0;
  font-family: 'JetBrains Mono', monospace;
  transition: all 0.3s ease;
}

.connect-form input:focus {
  background-color: rgba(255, 255, 255, 0.1);
  outline: none;
  box-shadow: 0 0 0 2px rgba(46, 204, 113, 0.3);
}

.connect-btn, .send-btn, .new-chat-btn {
  padding: 10px 15px;
  background-color: rgba(46, 204, 113, 0.2);
  color: #2ecc71;
  border: 1px solid rgba(46, 204, 113, 0.5);
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-family: 'JetBrains Mono', monospace;
}

.connect-btn:hover, .send-btn:hover, .new-chat-btn:hover {
  background-color: rgba(46, 204, 113, 0.3);
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.connect-btn:active, .send-btn:active, .new-chat-btn:active {
  transform: translateY(0);
  box-shadow: none;
}

.new-chat-btn {
  margin-top: 20px;
  width: 100%;
}

.error-message {
  color: #ff6b6b;
  font-size: 0.9rem;
  margin-top: 10px;
}

@media (max-width: 768px) {
  .chat-grid {
    grid-template-columns: 1fr;
  }
}
</style>