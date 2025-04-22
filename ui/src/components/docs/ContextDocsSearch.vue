<template>
    <div class="search-container" ref="searchContainer">
        <el-input
            v-model="searchQuery"
            :placeholder="t('search_docs')"
            class="search-input"
            @input="handleSearch"
            @keydown.enter.prevent="handleEnterKey"
            @keydown.up.prevent="handleKeyUp"
            @keydown.down.prevent="handleKeyDown"
            :loading="loading"
        >
            <template #prefix>
                <el-icon class="search-icon">
                    <Search />
                </el-icon>
            </template>
        </el-input>
        <div v-if="loading" class="loading-indicator">
            {{ t('searching') }}
        </div>
        <div v-if="showResults" class="search-results">
            <template v-if="searchResults.length > 0">
                <context-docs-link
                    v-for="(result, index) in searchResults"
                    :key="result.url"
                    class="search-result"
                    :class="{'selected': index === selectedIndex}"
                    :href="result.parsedUrl.replace(/^docs\//, '')"
                    use-raw
                    :data-index="index"
                    @click="resetSearch"
                >
                    <h4 class="result-title">
                        {{ result.title }}
                    </h4>
                    <p class="result-preview">
                        {{ result.preview }}
                    </p>
                </context-docs-link>
            </template>
            <div v-else class="no-results">
                {{ t("no_results_found") }}
            </div>
        </div>
    </div>
</template>

<script setup>
    import {ref, computed, onMounted, onUnmounted} from "vue";
    import {useStore} from "vuex";
    import {useI18n} from "vue-i18n";
    import {Search} from "@element-plus/icons-vue";
    import ContextDocsLink from "./ContextDocsLink.vue";

    const {t} = useI18n({useScope: "global"});
    const store = useStore();

    const searchQuery = ref("");
    const searchResults = ref([]);
    const loading = ref(false);
    const selectedIndex = ref(0);
    const searchContainer = ref(null);
    let searchTimeout = null;

    const showResults = computed(() => {
        return searchQuery.value.trim().length > 0;
    });

    const getTitle = (path) => {
        const parts = path.split("/");
        const lastPart = parts[parts.length - 1];
        return lastPart
            .split("-")
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(" ");
    };

    const handleKeyUp = (e) => {
        e.preventDefault();
        if (searchResults.value.length > 0) {
            selectedIndex.value = Math.max(0, selectedIndex.value - 1);
        }
    };

    const handleKeyDown = (e) => {
        e.preventDefault();
        if (searchResults.value.length > 0) {
            selectedIndex.value = Math.min(searchResults.value.length - 1, selectedIndex.value + 1);
        }
    };

    const handleEnterKey = (e) => {
        e.preventDefault();
        if (searchResults.value.length > 0) {
            const selectedResult = document.querySelector(`.search-result[data-index="${selectedIndex.value}"]`);
            if (selectedResult) {
                selectedResult.click();
            }
        }
    };

    const resetSearch = () => {
        searchQuery.value = "";
        searchResults.value = [];
    };

    const handleSearch = async () => {
        if (!searchQuery.value) {
            searchResults.value = [];
            selectedIndex.value = 0;
            return;
        }
        if (searchTimeout) {
            clearTimeout(searchTimeout);
        }
        searchTimeout = setTimeout(async () => {
            try {
                loading.value = true;
                const query = searchQuery.value.trim();
                const results = await store.dispatch("doc/search", query);
                
                const processedResults = (results || [])
                    .map(result => ({
                        ...result,
                        title: getTitle(result.parsedUrl),
                    }))
                    .slice(0, 10);
                searchResults.value = processedResults;
                selectedIndex.value = 0;
            } catch (error) {
                console.error("Error searching docs:", error);
                searchResults.value = [];
                selectedIndex.value = 0;
            } finally {
                loading.value = false;
            }
        }, 500);
    };

    const handleClickOutside = (event) => {
        if (searchContainer.value && !searchContainer.value.contains(event.target)) {
            resetSearch();
        }
    };

    onMounted(() => {
        document.addEventListener("click", handleClickOutside);
    });

    onUnmounted(() => {
        document.removeEventListener("click", handleClickOutside);
        if (searchTimeout) {
            clearTimeout(searchTimeout);
        }
    });
</script>

<style lang="scss" scoped>
    .search-container {
        position: relative;
        margin-bottom: 0;
        z-index: 1001;
    }

    .search-input {
        width: 100%;
        padding: 0.5rem;
        border: 1px solid var(--border-color);
        border-radius: 6px;
        font-size: 0.9rem;
        transition: border-color 0.2s;

        :deep(.el-input__wrapper) {
            background-color: #1e1e2e;
            box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.1);
            border-radius: 6px;
            padding: 0.5rem;

            &.is-focus {
                box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.2);
            }
        }

        :deep(.el-input__inner) {
            color: var(--ks-content-primary);
            font-size: 14px;
            height: 1.25rem;
            background: transparent;
            
            &::placeholder {
                color: rgba(255, 255, 255, 0.5);
            }
        }

        :deep(.el-input__prefix) {
            margin-right: 0.5rem;
            .search-icon {
                font-size: 1rem;
                color: rgba(255, 255, 255, 0.5);
            }
        }
    }

    .loading-indicator {
        position: absolute;
        right: 0.75rem;
        top: 50%;
        transform: translateY(-50%);
        color: var(--ks-content-secondary);
        font-size: 14px;
    }

    .search-results {
        position: absolute;
        top: 100%;
        left: 0;
        right: 0;
        background-color: var(--ks-background-card);
        border-radius: 6px;
        margin-top: 4px;
        max-height: 400px;
        overflow-y: auto;
        z-index: 1001;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
        padding: 4px 0;
    }

    .search-result {
        padding: 6px 12px;
        cursor: pointer;
        display: block;
        text-decoration: none;
        color: inherit;
        background: var(--ks-background-card);
        transition: background-color 0.2s;
        
        &:hover {
            background: var(--ks-background-hover);
            text-decoration: none;
            color: inherit;
        }

        &.selected {
            background: rgba(132, 5, 255, 0.1);
            border-left: 3px solid #8405FF;
        }

        .result-title {
            font-weight: 400;
            color: var(--ks-content-primary);
            margin-bottom: 2px;
            font-size: 14px;
        }

        .result-preview {
            font-size: 12px;
            color: var(--ks-content-secondary);
            margin: 0;
            opacity: 0.8;
        }
    }

    .no-results {
        color: var(--ks-content-secondary);
        text-align: center;
        cursor: default;
        padding: 6px 12px;
        font-size: 14px;
        
        &:hover {
            background: none;
        }
    }
</style> 