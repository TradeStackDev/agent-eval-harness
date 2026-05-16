# 🤖 agent-eval-harness

> **Evaluation framework for web-browsing AI agents. Define tasks, run episodes, score results.**

[![Python](https://img.shields.io/badge/Python-3.11+-3776ab?style=flat&logo=python&logoColor=white)](https://python.org)
[![Playwright](https://img.shields.io/badge/Playwright-1.44-2EAD33?style=flat)](https://playwright.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

`agent-eval-harness` provides a structured, reproducible framework for evaluating AI agents on web-based tasks. Define what success looks like, run your agent, and get a detailed score report.

Designed to complement `webenv-sim` — the harness handles **task definition**, **episode orchestration**, and **result scoring**; the simulator handles browser session management.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Task** | A YAML-defined goal with success criteria, constraints, and a starting URL |
| **Episode** | One agent attempt at a task (bounded by max steps and timeout) |
| **Scorer** | Evaluates a completed episode against success criteria |
| **Suite** | A named collection of tasks for benchmarking |
| **Report** | Aggregated results across a suite run |

---

## Quick Start

```bash
pip install agent-eval-harness
playwright install chromium
```

```python
from agent_eval import EvalHarness, Suite

harness = EvalHarness(
    agent=your_agent,           # Must implement .act(observation) -> action
    suite=Suite.load("suites/ecommerce.yaml"),
    headless=True,
    workers=4
)

report = await harness.run()
report.print_summary()
report.save("results/ecommerce_run_001.json")
```

---

## Defining Tasks

Tasks are defined in YAML with composable success criteria:

```yaml
# tasks/ecommerce/add_item_to_cart.yaml
id: ecommerce_add_to_cart_001
name: Add Specific Item to Cart
description: >
  Navigate to the store, find "Wireless Noise-Cancelling Headphones",
  and add one unit to the shopping cart.
start_url: https://sim-store.webenv.dev/
category: ecommerce
difficulty: medium
max_steps: 12
timeout_seconds: 45

success_criteria:
  - type: url_contains
    value: /cart
    weight: 0.2
  - type: dom_element_exists
    selector: ".cart-item"
    weight: 0.3
  - type: dom_text_contains
    selector: ".cart-item .product-name"
    value: "Wireless Noise-Cancelling Headphones"
    weight: 0.5

annotations:
  optimal_steps: 5
  requires_search: true
  requires_scroll: true
```

### Available Success Criteria

| Type | Description |
|------|-------------|
| `url_contains` | Final URL includes substring |
| `url_exact` | Final URL matches exactly |
| `dom_element_exists` | CSS selector resolves to ≥1 element |
| `dom_text_contains` | Element text includes value |
| `dom_attribute_equals` | Element attribute matches value |
| `page_title_contains` | Page title includes substring |
| `network_request_made` | Specific API call was triggered |
| `form_submitted` | Form submission detected |
| `custom` | Python callable for complex checks |

---

## Scoring

Each episode returns a detailed score breakdown:

```json
{
  "task_id": "ecommerce_add_to_cart_001",
  "success": true,
  "score": 0.92,
  "criteria_scores": {
    "url_contains": 1.0,
    "dom_element_exists": 1.0,
    "dom_text_contains": 0.8
  },
  "steps_taken": 7,
  "optimal_steps": 5,
  "efficiency_ratio": 0.71,
  "time_elapsed_s": 18.4,
  "actions": [ ... ]
}
```

### Suite Report

```
══════════════════════════════════════════════════
 EVAL SUITE: ecommerce-v1  (40 tasks, 4 workers)
══════════════════════════════════════════════════
 Success Rate      ████████████░░░░  74.2%
 Avg Score         ████████████░░░░  0.78
 Avg Steps         6.4 / 12 max
 Avg Time          22.1s / 45s max

 By Category:
   navigation      ██████████████░░  88.0%
   search          █████████████░░░  81.0%
   forms           ████████████░░░░  75.0%
   checkout        ██████████░░░░░░  62.5%
   multi-step      ████████░░░░░░░░  50.0%
══════════════════════════════════════════════════
```

---

## Built-in Task Suites

| Suite | Tasks | Categories |
|-------|-------|------------|
| `ecommerce-v1` | 40 | search, cart, checkout, account |
| `forms-v1` | 25 | login, signup, contact, survey |
| `navigation-v1` | 20 | link-following, pagination, menus |
| `search-v1` | 30 | keyword search, filtering, sorting |
| `multi-step-v1` | 15 | booking flows, multi-page forms |

Total: **130 tasks** across 5 suites.

---

## Headless Replay

Every episode is recorded and can be replayed for debugging:

```python
from agent_eval import EpisodeReplayer

replayer = EpisodeReplayer.from_file("results/episode_042.jsonl")

# Replay in headed browser at 0.5x speed
await replayer.replay(headless=False, speed=0.5)

# Export as annotated video
await replayer.export_video("replays/episode_042.mp4", annotate=True)
```

---

## CI Integration

Run evals in GitHub Actions:

```yaml
# .github/workflows/eval.yml
- name: Run eval suite
  run: |
    python -m agent_eval.cli run \
      --agent my_agent:MyAgent \
      --suite suites/ecommerce.yaml \
      --workers 4 \
      --output results/ \
      --fail-below 0.70
```

---

## License

MIT © Adam
