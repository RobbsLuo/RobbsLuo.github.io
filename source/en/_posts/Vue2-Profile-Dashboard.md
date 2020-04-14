---
title: Visualizing Developer Profiles in Vue 2
date: 2020-04-14 11:00:00
tags:
  - Career
  - Vue
  - Visualization
categories:
  - Career
description: The pipeline is wired up end to end. This final post covers what the Vue 2 dashboard actually shows, how it surfaces anomalies, and how it helps leads spot team risk early.
lang: en
---

The pipeline is wired up end to end. The last piece is how to draw the data in a way that's actually usable.

I'll be honest, I underestimated the visualization layer at first. The data was ready; how hard could a few charts be? After building it, I learned that "looks good" and "is useful" are very different things. The real value of a dashboard is making anomalies surface themselves.

## What the dashboard has to answer

Our dashboard has two audiences with different questions:

- **Team Leads**: Is my team at risk? Who's drifting? Which module is a single point of failure?
- **Individuals (the developers themselves)**: What does my profile look like? Where am I in the team?

So we split the dashboard into three sections:

1. **Personal profile page**: a radar chart across the four dimensions, plus trends
2. **Team view**: distributions, rankings (desensitized, not public), anomaly lists
3. **Risk monitor**: single-point dependencies, collaboration islands, sudden-change alerts

Let's go through each.

## Personal profile page: the radar chart is the centerpiece

A radar chart is the most intuitive way to render an individual profile. Each of the four dimensions (commit / code / collaboration / output) becomes a 0-100 normalized score, plotted on a four-axis radar.

```vue
<!-- ProfileChart.vue (simplified) -->
<template>
  <div ref="chart" style="width: 100%; height: 360px;"></div>
</template>

<script>
import * as echarts from 'echarts';

export default {
  props: {
    scores: Object,  // { commit: 78, code: 65, review: 92, output: 70 }
    baseline: Object, // team average, used as a reference ring
  },
  mounted() {
    const chart = echarts.init(this.$refs.chart);
    chart.setOption({
      tooltip: {},
      radar: {
        indicator: [
          { name: 'Commit', max: 100 },
          { name: 'Code', max: 100 },
          { name: 'Collaboration', max: 100 },
          { name: 'Output', max: 100 },
        ],
      },
      series: [{
        type: 'radar',
        data: [
          {
            value: Object.values(this.scores),
            name: 'Self',
            areaStyle: { color: 'rgba(64, 169, 255, 0.3)' },
          },
          {
            value: Object.values(this.baseline),
            name: 'Team average',
            lineStyle: { type: 'dashed' },
            areaStyle: { color: 'rgba(0,0,0,0)' },
          },
        ],
      }],
    });
  },
};
</script>
```

One detail I really want to hammer on: always draw a dashed "team average" line as a reference on the radar. Without it, a commit score of 78 is meaningless, high or low relative to what? With the reference ring, you can tell at a glance: "this person is very strong on collaboration but weak on output."

Next to the radar we put a 90-day trend line for each dimension. A static snapshot isn't very useful; the trend is where the signal lives.

## Team view: distribution beats ranking

In v2 we shipped a team ranking board. Within days it got torn apart. Any form of ranking pushes people to game the metrics, even if it's lead-only and never made public. The lead will leak it accidentally in 1:1s anyway.

So v3 became distribution-only:

- A **histogram + box plot** per dimension
- **Outliers** highlighted as scatter points (no names shown; click to drill in)
- **Monthly team-level trends**

```vue
<!-- TeamDistribution.vue -->
<template>
  <div class="team-dist">
    <MetricDistribution
      v-for="metric in metrics"
      :key="metric.key"
      :metric="metric.key"
      :title="metric.label"
      :data="teamData[metric.key]"
      :outliers="outliers[metric.key]"
      @click-outlier="onOutlierClick"
    />
  </div>
</template>

<script>
export default {
  data() {
    return {
      metrics: [
        { key: 'commit_score',  label: 'Commit' },
        { key: 'code_score',    label: 'Code' },
        { key: 'review_score',  label: 'Collaboration' },
        { key: 'output_score',  label: 'Output' },
      ],
    };
  },
  methods: {
    onOutlierClick({ metric, authorId }) {
      this.$router.push({
        name: 'profile',
        params: { id: authorId },
        query: { focus: metric },
      });
    },
  },
};
</script>
```

> Restraint beats feature-stuffing on a dashboard. Things the lead can drill into themselves shouldn't be listed outright; both privacy and data accuracy depend on that restraint.

## Risk monitor: where anomalies surface themselves

This is the part of the dashboard that's genuinely valuable. Every previous step in the pipeline exists so this one page can auto-surface risks.

