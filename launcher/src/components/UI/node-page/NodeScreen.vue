<template>
  <base-layout>
    <!-- Start Node main layouts -->
    <div class="w-full h-full grid grid-cols-24 relative">
      <div class="col-start-1 col-span-1 flex justify-center items-center">
        <SidebarSection />
      </div>
      <div class="col-start-2 col-end-17 w-full h-full relative">
        <NodeSection @open-expert="openExpertModal" @open-log="openLogPage" />
        <ExpertWindow v-if="isExpertModeOpen" :item="expertModeClient" @hide-modal="closeExpertMode" />
      </div>
      <div class="col-start-17 col-end-21 ml-1">
        <ServiceSection @open-expert="openExpertModal" @open-logs="openLogPage" />
      </div>
      <div class="col-start-21 col-end-25 px-1 flex flex-col justify-between">
        <div class="h-[60px] self-center w-full flex flex-col justify-center items-center">
          <button
            class="w-full h-[34px] rounded-full bg-[#264744] hover:bg-[#325e5a] px-2 py-1 text-gray-200 active:scale-95 shadow-md shadow-zinc-800 active:shadow-none transition-all duration-200 ease-in-out uppercase flex justify-center items-center"
            @click="alarmToggle"
            @mouseenter="
              footerStore.cursorLocation = nodeStore.infoAlarm
                ? `${$t('nodeSidebarVideo.stereumTutorial')}`
                : `${$t('nodeSidebarVideo.statBox')}`
            "
            @mouseleave="footerStore.cursorLocation = ''"
          >
            <img class="w-8" src="/img/icon/node-page-icons/access-tutorial-icon.png" alt="information" />
          </button>
        </div>
        <AlertSection :info-aralm="nodeStore.infoAlarm" />
      </div>
      <LogsSection
        v-if="isLogsPageActive"
        :client="nodeStore.clientToLogs"
        @close-log="closeLogPage"
        @export-log="exportLogs"
      />
    </div>

    <!-- End Node main layout -->
  </base-layout>
</template>
<script setup>
import SidebarSection from "./sections/SidebarSection";
import NodeSection from "./sections/NodeSection.vue";
import ServiceSection from "./sections/ServiceSection.vue";
import AlertSection from "./sections/AlertSection.vue";
import LogsSection from "./sections/LogsSection.vue";
import { ref, onMounted, onUnmounted, watchEffect } from "vue";
import ExpertWindow from "./sections/ExpertWindow.vue";
import { useNodeStore } from "@/store/theNode";
import ControlService from "@/store/ControlService";
import { useServices } from "@/store/services";
import { useNodeHeader } from "@/store/nodeHeader";
import { useControlStore } from "@/store/theControl";
import { useRefreshNodeStats } from "../../../composables/monitoring";
import { useListKeys } from "../../../composables/validators";
import { useRouter } from "vue-router";
import { useFooter } from "@/store/theFooter";
import { saveAs } from "file-saver";

const expertModeClient = ref(null);
const isExpertModeOpen = ref(false);
const isLogsPageActive = ref(false);
const refreshStats = ref(false);

let polling = null;
let pollingVitals = null;
let pollingNodeStats = null;
let pollingListingKeys = null;

const nodeStore = useNodeStore();
const headerStore = useNodeHeader();
const serviceStore = useServices();
const controlStore = useControlStore();
const router = useRouter();
const footerStore = useFooter();

//Computed & Watchers

watchEffect(() => {
  if (router.currentRoute.value.path !== "/node") {
    nodeStore.isLineHidden = true;
  } else {
    nodeStore.isLineHidden = false;
  }
});

watchEffect(() => {
  if (headerStore.openModalFromNodeAlert) {
    nodeStore.isLineHidden = true;
    expertModeClient.value = headerStore.selectedValidatorFromNodeAlert;
    expertModeClient.value.expertOptionsModal = true;
    isExpertModeOpen.value = true;
  }
});

watchEffect(() => {
  if (refreshStats.value) {
    updateConnectionStats();
    refreshStats.value = false;
  }
});

