---
title: 需求2 echarts刷新渲染动画问题，echarts5在进行页面重构时，会触发非正常label排序动画问题。
created: '2024-05-23T09:42:54.533Z'
modified: '2024-05-27T07:12:44.178Z'
---

# 需求2 echarts刷新渲染动画问题，echarts5在进行页面重构时，会触发非正常label排序动画问题。

* 方案一：
container-group index.vue
定位：bi/components/chart-display/echart.vue
解决方案：限制固定id的图表限制动画效果

