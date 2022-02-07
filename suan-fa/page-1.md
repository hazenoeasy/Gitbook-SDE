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

* 886 Possible Bipartition +

1. 通过BFS染色来做
2. 通过DFS来做
3. 特殊并查集 每个点和他的邻居不应该在同一个集合中。所以将它的所有邻居放在同一个union中。将点v的所有邻接点全部归并在一个集合，若有邻接点与该点v在同一个集合，说明冲突，不为二分图

* 2092 Find All People With Secret

1. Dijkstra's Alogrithm  我是先将所有点入站, 然后每次遍历后，update堆内元素。

{% tabs %}
{% tab title="my solution" %}
```
class Solution {
    // Dijkstra's Algorithm
    // 维护一个最小堆
    // 维护一个U 代表 未被选择的点
    class Node {
        int reachTime; // 未连通时，为Integer.Max_VALUE
        int number;
        Node(int number){
            this.number = number;
            this.reachTime = Integer.MAX_VALUE;
        }
        Node(int number,int reachTime){
            this.number = number;
           this.reachTime = reachTime;
        }
    }
    public List<Integer> findAllPeople(int n, int[][] meetings, int firstPerson) {
        Queue<Node> queue = new PriorityQueue<>((a,b)->a.reachTime-b.reachTime);   
        List<Node> nodeList = new ArrayList<>();
        boolean[] isVisited = new boolean[n];
        isVisited[0]=true;
        List<ArrayList<Node>> adjList = new ArrayList<>();
        List<Integer> result = new ArrayList<>();
        int limit =0;
        result.add(0);
        // 初始化 adjlist
        for(int i=0;i<n;i++){
            adjList.add(new ArrayList<>());
        }
        // 赋值adjlist
        for(int i=0;i<meetings.length;i++){
            adjList.get(meetings[i][0]).add(new Node(meetings[i][1],meetings[i][2]));
            adjList.get(meetings[i][1]).add(new Node(meetings[i][0],meetings[i][2]));
        }
        // 初始化
        for(int i=0;i<n;i++){
            Node node;
            if(i==firstPerson){
                node = new Node(i,0);
                nodeList.add(node);
            }else{
                node = new Node(i);
                nodeList.add(node);
            }
            // // 加入队列
            // queue.add(node);
        }
        for(int i=0;i<adjList.get(0).size();i++){
            Node temp1 = nodeList.get(adjList.get(0).get(i).number);
            Node temp2 = adjList.get(0).get(i);
           temp1.reachTime = Math.min(temp1.reachTime,temp2.reachTime);
            
        }
        for(int i=0;i<n;i++){
            queue.add(nodeList.get(i));
        }
        while(!queue.isEmpty() && queue.peek().reachTime != Integer.MAX_VALUE){
            Node topNode = queue.remove();
            result.add(topNode.number);
            limit = topNode.reachTime;
            isVisited[topNode.number]=true;
            List<Node> list = adjList.get(topNode.number);
            for(int i=0;i<list.size();i++){
                Node node = list.get(i);
                if(!isVisited[node.number]){
                    Node node_origin = nodeList.get(node.number);
                    if(node_origin.reachTime > node.reachTime && node.reachTime >= limit){
                        queue.remove(node_origin);
                        node_origin.reachTime = node.reachTime;
                        queue.add(node_origin);
                    }
                }
            }
        }
        return result;
    }
}
```
{% endtab %}

{% tab title="Dijkstra's Algorithm" %}
```
class Solution {
    public List<Integer> findAllPeople(int n, int[][] meetings, int firstPerson) {
        // 通过一个List 数组表示 图
        List<int[]>[] graph = new ArrayList[n];
        for (int i = 0; i < n; i++) {
            graph[i] = new ArrayList<>();
        }
        
        for (int[] meeting : meetings) {
            // 临街list 对应节点的邻接列表，插入元素以及其值
            graph[meeting[0]].add(new int[]{meeting[1], meeting[2]});
            graph[meeting[1]].add(new int[]{meeting[0], meeting[2]});
        }
        // 答案
        List<Integer> res = new ArrayList<>();
        // 按时间排序
        PriorityQueue<int[]> q = new PriorityQueue<>((o1, o2) -> (o1[1] - o2[1]));
        // 初始节点为 0， 时间为0 只插入两个元素
        q.offer(new int[]{0, 0});
        q.offer(new int[]{firstPerson, 0});
        boolean[] vis = new boolean[n];
        while (!q.isEmpty()) {
            // 提取出的点
            int[] cur = q.poll();
            int v = cur[0], t = cur[1];
            // 如果已经有这个点了
            if (vis[v]) continue;
            vis[v] = true;
            // res添加add
            res.add(v);
            遍历他的邻接
            for (int[] next : graph[cur[0]]) {
                
                int p = next[0], time = next[1];
                if (vis[p] || time < t) continue;
                q.offer(next);
            }
        }
        return res;
    }
}
```
{% endtab %}
{% endtabs %}

* 990 Satisfiability of Equality Equations

1. 并查集，并查集的题判断两个点是否属于同一个集合，可以跑两遍循环，第一遍加入集合，第二遍判断是否属于同一个集合。

* 834 Sum of Distances in Tree 没思路
*

