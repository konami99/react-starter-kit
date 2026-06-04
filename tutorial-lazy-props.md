# Button-Triggered Data Loading with Inertia Lazy Props

The cleanest way to load data on button click in this stack is **Inertia lazy props**. The prop is omitted from the initial page load and only fetched when you explicitly ask for it.

---

## How It Works

1. On the Rails side, wrap the prop in `InertiaRails.lazy { }` — it won't be evaluated on the first render
2. On the React side, call `router.reload({ only: ['posts'] })` on button click
3. Inertia sends a partial XHR request, Rails evaluates the lazy block, and the component re-renders with the data

---

## Implementation

### 1. Controller — `app/controllers/posts_controller.rb`

```ruby
class PostsController < InertiaController
  def index
    render inertia: "posts/index", props: {
      posts: InertiaRails.lazy { Post.all.as_json(only: %i[id title created_at]) }
    }
  end
end
```

`InertiaRails.lazy` means the block is **skipped on the first request**. The prop arrives as `null` in React until reloaded.

### 2. React Page — `app/frontend/pages/posts/index.tsx`

```tsx
import { router, usePage } from "@inertiajs/react"
import { useState } from "react"

import { Button } from "@/components/ui/button"
import AppLayout from "@/layouts/app-layout"

interface Post {
  id: number
  title: string
  created_at: string
}

export default function PostsIndex() {
  const { posts } = usePage<{ posts: Post[] | null }>().props
  const [loading, setLoading] = useState(false)

  const loadPosts = () => {
    setLoading(true)
    router.reload({
      only: ["posts"],
      onFinish: () => setLoading(false),
    })
  }

  return (
    <AppLayout>
      <div className="p-4">
        {!posts && (
          <Button onClick={loadPosts} disabled={loading}>
            {loading ? "Loading..." : "Load Posts"}
          </Button>
        )}

        {posts && (
          <ul className="mt-4 space-y-2">
            {posts.map((post) => (
              <li key={post.id} className="rounded border p-2">
                {post.title}
              </li>
            ))}
          </ul>
        )}
      </div>
    </AppLayout>
  )
}
```

---

## Key Details

**`only: ["posts"]`** — tells Inertia to fetch just this prop, not re-run the whole page. The string must match the key name in your Rails props hash exactly.

**`posts` starts as `null`** — because the lazy block never runs on the first request. Use this to decide whether to show the button or the list.

**`onFinish`** — fires after the reload completes (success or error), safe to use for resetting loading state. Use `onSuccess` if you only want to react to a successful load.

**The button hides itself** (`!posts && ...`) — once posts load they replace the button. If you want a refresh button that stays visible, remove that condition.

---

## Variant: Show a Loading Skeleton

If you want to show a skeleton immediately when the button is clicked (before data arrives):

```tsx
{!posts && !loading && (
  <Button onClick={loadPosts}>Load Posts</Button>
)}

{!posts && loading && (
  <div className="space-y-2 mt-4">
    {[1, 2, 3].map(i => (
      <div key={i} className="h-8 rounded bg-muted animate-pulse" />
    ))}
  </div>
)}

{posts && (
  <ul>...</ul>
)}
```

---

## Alternative: Plain Fetch (When You Don't Want a Full Page Reload)

If you need more control (e.g., infinite scroll, background polling) and don't want Inertia to update the page props, use a regular fetch against a JSON endpoint instead:

```ruby
# config/routes.rb
namespace :api do
  resources :posts, only: [:index]
end
```

```ruby
# app/controllers/api/posts_controller.rb
class Api::PostsController < ApplicationController
  def index
    render json: Post.all.as_json(only: %i[id title created_at])
  end
end
```

```tsx
const [posts, setPosts] = useState<Post[] | null>(null)
const [loading, setLoading] = useState(false)

const loadPosts = async () => {
  setLoading(true)
  const res = await fetch("/api/posts")
  const data = await res.json()
  setPosts(data)
  setLoading(false)
}
```

Use this approach when the loaded data is **local to the component** and shouldn't be reflected in the page URL or browser history. Use the lazy props approach when it's part of the page's canonical data.
