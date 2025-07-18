<template>
  <div class="curation-container view-container">
    <!-- Search Input Section -->
    <div class="search-section">
      <div class="search-input-row">
        <n-input
          v-model:value="searchQuery"
          type="textarea"
          placeholder="Describe the photos you want to curate... e.g., 'urban architecture with people' or 'nature landscapes at sunset'"
          :autosize="{ minRows: 1, maxRows: 2 }"
          class="search-input"
          @input="onSearchChange"
        />

        <div class="search-actions">
          <n-button
            type="primary"
            :loading="isSearching"
            :disabled="!hasSearchQuery"
            @click="performSearch"
            class="search-button"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M15.5 14h-.79l-.28-.27A6.471 6.471 0 0 0 16 9.5A6.5 6.5 0 1 0 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5S14 7.01 14 9.5S11.99 14 9.5 14z"
                  />
                </svg>
              </n-icon>
            </template>
            Search Photos
          </n-button>
          <n-button
            secondary
            @click="clearSearch"
            :disabled="!hasSearchQuery && candidatePhotos.length === 0"
            class="clear-button"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M19 6.41L17.59 5L12 10.59L6.41 5L5 6.41L10.59 12L5 17.59L6.41 19L12 13.41L17.59 19L19 17.59L13.41 12L19 6.41z"
                  />
                </svg>
              </n-icon>
            </template>
            Clear
          </n-button>
        </div>
        <!-- Usage Limit Warning Badge -->
        <!-- <div
          v-if="userStore.usageLimits.curation.exceeded"
          class="usage-limit-warning"
        >
          <WarningBadge
            message="Free usage ended"
            tooltip="This is a premium feature, purchase credits to continue using it."
          />
        </div> -->
      </div>
    </div>

    <!-- Main Curation Area -->
    <div
      class="curation-main"
      v-if="
        candidatePhotos.length > 0 || curatedPhotos.length > 0 || isSearching
      "
    >
      <!-- Left Side: Curation Area -->
      <div class="curation-area">
        <div class="area-header">
          <h3 class="area-title">Curation Area</h3>
          <div class="star-filter">
            <span class="filter-label">Min. Rating:</span>
            <div class="star-buttons">
              <n-rate
                v-model:value="minMatchScore"
                :count="3"
                size="small"
                clearable
                @update:value="onRatingChange"
                class="star-rating"
                :allow-half="false"
              />
            </div>
          </div>
        </div>

        <div
          class="photos-grid"
          v-if="filteredCandidatePhotos.length > 0 || isSearching"
        >
          <!-- Show skeletons when loading more photos -->
          <template v-if="isLoadingMore">
            <div
              v-for="n in 6"
              :key="`loading-skeleton-${n}`"
              class="photo-skeleton"
            >
              <n-skeleton height="100%" />
            </div>
          </template>
          <!-- Show skeletons while searching (only when no photos yet) -->
          <template v-if="isSearching && filteredCandidatePhotos.length === 0">
            <div v-for="n in 6" :key="`skeleton-${n}`" class="photo-skeleton">
              <n-skeleton height="100%" />
            </div>
          </template>
          <!-- Show actual photos -->
          <template v-if="filteredCandidatePhotos.length > 0">
            <PhotoCard
              v-for="photo in filteredCandidatePhotos"
              :key="photo.id"
              :photo="photo"
              :mode="'curation'"
              :selected="false"
              :isThinking="isThinking"
              @select="togglePhotoSelection"
              @info="showPhotoInfo"
              @move-to-selection="moveToSelection"
            />
          </template>
        </div>

        <div class="empty-area" v-else>
          <n-icon size="48" color="#6b7280">
            <svg viewBox="0 0 24 24">
              <path
                fill="currentColor"
                d="M21 19V5c0-1.1-.9-2-2-2H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2zM8.5 13.5l2.5 3.01L14.5 12l4.5 6H5l3.5-4.5z"
              />
            </svg>
          </n-icon>
          <p class="empty-text">No photos to review</p>
        </div>

        <div class="area-actions">
          <n-button
            :loading="isLoadingMore"
            @click="searchMorePhotos"
            class="search-more-button"
            size="large"
            :disabled="
              !hasMoreResults ||
              candidatePhotos.length == 0 ||
              isSearching ||
              isLoadingMore ||
              isThinking
            "
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"
                  />
                </svg>
              </n-icon>
            </template>
            Search More
          </n-button>
        </div>
      </div>

      <!-- Right Side: Selection Area -->
      <div class="selection-area">
        <div class="area-header">
          <h3 class="area-title">Selection Area</h3>
          <div class="area-stats">
            <span class="stats-text">{{ curatedPhotos.length }} curated</span>
            <n-button
              v-if="curatedPhotos.length > 0"
              size="small"
              @click="clearSelection"
              class="clear-selection-button"
            >
              Clear All
            </n-button>
          </div>
        </div>

        <div class="photos-grid" v-if="curatedPhotos.length > 0">
          <PhotoCard
            v-for="photo in curatedPhotos"
            :key="photo.id"
            :photo="photo"
            :mode="'selection'"
            @select="togglePhotoSelection"
            @info="showPhotoInfo"
            @move-to-curation="moveToCuration"
          />
        </div>

        <div class="empty-area" v-else>
          <n-icon size="48" color="#6b7280">
            <svg viewBox="0 0 24 24">
              <path
                fill="currentColor"
                d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-5 14H7v-2h7v2zm3-4H7v-2h10v2zm0-4H7V7h10v2z"
              />
            </svg>
          </n-icon>
          <p class="empty-text">Curated photos will appear here</p>
          <p class="empty-subtitle">
            Move photos from the left to start curating
          </p>
        </div>

        <div class="area-actions">
          <n-button
            :loading="isLoadingMore"
            @click="() => {}"
            size="large"
            :disabled="curatedPhotos.length == 0"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"
                  />
                </svg>
              </n-icon>
            </template>
            Take to Canvas
          </n-button>
          <n-button
            :loading="isLoadingMore"
            @click="() => {}"
            size="large"
            :disabled="curatedPhotos.length == 0"
          >
            <template #icon>
              <n-icon>
                <svg viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"
                  />
                </svg>
              </n-icon>
            </template>
            Add to Collection
          </n-button>
        </div>
      </div>
    </div>

    <!-- Empty State -->
    <div v-else class="empty-state">
      <div class="empty-content">
        <n-icon size="64" color="#6b7280" class="empty-icon">
          <svg viewBox="0 0 24 24">
            <path
              fill="currentColor"
              d="M9.5 3A6.5 6.5 0 0 1 16 9.5c0 1.61-.59 3.09-1.56 4.23l.27.27h.79l5 5l-1.5 1.5l-5-5v-.79l-.27-.27A6.516 6.516 0 0 1 9.5 16A6.5 6.5 0 0 1 3 9.5A6.5 6.5 0 0 1 9.5 3m0 2C7.01 5 5 7.01 5 9.5S7.01 14 9.5 14S14 11.99 14 9.5S11.99 5 9.5 5Z"
            />
          </svg>
        </n-icon>
        <h2 class="empty-title">Start Curating Photos</h2>
        <p class="empty-description">
          Enter a search topic above to find photos for curation
        </p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  computed,
  onMounted,
  onUnmounted,
  watch,
  defineOptions,
} from "vue";

