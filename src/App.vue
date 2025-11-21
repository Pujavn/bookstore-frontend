<template>
  <div class="app">
    <header v-if="currentView === 'categories'" class="app-header">
      <div class="app-header-inner">
        <h1>Gutenberg Project</h1>
        <p class="subtitle">
          A social cataloging website that allows you to freely search its
          database of books, annotations, and reviews.
        </p>
      </div>
    </header>

    <!-- Screen 1: Category selection -->
    <section v-if="currentView === 'categories'" class="categories">
      <h2>Select a Category</h2>
      <div class="category-grid">
        <button
          v-for="cat in categories"
          :key="cat.value"
          class="category-button"
          @click="selectCategory(cat)"
        >
          <div class="left">
            <img v-if="cat.icon" :src="cat.icon" alt="" class="category-icon" />
            <span>{{ cat.label }}</span>
          </div>
          <!-- in template -->
          <img
            src="/assets/icons/Next.svg"
            alt=""
            class="arrow-icon"
            aria-hidden="true"
          />
        </button>
      </div>
    </section>

    <!-- Screen 2: Books list -->
    <section v-else class="books-view">
      <div class="books-header">
        <button class="back-btn" @click="goBack">
          <img src="/assets/icons/Back.svg" alt="Back" class="back-icon" />
        </button>
        <div class="info">
          <h2>{{ selectedCategory?.label }} Books</h2>
          <p class="count" v-if="total !== null">{{ total }} books found</p>
          <p class="count" v-if="books.length">
            Showing {{ books.length }} books
          </p>
        </div>
      </div>

      <!-- Search box: title + author (search param) -->
      <div class="search-bar">
        <img src="/assets/icons/Search.svg" alt="" class="search-icon" />
        <input v-model="searchQuery" type="text" placeholder="Search" />
        <img
          v-if="searchQuery"
          src="/assets/icons/Cancel.svg"
          alt=""
          class="clear-icon"
          @click="searchQuery = ''"
        />
      </div>

      <!-- Error / loading -->
      <p v-if="error" class="error">{{ error }}</p>
      <p v-if="initialLoading && !error" class="loading">Loading books...</p>

      <!-- Books list -->
      <div v-if="!initialLoading && books.length" class="books-list">
        <article
          v-for="book in books"
          :key="book.id"
          class="book-card"
          @click="openBook(book)"
        >
          <img
            v-if="book.coverImage"
            :src="book.coverImage"
            :alt="`Cover image — ${book.title}`"
            class="book-cover"
            loading="lazy"
          />

          <div class="book-info">
            <h3 class="book-title">{{ book.title }}</h3>
            <p class="book-authors" v-if="book.authors?.length">
              {{ book.authors.join(", ") }}
            </p>
          </div>
        </article>

        <!-- Infinite scroll sentinel -->
        <div ref="loadMoreRef" class="load-more-sentinel">
          <p v-if="loadingMore">Loading more...</p>
          <p v-else-if="!hasMore">No more results.</p>
        </div>
      </div>

      <p v-if="!initialLoading && !books.length && !error" class="empty">
        No books found for this category/search.
      </p>
    </section>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, computed } from "vue";

// 🔗 Ignitesol Gutendex API base
// const BASE_URL = "http://skunkworks.ignitesol.com:8000";

const BASE_URL = import.meta.env.VITE_API_URL;

const categories = [
  { label: "Fiction", value: "fiction", icon: "/assets/icons/Fiction.svg" },
  { label: "Drama", value: "drama", icon: "/assets/icons/Drama.svg" },
  { label: "Humour", value: "humor", icon: "/assets/icons/Humour.svg" },
  { label: "Politics", value: "politics", icon: "/assets/icons/Politics.svg" },
  {
    label: "Philosophy",
    value: "philosophy",
    icon: "/assets/icons/Philosophy.svg",
  },
  { label: "History", value: "history", icon: "/assets/icons/History.svg" },
  {
    label: "Adventure",
    value: "adventure",
    icon: "/assets/icons/Adventure.svg",
  },
];

const currentView = ref("categories"); // 'categories' | 'books'
const selectedCategory = ref(null);

const books = ref([]);
const total = ref(null);
const currentPage = ref(1);

const initialLoading = ref(false);
const loadingMore = ref(false);
const error = ref(null);

const searchQuery = ref("");
let searchDebounce = null;

// For IntersectionObserver (infinite scroll)
const loadMoreRef = ref(null);
let observer = null;

// Whether there are more results to load
const hasMore = computed(() => {
  if (total.value === null) return true;
  return books.value.length < total.value;
});

function selectCategory(cat) {
  console.info("Selected category:", cat.value);
  selectedCategory.value = cat;
  currentView.value = "books";
  searchQuery.value = "";
  resetBooksState();
  fetchBooks();
}

function goBack() {
  currentView.value = "categories";
  selectedCategory.value = null;
  resetBooksState();
}

