<template>
  <div class="app">
    <header class="app-header">
      <h1>Bookstore Explorer</h1>
      <p class="subtitle">Browse Project Gutenberg books (Ignitesol Gutendex API)</p>
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
          {{ cat.label }}
        </button>
      </div>
    </section>

    <!-- Screen 2: Books list -->
    <section v-else class="books-view">
      <div class="books-header">
        <button class="back-btn" @click="goBack">← Back</button>
        <div class="info">
          <h2>{{ selectedCategory?.label }} Books</h2>
          <p class="count" v-if="total !== null">
            {{ total }} books found
          </p>
          <p v-if="books.length" class="count">
      Showing {{ books.length }} books
    </p>
        </div>
      </div>

      <!-- Search box: title + author (search param) -->
      <div class="search-bar">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search by title or author..."
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
              {{ book.authors.join(', ') }}
            </p>
            <p class="book-meta">
              <span v-if="book.languages?.length">
                Lang: {{ book.languages.join(', ') }}
              </span>
              <span v-if="book.download_count">
                · Downloads: {{ book.download_count }}
              </span>
            </p>
            <p class="book-tags" v-if="book.subjects?.length">
              {{ book.subjects.slice(0, 2).join(' • ') }}
            </p>
          </div>
        </article>

        <!-- Infinite scroll sentinel -->
        <div ref="loadMoreRef" class="load-more-sentinel">
          <p v-if="loadingMore">Loading more...</p>
          <p v-else-if="!nextPageUrl">No more results.</p>
        </div>
      </div>

      <p v-if="!initialLoading && !books.length && !error" class="empty">
        No books found for this category/search.
      </p>
    </section>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount} from 'vue'

// 🔗 Ignitesol Gutendex API base
const BASE_URL = 'http://skunkworks.ignitesol.com:8000'

// Category buttons – these map to `topic` filter
const categories = [
  { label: 'Fiction', value: 'fiction' },
  { label: 'Drama', value: 'drama' },
  { label: 'Humour', value: 'humor' },
  { label: 'Politics', value: 'politics' },
  { label: 'Philosophy', value: 'philosophy' },
  { label: 'History', value: 'history' },
  { label: 'Adventure', value: 'adventure' }
]

const currentView = ref('categories') // 'categories' | 'books'
const selectedCategory = ref(null)

const books = ref([])
const total = ref(null)
const nextPageUrl = ref(null)

const initialLoading = ref(false)
const loadingMore = ref(false)
const error = ref(null)

const searchQuery = ref('')
let searchDebounce = null

// For IntersectionObserver (infinite scroll)
const loadMoreRef = ref(null)
let observer = null

function selectCategory(cat) {
  selectedCategory.value = cat
  currentView.value = 'books'
  searchQuery.value = ''
  resetBooksState()
  fetchBooks()
}

function goBack() {
  currentView.value = 'categories'
  selectedCategory.value = null
  resetBooksState()
}

function resetBooksState() {
  books.value = []
  total.value = null
  nextPageUrl.value = null
  error.value = null
  initialLoading.value = false
  loadingMore.value = false
}

// Build initial URL based on category + search
function buildInitialUrl() {
  const params = new URLSearchParams()

  // topic → category (subjects/bookshelves)
  if (selectedCategory.value?.value) {
    params.append('topic', selectedCategory.value.value)
  }

  // search → title + author (Gutendex behaviour)
  if (searchQuery.value.trim()) {
    params.append('search', searchQuery.value.trim())
  }

  // only books with cover images
  params.append('mime_type', 'image')

  return `${BASE_URL}/books?${params.toString()}`
}

// Normalize Gutendex book to frontend shape
function normalizeBook(raw) {
  // coverImage: first image/* format
  let coverImage = null
  if (raw.formats) {
    for (const [mime, url] of Object.entries(raw.formats)) {
      if (mime.startsWith('image/')) {
        coverImage = url
        break
      }
    }
  }

  return {
    id: raw.id,
    title: raw.title,
    authors: raw.authors?.map(a => a.name) ?? [],
    languages: raw.languages ?? [],
    subjects: raw.subjects ?? [],
    download_count: raw.download_count ?? null,
    coverImage,
    formats: raw.formats || {}
  }
}

// Fetch first page
async function fetchBooks() {
  if (!selectedCategory.value) return

  initialLoading.value = true
  error.value = null

  try {
    const url = buildInitialUrl()
    const res = await fetch(url)
    if (!res.ok) throw new Error('Failed to fetch books')

    const data = await res.json()
    total.value = data.count ?? null
    nextPageUrl.value = data.next

    books.value = (data.results || []).map(normalizeBook)
  } catch (e) {
    console.error(e)
    error.value = 'Unable to load books. Please try again.'
  } finally {
    initialLoading.value = false
  }
}

// Fetch next page (infinite scroll)
async function loadMore() {
  if (!nextPageUrl.value || loadingMore.value) return
  loadingMore.value = true
  error.value = null

  try {
    const res = await fetch(nextPageUrl.value)
    if (!res.ok) throw new Error('Failed to load more books')

    const data = await res.json()
    nextPageUrl.value = data.next
    const newBooks = (data.results || []).map(normalizeBook)
    books.value = books.value.concat(newBooks)
  } catch (e) {
    console.error(e)
    error.value = 'Unable to load more books.'
  } finally {
    loadingMore.value = false
  }
}

// Open preferred format: HTML > PDF > TXT (formats is an object)
function openBook(book) {
  const formats = book.formats || {}

  let htmlUrl = null
  let pdfUrl = null
  let txtUrl = null

  for (const [mime, url] of Object.entries(formats)) {
    const lower = mime.toLowerCase()

    if (!htmlUrl && lower.startsWith('text/html')) {
      htmlUrl = url
    } else if (!pdfUrl && lower.includes('pdf')) {
      pdfUrl = url
    } else if (!txtUrl && lower.startsWith('text/plain')) {
      txtUrl = url
    }
  }

  const finalUrl = htmlUrl || pdfUrl || txtUrl

  if (finalUrl) {
    window.open(finalUrl, '_blank')
  } else {
    alert('No viewable version available')
  }
}

// Watch searchQuery with debounce → refetch from page 1
watch(searchQuery, () => {
  if (currentView.value !== 'books' || !selectedCategory.value) return

  if (searchDebounce) clearTimeout(searchDebounce)
  searchDebounce = setTimeout(() => {
    resetBooksState()
    fetchBooks()
  }, 500) // 0.5s debounce
})

// Setup IntersectionObserver for infinite scroll
onMounted(() => {
  observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          // Sentinel is visible -> load next page
          loadMore()
        }
      })
    },
    {
      root: null,          // viewport
      rootMargin: '200px', // start loading a bit before bottom
      threshold: 0
    }
  )

  // In case the ref is already set when mounted
  if (loadMoreRef.value) {
    observer.observe(loadMoreRef.value)
  }
})

// Watch the DOM ref – when the sentinel appears/disappears
watch(
  loadMoreRef,
  (el, oldEl) => {
    if (!observer) return

    if (oldEl) {
      observer.unobserve(oldEl)
    }
    if (el) {
      observer.observe(el)
    }
  }
)

onBeforeUnmount(() => {
  if (observer) {
    if (loadMoreRef.value) {
      observer.unobserve(loadMoreRef.value)
    }
    observer.disconnect()
  }
})

</script>
