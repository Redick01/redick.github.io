# 深度优先搜索和广度优先搜索 <!-- {docsify-ignore-all} -->


## 深度优先搜索代码模版

- **递归写法**

```java

Set<T> visited = new HashSet<>();

void dfs(TreeNode root, Set<T> visited) {
    // 终止条件
    if(visited.contain(node)) {
        // 已经访问过了
        return;
    }
    visited.add(root);
    for(TreeNode node : root.children()) {
        if(!visited.contain(node)) {
            dfs(node, visited);
        }
    }
}
```

- **非递归写法**

```java
void dfs(TreeNode root) {
    if(root == null) {
        return new ArrayList();
    }
    Set<T> visited = new HashSet<>();
    Stack<TreeNode> stack = new Stack<>();
    List<T> res = new ArrayList();
    stack.push(root);
    while(!stack.isEmpty()) {
        TreeNode node = stack.pop();
        visited.add(node);
        // 处理node
        process(ndioe);
        
        nodes = generate_related_nodes(node);
        stack.push(nodes);
    }
}
```

## 广度优先搜索代码模版

```java
void bfs(TreeNode graph, int start, int end) {
    Deque queue = new ArrayDeque();
    Set<T> visited = new HashSet<>();
    queue.add([start]);
    visited.add(start);
    while(!queue.isEmpty()) {
        node = queue.pop();
        visited.add(node);
        process(node);
        nodes = generate_related_nodes(node);
        queue.push(nodes);
    }
}
```