// Define component name for KeepAlive
defineOptions({
  name: "CurationView",
});

import PhotoCard from "../components/photoCards/PhotoCard.vue";
import { useUserStore } from "@/stores/userStore";
import { NRate } from "naive-ui";

import WarningBadge from "@/components/WarningBadge.vue";
import api from "@/utils/axios";
import { io } from "socket.io-client";

// Photo interface with curation-specific properties
interface CurationPhoto {
  id: string;
  url: string;
  title: string;
  rating: number;
  reasoning: string; // AI-generated reasoning for curation
  matchScore: number; // AI-generated match score (0 = not scored yet, 1-3 stars)
  matchingTags?: string[];
  width?: number;
  height?: number;
  thumbnailUrl: string;
  isUploading: boolean;
  file: any;
  needProcess: boolean;
  isDuplicate: boolean;
}

// Socket connection for real-time results
const socket = io(import.meta.env.VITE_API_WS_URL);

// Stores
const userStore = useUserStore();

// State
const searchQuery = ref("");
const isSearching = ref(false);
const isLoadingMore = ref(false);
const isThinking = ref(false); // Esperando insights
const hasMoreResults = ref(true);
const candidatePhotos = ref<CurationPhoto[]>([]);
const curatedPhotos = ref<CurationPhoto[]>([]);
const minMatchScore = ref<number | undefined>(2);

// Real-time results state
const iterationsRecord = ref<Record<number, { photos: CurationPhoto[] }>>({});
const currentIteration = ref(1);
const maxPageAttempts = ref(false);

// Computed properties
const hasSearchQuery = computed(() => searchQuery.value.trim().length > 0);

