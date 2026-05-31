# Persistent Layouts in Inertia (and how to vary them per page)

This doc explains how to keep a layout — like the app sidebar — **mounted across
Inertia navigations**, and how to make that persistent layout still **show
different things depending on the current page**.

## The core idea

Inertia has two layout styles:

- **Per-page layout (default in this repo today):** the page renders its own
  layout in its JSX (e.g. `<AppLayout>...</AppLayout>`). Because the layout lives
  *inside* the page component, Inertia tears it down and rebuilds it on every
  navigation. State (sidebar open/closed, scroll position) is lost or has to be
  re-read from storage.

- **Persistent layout:** the layout is set once at bootstrap
  (`entrypoints/inertia.tsx` → `layout: () => [PersistentLayout]`) and Inertia
  keeps that component **mounted** across navigations. Only the page
  (`{children}`) swaps. State inside the layout survives.

The trade-off that trips people up:

> A persistent layout is **the same component instance** across navigations.
> You **cannot** make it vary by giving each page a different layout tree — that
> would remount it and destroy the persistence you wanted.

So how do you make a persistent layout look different per page? **You don't swap
the component — you let the mounted component read the current page and
re-render reactively.**

## The key fact

`usePage()` from `@inertiajs/react` returns fresh values on every navigation:

```ts
const { component, url, props } = usePage()
// component → "dashboard/index", "settings/profiles/show", ...
// url       → "/dashboard", "/settings/profile", ...
// props     → shared props + the page's props
```

Any component that reads `usePage()` **re-renders when the page changes, without
unmounting**. That's the lever: the shell stays alive while its contents change.

---

## Technique 1 — Branch on `usePage().component` / `.url`

Decide *whether* to show the sidebar, or *what* to show in it, by reading the
current page name. Example: the persistent layout decides if a page gets the
sidebar shell at all.

```tsx
// layouts/persistent-layout.tsx
const SIDEBAR_ROUTE_PREFIXES = ["dashboard", "settings"]

export default function PersistentLayout({ children }: { children: ReactNode }) {
  useFlash()
  const { component } = usePage()
  const showSidebar = SIDEBAR_ROUTE_PREFIXES.some((p) => component.startsWith(p))

  return (
    <>
      {showSidebar ? (
        <AppShell variant="sidebar">
          <AppSidebar />
          <AppContent variant="sidebar" className="overflow-x-hidden">
            {children}
          </AppContent>
        </AppShell>
      ) : (
        children
      )}
      <Toaster richColors />
    </>
  )
}
```

You can push the same idea *inside* `AppSidebar` to vary its nav items by
section while the shell never remounts:

```tsx
export function AppSidebar() {
  const { component } = usePage()
  const inSettings = component.startsWith("settings")

  return (
    <Sidebar collapsible="icon" variant="inset">
      <SidebarContent>
        <NavMain items={inSettings ? settingsNavItems : mainNavItems} />
      </SidebarContent>
    </Sidebar>
  )
}
```

This is already how active highlighting works: `components/nav-main.tsx` reads
`usePage().url` and computes `isActive={page.url.startsWith(item.href)}` on each
render. The sidebar stays mounted; only the highlight moves.

---

## Technique 2 — Drive it from server-shared props

When the **backend** should decide what the layout shows (permissions, current
org, feature flags), share props from the controller and read them in the layout.

```ruby
# app/controllers/... (or InertiaController for app-wide)
inertia_share sidebar: -> { { variant: "settings", title: "Account" } }
```

```tsx
const { props } = usePage()
<AppShell variant={props.sidebar?.variant ?? "sidebar"}>...</AppShell>
```

`AppShell` already accepts a `variant` prop (`"header" | "sidebar"`), so this
slots in directly.

---

## Technique 3 — Let a page push data *up* via context

Some data is page-specific (breadcrumbs) but needs to appear in **persistent
chrome** (a header that never remounts). The page is `children` *inside* the
layout, so it can't pass props upward — use a small context instead.

```tsx
// persistent layout provides a slot
const [breadcrumbs, setBreadcrumbs] = useState<BreadcrumbItem[]>([])
return (
  <BreadcrumbContext.Provider value={setBreadcrumbs}>
    <AppShell variant="sidebar">
      <AppSidebar />
      <AppContent variant="sidebar">
        <AppSidebarHeader breadcrumbs={breadcrumbs} /> {/* persistent */}
        {children}
      </AppContent>
    </AppShell>
  </BreadcrumbContext.Provider>
)

// each page registers its breadcrumbs
function Dashboard() {
  useBreadcrumbs([{ title: "Dashboard", href: dashboard.index().url }])
  return <>{/* content */}</>
}
```

Now even the header persists, and pages just *declare* their breadcrumbs. More
plumbing, but maximum persistence.

> A lighter alternative used in the sidebar-migration plan: keep the **shell**
> persistent but let each page render `<AppSidebarHeader breadcrumbs={...} />`
> itself inside `{children}`. The header re-renders per page (fine — breadcrumbs
> differ), while only the sidebar + provider persist. No context needed.

---

## Technique 4 — Per-page layout property (accepts a remount)

Inertia supports `Page.layout = (page) => <Variant>{page}</Variant>`. Inertia
keeps a layout mounted **only when consecutive pages use the same layout
component**. If two sections use different shells, switching between them
*remounts*. Useful when two sections are genuinely different shells and you don't
need cross-section persistence — but it's the opposite of "keep the shell alive."

---

## The principle

> **Anything that must persist (stay mounted) needs a stable position in the
> tree. Anything that legitimately differs per page should be driven by reactive
> data — `usePage()` or server-shared props — not by swapping the component.**

| Need | Technique |
|---|---|
| Show/hide sidebar per route | 1 — branch on `usePage().component` |
| Different nav items / active state per section | 1 — `AppSidebar` reads `usePage()` |
| Backend decides layout variant | 2 — server-shared props |
| Page-owned data in persistent chrome (breadcrumbs) | 3 — context, or render in `children` |
| Two genuinely different shells, persistence not needed | 4 — per-page `Page.layout` |

## Relevant files in this repo

- `app/frontend/entrypoints/inertia.tsx` — sets the global persistent layout.
- `app/frontend/layouts/persistent-layout.tsx` — the always-mounted layout.
- `app/frontend/components/app-shell.tsx` — `SidebarProvider` + open/close state.
- `app/frontend/components/app-sidebar.tsx` — the sidebar nav.
- `app/frontend/components/nav-main.tsx` — reactive active highlighting via `usePage().url`.
- `app/frontend/components/app-sidebar-header.tsx` — breadcrumbs + `SidebarTrigger`.
