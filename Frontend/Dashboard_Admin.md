# Admin Dashboard - Frontend Design Document

## Overview

The Admin Dashboard is the central hub for organization administrators and owners to monitor, manage, and analyze their software development quality metrics. This comprehensive dashboard provides real-time insights into code quality, team performance, repository health, and system-wide statistics.

## Target Users

- **Organization Owners**: Full system oversight and configuration
- **Administrators**: Team and project management, quality monitoring
- **Team Leads**: Team performance tracking and member management

## Page Layout

### Header Section
- **Organization Selector**: Dropdown to switch between organizations (for users with multiple org memberships)
- **User Profile Menu**: Avatar, name, settings, logout
- **Notifications Bell**: Real-time alerts and notifications counter
- **Search Bar**: Global search for repositories, users, teams, projects

### Sidebar Navigation
```
├── 📊 Dashboard (Current Page)
├── 👥 Users & Teams
│   ├── Organization Members
│   ├── Teams
│   └── Onboarding
├── 📦 Repositories
├── 🎯 Projects
├── 📈 Quality Metrics
│   ├── DQS Scores
│   ├── SQS Scores
│   └── Code Coverage
├── 🔔 Alerts
├── 📝 Reports
├── 🏃 Sprints & Releases
├── 🎯 Goals & OKRs
├── 🔧 Technical Debt
├── 🔗 Integrations
│   └── GitHub
├── 📋 Audit Logs
└── ⚙️ Settings
```

---

## Dashboard Main Content

### 1. Key Metrics Overview (Top Section)

Display 4-6 large metric cards in a grid layout:

#### Card 1: Total Repositories
```
┌─────────────────────────┐
│ 📦 Repositories         │
│                         │
│      24                 │
│   ↑ 3 this month        │
└─────────────────────────┘
```
- **Data**: Total enabled repositories
- **Trend**: New repositories added this month
- **API**: `GET /api/dashboard/stats`

#### Card 2: Total Developers
```
┌─────────────────────────┐
│ 👥 Developers           │
│                         │
│      156                │
│   ↑ 12 this month       │
└─────────────────────────┘
```
- **Data**: Total organization members
- **Trend**: New members added this month
- **API**: `GET /api/dashboard/stats`

#### Card 3: Average SQS
```
┌─────────────────────────┐
│ 📊 Avg SQS              │
│                         │
│      78.5               │
│   ↑ 2.3 from last week  │
└─────────────────────────┘
```
- **Data**: Organization-wide Software Quality Score
- **Color Coding**: 
  - Green: > 70
  - Yellow: 50-70
  - Red: < 50
- **API**: `GET /api/dashboard/stats`

#### Card 4: Code Coverage
```
┌─────────────────────────┐
│ 🎯 Avg Coverage         │
│                         │
│      82.3%              │
│   ↑ 1.2% from last week │
└─────────────────────────┘
```
- **Data**: Average test coverage across all repositories
- **API**: `GET /api/dashboard/stats`

#### Card 5: Total Commits
```
┌─────────────────────────┐
│ 💻 Total Commits        │
│                         │
│      45,234             │
│   ↑ 1,234 this week     │
└─────────────────────────┘
```
- **Data**: Total commits across all repositories
- **API**: `GET /api/dashboard/stats`

#### Card 6: Open Alerts
```
┌─────────────────────────┐
│ 🔔 Open Alerts          │
│                         │
│      12                 │
│   3 Critical            │
└─────────────────────────┘
```
- **Data**: Count of unresolved alerts
- **Breakdown**: By severity (Critical, High, Medium, Low)
- **API**: `GET /api/alerts?status=OPEN`

---

### 2. Charts Section (Middle Section)

#### Chart 1: SQS Trend (Line Chart)
```
┌─────────────────────────────────────────────────┐
│ 📈 Software Quality Score Trend(Last 30 Days)   │
│                                                 │
│  90 ┤                                    ╭──╮   │
│  80 ┤                          ╭────╮   │  │    │
│  70 ┤                    ╭────╯    ╰───╯  │     │
│  60 ┤          ╭────────╯                 │     │
│  50 ┤    ╭────╯                           │     │
│     └────────────────────────────────────────   │
│      Day 1    Day 10    Day 20    Day 30        │
│                                                 │
│  [7 Days] [30 Days] [90 Days]                   │
└─────────────────────────────────────────────────┘
```
- **X-Axis**: Date
- **Y-Axis**: Average SQS Score (0-100)
- **Interaction**: Hover to see exact values
- **Time Range Selector**: 7 days, 30 days, 90 days
- **API**: `GET /api/dashboard/sqs-trend?days=30`