// Filter candidate photos based on minimum match score
const filteredCandidatePhotos = computed(() => {
  if (!minMatchScore.value) {
    return candidatePhotos.value;
  }

  const minScore = minMatchScore.value;
  return candidatePhotos.value.filter((photo) => {
    // Only filter photos that have been scored (matchScore > 0)
    // Photos with matchScore = 0 are still being analyzed
    return photo.matchScore === 0 || photo.matchScore >= minScore;
  });
});

// Socket event handlers
let socketListenersRegistered = false;

const registerSocketListeners = () => {
  if (socketListenersRegistered || !userStore.user?.id) return;

  const userId = userStore.user.id;
  socket.emit("join", { userId: userStore.user.id });

  // Socket for initial photo matches (fast response)
  socket.on("matches", (data) => {
    isThinking.value = true; // Esperar insights después de matches

    console.log("🎯 Matches received:", data);

    // Update iterations record with new photo results
    Object.entries(data.results).forEach(([iter, items]) => {
      const iterNum = parseInt(iter);
      if (!iterationsRecord.value[iterNum]) {
        iterationsRecord.value[iterNum] = { photos: [] };
      }

      const newPhotos = (items as any[]).map((item) => ({
        id: item.photo.id,
        url: item.photo.thumbnailUrl || item.photo.url,
        thumbnailUrl: item.photo.thumbnailUrl || item.photo.url,
        title: item.photo.title || `Photo ${item.photo.id}`,
        rating: item.photo.rating || Math.floor(Math.random() * 3) + 3,
        reasoning: "Analyzing...", // Temporary until insights arrive
        matchScore: 0, // Will be set when insights arrive
        matchingTags: item.photo.matchingTags || [],
        width: item.photo.width,
        height: item.photo.height,
        isUploading: false,
        file: null,
        needProcess: false,
        isDuplicate: false,
      }));

      iterationsRecord.value[iterNum].photos.push(...newPhotos);
    });

    // Update candidate photos with all results up to current iteration
    updateCandidatePhotos();

    hasMoreResults.value = data.hasMore;
    currentIteration.value = data.iteration + 1;
    isSearching.value = false;
    isLoadingMore.value = false;
  });

  // Socket for insights/reasoning (enriches existing photos)
  socket.on("insights", (data) => {
    isThinking.value = false;
    console.log("📊 Insights received:", data);

    // Enrich existing photos with reasoning and match scores
    Object.entries(data.results).forEach(([iter, richPhotos]) => {
      const iterNum = parseInt(iter);
      if (!iterationsRecord.value[iterNum]) return;

      iterationsRecord.value[iterNum].photos = iterationsRecord.value[
        iterNum
      ].photos.map((existing) => {
        const updated = (richPhotos as any[]).find(
          (item) => item.photo.id === existing.id
        );
        return updated
          ? {
              ...existing,
              reasoning: updated.reasoning || "No reasoning provided",
              matchScore: updated.matchScore || updated.photo?.matchScore || 0,
            }
          : existing;
      });
    });

    // Update candidate photos to reflect the new reasoning
    updateCandidatePhotos();

    console.log("✨ Photos enriched with insights");
  });

  socket.on("maxPageAttempts", () => {
    console.log("⚠️ Max page attempts reached");
    maxPageAttempts.value = true;
    hasMoreResults.value = false;
    isSearching.value = false;
    isLoadingMore.value = false;
  });

  socketListenersRegistered = true;
};

const updateCandidatePhotos = () => {
  const keys = Object.keys(iterationsRecord.value)
    .map(Number)
    .sort((a, b) => a - b);

  let allPhotos: CurationPhoto[] = [];
  for (let i = 0; i < currentIteration.value; i++) {
    const k = keys[i];
    if (k !== undefined && iterationsRecord.value[k]?.photos) {
      // Insert new photos at the beginning so newest are on top
      allPhotos.unshift(...iterationsRecord.value[k].photos);
    }
  }

  // Filtrar las fotos que ya están en curatedPhotos
  const curatedIds = new Set(curatedPhotos.value.map((p) => p.id));
  candidatePhotos.value = allPhotos.filter(
    (photo) => !curatedIds.has(photo.id)
  );
};

// Methods
const onSearchChange = () => {
  console.log("Search query changed:", searchQuery.value);
};

