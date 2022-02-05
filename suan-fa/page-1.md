# 图算法

### 拓扑排序

需要决定顺序 通过dfs实现

* 210 Course Schedule ||    +

{% tabs %}
{% tab title="DFS 方法" %}
```
三种颜色 表示状态 Black White Gray
通过isImpossible() 判断是否成环
递归dfs来遍历
要是遇到其他灰色，代表有环
要是白色，就继续dfs
结束遍历 将自己标为黑色

将List<Integer> 转化为int[]
this.topologicalOrder.stream().mapToInt(Integer::valueOf).toArray();
```
{% endtab %}

{% tab title="Second Tab" %}
根据入度判断

每次从入度为0的点入手
{% endtab %}
{% endtabs %}

* 269 Alien Dictionary +

也是拓扑排序，但是处理起来好麻烦
