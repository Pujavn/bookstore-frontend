# Bookstore Frontend

Vue 3 + Vite app that consumes the Gutenberg Bookstore API.

Features:
- Category selection (Fiction, Drama, Humour, etc.)
- Search on title/author while keeping category
- Infinite scroll (IntersectionObserver)
- Opens books in HTML → PDF → TXT order
- Uses only books with cover images (mime_type=image)

## Setup

```bash
npm install
cp .env.example .env   # if you create one, or set VITE_API_URL directly
npm run dev