// Shared API call for searching photos
const searchPhotosApi = async (isInitial = false) => {
  if (!hasSearchQuery.value) return;

  if (isInitial) {
    isSearching.value = true;
    candidatePhotos.value = [];
    iterationsRecord.value = {};
    currentIteration.value = 1;
    hasMoreResults.value = true;
    maxPageAttempts.value = false;
    console.log("Performing curation search:", searchQuery.value);
  } else {
    isLoadingMore.value = true;
  }

  try {
    const payload = {
      description: searchQuery.value,
      options: {
        iteration: currentIteration.value,
        pageSize: 6,
        searchMode: "curation",
        minMatchScore: minMatchScore.value,
      },
    };
    await api.post("/api/search/semantic", payload);
  } catch (error) {
    if (isInitial) {
      console.error("❌ Error performing curation search:", error);
      isSearching.value = false;
    } else {
      console.error("❌ Error loading more photos:", error);
      isLoadingMore.value = false;
    }
  }
};

const performSearch = async () => {
  await searchPhotosApi(true);
};

const searchMorePhotos = async () => {
  if (isLoadingMore.value || !hasMoreResults.value || !hasSearchQuery.value)
    return;
  await searchPhotosApi(false);
};

const clearSearch = () => {
  searchQuery.value = "";
  candidatePhotos.value = [];
  curatedPhotos.value = [];
  iterationsRecord.value = {};
  currentIteration.value = 1;
  hasMoreResults.value = true;
  maxPageAttempts.value = false;
};
const clearSelection = () => {
  // Move all curated photos back to candidates
  candidatePhotos.value.unshift(...curatedPhotos.value);
  curatedPhotos.value = [];
};

const moveToSelection = (photoId: string) => {
  const photoIndex = candidatePhotos.value.findIndex((p) => p.id === photoId);
  if (photoIndex === -1) return;

  const photo = candidatePhotos.value[photoIndex];
  candidatePhotos.value.splice(photoIndex, 1);

  // Add to curated with animation class
  curatedPhotos.value.push({
    ...photo,
    // Mark for animation (could be used in PhotoCard)
  });
};

const moveToCuration = (photoId: string) => {
  const photoIndex = curatedPhotos.value.findIndex((p) => p.id === photoId);
  if (photoIndex === -1) return;

  const photo = curatedPhotos.value[photoIndex];
  curatedPhotos.value.splice(photoIndex, 1);
  candidatePhotos.value.unshift(photo);
};

const togglePhotoSelection = (photoId: string) => {
  console.log("Photo selection toggled:", photoId);
};

const showPhotoInfo = (photo: any) => {
  console.log("Show photo info:", photo);
};

const onRatingChange = (rating: number | undefined) => {
  minMatchScore.value = rating;
  console.log("Min rating filter set to:", minMatchScore.value);

  // If we have candidate photos, show how many are filtered
  if (candidatePhotos.value.length > 0) {
    const filteredCount = filteredCandidatePhotos.value.length;
    console.log(
      `Showing ${filteredCount} of ${candidatePhotos.value.length} photos`
    );
  }
};

const setMinRating = (rating: number) => {
  minMatchScore.value = minMatchScore.value === rating ? undefined : rating;
  // Here you could filter photos by rating if needed
  console.log("Min rating filter set to:", minMatchScore.value);
};

// Lifecycle hooks
onMounted(() => {
  // Register socket listeners when user is available
  if (userStore.user?.id) {
    registerSocketListeners();
  }
});

onUnmounted(() => {
  // Clean up socket listeners
  socket.off("matches");
  socket.off("insights");
  socket.off("maxPageAttempts");
});

// Watch for user changes to register socket listeners
watch(
  () => userStore.user?.id,
  (userId: string | undefined) => {
    if (userId) {
      registerSocketListeners();
    }
  },
  { immediate: true }
);
</script>

<style scoped>
.curation-container {
  display: flex;
  flex-direction: column;
}

.search-section {
  background-color: transparent;
  border: none;
  border-radius: 0;
  padding: 0;
  margin-bottom: var(--spacing-lg);
}

.search-input-row {
  display: flex;
  gap: var(--spacing-lg);
  align-items: flex-start;
}

.search-input {
  font-size: var(--font-size-md);
  flex: 1;
}

.search-actions {
  display: flex;
  gap: var(--spacing-md);
  flex-shrink: 0;
}

.search-button {
  min-width: 140px;
  height: 40px;
}

.clear-button {
  min-width: 80px;
  height: 40px;
}

.curation-main {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: var(--spacing-2xl);
  min-height: 0; /* Allow grid items to shrink */
}

.curation-area,
.selection-area {
  display: flex;
  flex-direction: column;
  background-color: var(--bg-card);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.area-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--spacing-sm) var(--spacing-2xl);
  border-bottom: 1px solid var(--border-color);
  background-color: var(--bg-surface);
  min-height: 40px;
}