//Lifecycle Hooks
onMounted(() => {
  setTimeout(() => {
    refreshStats.value = true;
  }, 2000);

  updateConnectionStats();

  updateServiceLogs();
  polling = setInterval(updateServiceLogs, 10000); // refresh logs
  pollingVitals = setInterval(updateServerVitals, 1000); // refresh server vitals
  pollingNodeStats = setInterval(updateNodeStats, 1000); // refresh server vitals
  pollingListingKeys = setInterval(checkForListingKeys, 1000); // check for validators which need validator listing
});

onUnmounted(() => {
  clearInterval(polling);
  clearInterval(pollingVitals);
  clearInterval(pollingNodeStats);
  clearInterval(pollingListingKeys);
});

//Methods

const alarmToggle = () => {
  nodeStore.infoAlarm = !nodeStore.infoAlarm;
  footerStore.cursorLocation = "";
};

const checkForListingKeys = async () => {
  //is true when there is at least one validator service running without keys
  if (
    serviceStore.installedServices &&
    serviceStore.installedServices.length > 0 &&
    serviceStore.installedServices.some(
      (s) => s.category === "validator" && s.state === "running" && (!s.config.keys || !s.config.keys.length > 0)
    )
  ) {
    clearInterval(pollingListingKeys);
    useListKeys();
  }
};

const updateConnectionStats = async () => {
  const stats = await ControlService.getConnectionStats();
  controlStore.ServerName = stats.ServerName;
  controlStore.ipAddress = stats.ipAddress;
};
const updateServiceLogs = async () => {
  if (serviceStore.installedServices && serviceStore.installedServices.length > 0 && headerStore.refresh) {
    const data = await ControlService.getServiceLogs();
    nodeStore.serviceLogs = data;
  }
};
const updateServerVitals = async () => {
  try {
    if (serviceStore.installedServices && serviceStore.installedServices.length > 0 && headerStore.refresh) {
      const data = await ControlService.getServerVitals();
      controlStore.cpu = data.cpu;
      controlStore.availDisk = data.availDisk;
      controlStore.usedPerc = data.usedPerc;
    }
  } catch (e) {
    console.log("couldn't check server vitals");
  }
};
const openExpertModal = (item) => {
  nodeStore.isLineHidden = true;
  expertModeClient.value = item;
  expertModeClient.value.expertOptionsModal = true;
  isExpertModeOpen.value = true;
};

const updateNodeStats = async () => {
  await useRefreshNodeStats();
  nodeStore.isLineHidden = false;
};

const closeExpertMode = () => {
  isExpertModeOpen.value = false;
  nodeStore.isLineHidden = false;
  headerStore.openModalFromNodeAlert = false;
  headerStore.selectedValidatorFromNodeAlert = null;
};
// ********** LOGS **********

const exportLogs = async (client) => {
  const currentService = nodeStore.serviceLogs.find(
    (service) => service.config?.serviceID === client.config?.serviceID
  );

  const fileName = nodeStore.exportLogs ? `${client.name}_150_logs.txt` : `${client.name}_all_logs.txt`;

  // Select the data based on the condition
  const data = nodeStore.exportLogs ? currentService.logs.slice(-150).reverse() : currentService.logs.reverse();

  const lineByLine = data.map((line, index) => `#${data.length - index}: ${line}`).join("\n\n");
  const blob = new Blob([lineByLine], { type: "text/plain;charset=utf-8" });
  saveAs(blob, fileName);
};

const openLogPage = (item) => {
  nodeStore.clientToLogs = item;
  isLogsPageActive.value = true;
};

const closeLogPage = () => {
  nodeStore.clientToLogs = null;
  isLogsPageActive.value = false;
};
</script>

<style scoped>
.info-button {
  width: 98%;
  height: 7%;
  background: #264744;
  border-radius: 20px;
  box-shadow: 0 1px 3px 0px #1c1f22;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  margin-top: 3px;
  margin-bottom: 15px;
}
.info-button:hover {
  background: rgb(43, 84, 81);
}
.info-button:active {
  background: rgba(43, 84, 81, 0.5);
}
.info-button img {
  max-width: 19%;
}
</style>