#### Chart 2: Commit Activity (Bar Chart)
```
┌─────────────────────────────────────────────────┐
│ 💻 Commit Activity (Last 30 Days)               │
│                                                 │
│ 200 ┤     ██                                    │
│ 150 ┤  ██ ██    ██                              │
│ 100 ┤  ██ ██ ██ ██ ██    ██                     │
│  50 ┤  ██ ██ ██ ██ ██ ██ ██ ██                  │
│     └────────────────────────────────────────   │
│      Mon Tue Wed Thu Fri Sat Sun                │
│                                                 │
│  Legend: ■ Feature ■ Bugfix ■ Refactor ■ Other  │
└─────────────────────────────────────────────────┘
```
- **X-Axis**: Date
- **Y-Axis**: Number of commits
- **Stacked Bars**: By commit classification (Feature, Bugfix, Refactor, Test, Docs)
- **API**: `GET /api/dashboard/commit-trend?days=30`

#### Chart 3: Team Performance (Horizontal Bar Chart)
```
┌─────────────────────────────────────────────────┐
│ 👥 Team Performance (Avg DQS)                  │
│                                                 │
│  Frontend Team    ████████████████████ 85.2     │
│  Backend Team     ██████████████████ 78.5       │
│  DevOps Team      ███████████████ 72.3          │
│  Mobile Team      ████████████ 68.9             │
│  QA Team          ██████████ 65.4               │
│                                                 │
│  [View All Teams →]                             │
└─────────────────────────────────────────────────┘
```
- **X-Axis**: Average DQS Score
- **Y-Axis**: Team names
- **Color Coding**: Green (>70), Yellow (50-70), Red (<50)
- **API**: `GET /api/dashboard/top-teams?limit=5`

---

### 3. Data Tables Section (Bottom Section)

#### Table 1: Top Repositories
```
┌──────────────────────────────────────────────────────────────────────┐
│ 📦 Top Repositories by SQS                          [View All →]     │
├──────────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Repository       │ SQS      │ Coverage │ Commits  │ Last Updated    │
├──────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ frontend-app     │ 92.5 🟢  │ 88.2%    │ 1,234    │ 2 hours ago     │
│ backend-api      │ 87.3 🟢  │ 82.5%    │ 2,456    │ 5 hours ago     │
│ mobile-app       │ 78.9 🟢  │ 75.3%    │ 987      │ 1 day ago       │
│ data-pipeline    │ 72.1 🟢  │ 68.9%    │ 543      │ 3 days ago      │
│ auth-service     │ 68.5 🟡  │ 65.2%    │ 321      │ 1 week ago     │
└──────────────────┴──────────┴──────────┴──────────┴─────────────────┘
```
- **Columns**: Repository name, SQS score, Coverage %, Total commits, Last activity
- **Sorting**: Clickable column headers
- **Actions**: Click row to view repository details
- **API**: `GET /api/dashboard/top-repositories?limit=5`

#### Table 2: Repositories Needing Attention
```
┌─────────────────────────────────────────────────────────────────────┐
│ ⚠️ Repositories Needing Attention                   [View All →]    │
├──────────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Repository       │ SQS      │ Coverage │ Issues   │ Action          │
├──────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ legacy-system    │ 42.3 🔴  │ 35.2%    │ 12       │ [Review]        │
│ old-api          │ 48.9 🟡  │ 42.8%    │ 8        │ [Review]        │
│ prototype-v1     │ 51.2 🟡  │ 48.5%    │ 5        │ [Review]        │
└──────────────────┴──────────┴──────────┴──────────┴─────────────────┘
```
- **Filter**: Repositories with SQS < 60 or Coverage < 50%
- **API**: `GET /api/dashboard/bottom-repositories?limit=5`