function resetBooksState() {
  books.value = [];
  total.value = null;
  currentPage.value = 1;
  error.value = null;
  initialLoading.value = false;
  loadingMore.value = false;
}

function buildUrl(page = 1) {
  const params = new URLSearchParams();
  params.set("page", page.toString());

  // Category → topic filter
  if (selectedCategory.value?.value) {
    params.append("topic", selectedCategory.value.value);
  }

  // Search → send BOTH title and author (your Laravel backend supports this)
  if (searchQuery.value.trim()) {
    const term = searchQuery.value.trim();
    params.append("title", term);
    params.append("author", term);
  }

  params.append("mime_type", "image");

  return `${BASE_URL}/books?${params.toString()}`;
}

function normalizeBook(raw) {
  const imageFormat = raw.formats?.find((f) =>
    f.mime_type.toLowerCase().startsWith("image/")
  );

  return {
    id: raw.id,
    title: raw.title,
    authors: raw.authors?.map(a => typeof a === 'string' ? a : a.name) ?? [],
    languages: raw.languages,
    subjects: raw.subjects,
    download_count: raw.download_count,
    coverImage: imageFormat ? imageFormat.url : null,
    formats: raw.formats || [],
  };
}

// Fetch first page (or refreshed data)
async function fetchBooks() {
  if (!selectedCategory.value) return;

  initialLoading.value = true;
  error.value = null;
  currentPage.value = 1;
  books.value = [];

  try {
    const url = buildUrl(1);
    const res = await fetch(url);
    if (!res.ok) throw new Error("Failed to fetch books");

    const data = await res.json();
    total.value = data.count;

    const newBooks = (data.results || []).map(normalizeBook);
    books.value = newBooks;
  } catch (e) {
    console.error("fetchBooks error:", e);
    e?.message || "Unable to load books. Please try again.";
  } finally {
    initialLoading.value = false;
  }
}

// Fetch next page (infinite scroll)
async function loadMore() {
  if (loadingMore.value || initialLoading.value) return;
  if (!hasMore.value) return;

  loadingMore.value = true;
  error.value = null;

  const nextPage = currentPage.value + 1;

  try {
    const url = buildUrl(nextPage);
    const res = await fetch(url);
    if (!res.ok) throw new Error("Failed to load more books");

    const data = await res.json();
    const newBooks = (data.results || []).map(normalizeBook);

    books.value = books.value.concat(newBooks);
    currentPage.value = nextPage;
  } catch (e) {
    console.error(e);
    error.value = "Unable to load more books.";
  } finally {
    loadingMore.value = false;
  }
}

function openBook(book) {
  // Normalized into array: [{ mime_type, url }]
  const formats = normalizeFormats(book.formats);

  let htmlUrl = null;
  let pdfUrl = null;
  let txtUrl = null;

  for (const f of formats) {
    const mime = f.mime_type.toLowerCase();
    const url = f.url;

    if (!htmlUrl && mime.startsWith("text/html")) {
      htmlUrl = url;
    } else if (!pdfUrl && mime.includes("pdf")) {
      pdfUrl = url;
    } else if (!txtUrl && mime.startsWith("text/plain")) {
      txtUrl = url;
    }
  }

  const finalUrl = htmlUrl || pdfUrl || txtUrl;

  if (finalUrl) {
    window.open(finalUrl, "_blank");
  } else {
    alert("No viewable version available");
  }
}

function normalizeFormats(formats) {
  if (!formats) return [];

  // Case 1: Laravel API → array of { mime_type, url }
  if (Array.isArray(formats)) {
    return formats;
  }

  // Case 2: Gutendex → object { "text/html": "url", ... }
  return Object.entries(formats).map(([mime_type, url]) => ({
    mime_type,
    url,
  }));
}
// Watch searchQuery with debounce → refetch from page 1
watch(searchQuery, () => {
  if (currentView.value !== "books" || !selectedCategory.value) return;

  if (searchDebounce) clearTimeout(searchDebounce);
  searchDebounce = setTimeout(() => {
    fetchBooks();
  }, 500); // 0.5s debounce
});

// Setup IntersectionObserver for infinite scroll
onMounted(() => {
  observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          loadMore();
        }
      });
    },
    {
      root: null,
      rootMargin: "200px",
      threshold: 0,
    }
  );

  // If sentinel is already present
  if (loadMoreRef.value) {
    observer.observe(loadMoreRef.value);
  }
});

// Watch for sentinel element mounting/unmounting
watch(loadMoreRef, (el, oldEl) => {
  if (!observer) return;

  if (oldEl) {
    observer.unobserve(oldEl);
  }
  if (el) {
    observer.observe(el);
  }
});

onBeforeUnmount(() => {
  if (observer) {
    if (loadMoreRef.value) {
      observer.unobserve(loadMoreRef.value);
    }
    observer.disconnect();
  }
});
</script>