.area-title {
  font-size: var(--font-size-lg);
  font-weight: var(--font-weight-semibold);
  color: var(--text-primary);
  margin: 0;
}

.star-filter {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
}

.filter-label {
  font-size: var(--font-size-sm);
  color: var(--text-secondary);
  white-space: nowrap;
}

.star-rating {
  font-size: 16px;
}

.star-rating :deep(.n-rate__item) {
  margin-right: 2px;
}

.star-rating :deep(.n-rate__item .n-icon) {
  color: #fbbf24;
}

.star-buttons {
  display: flex;
  gap: var(--spacing-xs);
}

.area-stats {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
}

.stats-text {
  font-size: var(--font-size-sm);
  color: var(--text-secondary);
}

.clear-selection-button {
  font-size: var(--font-size-xs);
}

.photos-grid {
  flex: 1;
  display: grid !important;
  grid-template-columns: repeat(3, 1fr) !important;
  grid-auto-rows: max-content !important;
  gap: var(--spacing-lg) !important;
  padding: 12px;
  overflow-y: auto;
  align-content: start;
  place-items: start;
}

.photos-grid > * {
  width: 100%;
  box-sizing: border-box;
}

.empty-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-4xl);
  text-align: center;
}

.empty-text {
  font-size: var(--font-size-md);
  color: var(--text-secondary);
  margin: var(--spacing-md) 0 0 0;
}

.empty-subtitle {
  font-size: var(--font-size-sm);
  color: var(--text-tertiary);
  margin: var(--spacing-xs) 0 0 0;
}

.area-actions {
  padding: 12px;
  border-top: 1px solid var(--border-color);
  background-color: var(--bg-surface);
  display: flex;
  justify-content: center;
  gap: 24px;
}

.search-more-button {
  min-width: 160px;
}

.empty-state {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--spacing-6xl);
}

.empty-content {
  text-align: center;
  max-width: 400px;
}

.empty-icon {
  margin-bottom: var(--spacing-lg);
}

.empty-title {
  font-size: var(--font-size-2xl);
  font-weight: var(--font-weight-semibold);
  color: var(--text-primary);
  margin: 0 0 var(--spacing-md) 0;
}

.empty-description {
  font-size: var(--font-size-md);
  color: var(--text-secondary);
  margin: 0;
  line-height: var(--line-height-relaxed);
}

.photo-skeleton {
  width: 100%;
  aspect-ratio: 1;
  border-radius: var(--radius-md);
  overflow: hidden;
  background-color: var(--bg-surface);
}

.usage-limit-warning {
  display: flex;
  justify-content: center;
  margin-bottom: 16px;
}

.warning-icon {
  color: #f59e0b;
  flex-shrink: 0;
}

.warning-text {
  color: #f59e0b;
  white-space: nowrap;
}

.info-icon {
  color: #f59e0b;
  opacity: 0.8;
  cursor: help;
  flex-shrink: 0;
}

@media (max-width: 1024px) {
  .curation-main {
    grid-template-columns: 1fr;
    gap: var(--spacing-lg);
  }

  .photos-grid {
    grid-template-columns: repeat(3, 1fr) !important;
    gap: var(--spacing-md);
    padding: var(--spacing-lg);
  }
}

@media (max-width: 768px) {
  .curation-container {
    padding: var(--spacing-lg);
  }

  .search-section {
    padding: var(--spacing-lg);
    margin-bottom: var(--spacing-lg);
  }

  .search-input-row {
    flex-direction: column;
    gap: var(--spacing-md);
  }

  .search-actions {
    width: 100%;
  }

  .search-button {
    flex: 1;
  }

  .area-header {
    padding: var(--spacing-md) var(--spacing-lg);
  }

  .photos-grid {
    grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
    padding: var(--spacing-md);
  }

  .area-actions {
    padding: var(--spacing-md) var(--spacing-lg);
  }
}

@media (max-width: 480px) {
  .curation-container {
    padding: var(--spacing-md);
  }

  .search-section {
    padding: var(--spacing-md);
  }

  .area-header {
    flex-direction: column;
    align-items: stretch;
    gap: var(--spacing-sm);
  }

  .area-stats {
    justify-content: space-between;
  }

  .photos-grid {
    grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
    gap: var(--spacing-sm);
    padding: var(--spacing-sm);
  }

  .empty-area {
    padding: var(--spacing-2xl);
  }

  .loading-content {
    padding: var(--spacing-2xl);
    margin: var(--spacing-md);
  }
}
</style>
