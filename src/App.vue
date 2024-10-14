<script setup lang="ts">
import { onMounted, ref } from "vue";

declare const chrome: any;

interface Credential {
  id: number;
  siteName: string;
  username: string;
  password: string;
}

const credentials = ref<Credential[]>([]);
const newCredential = ref<Credential>({
  id: 0,
  siteName: "",
  username: "",
  password: "",
});
const isEditing = ref(false);
const editingId = ref<number | null>(null);

onMounted(() => {
  loadCredentials();
});

function loadCredentials() {
  chrome.storage.sync.get(["credentials"], (result: any) => {
    try {
      const storedCredentials = JSON.parse(result.credentials || "[]");
      credentials.value = Array.isArray(storedCredentials)
        ? storedCredentials
        : [];
    } catch (error) {
      console.error("Error parsing stored credentials:", error);
      credentials.value = [];
    }
  });
}

function saveCredential() {
  if (isEditing.value) {
    const index = credentials.value.findIndex(
      (cred) => cred.id === editingId.value
    );
    if (index !== -1) {
      credentials.value[index] = {
        ...newCredential.value,
        id: editingId.value!,
      };
    }
    isEditing.value = false;
    editingId.value = null;
  } else {
    const newId = Date.now();
    credentials.value = [
      ...credentials.value,
      { ...newCredential.value, id: newId },
    ];
  }
  chrome.storage.sync.set({ credentials: JSON.stringify(credentials.value) });
  newCredential.value = { id: 0, siteName: "", username: "", password: "" };
}

function editCredential(cred: Credential) {
  newCredential.value = { ...cred };
  isEditing.value = true;
  editingId.value = cred.id;
}

function deleteCredential(id: number) {
  credentials.value = credentials.value.filter((cred) => cred.id !== id);
  chrome.storage.sync.set({ credentials: JSON.stringify(credentials.value) });
}

function applyCredential(cred: Credential) {
  chrome.tabs.query({ active: true, lastFocusedWindow: true }, (tabs: any) => {
    const currentTab = tabs[0];
    console.log({
      currentTab,
      cred,
      url: currentTab.url,
    });
    if (
      currentTab &&
      currentTab.url &&
      currentTab.url.startsWith("http://localhost")
    ) {
      chrome.tabs.sendMessage(currentTab.id!, {
        action: "fillCredentials",
        username: cred.username,
        password: cred.password,
      });
      console.log("Credentials applied successfully.");
    } else {
      console.log("This feature only works on localhost sites.");
    }
  });
}
</script>

<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Localhost Password Storage</h1>

    <form @submit.prevent="saveCredential" class="mb-4">
      <input
        v-model="newCredential.siteName"
        placeholder="Site Name"
        required
        class="w-full mb-2 p-2 border rounded"
      />
      <input
        v-model="newCredential.username"
        placeholder="Username"
        required
        class="w-full mb-2 p-2 border rounded"
      />
      <input
        v-model="newCredential.password"
        type="password"
        placeholder="Password"
        required
        class="w-full mb-2 p-2 border rounded"
      />
      <button type="submit" class="w-full bg-blue-500 text-white p-2 rounded">
        {{ isEditing ? "Update" : "Save" }} Credential
      </button>
    </form>

    <ul>
      <li
        v-for="cred in credentials"
        :key="cred.id"
        class="mb-2 p-2 border rounded"
      >
        <div>
          <strong>{{ cred.siteName }}</strong>
          <br />
          Username: {{ cred.username }}
        </div>
        <div class="mt-2">
          <button
            @click="editCredential(cred)"
            class="bg-yellow-500 text-white p-1 rounded mr-2"
          >
            Edit
          </button>
          <button
            @click="deleteCredential(cred.id)"
            class="bg-red-500 text-white p-1 rounded mr-2"
          >
            Delete
          </button>
          <button
            @click="applyCredential(cred)"
            class="bg-green-500 text-white p-1 rounded"
          >
            Apply
          </button>
        </div>
      </li>
    </ul>
  </div>
</template>
