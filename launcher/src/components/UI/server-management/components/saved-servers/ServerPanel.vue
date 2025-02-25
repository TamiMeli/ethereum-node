<template>
  <div class="col-start-1 col-span-full row-start-1 row-span-full grid grid-cols-12 grid-rows-12 items-center bg-[#1b1b1d] p-2 rounded-md">
    <div class="col-start-1 col-end-7 row-start-1 row-span-1 flex justify-start items-center">
      <span class="text-md font-semibold text-gray-200 uppercase">{{ $t("multiServer.saveServerCon") }}</span>
    </div>
    <div class="col-start-7 col-span-full row-start-1 row-span-1 flex justify-start items-center relative">
      <label for="Search" class="sr-only"> {{ $t("multiServer.serch") }} </label>

      <input
        id="Search"
        ref="searchInputRef"
        v-model="searchQuery"
        type="text"
        :placeholder="`${t('multiServer.serchFor')}`"
        class="w-full h-8 rounded-md border-gray-200 py-2.5 pe-10 shadow-sm sm:text-sm px-2"
        @mouseenter="footerStore.cursorLocation = `${t('serverList.search')}`"
        @mouseleave="footerStore.cursorLocation = ''"
      />

      <span class="absolute inset-y-0 end-0 grid w-10 place-content-center">
        <button type="button" class="text-gray-600 hover:text-gray-700">
          <span class="sr-only">{{ $t("multiServer.serch") }} </span>

          <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="h-4 w-4">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M21 21l-5.197-5.197m0 0A7.5 7.5 0 105.196 5.196a7.5 7.5 0 0010.607 10.607z"
            />
          </svg>
        </button>
      </span>
    </div>

    <div
      class="w-full h-full max-h-[300px] col-start-1 col-span-full row-start-2 row-end-11 overflow-x-hidden overflow-y-auto flex flex-col justify-start items-center p-1 bg-black rounded-md space-y-2"
    >
      <ServerRow
        v-for="(server, index) in filteredServers"
        :key="server.name"
        :idx="index"
        :server="server"
        @select-server="selectServer"
        @quick-login="quickLogin"
        @mouseenter="footerStore.cursorLocation = `${server.name}`"
        @mouseleave="footerStore.cursorLocation = ''"
      />
    </div>
    <div class="col-start-1 col-span-full row-start-11 row-span-2 self-end grid grid-cols-12 gap-x-2">
      <button
        class="w-full h-[50px] self-end col-start-1 col-span-3 row-start-11 row-span-2 bg-gray-200 rounded-md px-1 flex justify-start items-center shadow-lg shadow-black active:shadow-none active:scale-95 cursor-pointer transition-all duration-200 pr-2 ease-in-out hover:bg-[#336666] text-gray-800 hover:text-gray-100"
        @click="openFileInput"
        @mouseenter="footerStore.cursorLocation = 'click to import server list'"
        @mouseleave="footerStore.cursorLocation = ''"
      >
        <img class="w-16 h-7" src="/img/icon/server-management-icons/import-config.png" alt="Add Icon" />
        <span class="text-sm text-left uppercase font-bold">import</span>
      </button>
      <input ref="fileInput" type="file" class="hidden" accept=".zip" @change="importConnections" />

      <button
        class="w-full h-[50px] self-end col-start-4 col-span-3 row-start-11 row-span-2 bg-gray-200 rounded-md px-1 pr-2 flex justify-start items-center shadow-lg shadow-black active:shadow-none active:scale-95 cursor-pointer transition-all duration-200 ease-in-out hover:bg-[#336666] text-gray-800 hover:text-gray-100"
        @click="exportConnections"
        @mouseenter="footerStore.cursorLocation = `click to export server list`"
        @mouseleave="footerStore.cursorLocation = ''"
      >
        <img class="w-16 h-7" src="/img/icon/server-management-icons/export-config.png" alt="Add Icon" />
        <span class="text-sm text-left uppercase font-bold">export</span>
      </button>
      <button
        class="w-full h-[50px] self-end col-start-10 col-span-full row-start-11 row-span-2 bg-gray-200 rounded-md px-4 py-2 flex justify-start items-center shadow-lg shadow-black active:shadow-none active:scale-95 cursor-pointer space-x-4 transition-all duration-200 ease-in-out hover:bg-[#336666] text-gray-800 hover:text-gray-100"
        @click="serverLogin"
        @mouseenter="footerStore.cursorLocation = `${t('serverList.addServer')}`"
        @mouseleave="footerStore.cursorLocation = ''"
      >
        <img
          class="w-7 h-7 border border-gray-500 bg-teal-500 rounded-full p-1"
          src="/img/icon/server-management-icons/plus.png"
          alt="Add Icon"
        />
        <span class="text-sm text-left uppercase font-bold">new</span>
      </button>

      <div
        class="w-full h-[50px] self-end col-start-7 col-end-10 row-start-11 row-span-2 flex justify-center items-center bg-[#093A4C] rounded-md px-2 py-1 cursor-pointer space-x-2 transition-all duration-200 ease-in-out hover:bg-[#336666] shadow-lg shadow-black active:shadow-none"
        @click="getToStereumPlusLogin"
      >
        <img class="w-full" src="/img/stereumPlus/logo.png" alt="Server Icon" />
      </div>
    </div>
  </div>
