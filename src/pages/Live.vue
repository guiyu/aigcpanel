<script setup lang="ts">
import { computed, onMounted, ref, watch } from "vue";
import Router from "../router";
import PageHeader from "../components/PageHeader.vue";
import { useLiveStore } from "../store/modules/live";
import { useServerStore } from "../store/modules/server";
import { EnumServerType } from "../types/Server";
import { Dialog } from "../lib/dialog";
import { mapError } from "../lib/error";
import { t } from "../lang";
import { functionToLabels } from "../lib/aigcpanel";

const tab = ref("");
const liveStore = useLiveStore();
const serverStore = useServerStore();
const taking = ref(false);
const selectedKey = ref("");

const syncTab = () => {
    tab.value = (Router.currentRoute.value.query.tab as string) || "monitor";
};

const localModels = computed(() => {
    return serverStore.records.filter(
        (s) =>
            s.type === EnumServerType.LOCAL ||
            s.type === EnumServerType.LOCAL_DIR ||
            s.type === EnumServerType.API,
    );
});

const selectedModel = computed(() => {
    return localModels.value.find((s) => s.key === selectedKey.value);
});

onMounted(() => {
    syncTab();
    selectedKey.value = liveStore.localConfig.serverKey || "";
});

watch(() => Router.currentRoute.value.query.tab, syncTab);

const doTakeover = async () => {
    if (!selectedKey.value) {
        Dialog.tipError(t("live.noLocalModel"));
        return;
    }
    taking.value = true;
    try {
        await serverStore.loadTakeover(selectedKey.value);
        liveStore.localConfig.serverKey = selectedKey.value;
        await liveStore.saveLocalConfig();
        await liveStore.configUpdate();
        Dialog.tipSuccess(t("live.takeoverOk"));
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        taking.value = false;
    }
};
</script>

<template>
    <div class="pb-device-container bg-white h-full relative select-none flex">
        <div
            class="p-6 w-52 flex-shrink-0 border-r border-solid border-gray-100 overflow-x-hidden overflow-y-auto"
        >
            <div
                class="p-2 rounded-lg mb-4 cursor-pointer"
                :class="tab === 'monitor' ? 'bg-gray-200' : ''"
                @click="tab = 'monitor'"
            >
                <div class="text-base flex items-center">
                    <i-mdi-broadcast
                        class="w-5 h-5 inline-block mr-1 text-rose-500"
                    />
                    {{ $t("live.service") }}
                </div>
            </div>
            <div
                class="p-2 rounded-lg mb-4 cursor-pointer"
                :class="tab === 'avatar' ? 'bg-gray-200' : ''"
                @click="tab = 'avatar'"
            >
                <div class="text-base flex items-center">
                    <i-mdi-account-circle
                        class="w-5 h-5 inline-block mr-1 text-blue-500"
                    />
                    {{ $t("live.avatar") }}
                </div>
            </div>
            <div
                class="p-2 rounded-lg mb-4 cursor-pointer"
                :class="tab === 'knowledge' ? 'bg-gray-200' : ''"
                @click="tab = 'knowledge'"
            >
                <div class="text-base flex items-center">
                    <i-mdi-book-open-variant
                        class="w-5 h-5 inline-block mr-1 text-amber-500"
                    />
                    {{ $t("live.knowledge") }}
                </div>
            </div>
            <div
                class="p-2 rounded-lg mb-4 cursor-pointer"
                :class="tab === 'event' ? 'bg-gray-200' : ''"
                @click="tab = 'event'"
            >
                <div class="text-base flex items-center">
                    <i-mdi-chat
                        class="w-5 h-5 inline-block mr-1 text-emerald-500"
                    />
                    {{ $t("live.event") }}
                </div>
            </div>
            <div
                class="p-2 rounded-lg mb-4 cursor-pointer"
                :class="tab === 'liveTalk' ? 'bg-gray-200' : ''"
                @click="tab = 'liveTalk'"
            >
                <div class="text-base flex items-center">
                    <i-mdi-history
                        class="w-5 h-5 inline-block mr-1 text-violet-500"
                    />
                    {{ $t("live.talkHistory") }}
                </div>
            </div>
        </div>
        <div class="flex-grow h-full overflow-y-auto">
            <div class="p-6">
                <PageHeader :title="$t('live.title')" />
                <div class="max-w-xl">
                    <div class="text-sm text-gray-500 mb-4">
                        {{ $t("live.takeoverDesc") }}
                    </div>
                    <div class="mb-3">
                        <div class="text-sm mb-1">
                            {{ $t("live.selectedModel") }}
                        </div>
                        <a-select
                            v-model="selectedKey"
                            :placeholder="$t('model.select')"
                            class="w-full"
                        >
                            <a-option
                                v-for="server in localModels"
                                :key="server.key"
                                :value="server.key"
                            >
                                {{ server.title }} v{{ server.version }}
                            </a-option>
                        </a-select>
                    </div>
                    <div
                        v-if="selectedModel"
                        class="text-xs text-gray-400 mb-3"
                    >
                        <a-tag
                            v-for="label in functionToLabels(
                                selectedModel.functions,
                            )"
                            :key="label"
                            class="mr-1"
                        >
                            {{ label }}
                        </a-tag>
                    </div>
                    <div
                        v-if="!localModels.length"
                        class="text-sm text-gray-400 mb-3"
                    >
                        {{ $t("live.noLocalModel") }}
                    </div>
                    <a-button
                        type="primary"
                        :loading="taking"
                        :disabled="!selectedKey"
                        @click="doTakeover"
                    >
                        {{ $t("live.takeoverAction") }}
                    </a-button>
                    <div
                        v-if="liveStore.server"
                        class="mt-4 text-sm text-gray-600"
                    >
                        {{ liveStore.server.title }}
                        ·
                        {{ liveStore.available ? $t("service.start") : $t("service.starting") }}
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped></style>
