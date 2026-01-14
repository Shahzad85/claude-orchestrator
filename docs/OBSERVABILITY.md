# Observability Guide

This guide covers the observability features added in v2.3 of Claude Orchestrator, including structured logging, Prometheus metrics, cost tracking, and the monitoring dashboard.

## Overview

The observability stack consists of four components:

1. **Structured Logging** - JSON-formatted logs for machine parsing
2. **Prometheus Metrics** - Quantitative metrics for monitoring
3. **Cost Tracking** - Claude API usage and cost estimation
4. **Web Dashboard** - Real-time visualization

## Quick Start

```bash
# Start the metrics server
~/.claude-orchestrator/scripts/metrics-server.sh &

# Start the dashboard (includes metrics server)
~/.claude-orchestrator/scripts/start-dashboard.sh

# View the dashboard
open http://localhost:3001
```

## Structured Logging

### Log Format

All orchestrator events are logged in JSON Lines format (one JSON object per line):

```json
{
  "timestamp": "2026-01-14T10:30:00.123Z",
  "event": "worker_state_change",
  "level": "info",
  "message": "Tab 2: WORKING -> PR_OPEN",
  "worker": {
    "tab": 2,
    "name": "auth-api",
    "branch": "feature/auth-api"
  },
  "state": {
    "from": "WORKING",
    "to": "PR_OPEN"
  }
}
```

### Log Files

| File | Description |
|------|-------------|
| `~/.claude/logs/orchestrator.jsonl` | Structured JSON logs |
| `~/.claude/orchestrator.log` | Human-readable text logs (backward compatible) |

### Event Types

| Event | Description |
|-------|-------------|
| `orchestrator_started` | Orchestrator process started |
| `orchestrator_stopped` | Orchestrator process stopped |
| `worker_state_change` | Worker transitioned between states |
| `worker_initialized` | Worker received initial instructions |
| `worker_nudged` | Orchestrator sent command to worker |
| `pr_detected` | PR detected for a worker |
| `pr_ci_status` | CI status check result |
| `review_started` | QA Guardian review initiated |
| `review_completed` | QA Guardian review finished |
| `agent_started` | Quality agent started |
| `agent_completed` | Quality agent finished |
| `pr_merged` | PR was merged |
| `worker_closed` | Worker tab closed |
| `project_status_change` | Project status updated |
| `project_workers_complete` | All workers merged |
| `notification_sent` | macOS notification sent |
| `error` | Error occurred |

### Log Levels

- `debug` - Detailed diagnostic information
- `info` - Normal operational events
- `warn` - Potential issues
- `error` - Errors requiring attention

### Configuration

Environment variables for logging:

```bash
export LOG_DIR="$HOME/.claude/logs"
export LOG_LEVEL="info"  # debug, info, warn, error
```

### Querying Logs

```bash
# View recent logs
tail -f ~/.claude/logs/orchestrator.jsonl | jq .

# Filter by event type
cat ~/.claude/logs/orchestrator.jsonl | jq 'select(.event == "worker_state_change")'

# Filter by worker
cat ~/.claude/logs/orchestrator.jsonl | jq 'select(.worker.name == "auth-api")'

# Count events by type
cat ~/.claude/logs/orchestrator.jsonl | jq -s 'group_by(.event) | map({event: .[0].event, count: length})'
```

## Prometheus Metrics

### Starting the Metrics Server

```bash
~/.claude-orchestrator/scripts/metrics-server.sh [port]
# Default port: 9090
```

### Metrics Endpoint

```bash
curl http://localhost:9090/metrics
```

### Available Metrics

#### Counters

| Metric | Description |
|--------|-------------|
| `orchestrator_projects_total` | Total projects started |
| `orchestrator_projects_completed_total` | Projects completed successfully |
| `orchestrator_projects_failed_total` | Projects that failed |
| `orchestrator_workers_spawned_total` | Workers spawned |
| `orchestrator_workers_merged_total` | Workers merged successfully |
| `orchestrator_prs_created_total` | PRs created |
| `orchestrator_prs_merged_total` | PRs merged |
| `orchestrator_reviews_total` | QA reviews run |
| `orchestrator_reviews_passed_total` | QA reviews passed |
| `orchestrator_reviews_failed_total` | QA reviews failed |
| `orchestrator_iterations_total` | Project iterations |

#### Gauges

| Metric | Description |
|--------|-------------|
| `orchestrator_active_workers` | Current active workers |
| `orchestrator_active_projects` | Current active projects |
| `orchestrator_up` | Orchestrator status (1=up, 0=down) |

#### Histograms

