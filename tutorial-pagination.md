# Pagination on the Posts Index Page

Uses **pagy** (the standard Rails pagination gem) + Inertia query params. Page state lives in the URL (`?page=2`) so links are shareable and the back button works.

---

## 1. Install Pagy

```bash
bundle add pagy
```

Create the initializer `config/initializers/pagy.rb`:

```ruby
require "pagy/extras/metadata"

Pagy::DEFAULT[:limit] = 20  # items per page
```

The `metadata` extra exposes pagination info (current page, total pages, etc.) as a plain hash you can pass to React.

---

## 2. Include Pagy in the Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < InertiaController
  include Pagy::Backend

  def index
    pagy, posts = pagy(Post.order(created_at: :desc))

    render inertia: "posts/index", props: {
      posts: posts.as_json(only: %i[id title created_at]),
      pagination: pagy_metadata(pagy)
    }
  end
end
```

`pagy_metadata` returns a hash like:

```json
{
  "count": 83,
  "page": 1,
  "limit": 20,
  "pages": 5,
  "next": 2,
  "prev": null
}
```

---

## 3. Read the Page Param from the URL

Pagy automatically reads `params[:page]`. When the user navigates to `?page=3`, Rails picks it up with no extra code.

---

## 4. React Page — `app/frontend/pages/posts/index.tsx`

```tsx
import { router, usePage } from "@inertiajs/react"

import { Button } from "@/components/ui/button"
import AppLayout from "@/layouts/app-layout"

interface Post {
  id: number
  title: string
  created_at: string
}

interface Pagination {
  count: number
  page: number
  limit: number
  pages: number
  next: number | null
  prev: number | null
}

export default function PostsIndex() {
  const { posts, pagination } = usePage<{
    posts: Post[]
    pagination: Pagination
  }>().props

  const goToPage = (page: number) => {
    router.get(
      window.location.pathname,
      { page },
      { preserveScroll: false }
    )
  }

  return (
    <AppLayout>
      <div className="p-4 space-y-4">
        <ul className="space-y-2">
          {posts.map((post) => (
            <li key={post.id} className="rounded border p-2">
              {post.title}
            </li>
          ))}
        </ul>

        <Paginator pagination={pagination} onPageChange={goToPage} />
      </div>
    </AppLayout>
  )
}

function Paginator({
  pagination,
  onPageChange,
}: {
  pagination: Pagination
  onPageChange: (page: number) => void
}) {
  const { page, pages, prev, next } = pagination

  if (pages <= 1) return null

  return (
    <div className="flex items-center gap-2">
      <Button
        variant="outline"
        size="sm"
        disabled={!prev}
        onClick={() => prev && onPageChange(prev)}
      >
        Previous
      </Button>

      <span className="text-sm text-muted-foreground">
        Page {page} of {pages}
      </span>

      <Button
        variant="outline"
        size="sm"
        disabled={!next}
        onClick={() => next && onPageChange(next)}
      >
        Next
      </Button>
    </div>
  )
}
```

---

## 5. Numbered Page Buttons (Optional)

Replace the `Paginator` component with this version to show individual page numbers:

```tsx
function Paginator({
  pagination,
  onPageChange,
}: {
  pagination: Pagination
  onPageChange: (page: number) => void
}) {
  const { page, pages, prev, next } = pagination

  if (pages <= 1) return null

  const pageNumbers = Array.from({ length: pages }, (_, i) => i + 1)

  return (
    <div className="flex items-center gap-1">
      <Button
        variant="outline"
        size="sm"
        disabled={!prev}
        onClick={() => prev && onPageChange(prev)}
      >
        ←
      </Button>

      {pageNumbers.map((n) => (
        <Button
          key={n}
          variant={n === page ? "default" : "outline"}
          size="sm"
          onClick={() => onPageChange(n)}
        >
          {n}
        </Button>
      ))}

      <Button
        variant="outline"
        size="sm"
        disabled={!next}
        onClick={() => next && onPageChange(next)}
      >
        →
      </Button>
    </div>
  )
}
```

For large page counts, add a windowing step to only show nearby pages (e.g. `pageNumbers.filter(n => Math.abs(n - page) <= 2)`).

---

## How Navigation Works

`router.get(pathname, { page: 2 })` triggers an Inertia visit to `?page=2`. Rails picks up `params[:page]`, pagy queries the right slice, and React re-renders with new props. The URL updates, so the back button and bookmarks work correctly.

`preserveScroll: false` scrolls back to the top on page change — set to `true` if you'd rather stay in place.

---

## Changing Items Per Page

Pass a `limit` param alongside `page`:

```ruby
pagy(Post.order(created_at: :desc), limit: params.fetch(:limit, 20).to_i)
```

```tsx
router.get(window.location.pathname, { page: 1, limit: 50 })
```
