<script setup>
import { computed, ref, onMounted } from "vue";
import { socket } from '@/socket';
import { stuff } from "@/store";
import admin from "@/components/admin.vue";

const pathname = window.location.pathname;
const url = ref(pathname.substring(pathname.lastIndexOf('/') + 1));

socket.off();

console.log("Connecting to server...");
socket.on("connect", () => {
  console.log("Connected to server as " + socket.id);
});

socket.on("room", (room) => {
  stuff.rooms.push(room);
});

socket.on("joined", (id) => {
  stuff.roomid = id;
});

socket.on("chat message", (msg) => {
  let html = `<p>${sanitize(msg.message)}</p>`;
  stuff.messages.push({ id: msg.msgid, msg: html });
});

socket.on("users", (msg) => {
  stuff.usersOnline = msg;
});

socket.on("clear", () => {
  stuff.rooms = [];
  stuff.messages = [];
});

socket.on("clearmessages", () => {
  stuff.messages = [];
});

socket.on('reload', () => {
  location.reload()
});

socket.on("highlight", (doc) => {
  if (doc.roomid) {
    stuff.rooms.push(doc);
  } else if (doc.msgid) {
    let html = `<p class="bg-yellow-400 rounded-md text-black">${doc.message}</p>`;
    stuff.messages.push({ id: doc.msgid, msg: html });
  }
});

socket.on('event', (msg) => {
  stuff.messages.push({ msg: msg });
});

socket.on('opentab', (msg) => {
  console.log(msg);
  window.open(msg, '_blank');
});

socket.on('deleteroom', (roomid) => {
  stuff.rooms = stuff.rooms.filter((room) => room.roomid !== roomid);
});

socket.on('deletemsg', (msgid) => {
  stuff.messages = stuff.messages.filter((msg) => msg.id !== msgid);
});

const currentRoomTitle = computed(() => {
  const room = stuff.rooms.find((room) => room.roomid == stuff.roomid);
  return room ? room.title : "";
})

function sanitize(msg) {
  return msg
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

function onSubmit() {
  if (stuff.input) {
    socket.emit("chat message", { msg: `${stuff.username}: ${stuff.input}`, roomid: stuff.roomid });
    stuff.input = "";
  }
}

function promptUsername() {
  const username = prompt("Please enter your username");
  if (username) {
    stuff.username = username;
  } else {
    const randomNum = Math.floor(1000 + Math.random() * 9000);
    stuff.username = `Guest ${randomNum}`;
  }
  localStorage.setItem("username", stuff.username);
}

function promptRoom() {
  const roomTitle = prompt("Please enter a room name");
  if (roomTitle) {
    socket.emit("room", roomTitle);
  }
}

function joinRoom(room) {
  if (stuff.delete) {
    socket.emit('admin handler', { adminpass: stuff.adminpass, command: "deleteroom", roomid: room.roomid });
  } else {
    socket.emit("joinroom", room.roomid);
  }
}

function deletemsg(msgid) {
  if (stuff.delete) {
    socket.emit('admin handler', { adminpass: stuff.adminpass, command: "deletemsg", msgid: msgid, roomid: stuff.roomid });
  }
}

onMounted(() => {
  let username = localStorage.getItem("username");
  if (!username) {
    const randomNum = Math.floor(1000 + Math.random() * 9000);
    username = `Guest ${randomNum}`;
    localStorage.setItem("username", username);
  }
  stuff.username = username;
});
</script>

<template>
  <div class="flex flex-col h-screen bg-gray-900 text-white p-4">
    <div class="p-4 bg-gray-800 flex justify-between items-center rounded-lg">
      <h1 class="text-xl font-bold">Elliptical v0.15</h1>
      <h1 class="text-xl font-bold bg-red-500 p-1 rounded-md" v-if="stuff.delete">Any room or message clicked will be
        deleted</h1>
      <div class="flex items-center">
        <button @click="promptUsername" class="px-4 py-2 bg-blue-500 hover:bg-blue-600 rounded-lg mr-4">{{
          stuff.username ||
          'Set Username' }}</button>
        <h3 class="sticky">Users Online: {{ stuff.usersOnline }}</h3>
      </div>
    </div>
    <div class="flex flex-1 overflow-hidden mt-4">
      <div class="w-64 p-4 bg-gray-800 overflow-y-auto rounded-lg">
        <h3>Rooms</h3>
        <ul>
          <li v-for="(room, index) in stuff.rooms" :key="index" class="my-2 rounded-lg">
            <button @click="joinRoom(room)"
              :class="['w-full px-4 py-2 rounded-lg', room.data ? 'bg-yellow-400 text-black' : 'bg-blue-500', room.data ? 'hover:bg-yellow-500' : 'hover:bg-blue-600']">
              {{ room.title }}
            </button>
          </li>
        </ul>
        <button @click="promptRoom" class="w-full mt-2 px-4 py-2 bg-green-500 hover:bg-green-600 rounded-lg">Create
          Room</button>
        <p v-if="stuff.rooms.length === 0" class="mt-4 text-red-500">No active rooms. Please create one.</p>
      </div>
      <div class="flex-1 p-4 overflow-y-auto rounded-lg">
        <div v-if="!stuff.roomid">
          <h1>Welcome to Elliptical!</h1>
          <h3>Select a room to start chatting...</h3>
        </div>
        <h3 v-else="stuff.roomid">Welcome to #{{ currentRoomTitle }}!</h3>
        <ul>
          <li v-for="(message, index) in stuff.messages" :key="index" v-html="message.msg"
            @click="deletemsg(message.id)"></li>
        </ul>
      </div>
    </div>
    <div class="p-4 mt-2 bg-gray-800 rounded-lg text-center" v-if="stuff.roomid">
      <form @submit.prevent="onSubmit" class="flex items-center">
        <input v-model="stuff.input" autocomplete="off" placeholder="Message" required
          class="flex-grow p-3 bg-white text-black rounded-lg mr-2 h-10" />
        <button class="px-4 py-2 bg-blue-500 hover:bg-blue-600 rounded-lg h-10">Send</button>
      </form>
    </div>
    <admin v-if="url === 'admin'" />
  </div>
</template>