We built three key anomaly detectors.

### 1. Single-point dependencies

A repo or module that only one person has touched in the last 90 days is a single point of failure. If that person gets sick, leaves, or takes vacation, the module stalls.

```sql
-- Simplified: modules with exactly one contributor in 90 days
SELECT
  repo_id,
  module_path,
  COUNT(DISTINCT author_email) AS contributor_count,
  GROUP_CONCAT(DISTINCT author_email) AS sole_authors
FROM git_commit_file
WHERE author_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
  AND is_noise = 0
GROUP BY repo_id, module_path
HAVING contributor_count = 1;
```

These show up as red cards on the panel, impossible to miss.

### 2. Collaboration islands

Someone who has neither given nor received a review in the past 30 days. They're effectively working "alone," either heads-down on an independent project, or already disengaging. Both are worth a lead's attention.

```javascript
// Build the collaboration graph
function buildCollabGraph(reviews) {
  const nodes = new Set();
  const edges = new Map();  // key: "a->b", value: count

  reviews.forEach(r => {
    nodes.add(r.reviewer);
    nodes.add(r.reviewee);
    const key = `${r.reviewer}->${r.reviewee}`;
    edges.set(key, (edges.get(key) || 0) + 1);
  });

  // Find nodes with zero degree
  const isolated = [...nodes].filter(n =>
    ![...edges.keys()].some(k => k.includes(n))
  );

  return { nodes: [...nodes], edges: [...edges.entries()], isolated };
}
```

### 3. Sudden-change alerts

A metric that drops sharply over a short window. For example:

- Collaboration score drops from 80 to 20 (last 14 days vs prior 14 days)
- Review participation goes from an average of 8 per week to 0
- Code volume drops more than 70%

A drop like this usually means the person is either heads-down on something big (a major refactor), or their state has shifted in a bad way. Both warrant a conversation.

```vue
<!-- AlertList.vue -->
<template>
  <div class="alert-list">
    <a-alert
      v-for="alert in alerts"
      :key="alert.id"
      :type="alert.severity"
      :message="alert.title"
      :description="alert.detail"
      show-icon
    />
  </div>
</template>
```

We deliberately didn't wire up email or DingTalk push for these. Push too much and it gets ignored; a "come look" dashboard is actually taken more seriously. That one took six months post-launch to figure out.

## A real scenario

Let me give a concrete one. About two months after the dashboard launched, a team lead came to me about someone on their team:

- Commit dimension: normal
- Code dimension: slight decline
- **Collaboration: dropped from top 30% to bottom 10%**
- Output: normal

Turns out that person had been loaned out to another team for the past month, de facto no longer in this lead's team, even though HR still had them there. Weekly standups hadn't surfaced it; the dashboard did.

That, to me, is the real value of visualization: making the invisible-but-should-be-visible things surface.

## A few Vue 2 engineering notes

A few small lessons from the Vue 2 layer:

- Import ECharts modules on demand, never the full package. The dashboard paints several charts on first paint, and a full import bloats the bundle to multiple MB.
  ```javascript
  // Import only radar and scatter
  import * as echarts from 'echarts/core';
  import { RadarChart, ScatterChart } from 'echarts/charts';
  import { TooltipComponent, GridComponent } from 'echarts/components';
  import { CanvasRenderer } from 'echarts/renderers';

  echarts.use([
    RadarChart, ScatterChart,
    TooltipComponent, GridComponent,
    CanvasRenderer,
  ]);
  ```
- Centralize state in Vuex, split per-page via modules. Don't let components self-fetch; otherwise navigating away and back loses all state.
- Use virtual scrolling for big tables (vue-virtual-scroller). Team views can run to hundreds of rows; without virtualization they lag.
- Aggregate on the backend, not the frontend. Don't make the browser `GROUP BY` over tens of millions of `git_commit_file` rows.

## Looking back at the whole system

Four posts in. Let me recap the full CoreTeam developer-profile pipeline:

1. **Metric definitions**: four dimensions (commit / code / collaboration / output), each constraining the others
2. **Data pipeline**: PyDriller parses Git into raw MySQL tables plus an author-mapping table
3. **Metric computation**: AirFlow four-layer DAG (prepare → clean → normalize → aggregate), every denoise rule explainable
4. **Visualization**: Vue 2 + ECharts. The point is surfacing anomalies, not displaying data.

After getting the whole chain running, the biggest takeaway is what I said in the first post: the hard part is always defining clearly what counts as contribution, not the tech. Tech choices are reversible; getting the definition wrong hurts for a long time.

Hope these four posts are useful to anyone building a similar system.
