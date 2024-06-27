---
title: 需求3 echarts数据看板，默认样式修改，满足之前的图表编辑逻辑
created: '2024-05-27T07:12:48.418Z'
modified: '2024-05-30T02:44:51.030Z'
---

# 需求3 echarts数据看板，默认样式修改，满足之前的图表编辑逻辑

方案
1.增加父传子参数判断当前组件是否为编辑组件 isEditPage，并让编辑页也传递new_switch id,可以通过new_switch 和 isEditPage来做唯一区分 
2. 修改为同一new_switch id发现样式仍然不生效，此时考虑是由于编辑页样式覆盖导致的问题。
3. 发现是echarts在进行渲染过程中存在一个样式初始化，而在编辑界面控制的变量会经过vuex和父子组件传值，最终在echarts渲染前传递到echarts组件，并通过优先级覆盖的方式将编辑界面接收到的值传递给echarts的option参数覆盖默认参数。

