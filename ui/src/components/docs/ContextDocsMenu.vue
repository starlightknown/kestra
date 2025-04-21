<template>
    <div class="docsMenuWrapper">
        <div class="search-container">
            <el-input
                v-model="searchQuery"
                :placeholder="t('Search')"
                class="search-input"
                @input="handleSearch"
                @keyup.enter="handleEnterKey"
                :loading="loading"
            >
                <template #prefix>
                    <el-icon class="search-icon">
                        <Search />
                    </el-icon>
                </template>
            </el-input>
            <div v-if="loading" class="loading-indicator">
                <span class="loading-spinner" />
            </div>
            <div v-if="showResults" class="search-results">
                <template v-if="searchResults.length > 0">
                    <context-docs-link
                        v-for="result in searchResults"
                        :key="result.url"
                        class="search-result"
                        :href="result.parsedUrl.replace(/^docs\//, '')"
                        use-raw
                        @click="() => {
                            searchQuery = '';
                            searchResults = [];
                            menuOpen = false;
                        }"
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
                    {{ t("No results found") }}
                </div>
            </div>
        </div>
        <el-button @click="menuOpen = !menuOpen" class="menuOpener">
            {{ t("documentationMenu") }} <MenuDown class="expandIcon" />
        </el-button>
        <ul v-if="menuOpen" class="docsMenu list-unstyled d-flex flex-column gap-3">
            <template v-if="rawStructure">
                <li v-for="[sectionName, children] in sectionsWithChildren" :key="sectionName">
                    <span class="text-secondary">
                        {{ sectionName.toUpperCase() }}
                    </span>
                    <recursive-toc :parent="{children}">
                        <template #default="{path, title}">
                            <context-docs-link @click="menuOpen = false" :href="path.slice(5)" use-raw>
                                {{ title.capitalize() }}
                            </context-docs-link>
                        </template>
                    </recursive-toc>
                </li>
            </template>
            <li v-else>
                Loading Menu...
            </li>
        </ul>
    </div>
</template>

