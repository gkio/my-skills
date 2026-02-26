# Presentation Layer — Full Patterns

## Table of Contents
1. [Query Key Factory](#query-key-factory)
2. [Parallel Queries](#parallel-queries)
3. [Optimistic Updates](#optimistic-updates)
4. [Prefetch on Hover](#prefetch-on-hover)
5. [Infinite / Paginated Lists](#infinite--paginated-lists)
6. [Compound Component Pattern](#compound-component-pattern)
7. [Skeleton & Error States](#skeleton--error-states)
8. [Mutation with UI Feedback](#mutation-with-ui-feedback)

---

## Query Key Factory

Central key definitions prevent cache bugs. Always define keys in the same file as the query hooks.

```tsx
// queries/campaign-keys.ts
export const campaignKeys = {
  all: ['campaigns'] as const,
  lists: () => [...campaignKeys.all, 'list'] as const,
  list: (filters: CampaignFilters) => [...campaignKeys.lists(), filters] as const,
  details: () => [...campaignKeys.all, 'detail'] as const,
  detail: (id: string) => [...campaignKeys.details(), id] as const,
}

// Invalidate all campaign queries:  qc.invalidateQueries({ queryKey: campaignKeys.all })
// Invalidate only detail queries:   qc.invalidateQueries({ queryKey: campaignKeys.details() })
// Invalidate one detail:            qc.invalidateQueries({ queryKey: campaignKeys.detail(id) })
```

---

## Parallel Queries

Use `useQueries` for a dynamic list of independent queries.

```tsx
// queries/use-campaign-details.ts
export function useCampaignDetails(ids: string[]) {
  return useQueries({
    queries: ids.map(id => ({
      queryKey: campaignKeys.detail(id),
      queryFn: () => fetchCampaign(id),
      staleTime: 60_000,
    })),
    combine: (results) => ({
      data: results.map(r => r.data).filter(Boolean),
      isPending: results.some(r => r.isPending),
      isError: results.some(r => r.isError),
    }),
  })
}

// Usage in container
function CampaignComparisonContainer({ ids }: { ids: string[] }) {
  const { data, isPending } = useCampaignDetails(ids)
  if (isPending) return <ComparisonSkeleton count={ids.length} />
  return <CampaignComparison campaigns={data} />
}
```

---

## Optimistic Updates

```tsx
// mutations/use-toggle-campaign-status.ts
export function useToggleCampaignStatus() {
  const qc = useQueryClient()

  return useMutation({
    mutationFn: ({ id, status }: { id: string; status: 'active' | 'paused' }) =>
      updateCampaignStatus(id, status),

    onMutate: async ({ id, status }) => {
      // Cancel outgoing refetches for this item
      await qc.cancelQueries({ queryKey: campaignKeys.detail(id) })

      // Snapshot for rollback
      const previous = qc.getQueryData(campaignKeys.detail(id))

      // Optimistically update
      qc.setQueryData(campaignKeys.detail(id), (old: Campaign) => ({
        ...old,
        status,
        statusLabel: status === 'active' ? 'Active' : 'Paused',
      }))

      return { previous, id }
    },

    onError: (_, __, context) => {
      // Rollback on failure
      if (context?.previous) {
        qc.setQueryData(campaignKeys.detail(context.id), context.previous)
      }
    },

    onSettled: (_, __, { id }) => {
      qc.invalidateQueries({ queryKey: campaignKeys.detail(id) })
    },
  })
}
```

---

## Prefetch on Hover

```tsx
// containers/campaign-list-container.tsx
function CampaignListContainer() {
  const qc = useQueryClient()
  const { data, isPending } = useCampaigns(filters)

  const prefetchCampaign = (id: string) => {
    qc.prefetchQuery({
      queryKey: campaignKeys.detail(id),
      queryFn: () => fetchCampaign(id),
      staleTime: 60_000,
    })
  }

  if (isPending) return <CampaignListSkeleton />

  return (
    <CampaignList
      items={data}
      onItemHover={prefetchCampaign}  // prefetch detail on hover
    />
  )
}

// CampaignList component just calls onItemHover — no query knowledge
function CampaignList({ items, onItemHover }: CampaignListProps) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onMouseEnter={() => onItemHover(item.id)}>
          <CampaignRow {...item} />
        </li>
      ))}
    </ul>
  )
}
```

---

## Infinite / Paginated Lists

```tsx
// queries/use-campaigns-infinite.ts
export function useCampaignsInfinite(filters: Omit<CampaignFilters, 'page'>) {
  return useInfiniteQuery({
    queryKey: campaignKeys.list({ ...filters, infinite: true }),
    queryFn: ({ pageParam }) => fetchCampaigns({ ...filters, page: pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.hasMore ? lastPage.page + 1 : undefined,
  })
}

// Container bridges query → pure list component
function CampaignInfiniteContainer() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useCampaignsInfinite(filters)

  const allItems = data?.pages.flatMap(p => p.items) ?? []

  return (
    <CampaignScrollList
      items={allItems}
      onLoadMore={fetchNextPage}
      hasMore={hasNextPage ?? false}
      isLoadingMore={isFetchingNextPage}
    />
  )
}

// Pure component — no query knowledge at all
function CampaignScrollList({ items, onLoadMore, hasMore, isLoadingMore }: Props) {
  return (
    <div>
      {items.map(item => <CampaignRow key={item.id} {...item} />)}
      {hasMore && (
        <button onClick={onLoadMore} disabled={isLoadingMore}>
          {isLoadingMore ? 'Loading...' : 'Load more'}
        </button>
      )}
    </div>
  )
}
```

---

## Compound Component Pattern

Replace boolean props with explicit composition.

```tsx
// components/metric-card/index.tsx
type MetricCardContextValue = { size: 'sm' | 'md' | 'lg' }
const MetricCardContext = createContext<MetricCardContextValue>({ size: 'md' })

function MetricCard({ children, size = 'md' }: MetricCardRootProps) {
  return (
    <MetricCardContext.Provider value={{ size }}>
      <div className="rounded-lg border bg-card p-4">{children}</div>
    </MetricCardContext.Provider>
  )
}

function MetricCardLabel({ children }: { children: React.ReactNode }) {
  return <p className="text-sm text-muted-foreground">{children}</p>
}

function MetricCardValue({ value, trend }: { value: string; trend: 'up' | 'down' | 'flat' }) {
  const { size } = useContext(MetricCardContext)
  return (
    <div className="flex items-center gap-2">
      <span className={cn('font-bold', size === 'lg' ? 'text-3xl' : 'text-2xl')}>{value}</span>
      <TrendIcon direction={trend} />
    </div>
  )
}

function MetricCardDelta({ value, percent }: { value: number; percent: number }) {
  return (
    <p className={cn('text-xs', value >= 0 ? 'text-green-600' : 'text-red-600')}>
      {value >= 0 ? '+' : ''}{percent.toFixed(1)}%
    </p>
  )
}

MetricCard.Label = MetricCardLabel
MetricCard.Value = MetricCardValue
MetricCard.Delta = MetricCardDelta

// Usage — flexible, no boolean flag proliferation
<MetricCard size="lg">
  <MetricCard.Label>Revenue</MetricCard.Label>
  <MetricCard.Value value={data.revenueFormatted} trend={data.revenueTrend} />
  <MetricCard.Delta value={data.revenueDelta} percent={data.revenueDeltaPercent} />
</MetricCard>
```

---

## Skeleton & Error States

Every component that has a container gets matching skeleton and error variants.

```tsx
// components/campaign-card.tsx — pure render
export function CampaignCard(props: CampaignCardProps) { ... }

// components/campaign-card-skeleton.tsx — loading state
export function CampaignCardSkeleton() {
  return (
    <div className="rounded-lg border p-4 animate-pulse">
      <div className="h-4 w-32 bg-muted rounded" />
      <div className="mt-2 h-3 w-16 bg-muted rounded" />
      <div className="mt-3 h-8 w-24 bg-muted rounded" />
    </div>
  )
}

// components/campaign-card-error.tsx — error state
export function CampaignCardError({ onRetry }: { onRetry?: () => void }) {
  return (
    <div className="rounded-lg border border-destructive/20 p-4">
      <p className="text-sm text-destructive">Failed to load campaign</p>
      {onRetry && (
        <button onClick={onRetry} className="mt-2 text-xs underline">
          Try again
        </button>
      )}
    </div>
  )
}

// containers/campaign-card-container.tsx — glue
export function CampaignCardContainer({ id }: { id: string }) {
  const { data, isPending, isError, refetch } = useCampaign(id)
  if (isPending) return <CampaignCardSkeleton />
  if (isError) return <CampaignCardError onRetry={refetch} />
  return <CampaignCard {...data} />
}
```

---

## Mutation with UI Feedback

```tsx
// The component receives callbacks — it knows nothing about mutations
function CampaignActions({ id, status }: { id: string; status: string }) {
  const { mutate: toggle, isPending } = useToggleCampaignStatus()

  return (
    <div className="flex gap-2">
      <Button
        variant="outline"
        size="sm"
        disabled={isPending}
        onClick={() => toggle({ id, status: status === 'active' ? 'paused' : 'active' })}
      >
        {isPending ? (
          <Loader2 className="h-4 w-4 animate-spin" />
        ) : status === 'active' ? (
          'Pause'
        ) : (
          'Resume'
        )}
      </Button>
    </div>
  )
}
```