</template>
<script setup>
import i18n from "@/includes/i18n";
import ControlService from "@/store/ControlService";
import { useServers } from "@/store/servers";
import { useControlStore } from "@/store/theControl";
import { useFooter } from "@/store/theFooter";
import { onMounted, ref, watch } from "vue";
import ServerRow from "./ServerRow.vue";
import JSZip from "jszip";
import { saveAs } from "file-saver";

const t = i18n.global.t;

const emit = defineEmits(["selectServer", "serverLogin", "quickLogin"]);

const footerStore = useFooter();
const serverStore = useServers();
const controlStore = useControlStore();
const searchQuery = ref("");
const searchInputRef = ref(null);
const filteredServers = ref(null);

const getFilteredServers = () => {
  if (!searchQuery.value) {
    return serverStore.savedServers?.savedConnections;
  }

  return serverStore.savedServers.savedConnections.filter((server) => server.name.toLowerCase().includes(searchQuery.value.toLowerCase()));
};

watch(
  [searchQuery, () => serverStore.refreshServers],
  async ([, refreshTrigger], [, oldRefreshTrigger]) => {
    if (refreshTrigger !== oldRefreshTrigger) {
      await loadStoredConnections();
    }

    filteredServers.value = null;
    setTimeout(() => {
      filteredServers.value = getFilteredServers();
    }, 10);
  },
  { deep: true }
);

//Lifecycle Hooks

onMounted(async () => {
  await loadStoredConnections();
  if (searchInputRef.value) {
    searchInputRef.value.focus();
  }
  filteredServers.value = getFilteredServers();
});

//Methods

const exportConnections = async () => {
  const zip = new JSZip();
  const connectionsJSON = JSON.stringify(serverStore.connections, null, 2);
  zip.file("connections.json", connectionsJSON);

  const blob = await zip.generateAsync({ type: "blob" });

  saveAs(blob, "connections.zip");
};

const fileInput = ref(null);

const openFileInput = () => {
  if (fileInput.value) {
    fileInput.value.click();
  }
};

const importConnections = async (event) => {
  const file = event.target.files[0];
  if (!file) {
    return;
  }

  const zip = new JSZip();
  try {
    const connect = await file.arrayBuffer();
    const zipContent = await zip.loadAsync(connect);

    const connectionsFile = zipContent.file("connections.json");
    if (!connectionsFile) {
      alert("No connections.json file found in the zip.");
      return;
    }

    const jsonData = await connectionsFile.async("string");

    const newConnections = JSON.parse(jsonData);

    if (Array.isArray(newConnections) && newConnections.length > 0) {
      const existingConnections = serverStore.savedServers.savedConnections;

      const uniqueConnections = newConnections.filter(
        (newConnection) =>
          !existingConnections.some((existingConnection) => JSON.stringify(existingConnection) === JSON.stringify(newConnection))
      );

      existingConnections.push(...uniqueConnections);

      const prevConf = await ControlService.readConfig();

      const conf = {
        ...prevConf,
        savedConnections: JSON.parse(JSON.stringify(existingConnections)),
      };

      await ControlService.writeConfig(conf);
    } else {
      console.error("Invalid or empty connections data.");
    }

    await loadStoredConnections(serverStore.savedServers);
  } catch (error) {
    console.error("An error occurred:", error);
  }
};

const getToStereumPlusLogin = () => {
  window.open("https://stereumplus.com/", "_blank");
};

const loadStoredConnections = async () => {
  serverStore.savedServers = await ControlService.readConfig();

  serverStore.selectedServerConnection = serverStore.savedServers?.savedConnections?.find((item) => item.host === controlStore.ipAddress);

  serverStore.refreshServers = false;
};

const selectServer = (server) => {
  emit("selectServer", server);
};

const quickLogin = (server) => {
  emit("quickLogin", server);
};

const serverLogin = () => {
  emit("serverLogin");
};
</script>

<style scoped>
::-webkit-scrollbar {
  width: 5px;
  background-color: transparent;
}

::-webkit-scrollbar-thumb {
  background-color: #7d838a;
  border-radius: 5px;
}
::-webkit-scrollbar-thumb:hover {
  background-color: #599ce9;
}

::-webkit-scrollbar-track {
  background-color: transparent;
}

::-webkit-scrollbar-track-piece {
  background-color: transparent;
}
</style>
