---
title: 可视化：Vue 2 把画像和异常亮出来
date: 2020-04-14 11:00:00
tags:
  - Career
  - Vue
  - 可视化
categories:
  - Career
description: 前三篇把数据链路跑通了，这一篇收尾——Vue 2 看板到底展示什么、怎么把异常亮出来、怎么帮管理者发现团队风险。
---

数据链路都跑通了，最后一篇讲讲怎么把这些数据画出来，并且真的能用。

老实说，可视化这块一开始我没太当回事——数据都算好了，画几个图表能有多难？做完才发现，做出来好看和做出来有用，中间隔着一个太平洋。看板真正的价值不是展示数据，是让异常自动浮出来。

## 看板要回答什么问题

我们的看板服务两类人，问题不同：

- Team Lead：我的团队现在有没有风险？谁状态不对？哪个模块成了单点？
- 个人（开发者本人）：我自己的画像长什么样？我在团队里大概什么位置？

所以看板分了三块：

1. 个人画像页：某个开发者四个维度的雷达图 + 趋势
2. 团队视图：分布、排名（脱敏，不公开）、异常清单
3. 风险监控页：单点依赖、协作孤岛、突变告警

下面一个个说。

## 个人画像页：雷达图是核心

雷达图是个人画像最直观的展示方式。四个维度（提交 / 代码 / 协作 / 产出）各算一个 0-100 的归一化分数，画一张五维或者四维的雷达图。

```vue
<!-- ProfileChart.vue 简化版 -->
<template>
  <div ref="chart" style="width: 100%; height: 360px;"></div>
</template>

<script>
import * as echarts from 'echarts';

export default {
  props: {
    scores: Object,  // { commit: 78, code: 65, review: 92, output: 70 }
    baseline: Object, // 团队均值，用于画参考圈
  },
  mounted() {
    const chart = echarts.init(this.$refs.chart);
    chart.setOption({
      tooltip: {},
      radar: {
        indicator: [
          { name: '提交', max: 100 },
          { name: '代码', max: 100 },
          { name: '协作', max: 100 },
          { name: '产出', max: 100 },
        ],
      },
      series: [{
        type: 'radar',
        data: [
          {
            value: Object.values(this.scores),
            name: '本人',
            areaStyle: { color: 'rgba(64, 169, 255, 0.3)' },
          },
          {
            value: Object.values(this.baseline),
            name: '团队均值',
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

这里有个细节我特别想强调：雷达图上一定要画一条"团队均值"的虚线作为参考。否则一个 78 分的提交维度到底是高是低，看的人完全没概念。配上参考圈，一眼就能看出"这个同学协作特别强，但产出偏弱"。

雷达图旁边再配一个 90 天的趋势线，看每个维度是不是在变化。单点静态的画像用处不大，趋势才有信号。

## 团队视图：分布比排名重要

第二版上线的时候，我们做了一份"团队内排名榜"，结果没两天就被吐槽了——任何形式的排名都会逼着大家刷数据，哪怕你不公开只给 Lead 看，Lead 跟人聊的时候也会无意泄露。

所以第三版改成只看分布：

- 每个维度的直方图 + 箱线图
- 标注离群点（散点图，不显示具体人名，点击才查）
- 团队整体的月度趋势

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
        { key: 'commit_score',  label: '提交维度' },
        { key: 'code_score',    label: '代码维度' },
        { key: 'review_score',  label: '协作维度' },
        { key: 'output_score',  label: '产出维度' },
      ],
    };
  },
  methods: {
    onOutlierClick({ metric, authorId }) {
      // 跳转到个人画像页
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

看板的克制比堆功能重要。该让 Lead 自己点进去看的，就不要直接列出来，隐私和数据准确性都靠这种克制保护。

## 风险监控：异常自动浮出来

这是整个看板真正有价值的一块。前面所有的工作，最后都是为了这一页能自动把风险亮出来。

我们做了几个关键的异常识别：

### 1. 单点依赖

某个仓库 / 模块，过去 90 天内只有一个人在动 → 这就是单点依赖。这个人生病、离职、休假，对应模块就停摆。

```sql
-- 简化逻辑：90 天内某模块提交者只有一个人
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

