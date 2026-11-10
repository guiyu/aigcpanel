<script setup lang="ts">
import { computed, ref } from "vue";
import { AppConfig } from "../../config";
import { t } from "../../lang";
import { functionToLabels } from "../../lib/aigcpanel";
import { Modal } from "@arco-design/web-vue";
import { Dialog } from "../../lib/dialog";
import { isDev } from "../../lib/env";
import { mapError } from "../../lib/error";
import { VersionUtil } from "../../lib/util";
import { useServerStore } from "../../store/modules/server";
import {
    EnumServerStatus,
    EnumServerType,
    ServerRecord,
} from "../../types/Server";
import ServerActionLog from "./ServerActionLog.vue";

/**
 * 开源版能在本机做的数字人模型：部署（导入 config.json）、运维（启停/接管/日志）、管理（目录/卸载）。
 * 云端模型市场和 Pro 工作流不在这里。
 */
const CATALOG = [
    {
        name: "MuseTalk",
        repo: "https://github.com/TMElyralab/MuseTalk",
        descKey: "model.dhMuseTalk",
    },
    {
        name: "LatentSync",
        repo: "https://github.com/bytedance/LatentSync",
        descKey: "model.dhLatentSync",
    },
    {
        name: "Wav2Lip",
        repo: "https://github.com/Rudrabha/Wav2Lip",
        descKey: "model.dhWav2Lip",
    },
    {
        name: "Heygem",
        repo: "https://github.com/duixcom/Duix.Heygem",
        descKey: "model.dhHeygem",
    },
];

const serverStore = useServerStore();
const deploying = ref(false);
const busyKey = ref("");

const installed = computed(() => {
    return serverStore.records.filter((record) => {
        if (
            record.type !== EnumServerType.LOCAL &&
            record.type !== EnumServerType.LOCAL_DIR
        ) {
            return false;
        }
        const functions = Array.isArray(record.functions)
            ? record.functions
            : [];
        return functions.includes("videoGen");
    });
});

const functionNames = (raw: any): string[] => {
    if (Array.isArray(raw)) {
        return raw.map((item) => String(item));
    }
    if (raw && typeof raw === "object") {
        return Object.keys(raw);
    }
    return [];
};

const statusText = (record: ServerRecord) => {
    const runtime = record.runtime?.status;
    const status = runtime || record.status;
    if (status === EnumServerStatus.RUNNING) {
        return t("status.running");
    }
    if (status === EnumServerStatus.STARTING) {
        return t("service.starting");
    }
    if (status === EnumServerStatus.ERROR) {
        return t("model.dhStatusError");
    }
    if (status === EnumServerStatus.PREPARING) {
        return t("status.preparingWeights");
    }
    return t("status.stopped");
};

const deployFromConfig = async () => {
    const configPath = await $mapi.file.openFile({
        filters: [{ name: "config.json", extensions: ["json"] }],
    });
    if (!configPath) {
        return;
    }
    if (!/^[a-zA-Z0-9\/:\-\\._]+$/.test(configPath)) {
        const examplePath = $mapi.app.isPlatform("win")
            ? "D:\\aigcpanel\\models"
            : "/aigcpanel/models";
        Dialog.tipError(
            t("error.modelPathInvalid", {
                path: configPath,
                example: examplePath,
            }),
        );
        return;
    }
    deploying.value = true;
    try {
        const content = await $mapi.file.read(configPath, {
            isDataPath: false,
        });
        const json = JSON.parse(content);
        const functions = functionNames(json.functions);
        if (!functions.includes("videoGen")) {
            Dialog.tipError(t("model.dhNotVideoGen"));
            return;
        }
        const name = json.name || "";
        const version = json.version || "";
        if (!name || !version) {
            Dialog.tipError(t("error.modelDirIdentifyFailed"));
            return;
        }
        const exists = await serverStore.getByNameVersion(name, version);
        if (exists) {
            Dialog.tipError(t("error.modelVersionExists"));
            return;
        }
        const serverPath = configPath.replace(/[\/\\]config.json$/, "");
        const target = await $mapi.file.fullPath(`model/${name}-${version}`);
        if (await $mapi.file.exists(target)) {
            Dialog.tipError(t("error.modelVersionExists"));
            return;
        }
        const isPlaceholder = (v: string) => !v || v.startsWith("__");
        const platformName = json.platformName || "";
        const platformArch = json.platformArch || "";
        if (
            !isPlaceholder(platformName) &&
            $mapi.app.platformName() !== platformName &&
            !isDev
        ) {
            Dialog.tipError(t("error.modelPlatformMismatch"));
            return;
        }
        if (
            !isPlaceholder(platformArch) &&
            $mapi.app.platformArch() !== platformArch &&
            !isDev
        ) {
            Dialog.tipError(t("error.modelArchMismatch"));
            return;
        }
        if (!VersionUtil.match(AppConfig.version, json.serverRequire || "*")) {
            Dialog.tipError(t("error.softwareVersionMismatch"));
            return;
        }
        await serverStore.add({
            key: serverStore.generateServerKey({ name, version } as any),
            name,
            title: json.title || name,
            version,
            type: EnumServerType.LOCAL_DIR,
            autoStart: json.entry === "__EasyServer__",
            functions,
            localPath: serverPath,
            settings: json.settings || [],
            setting: json.setting || {},
            config: json,
        } as ServerRecord);
        Dialog.tipSuccess(t("model.dhDeployed"));
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        deploying.value = false;
    }
};