| Metric | Description |
|--------|-------------|
| `orchestrator_prd_generation_seconds` | Time to generate PRD |
| `orchestrator_worker_execution_seconds` | Worker execution time |
| `orchestrator_review_seconds` | QA review time |
| `orchestrator_project_total_seconds` | Total project time |

### Prometheus Integration

Add to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'claude-orchestrator'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 15s
```

### Grafana Dashboard

Import the dashboard from `dashboard/grafana.json` or create custom panels using these queries:

```promql
# Active workers over time
orchestrator_active_workers

# Projects per hour
rate(orchestrator_projects_completed_total[1h])

# Review pass rate
orchestrator_reviews_passed_total / orchestrator_reviews_total

# Average worker execution time
rate(orchestrator_worker_execution_seconds_sum[5m]) / rate(orchestrator_worker_execution_seconds_count[5m])
```

## Cost Tracking

### How It Works

Cost tracking estimates Claude API usage by:
1. Parsing token counts from Claude Code output
2. Estimating tokens from text length when counts unavailable
3. Applying model-specific pricing

### Pricing (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Claude Opus 4.5 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude 3.5 Haiku | $0.80 | $4.00 |

### Recording Usage

```bash
source ~/.claude-orchestrator/scripts/cost-tracker.sh

# Record token usage
record_usage "my-project" "worker-1" "claude-sonnet-4-20250514" 10000 5000
```

### Viewing Costs

```bash
source ~/.claude-orchestrator/scripts/cost-tracker.sh

# View project summary
get_project_summary "my-project"

# View all projects
get_all_costs

# Generate markdown report
generate_cost_report "my-project" > report.md
```

### Budget Alerts

Set a budget threshold to receive notifications when costs exceed the limit:

```bash
source ~/.claude-orchestrator/scripts/cost-tracker.sh

# Set threshold to $10
set_budget_threshold 10.00
```

When the threshold is exceeded, you'll receive a macOS notification.

### Cost Files

| File | Description |
|------|-------------|
| `~/.claude/costs/pricing.json` | Model pricing configuration |
| `~/.claude/costs/{project}.json` | Per-project cost data |

## Web Dashboard

### Starting the Dashboard

```bash
~/.claude-orchestrator/scripts/start-dashboard.sh [dashboard-port] [metrics-port]
# Defaults: 3001, 9090
```

### Dashboard Features

1. **Status Overview** - Orchestrator online/offline status
2. **Key Metrics** - Active workers, completed projects, PRs merged
3. **Worker Cards** - Real-time worker status with state badges
4. **Cost Breakdown** - Token usage and cost estimates
5. **Live Logs** - Streaming log viewer with filtering

### Dashboard URL

```
http://localhost:3001
```

### Customization

Edit `~/.claude-orchestrator/dashboard/index.html` to:
- Change colors (CSS variables)
- Add new metrics panels
- Modify polling interval (default: 5 seconds)

## Best Practices

### Log Retention

Configure log rotation to prevent disk usage issues:

```bash
# In your shell profile
export LOG_MAX_SIZE=10485760  # 10MB
export LOG_MAX_FILES=5

# Manual rotation
source ~/.claude-orchestrator/scripts/logging.sh
rotate_logs
```

### Alerting

Set up alerts based on metrics:

1. **High error rate**: `rate(orchestrator_reviews_failed_total[5m]) > 0.5`
2. **Stuck workers**: `orchestrator_active_workers > 10`
3. **Budget exceeded**: Check `~/.claude/costs/` files

### Debugging

When issues occur:

1. Check structured logs for the event timeline:
   ```bash
   cat ~/.claude/logs/orchestrator.jsonl | jq 'select(.level == "error")'
   ```

2. Check metrics for patterns:
   ```bash
   curl localhost:9090/metrics | grep -E "failed|error"
   ```

3. Review cost data for unusual API usage:
   ```bash
   source ~/.claude-orchestrator/scripts/cost-tracker.sh
   get_all_costs
   ```

## Troubleshooting

### Metrics server won't start

```bash
# Check if port is in use
lsof -i :9090

# Try a different port
~/.claude-orchestrator/scripts/metrics-server.sh 9091
```

### Logs not appearing

```bash
# Check log directory exists
ls -la ~/.claude/logs/

# Check logging is enabled
grep STRUCTURED_LOGGING ~/.claude-orchestrator/scripts/orchestrator-loop.sh
```

### Dashboard shows "Offline"

1. Ensure metrics server is running
2. Check browser console for CORS errors
3. Verify metrics endpoint: `curl localhost:9090/metrics`

### Cost tracking inaccurate

Token counts are estimates when not available from Claude Code output. For precise costs:
1. Check Claude Code output for actual token usage
2. Review the `events` array in cost files for per-call data
