# GitHub Profile Metrics Dashboard - Template Repository Specification

## Overview

Build a **GitHub Template Repository** that allows users to easily set up automated GitHub metrics tracking and visualization for their profile. Users should be able to click "Use this template", configure one secret, enable GitHub Pages, and have a fully functional metrics dashboard.

**Repository Name Suggestion:** `github-profile-metrics`

---

## Goals

1. Zero-friction setup (Use this template → Configure secret → Done)
2. Works for ANY GitHub user without code changes
3. Self-configuring (automatically detects the repository owner)
4. Clean, professional dashboard visualization
5. Daily automated updates via GitHub Actions

---

## Repository Structure

```
github-profile-metrics/
├── .github/
│   └── workflows/
│       └── update-metrics.yml      # Main automation workflow
├── docs/
│   ├── index.html                  # Dashboard visualization (GitHub Pages)
│   └── metrics-history.json        # Starter file with empty metrics array
├── README.md                       # Template setup instructions
├── PROFILE_README_TEMPLATE.md      # Sample README users can copy to their profile
└── LICENSE                         # MIT License
```

---

## File Specifications

### 1. `.github/workflows/update-metrics.yml`

**Purpose:** Automated daily metrics collection via GitHub Actions

**Key Requirements:**
- Must auto-detect username from `github.repository_owner` (NO hardcoded usernames)
- Runs daily at midnight UTC + manual trigger support
- Collects: repositories, stars, forks, watchers, followers, 14-day views
- Updates both README.md and metrics-history.json
- Rolling 365-day data retention
- Proper rate limiting and error handling