const doStart = async (record: ServerRecord) => {
    busyKey.value = record.key;
    try {
        if (record.autoStart) {
            await serverStore.loadTakeover(record.key);
        } else {
            await serverStore.start(record);
        }
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        busyKey.value = "";
    }
};

const doStop = async (record: ServerRecord) => {
    busyKey.value = record.key;
    try {
        await serverStore.stop(record);
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        busyKey.value = "";
    }
};

const doTakeover = async (record: ServerRecord) => {
    busyKey.value = record.key;
    try {
        await serverStore.loadTakeover(record.key);
        Dialog.tipSuccess(t("live.takeoverOk"));
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        busyKey.value = "";
    }
};

const doOpenDir = (record: ServerRecord) => {
    if (!record.localPath) {
        return;
    }
    $mapi.app.openPath(record.localPath);
};

const askRemove = () => {
    return new Promise<boolean>((resolve) => {
        Modal.confirm({
            title: t("common.tip"),
            content: t("model.dhRemoveConfirm"),
            okText: t("common.confirm"),
            cancelText: t("common.cancel"),
            onOk: () => resolve(true),
            onCancel: () => resolve(false),
        });
    });
};

const doRemove = async (record: ServerRecord) => {
    if (!(await askRemove())) {
        return;
    }
    busyKey.value = record.key;
    try {
        await serverStore.delete(record);
        Dialog.tipSuccess(t("model.dhRemoved"));
    } catch (e) {
        Dialog.tipError(mapError(e));
    } finally {
        busyKey.value = "";
    }
};

const openRepo = (url: string) => {
    window.$mapi.app.openExternal(url);
};
</script>

<template>
    <div class="mb-5 rounded-xl border border-gray-100 p-4">
        <div class="flex items-start gap-3">
            <div class="flex-grow">
                <div class="text-base font-medium">
                    {{ $t("model.dhTitle") }}
                </div>
                <div class="mt-1 text-xs leading-5 text-gray-400">
                    {{ $t("model.dhDesc") }}
                </div>
            </div>
            <a-button type="primary" :loading="deploying" @click="deployFromConfig">
                <template #icon>
                    <icon-plus />
                </template>
                {{ $t("model.dhDeploy") }}
            </a-button>
        </div>

        <div class="mt-3 grid grid-cols-2 gap-2 lg:grid-cols-4">
            <button
                v-for="item in CATALOG"
                :key="item.name"
                type="button"
                class="rounded-lg bg-gray-50 px-3 py-2 text-left hover:bg-gray-100"
                @click="openRepo(item.repo)"
            >
                <div class="text-sm">{{ item.name }}</div>
                <div class="text-xs text-gray-400">{{ $t(item.descKey) }}</div>
            </button>
        </div>

        <div v-if="!installed.length" class="mt-4 text-sm text-gray-400">
            {{ $t("model.dhEmpty") }}
        </div>
        <div v-else class="mt-4 flex flex-col gap-2">
            <div
                v-for="record in installed"
                :key="record.key"
                class="flex flex-wrap items-center gap-2 rounded-lg border border-gray-100 px-3 py-2"
            >
                <div class="min-w-0 flex-grow">
                    <div class="truncate text-sm">
                        {{ record.title }}
                        <span class="ml-1 rounded-full bg-gray-100 px-2 text-xs">
                            v{{ record.version }}
                        </span>
                    </div>
                    <div class="truncate text-xs text-gray-400">
                        {{ record.localPath }}
                        ·
                        {{ statusText(record) }}
                        ·
                        {{ functionToLabels(record.functions || []).join(" / ") }}
                    </div>
                </div>
                <a-button
                    size="small"
                    :loading="busyKey === record.key"
                    @click="doStart(record)"
                >
                    {{ $t("service.start") }}
                </a-button>
                <a-button size="small" @click="doStop(record)">
                    {{ $t("common.stopService") }}
                </a-button>
                <a-button
                    size="small"
                    type="primary"
                    :loading="busyKey === record.key"
                    @click="doTakeover(record)"
                >
                    {{ $t("live.takeoverAction") }}
                </a-button>
                <ServerActionLog :record="record" />
                <a-button size="small" @click="doOpenDir(record)">
                    {{ $t("common.openPath") }}
                </a-button>
                <a-button size="small" status="danger" @click="doRemove(record)">
                    {{ $t("common.delete") }}
                </a-button>
            </div>
        </div>
    </div>
</template>
