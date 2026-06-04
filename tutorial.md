# React Starter Kit — Tutorial

A practical guide to understanding and building with this codebase.

---

## What This Is

A full-stack starter kit built on **Rails 8.1 + React 19 + Inertia.js**. It ships with authentication, a settings UI, and a dashboard shell ready to build on.

The core idea: Rails handles routing and data, React handles rendering — but there's no REST API between them. Inertia.js is the bridge that makes this work seamlessly.

---

## Stack at a Glance

| Layer | Technology |
|---|---|
| Backend | Rails 8.1, Ruby 4.0 |
| Frontend | React 19, TypeScript |
| Bridge | Inertia.js 3.0 |
| Styling | Tailwind CSS v4, shadcn/ui |
| Build | Vite |
| Database | SQLite (Solid Queue, Solid Cache, Solid Cable) |
| Auth | Authentication Zero |
| Deployment | Kamal (Docker) |

---

## Getting Started

```bash
# First time setup — installs gems, npm packages, prepares the database
bin/setup

# Start the dev server (Rails + Vite simultaneously)
bin/dev
```

Visit `http://localhost:3000`. Sign up at `/sign_up` to create your first account.

To create an account from the console instead:

```bash
bin/rails console
User.create!(email: "you@example.com", password: "password123", password_confirmation: "password123")
```

---

## The Inertia.js Mental Model

This is the most important concept in the codebase.

In a traditional Rails app, a controller renders an HTML view. Here, the controller renders a **React component** instead — and passes data to it as props. There's no separate API, no `fetch()` calls for page data, no Redux store. Rails props flow directly into React.

```
Browser requests /dashboard
  → Rails DashboardController#index runs
  → Inertia renders app/frontend/pages/dashboard/index.tsx
  → Props (auth.user, etc.) arrive as React props
  → React renders the component
```

Clicking a link doesn't do a full page reload. Inertia intercepts it, fetches only the new component + props via XHR, and swaps the page — like a SPA, but driven by Rails routing.

---

## Directory Tour

```
app/
  controllers/          # Rails controllers — define what data pages receive
  models/               # User, Session (Authentication Zero)
  frontend/
    pages/              # React page components (one per route)
    components/         # Reusable UI (shadcn/ui based)
    layouts/            # Page wrapper layouts
    hooks/              # Custom React hooks
    lib/                # Utilities
    routes/             # Type-safe route helpers (typelizer generated)
    types/              # Shared TypeScript types
    entrypoints/        # Vite entry — inertia.tsx bootstraps the app
config/
  routes.rb             # All URL routes
  initializers/
    inertia_rails.rb    # Inertia config (SSR toggle, parent controller)
db/
  schema.rb             # users, sessions tables
```

---

## Adding a New Page

Here's the full cycle for adding a `/posts` page.

**1. Add a route** in `config/routes.rb`:

```ruby
resources :posts, only: [:index]
```

**2. Create the controller** `app/controllers/posts_controller.rb`:

```ruby
class PostsController < InertiaController
  def index
    render inertia: "posts/index", props: {
      posts: Post.all.as_json(only: %i[id title created_at])
    }
  end
end
```

> Extending `InertiaController` (not `ApplicationController` directly) ensures auth is required and `auth.user` is shared automatically.

**3. Create the React page** `app/frontend/pages/posts/index.tsx`:

```tsx
import AppLayout from "@/layouts/app-layout"

interface Post {
  id: number
  title: string
  created_at: string
}

export default function PostsIndex({ posts }: { posts: Post[] }) {
  return (
    <AppLayout>
      <ul>
        {posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </AppLayout>
  )
}
```

That's it. No API endpoint, no fetch, no state management for page data.

---

## Authentication

Authentication Zero is pre-wired. Here's how it works:

- `ApplicationController` runs `before_action :authenticate` on every request
- It checks `cookies.signed[:session_token]` and sets `Current.session`
- `Current.user` is available anywhere via Rails `CurrentAttributes`
- `InertiaController` shares `auth.user` and `auth.session` as Inertia shared props — available in every React page

To access the current user in a React component:

```tsx
import { usePage } from "@inertiajs/react"

const { auth } = usePage().props
console.log(auth.user.email)
```

To make a page public (no login required), mirror `HomeController`:

```ruby
class HomeController < InertiaController
  skip_before_action :authenticate
  before_action :perform_authentication  # still sets Current.user if logged in
end
```

---

## Shared Data (Inertia Props)

`InertiaController` shares data to every page automatically:

```ruby
# app/controllers/inertia_controller.rb
inertia_share auth: {
  user: -> { Current.user.as_json(only: %i[id name email verified created_at updated_at]) },
  session: -> { Current.session.as_json(only: %i[id]) }
}
```

To add more shared data (e.g., flash messages, feature flags), add keys here.

---

## Type-Safe Routes

Routes are generated as TypeScript helpers via typelizer. Instead of hardcoding strings:

```tsx
// Don't do this
<Link href="/dashboard">Dashboard</Link>

// Do this
import { dashboard } from "@/routes"
<Link href={dashboard.index().url}>Dashboard</Link>
```

After changing `config/routes.rb`, regenerate the route types:

```bash
bin/rails typelizer:generate
```

---

## Styling

Tailwind CSS v4 + shadcn/ui components. Add new shadcn components with:

```bash
npx shadcn@latest add button
```

Components land in `app/frontend/components/ui/`.

---

## Key Commands

```bash
bin/dev                        # Start dev server
bin/rails console              # Rails REPL
bin/rails db:migrate           # Run migrations
bin/rails routes               # List all routes
npm run type-check             # TypeScript check
npm run lint                   # ESLint
npm run format                 # Prettier
```

---

## What's Pre-Built

| Feature | Location |
|---|---|
| Sign up / Sign in / Sign out | `app/frontend/pages/sessions/`, `users/` |
| Email verification | `app/frontend/pages/identity/` |
| Password reset | `app/frontend/pages/identity/` |
| Dashboard shell | `app/frontend/pages/dashboard/` |
| Profile settings | `app/frontend/pages/settings/` |
| Password & email change | `app/frontend/pages/settings/` |
| Active sessions list | `app/frontend/pages/settings/sessions/` |
| Appearance toggle (dark mode) | `app/frontend/pages/settings/appearance/` |
| Sidebar layout | `app/frontend/layouts/app/` |
