<template>
  <div class="app">
    <header class="app-header">
      <h1>Gutenberg Project</h1>
      <p class="subtitle">
        A social cataloging website that allows you to freely search its
        database of books, annotations, and reviews.
      </p>
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
          <img v-if="cat.icon" :src="cat.icon" alt="" class="category-icon" />
          <span>{{ cat.label }}</span>
        </button>
      </div>
    </section>

    <!-- Screen 2: Books list -->
    <section v-else class="books-view">
      <div class="books-header">
        <button class="back-btn" @click="goBack">
          <img :src="BackIcon" alt="Back" class="back-icon" />
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
        <img :src="SearchIcon" alt="" class="search-icon" />
        <input v-model="searchQuery" type="text" placeholder="Search" />
        <img
          v-if="searchQuery"
          :src="CancelIcon"
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
            alt="Cover"
            class="book-cover"
          />

          <div class="book-info">
            <h3 class="book-title">{{ book.title }}</h3>
            <p class="book-authors" v-if="book.authors?.length">
              {{ book.authors.join(", ") }}
            </p>
            <p class="book-meta">
              <span v-if="book.languages?.length">
                Lang: {{ book.languages.join(", ") }}
              </span>
              <span v-if="book.download_count">
                · Downloads: {{ book.download_count }}
              </span>
            </p>
            <p class="book-tags" v-if="book.subjects?.length">
              {{ book.subjects.slice(0, 2).join(" • ") }}
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
// Icons
import BackIcon from "./assets/icons/Back.svg";
import FictionIcon from "./assets/icons/Fiction.svg";
import DramaIcon from "./assets/icons/Drama.svg";
import HumorIcon from "./assets/icons/Humour.svg";
import HistoryIcon from "./assets/icons/History.svg";
import AdventureIcon from "./assets/icons/Adventure.svg";
import PoliticsIcon from "./assets/icons/Politics.svg";
import PhilosophyIcon from "./assets/icons/Philosophy.svg";
import SearchIcon from "./assets/icons/Search.svg";
import CancelIcon from "./assets/icons/Cancel.svg";

import { ref, watch, onMounted, onBeforeUnmount, computed } from "vue";

// 🔗 Ignitesol Gutendex API base
// const BASE_URL = "http://skunkworks.ignitesol.com:8000";

const BASE_URL = import.meta.env.VITE_API_URL;

const categories = [
  { label: "Fiction", value: "fiction", icon: FictionIcon },
  { label: "Drama", value: "drama", icon: DramaIcon },
  { label: "Humour", value: "humor", icon: HumorIcon },
  { label: "Politics", value: "politics", icon: PoliticsIcon },
  { label: "Philosophy", value: "philosophy", icon: PhilosophyIcon },
  { label: "History", value: "history", icon: HistoryIcon },
  { label: "Adventure", value: "adventure", icon: AdventureIcon },
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

// // Build URL based on page + category + search
// function buildUrl(page = 1) {
//   const params = new URLSearchParams();

//   params.set("page", page.toString());

//   // topic → category (subjects/bookshelves)
//   if (selectedCategory.value?.value) {
//     params.append("topic", selectedCategory.value.value);
//   }

//   // search → title + author (Gutendex behaviour)
//   if (searchQuery.value.trim()) {
//     params.append("search", searchQuery.value.trim());
//   }

//   // only books with cover images
//   params.append("mime_type", "image");

//   return `${BASE_URL}/books?${params.toString()}`;
// }

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

  // Only books with cover images
  params.append("mime_type", "image");

  return `${BASE_URL}/books?${params.toString()}`;
}
// // Normalize Gutendex book to frontend shape
// function normalizeBook(raw) {
//   // coverImage: first image/* format
//   let coverImage = null;
//   if (raw.formats) {
//     for (const [mime, url] of Object.entries(raw.formats)) {
//       if (mime.startsWith("image/")) {
//         coverImage = url;
//         break;
//       }
//     }
//   }

//   return {
//     id: raw.id,
//     title: raw.title,
//     authors: raw.authors?.map((a) => a.name) ?? [],
//     languages: raw.languages ?? [],
//     subjects: raw.subjects ?? [],
//     download_count: raw.download_count ?? null,
//     coverImage,
//     formats: raw.formats || {},
//   };
// }
// function normalizeBook(raw) {
//   // Normalize formats (works for both Laravel + Gutendex)
//   const formatsArray = normalizeFormats(raw.formats);

//   // coverImage: first image/* format
//   let coverImage = null;
//   const imageFormat = formatsArray.find((f) =>
//     f.mime_type.toLowerCase().startsWith("image/")
//   );
//   if (imageFormat) {
//     coverImage = imageFormat.url;
//   }

//   return {
//     id: raw.id,
//     title: raw.title,
//     authors:
//       raw.authors?.map((a) => (typeof a === "string" ? a : a.name)) ?? [],
//     languages: raw.languages ?? [],
//     subjects: raw.subjects ?? [],
//     download_count: raw.download_count ?? null,
//     coverImage,
//     // keep normalized formats so openBook() works for both backends
//     formats: formatsArray,
//   };
// }

function normalizeBook(raw) {
  // Find first image format for cover
  const imageFormat = raw.formats?.find(f => 
    f.mime_type.toLowerCase().startsWith("image/")
  );

  return {
    id: raw.id,
    title: raw.title,
    authors: raw.authors.map(a => a.name),
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
    console.error(e);
    error.value = "Unable to load books. Please try again.";
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

// // Open preferred format: HTML > PDF > TXT (formats is an object)
// function openBook(book) {
//   const formats = book.formats || {};

//   let htmlUrl = null;
//   let pdfUrl = null;
//   let txtUrl = null;

//   for (const [mime, url] of Object.entries(formats)) {
//     const lower = mime.toLowerCase();

//     if (!htmlUrl && lower.startsWith("text/html")) {
//       htmlUrl = url;
//     } else if (!pdfUrl && lower.includes("pdf")) {
//       pdfUrl = url;
//     } else if (!txtUrl && lower.startsWith("text/plain")) {
//       txtUrl = url;
//     }
//   }

//   const finalUrl = htmlUrl || pdfUrl || txtUrl;

//   if (finalUrl) {
//     window.open(finalUrl, "_blank");
//   } else {
//     alert("No viewable version available");
//   }
// }

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
