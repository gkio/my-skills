# Presentation Layer — Anti-Patterns & Refactoring Guide

## Table of Contents
1. [useEffect for Data Fetching](#useeffect-for-data-fetching)
2. [Business Logic in Components](#business-logic-in-components)
3. [Raw API Objects as Props](#raw-api-objects-as-props)
4. [Boolean Prop Explosion](#boolean-prop-explosion)
5. [Permission Logic in UI](#permission-logic-in-ui)
6. [Multiple useState for Server Data](#multiple-usestate-for-server-data)

---

## useEffect for Data Fetching

### Before (Wrong)
```tsx
function CampaignList() {
  const [campaigns, setCampaigns] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    setLoading(true)
    fetch('/api/campaigns')
      .then(r => r.json())
      .then(data => { setCampaigns(data); setLoading(false) })
      .catch(err => { setError(err); setLoading(false) })
  }, [])  // ← no cleanup, no deduplication, no caching

  if (loading) return <Spinner />
  if (error) return <ErrorMessage />
  return <div>{campaigns.map(c => <CampaignRow key={c.id} {...c} />)}</div>
}
```

### After (Correct)
```tsx
// queries/use-campaigns.ts
export function useCampaigns() {
  return useQuery({
    queryKey: campaignKeys.lists(),
    queryFn: fetchCampaigns,
    staleTime: 30_000,
  })
}

// containers/campaign-list-container.tsx
function CampaignListContainer() {
  const { data, isPending, isError } = useCampaigns()
  if (isPending) return <CampaignListSkeleton />
  if (isError) return <CampaignListError />
  return <CampaignList campaigns={data} />
}

// components/campaign-list.tsx — pure render
function CampaignList({ campaigns }: { campaigns: CampaignRow[] }) {
  return <div>{campaigns.map(c => <CampaignRow key={c.id} {...c} />)}</div>
}
```

**Why:** TanStack Query gives you deduplication, caching, background refetch, retry, and devtools for free. `useEffect` gives you none of that.

---

## Business Logic in Components

### Before (Wrong)
```tsx
function CampaignCard({ campaign }: { campaign: RawCampaign }) {
  // ❌ All of this belongs in the backend
  const revenue = campaign.orders.reduce((sum, o) => sum + o.total, 0)
  const spend = campaign.adSets.reduce((sum, a) => sum + a.spend, 0)
  const roi = spend > 0 ? ((revenue - spend) / spend) * 100 : 0
  const status = campaign.endDate && new Date(campaign.endDate) < new Date()
    ? 'ended'
    : campaign.startDate && new Date(campaign.startDate) > new Date()
    ? 'scheduled'
    : 'active'
  const statusLabel = { active: 'Active', ended: 'Ended', scheduled: 'Scheduled' }[status]

  return (
    <div>
      <Badge>{statusLabel}</Badge>
      <p>{roi.toFixed(1)}% ROI</p>
      <p>${revenue.toLocaleString()}</p>
    </div>
  )
}
```

### After (Correct)

Backend returns ready-to-render data:
```tsx
// Backend API response shape (backend's responsibility)
type CampaignCard = {
  id: string
  name: string
  statusLabel: 'Active' | 'Ended' | 'Scheduled'   // backend computes
  roiPercent: number                                // backend computes
  revenueFormatted: string                          // "$12,400"
  trend: 'up' | 'down' | 'flat'                    // backend computes
}

// Component is trivial — just render
function CampaignCard(props: CampaignCard) {
  return (
    <div>
      <Badge>{props.statusLabel}</Badge>
      <p>{props.roiPercent.toFixed(1)}% ROI</p>
      <p>{props.revenueFormatted}</p>
    </div>
  )
}
```

---

## Raw API Objects as Props

### Before (Wrong)
```tsx
// ❌ Passing entire raw API object — component now knows API shape
function CampaignDashboard() {
  const { data } = useCampaign(id)
  return <CampaignCard campaign={data} />  // raw object
}

function CampaignCard({ campaign }: { campaign: ApiCampaignResponse }) {
  // component must navigate nested structure
  return <p>{campaign.meta.display.label}</p>
}
```

### After (Correct)
```tsx
// Container maps API shape → flat typed props
function CampaignCardContainer({ id }: { id: string }) {
  const { data, isPending, isError } = useCampaign(id)
  if (isPending) return <CampaignCardSkeleton />
  if (isError) return <CampaignCardError />

  // Explicit mapping — component gets flat, predictable props
  return (
    <CampaignCard
      name={data.name}
      statusLabel={data.statusLabel}
      revenueFormatted={data.revenueFormatted}
      trend={data.trend}
    />
  )
}

// Component has zero knowledge of API structure
function CampaignCard({ name, statusLabel, revenueFormatted, trend }: CampaignCardProps) {
  return <p>{name}</p>
}
```

---

## Boolean Prop Explosion

### Before (Wrong)
```tsx
// ❌ 6 boolean props = 64 possible states, impossible to reason about
<MetricCard
  value={revenue}
  isLoading={isPending}
  isError={isError}
  isCompact={isMobile}
  showDelta={hasComparison}
  showTrend={showTrendEnabled}
  isHighlighted={isSelected}
/>
```

### After (Correct)
```tsx
// ✅ Composable — caller decides what to show
// Loading state handled by container, not by prop
// Compact variant = different component or CSS class

// Container handles loading/error:
if (isPending) return <MetricCardSkeleton />

// Composition handles what to render:
<MetricCard className={cn(isMobile && 'compact')}>
  <MetricCard.Label>Revenue</MetricCard.Label>
  <MetricCard.Value value={data.revenueFormatted} trend={data.trend} />
  {hasComparison && (
    <MetricCard.Delta percent={data.revenueDeltaPercent} />
  )}
</MetricCard>
```

---

## Permission Logic in UI

### Before (Wrong)
```tsx
// ❌ Frontend deciding permissions from raw user/resource data
function CampaignActions({ campaign, user }) {
  const canEdit = user.role === 'admin' || campaign.ownerId === user.id
  const canDelete = user.role === 'admin' && campaign.status !== 'active'
  const canPublish = user.role !== 'viewer' && campaign.status === 'draft'

  return (
    <div>
      {canEdit && <Button>Edit</Button>}
      {canDelete && <Button variant="destructive">Delete</Button>}
      {canPublish && <Button>Publish</Button>}
    </div>
  )
}
```

### After (Correct)
```tsx
// Backend returns resolved boolean flags
type CampaignPermissions = {
  canEdit: boolean    // backend resolves
  canDelete: boolean  // backend resolves
  canPublish: boolean // backend resolves
}

// Component just renders — no permission logic
function CampaignActions({ canEdit, canDelete, canPublish, onEdit, onDelete, onPublish }) {
  return (
    <div>
      {canEdit && <Button onClick={onEdit}>Edit</Button>}
      {canDelete && <Button variant="destructive" onClick={onDelete}>Delete</Button>}
      {canPublish && <Button onClick={onPublish}>Publish</Button>}
    </div>
  )
}

// Container connects mutations and passes resolved permissions
function CampaignActionsContainer({ campaignId }) {
  const { data } = useCampaign(campaignId)
  const { mutate: deleteCampaign } = useDeleteCampaign()
  const { mutate: publishCampaign } = usePublishCampaign()

  return (
    <CampaignActions
      canEdit={data.permissions.canEdit}
      canDelete={data.permissions.canDelete}
      canPublish={data.permissions.canPublish}
      onDelete={() => deleteCampaign({ id: campaignId })}
      onPublish={() => publishCampaign({ id: campaignId })}
    />
  )
}
```

---

## Multiple useState for Server Data

### Before (Wrong)
```tsx
function Dashboard() {
  // ❌ Managing server state manually
  const [campaigns, setCampaigns] = useState([])
  const [metrics, setMetrics] = useState(null)
  const [isCampaignsLoading, setIsCampaignsLoading] = useState(true)
  const [isMetricsLoading, setIsMetricsLoading] = useState(true)
  const [campaignsError, setCampaignsError] = useState(null)
  const [metricsError, setMetricsError] = useState(null)

  useEffect(() => { /* fetch campaigns */ }, [])
  useEffect(() => { /* fetch metrics */ }, [])
  // ...
}
```

### After (Correct)
```tsx
// Each query hook is independent, parallel by default
function DashboardContainer() {
  const campaigns = useCampaigns(filters)
  const metrics = useDashboardMetrics(dateRange)

  // Both fetch in parallel automatically
  const isPending = campaigns.isPending || metrics.isPending
  if (isPending) return <DashboardSkeleton />

  return (
    <Dashboard
      campaigns={campaigns.data}
      metrics={metrics.data}
    />
  )
}
```
