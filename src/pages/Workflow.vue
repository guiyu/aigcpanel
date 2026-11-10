<script setup lang="ts">
import PageHeader from "../components/PageHeader.vue";
</script>

<template>
    <div class="bg-white min-h-full select-none">
        <div class="p-6">
            <PageHeader :title="$t('workflow.title')" />
            <div class="text-sm text-gray-500">
                {{ $t("common.notInBuild") }}
            </div>
        </div>
    </div>
</template>
