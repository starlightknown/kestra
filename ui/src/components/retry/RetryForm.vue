<template>
    <el-collapse class="collapse retry-collapse">
        <el-collapse-item 
            name="retry" 
            :title="$t('no_code.sections.retry')"
        >
            <template #icon>
                <el-button 
                    @click.prevent.stop="openRetryForm" 
                    type="primary" 
                    :icon="Plus"
                >
                    {{ $t('add') }}
                </el-button>
            </template>

            <div 
                v-if="hasRetry" 
                @click="openRetryForm" 
                class="d-flex my-2 p-2 rounded element"
            >
                <div class="flex-grow-1 label">
                    {{ displayValue }}
                </div>
                <el-button
                    @click.stop.prevent="removeRetry"
                    :icon="DeleteOutline"
                    size="small"
                    class="border-0"
                />
            </div>
        </el-collapse-item>
    </el-collapse>

    <el-drawer
        v-model="isDrawerOpen"
        :title="$t('no_code.sections.retry')"
        direction="rtl"
        size="50%"
        :destroy-on-close="false"
    >
        <div class="retry-form p-4">
            <el-form label-position="top">
                <!-- Full class name input -->
                <el-form-item>
                    <template #label>
                        <span>{{ $t('class') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            string
                        </el-tag>
                    </template>
                    <el-input 
                        v-model="retryClassname" 
                        class="w-100" 
                        disabled
                    />
                </el-form-item>

                <el-form-item required>
                    <template #label>
                        <span>{{ $t('type') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            string
                        </el-tag>
                    </template>
                    <el-select v-model="retryData.type" class="w-100" @change="updateClassName">
                        <el-option value="constant" :label="$t('constant')" />
                        <el-option value="exponential" :label="$t('exponential')" />
                        <el-option value="random" :label="$t('random')" />
                    </el-select>
                </el-form-item>

                <el-form-item>
                    <template #label>
                        <span>{{ $t('behavior') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            string
                        </el-tag>
                    </template>
                    <el-select v-model="retryData.behavior" class="w-100">
                        <el-option value="RETRY_FAILED_TASK" :label="$t('retry_failed_task')" />
                        <el-option value="CREATE_NEW_EXECUTION" :label="$t('create_new_execution')" />
                    </el-select>
                </el-form-item>

                <el-form-item>
                    <template #label>
                        <span>{{ $t('max_retries') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            integer
                        </el-tag>
                    </template>
                    <el-input-number v-model="retryData.maxAttempt" :min="1" class="w-100" />
                </el-form-item>

                <el-form-item>
                    <template #label>
                        <span>{{ $t('interval') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            duration
                        </el-tag>
                    </template>
                    <el-input v-model="retryData.interval" class="w-100" />
                </el-form-item>

                <el-form-item v-if="retryData.type !== 'constant'">
                    <template #label>
                        <span>{{ $t('max_interval') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            duration
                        </el-tag>
                    </template>
                    <el-input v-model="retryData.maxInterval" class="w-100" />
                </el-form-item>

                <el-form-item v-if="retryData.type === 'exponential'">
                    <template #label>
                        <span>{{ $t('delay_factor') }}</span>
                        <el-tag disable-transitions size="small" class="ms-2 type-tag">
                            number
                        </el-tag>
                    </template>
                    <el-input-number v-model="retryData.delayFactor" :min="1" class="w-100" />
                </el-form-item>

                <el-form-item class="mt-4">
                    <el-button type="primary" @click="saveRetry">
                        {{ $t('save') }}
                    </el-button>
                    <el-button @click="isDrawerOpen = false">
                        {{ $t('cancel') }}
                    </el-button>
                </el-form-item>
            </el-form>
        </div>
    </el-drawer>
</template>

<script setup lang="ts">
    import {ref, computed, watch, onMounted} from "vue";
    import {YamlUtils as YAML_UTILS} from "@kestra-io/ui-libs";
    import {DeleteOutline, Plus} from "../code/utils/icons";
    import {useI18n} from "vue-i18n";

    const {t} = useI18n({useScope: "global"});

    const props = defineProps({
        modelValue: {
            type: [String, Object],
            default: null
        }
    });

    const emits = defineEmits(["update:modelValue"]);

    // UI state
    const isDrawerOpen = ref(false);
    
    // Retry data
    const retryData = ref({
        type: "constant",
        behavior: "RETRY_FAILED_TASK",
        maxAttempt: 5,
        interval: "PT1M",
        maxInterval: "PT1H",
        delayFactor: 2,
    });

    const hasRetry = computed(() => {
        return !!props.modelValue;
    });

    const displayValue = computed(() => {
        if (!props.modelValue) return "";
        
        let parsed;
        if (typeof props.modelValue === "string") {
            try {
                parsed = YAML_UTILS.parse(props.modelValue);
                if (!parsed) {
                    return "Invalid configuration";
                }
            } catch {
                return "Invalid configuration";
            }
        } else {
            parsed = props.modelValue;
        }
        
        return `${parsed.type} (${t("max_retries")}: ${parsed.maxAttempt})`;
    });

    const openRetryForm = () => {
        if (props.modelValue) {
            if (typeof props.modelValue === "string") {
                try {
                    const parsed = YAML_UTILS.parse(props.modelValue);
                    if (parsed) {
                        retryData.value = {...getDefaultValues(), ...parsed};
                    }
                } catch {
                    // If parsing fails, use defaults
                    retryData.value = getDefaultValues();
                }
            } else {
                retryData.value = {...getDefaultValues(), ...props.modelValue};
            }
        } else {
            retryData.value = getDefaultValues();
        }
        
        updateClassName();
        isDrawerOpen.value = true;
    };

    const getDefaultValues = () => {
        return {
            type: "constant",
            behavior: "RETRY_FAILED_TASK",
            maxAttempt: 5,
            interval: "PT1M",
            maxInterval: "PT1H",
            delayFactor: 2,
        };
    };

    const saveRetry = () => {
        // Clean up unnecessary properties based on type
        const cleanedData = {...retryData.value};
        
        if (retryData.value.type === "constant") {
            delete cleanedData.maxInterval;
            delete cleanedData.delayFactor;
        } else if (retryData.value.type === "random") {
            delete cleanedData.delayFactor;
        }
        
        // Convert to YAML and emit the updated value
        const yamlData = YAML_UTILS.stringify(cleanedData);
        emits("update:modelValue", yamlData);
        
        isDrawerOpen.value = false;
    };

    const removeRetry = () => {
        emits("update:modelValue", "");
    };

    // Watch for external changes to modelValue
    watch(() => props.modelValue, (newValue) => {
        if (!newValue) {
            retryData.value = getDefaultValues();
        } else if (typeof newValue === "string" && newValue) {
            try {
                const parsed = YAML_UTILS.parse(newValue);
                if (parsed) {
                    retryData.value = {...getDefaultValues(), ...parsed};
                }
            } catch {
                // If parsing fails, keep current state
            }
        } else if (newValue) {
            retryData.value = {...getDefaultValues(), ...newValue};
        }
    });

    const retryClassname = ref("");

    const updateClassName = () => {
        retryClassname.value = `io.kestra.core.models.tasks.retrys.${retryData.value.type.charAt(0).toUpperCase() + retryData.value.type.slice(1)}`;
    };
    
    // Update the class name when opening the form
    watch(() => retryData.value.type, (_unused) => {
        updateClassName();
    });
    
    // Initialize the class name on component creation
    onMounted(() => {
        updateClassName();
    });
</script>

<style scoped lang="scss">
    @import "../code/styles/code.scss";
    
    .retry-collapse {
        margin-bottom: 1rem;
    }
    
    .element {
        cursor: pointer;
        background-color: $code-card-color;
        border: 1px solid $code-border-color;

        & > .label {
            color: inherit;
            font-size: $code-font-sm;
        }
    }
    
    .type-tag {
        background-color: #E9E9EB;
        color: #909399;
    }
    
    .w-100 {
        width: 100%;
    }
</style> 