#### Table 3: Top Developers
```
┌──────────────────────────────────────────────────────────────────────┐
│ 🏆 Top Developers by DQS                            [View All →]     │
├──────────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Developer        │ DQS      │ Commits  │ Team     │ Last Active     │
├──────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ 👤 Alice Johnson │ 94.2 🟢  │ 456      │ Frontend │ 1 hour ago      │
│ 👤 Bob Smith     │ 89.7 🟢  │ 523      │ Backend  │ 3 hours ago     │
│ 👤 Carol White   │ 85.3 🟢  │ 389      │ DevOps   │ 5 hours ago     │
│ 👤 David Brown   │ 82.1 🟢  │ 412      │ Mobile   │ 1 day ago       │
│ 👤 Eve Davis     │ 78.9 🟢  │ 345      │ QA       │ 2 days ago      │
└──────────────────┴──────────┴──────────┴──────────┴─────────────────┘
```
- **Columns**: Developer name (with avatar), DQS score, Commit count, Team, Last activity
- **API**: `GET /api/dashboard/top-developers?limit=5`

#### Table 4: Recent Activity Feed
```
┌──────────────────────────────────────────────────────────────────────┐
│ 📋 Recent Activity                                  [View All →]     │
├──────────────────────────────────────────────────────────────────────┤
│ 💻 Alice Johnson committed to frontend-app                           │
│    "feat: Add user authentication flow"                              │
│    2 hours ago • Feature                                             │
├──────────────────────────────────────────────────────────────────────┤
│ 🐛 Bob Smith committed to backend-api                                │
│    "fix: Resolve database connection timeout"                        │
│    3 hours ago • Bugfix                                              │
├──────────────────────────────────────────────────────────────────────┤
│ 🔔 New alert: High anomaly detected in data-pipeline                 │
│    Anomaly score: 0.87 • Critical                                    │
│    5 hours ago                                                       │
├──────────────────────────────────────────────────────────────────────┤
│ 🎯 Sprint "Q1 2026 Sprint 3" completed                               │
│    Frontend Team • 45 commits • 12 features                          │
│    1 day ago                                                         │
└──────────────────────────────────────────────────────────────────────┘
```
- **Types**: Commits, Alerts, Sprint completions, Goal achievements
- **Real-time**: Updates via WebSocket
- **API**: `GET /api/dashboard/recent-activity?limit=10`

---

### 4. Alerts & Notifications Panel (Right Sidebar)

```
┌─────────────────────────────────────┐
│ 🔔 Alerts & Notifications           │
│                                     │
│ ⚠️ CRITICAL                         │
│ Anomaly detected in auth-service    │
│ Score: 0.92 • 2 hours ago           │
│ [Acknowledge] [View Details]        │
│                                     │
│ ⚠️ HIGH                             │
│ Low SQS: legacy-system              │
│ Score: 42.3 • 5 hours ago           │
│ [Acknowledge] [View Details]        │
│                                     │
│ ℹ️ MEDIUM                           │
│ Coverage dropped below 70%          │
│ Repository: mobile-app              │
│ [View Report]                       │
│                                     │
│ ✅ SUCCESS                          │
│ Goal achieved: Increase DQS to 80   │
│ Frontend Team • 1 day ago           │
│                                     │
│ [View All Alerts →]                 │
└─────────────────────────────────────┘
```
- **Filter**: By severity (Critical, High, Medium, Low)
- **Actions**: Acknowledge, Resolve, View details
- **API**: `GET /api/dashboard/alerts`

---

## API Endpoints Reference

### Dashboard Statistics
```typescript
GET /api/dashboard/stats
Response: {
  totalRepositories: number;
  totalTeams: number;
  totalDevelopers: number;
  totalProjects: number;
  totalCommits: number;
  bugFixCommits: number;
  avgCoverage: number;
  avgSQS: number;
  riskyModulesCount: number;
}
```

### SQS Trend
```typescript
GET /api/dashboard/sqs-trend?days=30
Response: Array<{
  date: string; // YYYY-MM-DD
  value: number; // Average SQS score
}>
```

### Commit Trend
```typescript
GET /api/dashboard/commit-trend?days=30
Response: Array<{
  date: string; // YYYY-MM-DD
  value: number; // Commit count
}>
```

### Top Repositories
```typescript
GET /api/dashboard/top-repositories?limit=5
Response: Array<{
  id: string;
  name: string;
  fullName: string;
  sqs: number;
  coverage: number;
  commits: number;
}>
```

### Bottom Repositories
```typescript
GET /api/dashboard/bottom-repositories?limit=5
Response: Array<{
  id: string;
  name: string;
  fullName: string;
  sqs: number;
  coverage: number;
  commits: number;
}>
```

