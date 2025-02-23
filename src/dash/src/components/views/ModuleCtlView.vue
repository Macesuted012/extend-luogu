<script setup lang="ts">
import { inject, ref } from 'vue'
import type { ModulesReadonly, ExecuteState } from '@core/types'
import {
    kModuleCtl,
    kShowConfig,
    kShowInterface
} from '@/utils/injectionSymbols'
import Await from '@comp/utils/Await.vue'
import TextCheckbox from '@comp/utils/TextCheckbox.vue'

const emits = defineEmits<{
    (e: 'uninstallModule', id: string): void
}>()

const moduleCtl = inject(kModuleCtl)!
const showConfig = inject(kShowConfig)!
const showInterface = inject(kShowInterface)!
const { utils, schemas, modules } = window.exlg

const modulesRo = ref<ModulesReadonly | null>()

function toggleModule(id: string) {
    moduleCtl.storage.do(id, (mod) => {
        mod.active = !mod.active
        return mod
    })
}

function updateModuleCache() {
    modulesRo.value = moduleCtl.storage.getAll()
}

function uninstall(id: string) {
    utils.simpleAlert(`确定要删除模块 ${id}？`, {
        onAccept: () => {
            moduleCtl.storage.del(id)
            emits('uninstallModule', id)
            updateModuleCache()
        }
    })
}

const executeStateIcons: Record<ExecuteState, string> = {
    done: '✨',
    threw: '💥',
    inactive: '❄️',
    mismatched: '🌙',
    storageBroken: '💥',
    notExported: '💥',
    unwrapThrew: '💥'
}

const executeStateTexts: Record<ExecuteState, string> = {
    done: '已加载',
    threw: '出错了',
    inactive: '未开启',
    mismatched: '未匹配',
    storageBroken: '数据错误',
    notExported: '无导出',
    unwrapThrew: '解包错误'
}

updateModuleCache()

defineExpose({
    updateModuleCache
})

const showId = ref(false)
</script>

<template>
    <div class="root">
        <div>
            <TextCheckbox text="🆔" title="显示 ID" v-model="showId" />
            <ul class="module-list">
                <li v-for="mod of modulesRo" :key="mod.id" class="module-entry">
                    <span>
                        <Await
                            :promise="modules[mod.id]?.runtime?.executeState"
                        >
                            <template #first>🕒</template>
                            <template #then="{ result }">
                                <!-- FIXME: <https://segmentfault.com/q/1010000042083565> -->
                                <span
                                    class="execute-state exlg-tooltip"
                                    :data-exlg-tooltip="
                                        /* @ts-expect-error */
                                        executeStateTexts[result]
                                    "
                                >
                                    {{
                                        /* @ts-expect-error */
                                        executeStateIcons[result]
                                    }}
                                </span>
                            </template>
                        </Await>
                        {{ showId ? mod.id : mod.metadata.display }}
                        <span class="module-version">
                            @{{ mod.metadata.version }}
                        </span>
                    </span>
                    <span style="white-space: nowrap">
                        <span
                            v-if="
                                Object.keys(modules[mod.id].runtime.interfaces)
                                    .length
                            "
                            class="emoji-button"
                            title="执行命令"
                            @click="showInterface(mod.id)"
                        >
                            💈
                        </span>
                        <span
                            v-if="schemas[mod.id]"
                            class="emoji-button"
                            title="配置"
                            @click="showConfig(mod.id)"
                        >
                            ⚙️
                        </span>
                        <span
                            class="emoji-button"
                            title="卸载"
                            @click="uninstall(mod.id)"
                        >
                            🗑️
                        </span>
                        <input
                            class="exlg-checkbox"
                            type="checkbox"
                            :checked="mod.active"
                            @change="toggleModule(mod.id)"
                        />
                    </span>
                </li>
            </ul>
        </div>
    </div>
</template>

<style>
.module-list {
    list-style: none;
    padding: 0;
}

.module-entry {
    display: flex;
    justify-content: space-between;
}

.module-version {
    color: var(--accent-color);
}

.emoji-button {
    user-select: none;
    cursor: pointer;
}

.execute-state {
    user-select: none;
}
.execute-state:after {
    right: 100%;
    width: max-content;
}
</style>