```yaml
name: Update GitHub Metrics

on:
  schedule:
    - cron: "0 0 * * *"    # Daily at midnight UTC
  workflow_dispatch:       # Manual trigger for testing

jobs:
  update-metrics:
    runs-on: ubuntu-latest

    permissions:
      contents: write

    timeout-minutes: 15

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Verify dependencies
        run: |
          echo "Checking gh CLI..."
          gh --version
          echo "Checking jq..."
          jq --version

      - name: Collect metrics via GraphQL
        env:
          GH_TOKEN: ${{ secrets.METRICS_TOKEN }}
          USERNAME: ${{ github.repository_owner }}
        run: |
          set -e

          echo "Fetching metrics for $USERNAME..."

          # GraphQL query for stars, forks, watchers, repo count, followers
          QUERY='
          query($username: String!, $cursor: String) {
            user(login: $username) {
              followers {
                totalCount
              }
              repositories(first: 100, after: $cursor, ownerAffiliations: OWNER, isFork: false) {
                totalCount
                pageInfo {
                  hasNextPage
                  endCursor
                }
                nodes {
                  name
                  stargazerCount
                  forkCount
                  watchers {
                    totalCount
                  }
                }
              }
            }
          }'

          total_stars=0
          total_forks=0
          total_watchers=0
          total_repos=0
          total_followers=0
          cursor="null"
          page=1
          empty_pages=0
          max_pages=50  # Safety limit

          while true; do
            # Safety check: prevent infinite loops
            if [ "$page" -gt "$max_pages" ]; then
              echo "Warning: Reached max page limit ($max_pages). Breaking to prevent infinite loop."
              break
            fi
            echo "Fetching page $page..."

            if [ "$cursor" = "null" ]; then
              full_response=$(gh api graphql \
                -f query="$QUERY" \
                -f username="$USERNAME" \
                --jq '.data.user' 2>/dev/null || echo "{}")
              response=$(echo "$full_response" | jq '.repositories // {}')
              # Get followers only on first page
              total_followers=$(echo "$full_response" | jq '.followers.totalCount // 0')
            else
              response=$(gh api graphql \
                -f query="$QUERY" \
                -f username="$USERNAME" \
                -f cursor="$cursor" \
                --jq '.data.user.repositories' 2>/dev/null || echo "{}")
            fi

            # Parse response
            has_next=$(echo "$response" | jq -r '.pageInfo.hasNextPage // false')
            cursor=$(echo "$response" | jq -r '.pageInfo.endCursor // "null"')

            # Set total_repos only once (same value on all pages)
            if [ "$page" -eq 1 ]; then
              total_repos=$(echo "$response" | jq '.totalCount // 0')
            fi

            # Sum metrics from this page with null/empty safety
            page_stars=$(echo "$response" | jq '[.nodes[]?.stargazerCount // 0] | add // 0')
            page_forks=$(echo "$response" | jq '[.nodes[]?.forkCount // 0] | add // 0')
            page_watchers=$(echo "$response" | jq '[.nodes[]?.watchers?.totalCount // 0] | add // 0')

            # Ensure values are numeric (default to 0 if empty or null)
            page_stars=${page_stars:-0}
            page_forks=${page_forks:-0}
            page_watchers=${page_watchers:-0}
            [[ "$page_stars" =~ ^[0-9]+$ ]] || page_stars=0
            [[ "$page_forks" =~ ^[0-9]+$ ]] || page_forks=0
            [[ "$page_watchers" =~ ^[0-9]+$ ]] || page_watchers=0

            total_stars=$((total_stars + page_stars))
            total_forks=$((total_forks + page_forks))
            total_watchers=$((total_watchers + page_watchers))

            echo "  Page $page: +$page_stars stars, +$page_forks forks, +$page_watchers watchers"

            # Track consecutive empty pages to detect potential infinite loops
            if [ "$page_stars" -eq 0 ] && [ "$page_forks" -eq 0 ] && [ "$page_watchers" -eq 0 ]; then
              empty_pages=$((empty_pages + 1))
              if [ "$empty_pages" -ge 3 ]; then
                echo "Warning: $empty_pages consecutive empty pages. Breaking to prevent potential infinite loop."
                break
              fi
            else
              empty_pages=0
            fi

            # Break if no more pages or invalid response
            if [ "$has_next" != "true" ]; then
              break
            fi

            # Also break if cursor is empty or null
            if [ -z "$cursor" ] || [ "$cursor" = "null" ] || [ "$cursor" = "" ]; then
              echo "Warning: Invalid cursor received. Breaking pagination."
              break
            fi
            page=$((page + 1))
            sleep 1  # Rate limit protection
          done

          echo ""
          echo "Fetching traffic views..."

          # Get repo names for traffic API (REST only, with pagination)
          total_views=0
          traffic_page=1
          views_data=""

          while true; do
            repos=$(gh api "users/$USERNAME/repos?per_page=100&page=$traffic_page&type=owner" --jq '.[].name' 2>/dev/null || echo "")

            # Break if no repos returned
            [ -z "$repos" ] && break

            for repo in $repos; do
              views=$(gh api "repos/$USERNAME/$repo/traffic/views" --jq '.count // 0' 2>/dev/null || echo "0")
              # Ensure views is numeric
              [[ "$views" =~ ^[0-9]+$ ]] || views=0
              total_views=$((total_views + views))
              if [ "$views" -gt 0 ]; then
                echo "  $repo: $views views"
                views_data="${views_data}${views} ${repo}"$'\n'
              fi
              sleep 0.5  # Rate limit protection
            done

            # Check if we got less than 100 repos (last page)
            repo_count=$(echo "$repos" | wc -l)
            [ "$repo_count" -lt 100 ] && break

            traffic_page=$((traffic_page + 1))
          done

          # Sort by views (descending) and take top 10
          TOP_REPOS_COUNT=10
          echo ""
          echo "=== Top $TOP_REPOS_COUNT Repositories by Views ==="
          top_repos_table="| Repository | Views |"$'\n'
          top_repos_table+="|------------|-------|"$'\n'

          top_repos=$(echo "$views_data" | sort -t' ' -k1 -nr | head -n $TOP_REPOS_COUNT)
          while IFS= read -r line; do
            [ -z "$line" ] && continue
            repo_views=$(echo "$line" | cut -d' ' -f1)
            repo_name=$(echo "$line" | cut -d' ' -f2-)
            echo "  $repo_name: $repo_views views"
            top_repos_table+="| [$repo_name](https://github.com/$USERNAME/$repo_name) | $repo_views |"$'\n'
          done <<< "$top_repos"

          # Export top repos table (use delimiter for multiline)
          echo "TOP_REPOS_TABLE<<EOF" >> $GITHUB_ENV
          echo "$top_repos_table" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

          echo ""
          echo "=== Final Metrics ==="
          echo "Repositories: $total_repos"
          echo "Stars: $total_stars"
          echo "Forks: $total_forks"
          echo "Watchers: $total_watchers"
          echo "Followers: $total_followers"
          echo "Views (14 days): $total_views"

          # Export to environment
          echo "TOTAL_REPOS=$total_repos" >> $GITHUB_ENV
          echo "TOTAL_STARS=$total_stars" >> $GITHUB_ENV
          echo "TOTAL_FORKS=$total_forks" >> $GITHUB_ENV
          echo "TOTAL_WATCHERS=$total_watchers" >> $GITHUB_ENV
          echo "TOTAL_FOLLOWERS=$total_followers" >> $GITHUB_ENV
          echo "TOTAL_VIEWS=$total_views" >> $GITHUB_ENV
          echo "LAST_UPDATED=$(date -u '+%Y-%m-%d %H:%M UTC')" >> $GITHUB_ENV
          echo "USERNAME=$USERNAME" >> $GITHUB_ENV

      - name: Update README
        run: |
          sed -i "s|<!--TOTAL_REPOS-->.*<!--/TOTAL_REPOS-->|<!--TOTAL_REPOS-->${TOTAL_REPOS}<!--/TOTAL_REPOS-->|" README.md
          sed -i "s|<!--TOTAL_STARS-->.*<!--/TOTAL_STARS-->|<!--TOTAL_STARS-->${TOTAL_STARS}<!--/TOTAL_STARS-->|" README.md
          sed -i "s|<!--TOTAL_FORKS-->.*<!--/TOTAL_FORKS-->|<!--TOTAL_FORKS-->${TOTAL_FORKS}<!--/TOTAL_FORKS-->|" README.md
          sed -i "s|<!--TOTAL_WATCHERS-->.*<!--/TOTAL_WATCHERS-->|<!--TOTAL_WATCHERS-->${TOTAL_WATCHERS}<!--/TOTAL_WATCHERS-->|" README.md
          sed -i "s|<!--TOTAL_FOLLOWERS-->.*<!--/TOTAL_FOLLOWERS-->|<!--TOTAL_FOLLOWERS-->${TOTAL_FOLLOWERS}<!--/TOTAL_FOLLOWERS-->|" README.md
          sed -i "s|<!--TOTAL_VIEWS-->.*<!--/TOTAL_VIEWS-->|<!--TOTAL_VIEWS-->${TOTAL_VIEWS}<!--/TOTAL_VIEWS-->|" README.md
          sed -i "s|<!--LAST_UPDATED-->.*<!--/LAST_UPDATED-->|<!--LAST_UPDATED-->${LAST_UPDATED}<!--/LAST_UPDATED-->|" README.md

          # Replace top repos table (multiline)
          awk -v table="$TOP_REPOS_TABLE" '
            /<!--TOP_REPOS_START-->/ { print; print table; skip=1; next }
            /<!--TOP_REPOS_END-->/ { skip=0 }
            !skip { print }
          ' README.md > README.tmp && mv README.tmp README.md

      - name: Update metrics history (rolling 365 days)
        run: |
          python3 << 'PYTHON_EOF'
          import json
          import os
          from datetime import datetime, timedelta

          # Get today's date
          today = datetime.now().strftime("%Y-%m-%d")

          # Metrics history file path (in docs/ for GitHub Pages)
          metrics_file = 'docs/metrics-history.json'

          # Load existing metrics history
          try:
              with open(metrics_file, 'r') as f:
                  data = json.load(f)
          except FileNotFoundError:
              data = {"metrics": []}

          # Create today's metric entry
          today_metric = {
              "date": today,
              "repositories": int(os.environ.get('TOTAL_REPOS', 0)),
              "stars": int(os.environ.get('TOTAL_STARS', 0)),
              "forks": int(os.environ.get('TOTAL_FORKS', 0)),
              "watchers": int(os.environ.get('TOTAL_WATCHERS', 0)),
              "followers": int(os.environ.get('TOTAL_FOLLOWERS', 0)),
              "views_14d": int(os.environ.get('TOTAL_VIEWS', 0))
          }

          # Remove today's entry if it exists (to avoid duplicates)
          data["metrics"] = [m for m in data["metrics"] if m["date"] != today]

          # Add today's data
          data["metrics"].append(today_metric)

          # Keep only last 365 days
          cutoff_date = (datetime.now() - timedelta(days=365)).strftime("%Y-%m-%d")
          data["metrics"] = [m for m in data["metrics"] if m["date"] >= cutoff_date]

          # Save updated metrics history
          with open(metrics_file, 'w') as f:
              json.dump(data, f, indent=2)

          print(f"Updated {metrics_file} with {len(data['metrics'])} days of data")
          PYTHON_EOF
        env:
          TOTAL_REPOS: ${{ env.TOTAL_REPOS }}
          TOTAL_STARS: ${{ env.TOTAL_STARS }}
          TOTAL_FORKS: ${{ env.TOTAL_FORKS }}
          TOTAL_WATCHERS: ${{ env.TOTAL_WATCHERS }}
          TOTAL_FOLLOWERS: ${{ env.TOTAL_FOLLOWERS }}
          TOTAL_VIEWS: ${{ env.TOTAL_VIEWS }}

      - name: Update dashboard username
        run: |
          # Replace placeholder username in dashboard HTML
          sed -i "s|{{USERNAME}}|${USERNAME}|g" docs/index.html
          sed -i "s|@foyzulkarim|@${USERNAME}|g" docs/index.html

      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Check if there are changes
          if ! git diff --quiet README.md docs/metrics-history.json docs/index.html 2>/dev/null; then
            git add README.md docs/metrics-history.json docs/index.html
            git commit -m "chore: update GitHub metrics [skip ci]"
            # Pull with rebase to handle any concurrent updates, then push
            git pull --rebase origin ${{ github.ref_name }} || true
            git push
            echo "Metrics updated successfully!"
          else
            echo "No changes to commit."
          fi
```

