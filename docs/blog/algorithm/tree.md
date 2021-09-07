# 树 <!-- {docsify-ignore-all} -->

## 树结构代码

```java
public class TreeNode {
    public int val;
    public TreeNode left, right;
    public TreeNode(int val) {
        this.val = val;
        this.left = null;
        this.right = null;
    }
}
```

## 二叉树的遍历

1. 前序Pre-order：根-左-右
2. 中序In-order：左-根-右
3. 后续Post-order：左-右-根

- **递归实现**


```java
/**
 *前序
 */
public void preOrder(TreeNode root) {
    if (root != null) {
        System.out.print(root.val + "  ");
        preOrder(root.left);
        preOrder(root.right);
    }
}

/**
 * 中序遍历
 * @param tree
 */
public void inOrder(Node tree) {
    if (tree == null) {
        return;
    }

    inOrder(tree.left);

    System.out.print(tree.val + "  ");

    inOrder(tree.right);
}

/**
 * 后序遍历
 * @param tree
 */
public void postOrder(Node tree) {
    if (tree == null) {
        return;
    }

    postOrder(tree.left);

    postOrder(tree.right);

    System.out.print(tree.data + "  ");
}

    /**
     * 层次遍历
     * @param tree
     */
public void BFSOrder(Node tree) {
    if (tree == null) {
        return;
    }

    Queue<Node> queue = new LinkedList<>();
    Node temp = null;
    // 使用 offer 和 poll 优于 add 和 remove 之处在于前者可以通过返回值判断成功与否，而不抛出异常
    queue.offer(tree);
    while (!queue.isEmpty()) {
        temp = queue.poll();
        System.out.print(temp.data + "  ");
        if (temp.left != null) {
            queue.offer(temp.left);
        }

        if (temp.right != null) {
            queue.offer(temp.right);
        }
    }
}

/**
 * 求树的高度，从 1 开始
 * @param tree
 * @return
 */
public int treeHeight (Node tree) {
    if (tree == null) {
        return 0;
    }

    int lHeight = treeHeight(tree.left);
    int rHeight = treeHeight(tree.right);

    return lHeight > rHeight ? lHeight + 1 : rHeight + 1;
}
```

## 二叉搜索树

二叉搜索树，也叫二叉查找树、有序二叉树、排序二叉树、是指一颗空树活着具有下列性质的二叉树：

> 1.左子树上**所有节点**的值均小于它的跟节点的值；
> 2.柚子树上**所有节点**的值均大于它的跟节点的值；
> 3.以此类推：左、右子树也分别为二叉搜索树。（重复性）

## AVL树

1. 平衡二叉搜索树
2. 每个节点存平衡因子 balance factor={-1, 0, 1}
3. 四种旋转操作
（1）左旋
（2）右旋
（3）左右旋
（4）右左旋

![avatar](../../../_media/image/algorithm/WechatIMG162.png)

不足：节点需要存储额外信息、切调整次数频繁

## 红黑树

红黑树是一种`近似平衡`的二叉搜索树，它能够确保任何一个节点的左右子树的`高度差小于两倍`。具体来说，红黑树是满足如下条件的二叉搜索树：

- 每个节点要么是红色，要么是黑色
- 根结点是黑色
- 每个叶子结点（NIL节点，空节点）是黑色的
- 不能有相邻接的两个红色节点
- 从任一个节点到其每个叶子的所有路径都包含相同数目的黑色节点。