### Top Developers
```typescript
GET /api/dashboard/top-developers?limit=5
Response: Array<{
  id: string;
  name: string;
  email: string;
  avatarUrl: string | null;
  dqs: number;
  commits: number;
}>
```

### Top Teams
```typescript
GET /api/dashboard/top-teams?limit=5
Response: Array<{
  id: string;
  name: string;
  avgDqs: number;
  memberCount: number;
}>
```

### Recent Activity
```typescript
GET /api/dashboard/recent-activity?limit=10
Response: Array<{
  id: string;
  type: 'commit' | 'alert' | 'sprint' | 'goal';
  message: string;
  author: string;
  authorAvatar: string | null;
  repository: string;
  classification: string | null;
  timestamp: Date;
}>
```

### Alerts
```typescript
GET /api/dashboard/alerts
Response: Array<{
  id: string;
  type: 'alert' | 'low_score';
  severity: 'high' | 'warning';
  title: string;
  description: string;
  timestamp: Date;
}>
```

---

## UI Components & Libraries

### Recommended Tech Stack
- **Framework**: React 18+ with TypeScript
- **State Management**: Redux Toolkit or Zustand
- **UI Library**: Material-UI (MUI) or Ant Design
- **Charts**: Recharts or Chart.js
- **Data Tables**: TanStack Table (React Table v8)
- **Real-time**: Socket.io-client
- **HTTP Client**: Axios or React Query
- **Routing**: React Router v6
- **Forms**: React Hook Form + Zod validation
- **Date Handling**: date-fns or Day.js

### Component Structure
```
src/
├── pages/
│   └── Dashboard/
│       ├── Dashboard.tsx
│       ├── Dashboard.styles.ts
│       └── components/
│           ├── MetricCard.tsx
│           ├── SQSTrendChart.tsx
│           ├── CommitActivityChart.tsx
│           ├── TeamPerformanceChart.tsx
│           ├── RepositoriesTable.tsx
│           ├── DevelopersTable.tsx
│           ├── ActivityFeed.tsx
│           └── AlertsPanel.tsx
├── components/
│   ├── Layout/
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx
│   │   └── Layout.tsx
│   └── common/
│       ├── Card.tsx
│       ├── Badge.tsx
│       ├── Avatar.tsx
│       └── Tooltip.tsx
├── hooks/
│   ├── useDashboardStats.ts
│   ├── useSQSTrend.ts
│   ├── useCommitTrend.ts
│   └── useWebSocket.ts
├── services/
│   └── api/
│       └── dashboard.ts
├── types/
│   └── dashboard.types.ts
└── utils/
    ├── formatters.ts
    └── colors.ts
```

---

## Color Scheme & Design Tokens

### Score Color Coding
```typescript
const getScoreColor = (score: number) => {
  if (score >= 80) return '#10B981'; // Green - Excellent
  if (score >= 70) return '#3B82F6'; // Blue - Good
  if (score >= 50) return '#F59E0B'; // Yellow - Fair
  return '#EF4444'; // Red - Poor
};
```

### Severity Colors
```typescript
const severityColors = {
  CRITICAL: '#DC2626', // Red
  HIGH: '#F59E0B',     // Orange
  MEDIUM: '#3B82F6',   // Blue
  LOW: '#6B7280',      // Gray
};
```

### Chart Colors
```typescript
const chartColors = {
  primary: '#3B82F6',   // Blue
  secondary: '#8B5CF6', // Purple
  success: '#10B981',   // Green
  warning: '#F59E0B',   // Yellow
  danger: '#EF4444',    // Red
  info: '#06B6D4',      // Cyan
};
```

---

## Responsive Design

### Breakpoints
- **Mobile**: < 640px (Stack all cards vertically)
- **Tablet**: 640px - 1024px (2-column grid for cards)
- **Desktop**: 1024px - 1536px (3-column grid for cards)
- **Large Desktop**: > 1536px (Full layout with sidebar)

### Mobile Adaptations
- Collapse sidebar to hamburger menu
- Stack metric cards vertically
- Simplify charts (reduce data points)
- Hide right alerts panel (move to separate tab)
- Use accordion for tables

---

## Real-time Updates

### WebSocket Events
```typescript
// Connect to WebSocket
const socket = io('ws://localhost:3000', {
  auth: { token: authToken }
});

// Listen for events
socket.on('commit.processed', (data) => {
  // Update commit count and activity feed
});

socket.on('alert.created', (data) => {
  // Show new alert notification
});

socket.on('score.calculated', (data) => {
  // Update SQS/DQS displays
});
```