<script setup>
    import {ref, computed, watch} from "vue";
    import {useStore} from "vuex";
    import {useI18n} from "vue-i18n";
    import {Search} from "@element-plus/icons-vue";
    import MenuDown from "vue-material-design-icons/MenuDown.vue";
    import RecursiveToc from "./RecursiveToc.vue";
    import ContextDocsLink from "./ContextDocsLink.vue";

    const {t} = useI18n({useScope: "global"});
    const store = useStore();
    const menuOpen = ref(false);
    const searchQuery = ref("");
    const searchResults = ref([]);
    const loading = ref(false);
    const allDocs = ref([]);

    const getTitle = (path) => {
        const parts = path.split("/");
        const lastPart = parts[parts.length - 1];
        return lastPart
            .split("-")
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(" ");
    };

    const getSearchScore = (result, query) => {
        const searchLower = query.toLowerCase();
        const title = getTitle(result.parsedUrl).toLowerCase();
        const path = result.parsedUrl.toLowerCase();

        let score = 0;

        if (title.includes(searchLower)) {
            score += 10;
            if (title.startsWith(searchLower)) {
                score += 5;
            }
        }

        if (path.includes(searchLower)) {
            score += 3;
            if (path.split("/").some(part => part.startsWith(searchLower))) {
                score += 2;
            }
        }

        const words = searchLower.split(" ");
        words.forEach(word => {
            if (title.includes(word)) score += 2;
            if (path.includes(word)) score += 1;
        });

        return score;
    };

    const showResults = computed(() => {
        return searchQuery.value.trim().length > 0;
    });

    const handleSearch = async () => {
        if (!searchQuery.value) {
            searchResults.value = [];
            return;
        }

        try {
            loading.value = true;

            if (allDocs.value.length === 0) {
                const results = await store.dispatch("doc/search", "");
                allDocs.value = results || [];
            }

            const query = searchQuery.value.trim();
            const filteredResults = allDocs.value
                .map(result => ({
                    ...result,
                    title: getTitle(result.parsedUrl),
                    score: getSearchScore(result, query)
                }))
                .filter(result => result.score > 0)
                .sort((a, b) => b.score - a.score)
                .slice(0, 10);

            searchResults.value = filteredResults;
        } catch (error) {
            console.error("Error searching docs:", error);
            searchResults.value = [];
        } finally {
            loading.value = false;
        }
    };

    const handleEnterKey = () => {
        if (searchResults.value.length > 0) {
            const result = searchResults.value[0];
            searchQuery.value = "";
            searchResults.value = [];
            menuOpen.value = false;
            store.commit("doc/setDocPath", result.parsedUrl.replace(/^docs\//, ""));
        }
    };

    const SECTIONS = {
        "Get Started with Kestra": [
            "Getting Started",
            "Tutorial",
            "Architecture",
            "Installation Guide",
            "User Interface"
        ],
        "Build with Kestra": [
            "Concepts",
            "Workflow Components",
            "Expressions",
            "Version Control & CI/CD",
            "Plugin Developer Guide",
            "How-to Guides"
        ],
        "Scale with Kestra": [
            "Enterprise Edition",
            "Task Runners",
            "Best Practices"
        ],
        "Manage Kestra": [
            "Administrator Guide",
            "Configuration Guide",
            "Migration Guide",
            "Terraform Provider",
            "API Reference"
        ]
    };

    const rawStructure = ref(undefined);

    watch(menuOpen, async (val) => {
        if(!val || rawStructure.value !== undefined){
            return;
        }
        rawStructure.value = await store.dispatch("doc/children");
    });

    const toc = computed(() => {
        if (rawStructure.value === undefined) {
            return undefined;
        }

        const childrenWithMetadata = Object.entries(rawStructure.value)
            .reduce((acc, [url, metadata]) => {
                if(!metadata || metadata.hideSidebar){
                    return acc;
                }

                acc[url] = {
                    ...metadata,
                    path: url
                };

                return acc
            }, {});

        for(const url in childrenWithMetadata){
            const metadata = childrenWithMetadata[url];
            const split = url.split("/");
            const parentUrl = split.slice(0, split.length - 1).join("/");
            const parent = childrenWithMetadata[parentUrl];
            if (parent !== undefined) {
                parent.children = [...(parent.children ?? []), metadata];
            }
        }

        return Object.entries(childrenWithMetadata)[0]?.[1]?.children;
    });

    const sectionsWithChildren = computed(() => {
        if (toc.value === undefined) {
            return undefined;
        }

        return Object.entries(SECTIONS).map(([section, childrenTitles]) => [section, toc.value.filter(({title}) => childrenTitles.includes(title))]);
    });
</script>

<style lang="scss" scoped>
    ul > li > span:first-child {
        font-size: 12px;
    }

    .docsMenu{
        position: absolute;
        z-index: 1000;
        padding: .5rem 1rem;
        left: 1rem;
        top: 100%;
        right: 1rem;
        background-color: var(--ks-background-card);
        border-radius: 6px;
    }

    .docsMenuWrapper{
        position: relative;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .menuOpener{
        flex: 1;
        margin: 0;
    }

    .expandIcon{
        margin-left: 1rem;
    }

    .search-container {
        position: relative;
        margin-bottom: 1rem;
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
    }

    .loading-spinner {
        display: inline-block;
        width: 1rem;
        height: 1rem;
        border: 2px solid var(--border-color);
        border-top-color: var(--primary-color);
        border-radius: 50%;
        animation: spin 1s linear infinite;
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
        z-index: 1000;
        box-shadow: 0 2px 12px 0 rgba(0,0,0,.1);
        padding: 4px 0;
    }

    .search-result {
        padding: 6px 12px;
        cursor: pointer;
        display: block;
        text-decoration: none;
        color: inherit;
        
        &:hover {
            background: var(--ks-background-hover);
            text-decoration: none;
            color: inherit;
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

    @keyframes spin {
        to {
            transform: rotate(360deg);
        }
    }
</style>