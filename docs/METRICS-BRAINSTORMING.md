# GitHub Metrics Expansion - Brainstorming Proposal

> **Date**: January 6, 2026
> **Status**: Draft for Review
> **Current Branch**: `claude/understand-cicd-metrics-40zFa`

---

## Executive Summary

The current metrics system elegantly tracks repository-level statistics (repos, stars, forks, watchers, followers, views) using GitHub APIs and displays them via GitHub Pages with no external database required. This proposal explores additional metrics that could leverage the same architecture.

---

## Current System Architecture

```
GitHub Actions (Daily Cron)
         │
         ├─► GitHub GraphQL API ─► Aggregate Metrics
         └─► GitHub REST API    ─► Traffic Data
                                        │
                                        ▼
                              docs/metrics-history.json
                                        │
                                        ▼
                              GitHub Pages (Chart.js)
```

**Key Strengths:**
- Zero infrastructure cost (uses GitHub's free tiers)
- Automatic daily updates via Actions
- Rolling 365-day history prevents unbounded growth
- Client-side rendering = no backend needed

---

## Proposed Additional Metrics

### Category 1: Contribution Activity

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Total Commits (Year)** | Commits made across all repos in the past year | GraphQL `contributionsCollection` | Low |
| **Commit Streak** | Current and longest consecutive days with commits | GraphQL `contributionCalendar` | Medium |
| **Contribution Heatmap Data** | Daily contribution counts for calendar visualization | GraphQL `contributionCalendar` | Medium |
| **Pull Requests Opened** | PRs created across all repos | GraphQL `pullRequestContributions` | Low |
| **Pull Requests Merged** | Successfully merged PRs | GraphQL with filters | Low |
| **Issues Opened** | Issues created across repos | GraphQL `issueContributions` | Low |
| **Code Reviews** | Review comments and approvals given | GraphQL `pullRequestReviewContributions` | Medium |

**Visualization Ideas:**
- Line chart: Weekly commit activity over time
- Heatmap calendar: Daily contribution intensity (like GitHub profile)
- Stacked bar: PRs opened vs merged ratio

---

### Category 2: Language & Technology Distribution

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Primary Languages** | Top languages by bytes of code | GraphQL `repository.languages` | Low |
| **Language Trends** | How language usage changes over time | Daily snapshots | Medium |
| **Technology Stack** | Detected frameworks from repo topics/files | REST API topics | Medium |

**Visualization Ideas:**
- Doughnut/Pie chart: Language distribution
- Stacked area chart: Language trends over time
- Tag cloud: Technology keywords

---

### Category 3: Repository Health & Activity

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Open Issues Count** | Total open issues across repos | GraphQL aggregation | Low |
| **Closed Issues (30d)** | Issues closed in last 30 days | GraphQL with date filter | Medium |
| **Issue Response Time** | Average time to first response | REST API with processing | High |
| **PR Merge Rate** | Percentage of PRs that get merged | GraphQL stats | Medium |
| **Code Frequency** | Lines added/deleted per week | REST `/stats/code_frequency` | Medium |
| **Release Activity** | Number of releases published | GraphQL `releases` | Low |

**Visualization Ideas:**
- Gauge: Issue resolution rate
- Line chart: Open vs closed issues over time
- Bar chart: Monthly release count

---

### Category 4: Community & Engagement

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Unique Contributors** | Number of people who contributed | GraphQL `contributors` | Medium |
| **Sponsors Count** | GitHub Sponsors supporters | GraphQL `sponsorsListing` | Low |
| **Sponsorship Tiers** | Breakdown by tier | GraphQL | Medium |
| **Discussion Activity** | Posts in GitHub Discussions | GraphQL `discussions` | Medium |
| **Gists Count** | Public gists created | REST API `/users/{user}/gists` | Low |
| **Organizations** | Organization memberships | GraphQL `organizations` | Low |

**Visualization Ideas:**
- Counter cards: Sponsors, Orgs, Gists
- Line chart: Sponsor count over time
- Network graph: Contributor connections

---

### Category 5: Growth & Trends

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Star Velocity** | Stars gained per day/week | Calculated from history | Low |
| **Follower Growth Rate** | Net followers gained per period | Calculated from history | Low |
| **Fork-to-Star Ratio** | Indicates code reuse vs appreciation | Calculated | Low |
| **Repo Creation Timeline** | When repos were created | GraphQL `createdAt` | Low |
| **Most Starred This Month** | Top gaining repos | Calculated from snapshots | Medium |

**Visualization Ideas:**
- Sparklines: Quick trend indicators
- Leaderboard: Top growing repos
- Trend arrows with percentages

---

### Category 6: Content & Documentation

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **README Quality Score** | Presence of badges, sections, etc. | File content analysis | Medium |
| **License Distribution** | Types of licenses used | GraphQL `licenseInfo` | Low |
| **Topics Coverage** | Repos with/without topics | GraphQL `repositoryTopics` | Low |
| **Wiki Usage** | Repos with wikis enabled | GraphQL `hasWikiEnabled` | Low |

**Visualization Ideas:**
- Pie chart: License types
- Progress bar: Documentation coverage
- Word cloud: Common topics

---

### Category 7: Security & Maintenance

| Metric | Description | API Source | Complexity |
|--------|-------------|------------|------------|
| **Dependabot Alerts** | Open security vulnerabilities | GraphQL `vulnerabilityAlerts` | Medium |
| **Last Commit Age** | Average/median days since last commit | GraphQL `pushedAt` | Low |
| **Archived Repos** | Count of archived repositories | GraphQL `isArchived` | Low |
| **Branch Protection** | Repos with protected branches | GraphQL `branchProtectionRules` | Medium |

**Visualization Ideas:**
- Status indicators: Security health
- Histogram: Repo activity freshness
- Progress ring: Protected branch coverage

---

## Implementation Priority Matrix

### Quick Wins (Low effort, High value)
1. **Total Commits (Year)** - Single GraphQL query addition
2. **Primary Languages** - Already available in repo data
3. **Follower/Star Growth Rate** - Pure calculation from existing data
4. **Open Issues Count** - Simple aggregation
5. **License Distribution** - Easy query addition

### Medium Effort, High Value
1. **Contribution Heatmap** - Powerful visual, moderate API work
2. **PR Statistics** - Valuable for developers
3. **Language Trends Over Time** - Unique insight
4. **Release Activity** - Shows project momentum

### Advanced Features (Higher effort)
1. **Issue Response Time** - Requires complex data processing
2. **Contributor Network** - Graph visualization complexity
3. **README Quality Scoring** - Content analysis logic

---

## Proposed Dashboard Enhancements

### Option A: Single Page Expansion
Add more chart sections to existing `index.html`:
- Pros: Simple, consistent experience
- Cons: Could become cluttered

### Option B: Tabbed Dashboard
Create category tabs (Overview, Activity, Languages, Community):
- Pros: Organized, scalable
- Cons: More complex navigation

### Option C: Multiple Pages
Separate pages for different metric categories:
- Pros: Clean separation, deep-dive possible
- Cons: Fragmented experience

**Recommendation**: Start with Option A for 3-5 new metrics, evolve to Option B if scope grows.

---

## Data Storage Considerations

### Current Structure
```json
{
  "metrics": [
    {
      "date": "2026-01-06",
      "repositories": 125,
      "stars": 1565,
      ...
    }
  ]
}
```

### Proposed Extended Structure
```json
{
  "metrics": [...],
  "languages": {
    "2026-01-06": {
      "JavaScript": 45.2,
      "Python": 30.1,
      "TypeScript": 24.7
    }
  },
  "contributions": {
    "2026-01-06": {
      "commits": 1250,
      "prs_opened": 45,
      "prs_merged": 42,
      "issues_opened": 23
    }
  },
  "community": {
    "2026-01-06": {
      "sponsors": 5,
      "gists": 12,
      "organizations": 3
    }
  }
}
```

**Alternative**: Multiple JSON files per category for better modularity:
- `metrics-history.json` (existing)
- `languages-history.json`
- `contributions-history.json`
- `community-history.json`

---

## API Rate Limit Considerations

| API Type | Rate Limit | Current Usage | Headroom |
|----------|------------|---------------|----------|
| GraphQL | 5,000 points/hour | ~200 points | Plenty |
| REST (authenticated) | 5,000 requests/hour | ~100 requests | Plenty |
| REST (traffic) | Per-repo limits | Used sparingly | OK |

**Assessment**: Current architecture has significant headroom for expansion.

---

## Questions for Discussion

1. **Scope Priority**: Which metric categories are most interesting?
   - [ ] Contribution Activity
   - [ ] Languages
   - [ ] Repository Health
   - [ ] Community
   - [ ] Growth Trends
   - [ ] Security

2. **Time Range**: Should we track weekly aggregates for some metrics (less storage, smoother trends)?

3. **Privacy**: Any metrics that should be opt-in or excluded?

4. **Visualization Style**: Prefer more charts or more summary cards/numbers?

5. **Update Frequency**: Daily is fine for most, but some (like commits) could be real-time badge updates?

---

## Next Steps

1. **Review this proposal** - Add comments, prioritize metrics
2. **Select Phase 1 metrics** - Pick 3-5 quick wins to implement first
3. **Design data schema** - Finalize JSON structure for new metrics
4. **Update workflow** - Add new API queries
5. **Enhance dashboard** - Add new visualizations
6. **Test & iterate** - Validate data accuracy

---

## Appendix: GraphQL Query Examples

### Contribution Activity Query
```graphql
query($username: String!) {
  user(login: $username) {
    contributionsCollection {
      totalCommitContributions
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      contributionCalendar {
        totalContributions
        weeks {
          contributionDays {
            date
            contributionCount
          }
        }
      }
    }
  }
}
```

### Language Distribution Query
```graphql
query($username: String!, $cursor: String) {
  user(login: $username) {
    repositories(first: 100, after: $cursor, ownerAffiliations: OWNER) {
      nodes {
        languages(first: 10, orderBy: {field: SIZE, direction: DESC}) {
          edges {
            size
            node {
              name
              color
            }
          }
        }
      }
    }
  }
}
```

---

*This document is a living proposal. Please add comments or suggestions directly to this file or create issues for discussion.*