这种在面板上用红色卡片直接列出来，一眼就能看到。

### 2. 协作孤岛

某同学过去 30 天既没评审别人，也没被别人评审。这种基本就是在"单干"——要么是在做一个独立项目，要么是已经被边缘化了。两种情况都值得 Lead 关注。

```javascript
// 协作网络的节点
function buildCollabGraph(reviews) {
  const nodes = new Set();
  const edges = new Map();  // key: "a->b", value: 次数

  reviews.forEach(r => {
    nodes.add(r.reviewer);
    nodes.add(r.reviewee);
    const key = `${r.reviewer}->${r.reviewee}`;
    edges.set(key, (edges.get(key) || 0) + 1);
  });

  // 找出度为 0 的孤立节点
  const isolated = [...nodes].filter(n =>
    ![...edges.keys()].some(k => k.includes(n))
  );

  return { nodes: [...nodes], edges: [...edges.entries()], isolated };
}
```

### 3. 突变告警

某个指标在短时间内大幅下跌。比如：

- 协作维度分数从 80 跌到 20（过去 14 天 vs 前 14 天）
- 评审参与次数从平均每周 8 次跌到 0
- 代码量突然下降 70% 以上

这种突变通常意味着：这个人要么在憋大招（搞一个大重构），要么状态已经不对了。两种情况 Lead 都该聊一聊。

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

我们最终没接邮件 / 钉钉推送，故意不接。理由很简单：推送一多就会被忽略，看板这种"主动来看"的形式反而更容易被认真对待。这一点也是上线半年之后才想明白的。

## 一个真实场景

讲一个具体场景吧。看板上线大概两个月后，某 Team Lead 找我说，他在面板上看到自己组里某同学：

- 提交维度还正常
- 代码维度稍微下降
- 协作维度从前 30% 跌到了倒数 10%
- 产出维度正常

聊完发现，那位同学最近一个月都在帮另一个组做支援，已经事实上不在这个 Lead 的组里了——但人事关系还没走。这种情况靠以前的人工周会是发现不了的，看板倒先亮了出来。

我觉得这才是可视化的真正价值：不是为了好看，是让那些"看不出但应该被看到"的事情浮出来。

## Vue 2 的工程小细节

最后补几个 Vue 2 工程上的小经验：

- ECharts 用按需引入，别整包 import。看板首屏本来就要画好几个图，整包 import 会让 bundle 直接膨胀到几 MB。
  ```javascript
  // 只引入雷达图和散点图
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
- 数据用 Vuex 集中管理，分页面按 module 拆。不要让某个组件自己 fetch 数据，否则切换页面回来之后状态全丢。
- 大表用虚拟滚动（vue-virtual-scroller），团队视图动辄几百行，不虚拟化会卡。
- 后端聚合过的数据再给前端，不要把原始 `git_commit_file` 这种几千万行的表让前端去 group by。

## 整套系统回顾

四篇写完了，回顾一下整个 CoreTeam 用户画像系统的链路：

1. 指标定义：四维度（提交 / 代码 / 协作 / 产出），互为牵制
2. 数据管道：PyDriller 解析 Git → MySQL 原始表 + 作者映射表
3. 指标计算：AirFlow 四层 DAG（准备 → 清洗 → 归一 → 汇总），去噪规则可解释
4. 可视化：Vue 2 + ECharts，重点不在展示数据，在亮出异常

整套链路跑通之后，最大体会就是开篇那句话——做工程师画像，技术不是最难的，最难的永远是定义清楚"什么算贡献"。技术方案都好换，定义错了改回来要伤一阵子。

希望这四篇对想做类似系统的同学有点参考。