---

## Loading States

### Initial Load
- Show skeleton loaders for all cards and charts
- Display "Loading..." text in tables
- Disable interactive elements

### Refresh/Update
- Show subtle loading indicator in top-right corner
- Keep existing data visible while fetching updates
- Use optimistic updates for user actions

---

## Error Handling

### API Errors
```typescript
// Display error toast notification
toast.error('Failed to load dashboard data. Please try again.');

// Show error state in component
<ErrorState 
  message="Unable to load statistics"
  retry={() => refetch()}
/>
```

### Empty States
```typescript
// No data available
<EmptyState
  icon={<ChartIcon />}
  title="No data available"
  description="Start by connecting a repository"
  action={<Button>Connect Repository</Button>}
/>
```

---

## Performance Optimization

### Data Fetching
- Use React Query for caching and automatic refetching
- Implement pagination for large tables
- Lazy load charts and heavy components
- Debounce search inputs

### Rendering
- Memoize expensive calculations with `useMemo`
- Use `React.memo` for pure components
- Virtualize long lists with `react-window`
- Code-split routes with `React.lazy`

---

## Accessibility (a11y)

### Requirements
- ARIA labels for all interactive elements
- Keyboard navigation support (Tab, Enter, Escape)
- Screen reader announcements for dynamic updates
- Color contrast ratio ≥ 4.5:1 for text
- Focus indicators on all focusable elements
- Alt text for all images and icons

### Example
```tsx
<button
  aria-label="Acknowledge alert"
  onClick={handleAcknowledge}
  className="btn-primary"
>
  Acknowledge
</button>
```

---

## Testing Strategy

### Unit Tests
- Test individual components with Jest + React Testing Library
- Mock API calls with MSW (Mock Service Worker)
- Test utility functions and formatters

### Integration Tests
- Test complete user flows (e.g., viewing dashboard → clicking repository → viewing details)
- Test WebSocket connections and real-time updates

### E2E Tests
- Use Playwright or Cypress
- Test critical paths (login → dashboard → view alerts)

---

## Security Considerations

### Authentication
- Require valid JWT token for all API calls
- Redirect to login if token expired
- Implement token refresh mechanism

### Authorization
- Check user role before displaying admin-only features
- Disable actions user doesn't have permission for
- Validate permissions on backend for all mutations

### Data Protection
- Never expose sensitive data in client-side code
- Sanitize user inputs to prevent XSS
- Use HTTPS for all API calls

---

## Future Enhancements

### Phase 2 Features
- **Custom Dashboards**: Allow users to create custom dashboard layouts
- **Export Reports**: Export dashboard data as PDF or CSV
- **Scheduled Reports**: Email daily/weekly summary reports
- **Advanced Filters**: Filter by date range, team, repository, etc.
- **Comparison View**: Compare metrics across time periods
- **Drill-down Analysis**: Click any metric to see detailed breakdown
- **Dark Mode**: Toggle between light and dark themes
- **Mobile App**: Native mobile app for on-the-go monitoring

### Phase 3 Features
- **AI Insights**: ML-powered recommendations and predictions
- **Anomaly Detection**: Automatic detection of unusual patterns
- **Predictive Analytics**: Forecast future trends
- **Integration Hub**: Connect with Jira, Slack, Teams, etc.
- **Custom Alerts**: User-defined alert rules and thresholds
- **Collaboration**: Comments and discussions on metrics

---


## Support & Documentation

### Developer Resources
- **API Documentation**: `/docs/api`
- **Component Storybook**: `/storybook`
- **Design System**: `/docs/design-system`
- **Contributing Guide**: `/CONTRIBUTING.md`

### User Help
- **User Guide**: In-app help tooltips and documentation
- **Video Tutorials**: Screen recordings of common tasks
- **FAQ**: Frequently asked questions

---

## Conclusion

This admin dashboard provides a comprehensive view of software development quality metrics, enabling data-driven decision-making and continuous improvement. The design prioritizes usability, performance, and real-time insights to help teams deliver high-quality software.

**Key Success Metrics:**
- Dashboard load time < 2 seconds
- Real-time updates within 1 second
- 100% accessibility compliance
- Mobile-responsive on all devices
- 95%+ user satisfaction score

Good luck with the implementation! 🚀