---

### 2. `docs/index.html`

**Purpose:** Interactive dashboard with Chart.js visualizations

**Key Requirements:**
- Uses `{{USERNAME}}` placeholder that gets replaced on first run
- Responsive design (mobile + desktop)
- 6 charts: Repositories, Stars, Forks, Followers, Watchers, Views
- Summary cards with daily change indicators
- Loads data from `metrics-history.json`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GitHub Metrics Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
            background: #fff;
            color: #333;
            padding: 20px;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
        }

        h1 {
            text-align: center;
            margin-bottom: 10px;
            font-size: 28px;
        }

        .subtitle {
            text-align: center;
            color: #666;
            margin-bottom: 30px;
            font-size: 14px;
        }

        .error-message {
            background: #fee;
            border: 1px solid #f99;
            color: #c33;
            padding: 15px;
            border-radius: 4px;
            margin-bottom: 20px;
            display: none;
        }

        .loading {
            text-align: center;
            padding: 40px;
            font-size: 16px;
            color: #666;
        }

        .charts-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(500px, 1fr));
            gap: 30px;
            margin-bottom: 40px;
        }

        .chart-container {
            background: #f9f9f9;
            border: 1px solid #eee;
            border-radius: 8px;
            padding: 20px;
            position: relative;
            height: 400px;
        }

        .chart-title {
            font-weight: 600;
            font-size: 16px;
            margin-bottom: 15px;
            color: #333;
        }

        canvas {
            max-width: 100%;
        }

        .stats-summary {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
            gap: 15px;
            margin-bottom: 30px;
        }

        .stat-card {
            background: #f0f4f8;
            padding: 15px;
            border-radius: 6px;
            text-align: center;
        }

        .stat-label {
            font-size: 12px;
            color: #666;
            text-transform: uppercase;
            letter-spacing: 0.5px;
            margin-bottom: 5px;
        }

        .stat-value {
            font-size: 24px;
            font-weight: bold;
            color: #333;
        }

        .stat-change {
            font-size: 12px;
            color: #666;
            margin-top: 5px;
        }

        .stat-change.positive {
            color: #28a745;
        }

        .stat-change.negative {
            color: #dc3545;
        }

        .footer {
            text-align: center;
            color: #999;
            font-size: 12px;
            margin-top: 40px;
            padding-top: 20px;
            border-top: 1px solid #eee;
        }

        @media (max-width: 768px) {
            .charts-grid {
                grid-template-columns: 1fr;
            }

            h1 {
                font-size: 22px;
            }

            .chart-container {
                height: 300px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📊 GitHub Metrics Dashboard</h1>
        <p class="subtitle">Historical tracking of GitHub statistics for @{{USERNAME}}</p>

        <div class="error-message" id="errorMessage"></div>
        <div class="loading" id="loadingMessage">Loading metrics data...</div>

        <div id="dashboard" style="display: none;">
            <div class="stats-summary" id="statsSummary"></div>

            <div class="charts-grid">
                <div class="chart-container">
                    <div class="chart-title">Repositories</div>
                    <canvas id="reposChart"></canvas>
                </div>

                <div class="chart-container">
                    <div class="chart-title">Stars</div>
                    <canvas id="starsChart"></canvas>
                </div>

                <div class="chart-container">
                    <div class="chart-title">Forks</div>
                    <canvas id="forksChart"></canvas>
                </div>

                <div class="chart-container">
                    <div class="chart-title">Followers</div>
                    <canvas id="followersChart"></canvas>
                </div>

                <div class="chart-container">
                    <div class="chart-title">Watchers</div>
                    <canvas id="watchersChart"></canvas>
                </div>

                <div class="chart-container">
                    <div class="chart-title">Views (14 days)</div>
                    <canvas id="viewsChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <div class="footer" id="footer"></div>

    <script>
        const chartConfigs = {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: {
                    display: false
                }
            },
            scales: {
                y: {
                    beginAtZero: false,
                    ticks: {
                        color: '#666'
                    },
                    grid: {
                        color: '#eee'
                    }
                },
                x: {
                    ticks: {
                        color: '#666'
                    },
                    grid: {
                        color: '#eee'
                    }
                }
            }
        };

        const colors = {
            repos: '#3b82f6',
            stars: '#f59e0b',
            forks: '#10b981',
            followers: '#8b5cf6',
            watchers: '#ec4899',
            views: '#06b6d4'
        };

        let charts = {};

        async function loadMetrics() {
            try {
                const response = await fetch('metrics-history.json');
                if (!response.ok) throw new Error('Failed to load metrics data');

                const data = await response.json();
                const metrics = data.metrics || [];

                if (metrics.length === 0) {
                    throw new Error('No metrics data available yet. Run the workflow manually or wait for the scheduled run.');
                }

                const dates = metrics.map(m => m.date);
                const repos = metrics.map(m => m.repositories);
                const stars = metrics.map(m => m.stars);
                const forks = metrics.map(m => m.forks);
                const followers = metrics.map(m => m.followers);
                const watchers = metrics.map(m => m.watchers);
                const views = metrics.map(m => m.views_14d);

                renderCharts(dates, repos, stars, forks, followers, watchers, views);
                renderStats(metrics);
                document.getElementById('loadingMessage').style.display = 'none';
                document.getElementById('dashboard').style.display = 'block';

            } catch (error) {
                console.error('Error loading metrics:', error);
                document.getElementById('loadingMessage').style.display = 'none';
                document.getElementById('errorMessage').style.display = 'block';
                document.getElementById('errorMessage').innerHTML = `Error: ${error.message}`;
            }
        }

        function renderCharts(dates, repos, stars, forks, followers, watchers, views) {
            const chartData = {
                repos: { canvas: 'reposChart', data: repos, color: colors.repos, label: 'Repositories' },
                stars: { canvas: 'starsChart', data: stars, color: colors.stars, label: 'Stars' },
                forks: { canvas: 'forksChart', data: forks, color: colors.forks, label: 'Forks' },
                followers: { canvas: 'followersChart', data: followers, color: colors.followers, label: 'Followers' },
                watchers: { canvas: 'watchersChart', data: watchers, color: colors.watchers, label: 'Watchers' },
                views: { canvas: 'viewsChart', data: views, color: colors.views, label: 'Views' }
            };

            Object.entries(chartData).forEach(([key, config]) => {
                const ctx = document.getElementById(config.canvas).getContext('2d');
                charts[key] = new Chart(ctx, {
                    type: 'line',
                    data: {
                        labels: dates,
                        datasets: [{
                            label: config.label,
                            data: config.data,
                            borderColor: config.color,
                            backgroundColor: config.color + '15',
                            borderWidth: 2,
                            fill: true,
                            tension: 0.4,
                            pointRadius: 4,
                            pointBackgroundColor: config.color,
                            pointBorderColor: '#fff',
                            pointBorderWidth: 2,
                            pointHoverRadius: 6
                        }]
                    },
                    options: chartConfigs
                });
            });
        }

        function renderStats(metrics) {
            if (metrics.length < 2) {
                document.getElementById('statsSummary').innerHTML = '<p style="grid-column: 1/-1; text-align: center; color: #999;">Not enough data to show changes yet. Check back after a few days!</p>';
                return;
            }

            const latest = metrics[metrics.length - 1];
            const previous = metrics[metrics.length - 2];

            const stats = [
                { label: 'Repositories', value: latest.repositories, key: 'repositories' },
                { label: 'Stars', value: latest.stars, key: 'stars' },
                { label: 'Forks', value: latest.forks, key: 'forks' },
                { label: 'Followers', value: latest.followers, key: 'followers' },
                { label: 'Watchers', value: latest.watchers, key: 'watchers' }
            ];

            let html = '';
            stats.forEach(stat => {
                const change = latest[stat.key] - previous[stat.key];
                const changeClass = change >= 0 ? 'positive' : 'negative';
                const changeText = change >= 0 ? `+${change}` : `${change}`;

                html += `
                    <div class="stat-card">
                        <div class="stat-label">${stat.label}</div>
                        <div class="stat-value">${stat.value.toLocaleString()}</div>
                        <div class="stat-change ${changeClass}">${changeText} since yesterday</div>
                    </div>
                `;
            });

            document.getElementById('statsSummary').innerHTML = html;
        }

        function formatDate() {
            const now = new Date();
            return now.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric', hour: '2-digit', minute: '2-digit', timeZone: 'UTC' });
        }

        document.getElementById('footer').innerHTML = `Last updated: ${formatDate()} UTC`;

        // Load metrics on page load
        loadMetrics();
    </script>
</body>
</html>
```

---

### 3. `docs/metrics-history.json`

**Purpose:** Starter file for metrics storage

```json
{
  "metrics": []
}
```

---

### 4. `README.md`

**Purpose:** Template README that serves as BOTH setup instructions AND the user's profile README after configuration

**Design Decision:** The README should work in two modes:
1. Before first run: Shows setup instructions
2. After first run: Shows actual metrics (placeholders get replaced)

```markdown
# GitHub Profile Metrics

Automated GitHub metrics tracking with a beautiful dashboard.

## Quick Setup

1. **Use this template** → Click the green "Use this template" button above
2. **Name your repository** → Must be `<your-username>/<your-username>` for profile README, or any name for standalone metrics
3. **Create a Personal Access Token:**
   - Go to [GitHub Settings → Developer Settings → Personal Access Tokens → Tokens (classic)](https://github.com/settings/tokens)
   - Click "Generate new token (classic)"
   - Give it a name like "Metrics Token"
   - Select scopes: `repo` (full), `read:user`
   - Generate and copy the token
4. **Add the secret:**
   - Go to your new repository → Settings → Secrets and variables → Actions
   - Click "New repository secret"
   - Name: `METRICS_TOKEN`
   - Value: paste your token
5. **Enable GitHub Pages:**
   - Go to Settings → Pages
   - Source: "Deploy from a branch"
   - Branch: `main`, folder: `/docs`
   - Save
6. **Run the workflow:**
   - Go to Actions → "Update GitHub Metrics"
   - Click "Run workflow"
   - Wait ~2 minutes for completion

Your dashboard will be live at: `https://<your-username>.github.io/<repo-name>/`

---

## My GitHub Stats

| Metric | Count |
|--------|-------|
| Repositories | <!--TOTAL_REPOS-->-<!--/TOTAL_REPOS--> |
| Stars | <!--TOTAL_STARS-->-<!--/TOTAL_STARS--> |
| Forks | <!--TOTAL_FORKS-->-<!--/TOTAL_FORKS--> |
| Watchers | <!--TOTAL_WATCHERS-->-<!--/TOTAL_WATCHERS--> |
| Followers | <!--TOTAL_FOLLOWERS-->-<!--/TOTAL_FOLLOWERS--> |
| Views (14 days) | <!--TOTAL_VIEWS-->-<!--/TOTAL_VIEWS--> |

<sub>Last updated: <!--LAST_UPDATED-->Never (run workflow to populate)<!--/LAST_UPDATED--></sub>

---

### Top Repositories (by views, last 14 days)

<!--TOP_REPOS_START-->
*Run the workflow to see your top repositories*
<!--TOP_REPOS_END-->

---

## Features

- 📊 **Interactive Dashboard** - Beautiful Chart.js visualizations
- 🔄 **Daily Updates** - Automatic metrics collection via GitHub Actions
- 📈 **Historical Tracking** - 365 days of rolling data
- 📱 **Responsive Design** - Works on desktop and mobile
- 🔒 **Private Data Safe** - Only collects public metrics
- ⚡ **Zero Maintenance** - Set it and forget it

## How It Works

1. GitHub Actions runs daily at midnight UTC
2. Collects metrics via GitHub's GraphQL and REST APIs
3. Updates this README with latest numbers
4. Appends data to `metrics-history.json`
5. Dashboard reads JSON and renders charts

## Customization

### Change Update Frequency

Edit `.github/workflows/update-metrics.yml`:

```yaml
on:
  schedule:
    - cron: "0 */6 * * *"  # Every 6 hours instead of daily
```

### Modify Dashboard Colors

Edit `docs/index.html` and change the `colors` object:

```javascript
const colors = {
    repos: '#3b82f6',     // Blue
    stars: '#f59e0b',     // Amber
    forks: '#10b981',     // Green
    followers: '#8b5cf6', // Purple
    watchers: '#ec4899',  // Pink
    views: '#06b6d4'      // Cyan
};
```

## Token Permissions Explained

| Scope | Why Needed |
|-------|------------|
| `repo` | Access traffic data (views) for your repositories |
| `read:user` | Read your profile information (followers) |

## Troubleshooting

**Workflow fails with 401 error:**
- Your `METRICS_TOKEN` secret may be expired or invalid
- Generate a new token and update the secret

**Dashboard shows "No data":**
- Wait for the first workflow run to complete
- Check Actions tab for any errors

**Views always show 0:**
- Traffic data requires `repo` scope on your token
- Traffic data is only available for repos you own

## Credits

Built with:
- [GitHub Actions](https://github.com/features/actions)
- [Chart.js](https://www.chartjs.org/)
- [GitHub GraphQL API](https://docs.github.com/graphql)

## License

MIT License - feel free to use and modify!
```

---

### 5. `PROFILE_README_TEMPLATE.md`

**Purpose:** A cleaner template users can copy if they want a minimal profile README

```markdown
# Hi there 👋

<!-- Add your personal intro here -->

## GitHub Stats

| Metric | Count |
|--------|-------|
| Repositories | <!--TOTAL_REPOS-->-<!--/TOTAL_REPOS--> |
| Stars | <!--TOTAL_STARS-->-<!--/TOTAL_STARS--> |
| Forks | <!--TOTAL_FORKS-->-<!--/TOTAL_FORKS--> |
| Watchers | <!--TOTAL_WATCHERS-->-<!--/TOTAL_WATCHERS--> |
| Followers | <!--TOTAL_FOLLOWERS-->-<!--/TOTAL_FOLLOWERS--> |
| Views (14 days) | <!--TOTAL_VIEWS-->-<!--/TOTAL_VIEWS--> |

<sub>Last updated: <!--LAST_UPDATED-->-<!--/LAST_UPDATED--></sub>

### 📈 [View Full Metrics Dashboard →](https://<YOUR_USERNAME>.github.io/<YOUR_REPO>/)

### Top Repositories

<!--TOP_REPOS_START-->
<!--TOP_REPOS_END-->

---

<!-- Add more sections: About me, Tech stack, Contact, etc. -->
```

---

### 6. `LICENSE`

```
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## Repository Settings to Configure

After creating the repository:

1. **Mark as Template Repository:**
   - Settings → General → Check "Template repository"

2. **Set Repository Description:**
   ```
   📊 GitHub profile metrics dashboard with automated daily tracking. Use this template to track your stars, followers, forks, and more!
   ```

3. **Add Topics:**
   ```
   github-profile, github-metrics, github-actions, dashboard, chart-js, github-stats, profile-readme
   ```

4. **Enable GitHub Pages:**
   - Settings → Pages → Source: Deploy from branch → `main` → `/docs`

---

## Testing Checklist

Before publishing, verify:

- [ ] Workflow runs successfully with a test GitHub account
- [ ] Dashboard loads and displays placeholder state correctly
- [ ] After first run, README placeholders are replaced
- [ ] Dashboard shows charts with data
- [ ] Mobile responsive design works
- [ ] "Use this template" button appears (after marking as template)

---

## User Journey

1. User discovers the template (via GitHub Explore, search, or sharing)
2. Clicks "Use this template"
3. Names repo `username/username` (or any name)
4. Creates PAT with required scopes
5. Adds `METRICS_TOKEN` secret
6. Enables GitHub Pages
7. Triggers workflow manually (or waits for midnight UTC)
8. Dashboard is live, README shows real metrics
9. Daily updates happen automatically

---

## Original Implementation Reference

This template is based on the metrics system from:
- Repository: `foyzulkarim/foyzulkarim`
- Live Dashboard: https://foyzulkarim.github.io/foyzulkarim/

---

## Notes for the AI Agent

1. **Do NOT hardcode any usernames** - Always use `${{ github.repository_owner }}`
2. **Test the workflow** - Ensure it handles accounts with 0 repos gracefully
3. **Keep the HTML self-contained** - No external CSS files, only Chart.js CDN
4. **Preserve HTML comment markers** - They're used for sed replacements
5. **The `{{USERNAME}}` placeholder** in index.html gets replaced on first workflow run
