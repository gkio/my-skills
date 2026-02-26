---
name: presentation-layer
description: >
  Enforces the pure presentation layer pattern for React frontend development.
  Use when building, reviewing, or refactoring React UI components to ensure
  strict separation: TanStack Query hooks own ALL data fetching, caching, and
  mutations — UI components are purely for rendering beautiful pixels.
  Triggers on: building components that fetch data, reviewing components with
  useEffect/useState for server state, creating new features with UI + data,
  refactoring data-fetching components, implementing mutations with UI feedback.
---

# Presentation Layer Pattern

> **Query hooks own data. Components own pixels. Nothing else.**

## ⚠️ The #1 Rule — SRP: One File, One Job

> **Every file does exactly one thing. Max 100–150 lines. No exceptions.**

If a file is growing past 150 lines, it is doing more than one job. Split it.

| File type | Its one job |
|-----------|-------------|
| `use-campaigns.ts` | Fetch and cache campaigns |
| `use-update-campaign.ts` | Mutate a campaign |
| `campaign-card.tsx` | Render a campaign card |
| `campaign-card-skeleton.tsx` | Render the loading state |
| `campaign-card-container.tsx` | Connect query to component |

**Violations to split immediately:**
- A query file that also has mutation logic → split into two files
- A component that renders two different sections → split into two components
- A container that holds UI logic → extract to a component
- A hook that fetches AND transforms business data → split fetch from transform

## The Four Layers

```
queries/          ← TanStack Query: fetch, cache, invalidate
mutations/        ← TanStack Mutation: create, update, delete
components/       ← Pure render: receive props, return JSX
containers/       ← Bridge: query → component (thin glue only)
```

## Core Rules

1. **One file = one job, 100–150 LOC max** — if it grows, split it
2. **No `useEffect` for data** — TanStack Query handles all server state
3. **No business logic in components** — backend returns ready-to-render data
4. **No transformations in components** — use query `select` for shape changes
5. **Props are primitives** — strings, numbers, booleans, not raw API objects
6. **Composition over boolean flags** — no `isCompact`, `isLoading`, `showDelta` props

## Quick Patterns

### Query Hook

```tsx
// queries/use-campaigns.ts
const keys = {
  all: ['campaigns'] as const,
  list: (f: Filters) => ['campaigns', 'list', f] as const,
  detail: (id: string) => ['campaigns', 'detail', id] as const,
}

export function useCampaigns(filters: Filters) {
  return useQuery({
    queryKey: keys.list(filters),
    queryFn: () => fetchCampaigns(filters),
    staleTime: 30_000,
  })
}
```

### Mutation Hook

```tsx
// mutations/use-update-campaign.ts
export function useUpdateCampaign() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: UpdateCampaignPayload) => updateCampaign(payload),
    onSuccess: (_, { id }) => {
      qc.invalidateQueries({ queryKey: keys.detail(id) })
      qc.invalidateQueries({ queryKey: keys.all })
    },
  })
}
```

### Pure Component

```tsx
// components/campaign-card.tsx
// Backend already computed: statusLabel, revenueFormatted, roiPercent, trend
type CampaignCardProps = {
  name: string
  statusLabel: string       // "Active" — backend computed
  revenueFormatted: string  // "$12,400" — backend formatted
  roiPercent: number        // 24.5 — backend computed
  trend: 'up' | 'down' | 'flat'
}

export function CampaignCard(props: CampaignCardProps) {
  return (
    <div className="rounded-lg border p-4">
      <p className="font-medium">{props.name}</p>
      <Badge>{props.statusLabel}</Badge>
      <p className="text-2xl font-bold">{props.revenueFormatted}</p>
      <TrendIndicator value={props.roiPercent} direction={props.trend} />
    </div>
  )
}
```

### Container (Thin Glue)

```tsx
// containers/campaign-card-container.tsx
export function CampaignCardContainer({ id }: { id: string }) {
  const { data, isPending, isError } = useCampaign(id)

  if (isPending) return <CampaignCardSkeleton />
  if (isError) return <CampaignCardError />

  return <CampaignCard {...data} />  // data is already ready-to-render
}
```

## Anti-Pattern Checklist

When reviewing a component, flag any of these immediately:

| Found in component | Fix |
|--------------------|-----|
| `useEffect` + `fetch`/`setState` | Move to `useQuery` hook |
| `.reduce()` computing totals | Add computed field to backend response |
| `.filter()` for business rules | Compute server-side, pass as prop |
| Permission checks (`if role === ...`) | Backend returns `canEdit: boolean` |
| Status derivation from raw fields | Backend returns `statusLabel: string` |
| `useState` for server data | Use `useQuery` |
| Multiple boolean props (`isX`, `showY`) | Use compound components / variants |
| Spreading raw API object as props | Map to typed, flat props |

## Composition Pattern (No Boolean Props)


```tsx
// ❌ Bad — boolean prop explosion
<MetricCard value={v} isCompact isLoading showDelta isSelected />

// ✅ Good — compose explicit pieces
<MetricCard>
  <MetricCardLabel>Revenue</MetricCardLabel>
  <MetricCardValue value={data.revenueFormatted} trend={data.revenueTrend} />
</MetricCard>
```

## select — Only Legitimate Client Transform

`select` is allowed for **UI shape adaptation only**, not business logic:

```tsx
// ✅ OK — adapting shape for a dropdown
export function useCampaignOptions() {
  return useQuery({
    queryKey: keys.all,
    queryFn: fetchCampaigns,
    select: (data) => data.map(c => ({ value: c.id, label: c.name })),
  })
}

// ❌ Not OK — business computation belongs in backend
select: (data) => data.map(c => ({
  ...c,
  roi: ((c.revenue - c.spend) / c.spend) * 100,  // ← backend's job
}))
```

## Component Hierarchy (Atomic Design)

Every component belongs to exactly one level. Level determines what it may import.

| Atomic Level | Maps To | One Job | May Import From |
|---|---|---|---|
| **Atom** | Leaf component (`Button`, `Badge`, `TrendIcon`) | Render one UI primitive | Quarks (tokens) only |
| **Molecule** | Composed component (`MetricCard`, `CampaignRow`) | Combine atoms into one unit | Atoms only |
| **Organism** | Feature section — **containers live here** | Connect data to a section | Molecules + Atoms + Query/Mutation hooks |
| **Template** | Layout shell (`DashboardLayout`) | Define page structure | Organisms |
| **Page** | Route component | Compose the full view | Templates + Organisms |

**The import direction is one-way, always downward:**
```
Page → Template → Organism/Container → Molecule → Atom
```

```tsx
// ✅ Correct — Molecule imports Atoms only
import { Badge } from '@/components/atoms/badge'
import { TrendIcon } from '@/components/atoms/trend-icon'
export function CampaignRow({ name, statusLabel, trend }: Props) { ... }

// ❌ Wrong — Atom reaching up into a Molecule
// atoms/button.tsx
import { FormField } from '@/components/molecules/form-field' // NEVER
```

## Orthogonality Test

Before shipping any component, answer these three questions:

1. **"If the API renames a field, which files change?"**
   → Only the container. If the component file changes too, the container is leaking API shape into the component.

2. **"Can I render this component with a plain mock object in isolation?"**
   → Yes, always. If it requires a query provider or real data, it has hidden coupling.

3. **"Does the same value get computed in both the query `select` and a component conditional?"**
   → If yes, you have duplicate logic that will drift. Pick one place — backend or `select`, never the component.

**Non-orthogonality red flags in frontend:**

| Symptom | What it means |
|---|---|
| API field rename breaks a component file | Container leaks API shape |
| Two components compute the same status label differently | Missing single source of truth |
| Query hook also handles optimistic UI state | Hook doing two jobs — split it |
| Container has conditional rendering logic | Business rule crept into the glue layer |
| `select` computes both shape AND a business value | Mix of UI transform + business logic |

## File Structure

```
queries/
  use-campaigns.ts          ← one query domain per file
  use-campaign-detail.ts    ← separate file if detail needs differ
mutations/
  use-update-campaign.ts    ← one mutation per file
  use-delete-campaign.ts
components/
  atoms/
    badge.tsx
    trend-icon.tsx
  molecules/
    campaign-row.tsx        ← composes atoms, ~50-80 LOC
    metric-card/
      index.tsx             ← compound root
      metric-card-label.tsx ← each sub-component its own file
      metric-card-value.tsx
  organisms/
    campaign-card-container.tsx   ← query → molecule (thin glue, ~30 LOC)
    campaign-list-container.tsx
```

## Detailed Reference

- **[patterns.md](references/patterns.md)** — Full code examples: parallel queries, optimistic updates, prefetch on hover, infinite scroll
- **[anti-patterns.md](references/anti-patterns.md)** — Before/after refactoring examples for common violations
