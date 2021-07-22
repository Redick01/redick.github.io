# 递归与相关题目

## 递归算法的处理思路

- **算法流程**

  > 1.终止条件

  > 2.当前层处理逻辑

  > 3.进入下一层

  > 4.清理当前层状态

**注意：**

    1. 不要进行人肉递归
    2. 找到最近最简单方法，将其拆解成可重复解决的问题（重复字问题）
    3. 学会数学归纳法

- **递归代码模版**

```java
public class Recursion {

    private static final long MAX_LEVEL = Long.MAX_VALUE;

    private void recursion(Object level, String... s) {
        // 1.   终止条件
        if ((long)level > MAX_LEVEL) {
            // 处理结果
            process_result
            return;
        }
        // 2.处理当前层逻辑
        process(level, data);
        // 3.进入下一层
        recursion((long)level + 1, s);
        // 
    }
}
```

## 相关题目

- 二叉树的前中后序遍历
- 爬楼梯问题（斐波那契数列）
- N叉树的前中后序遍历
- 生成括号