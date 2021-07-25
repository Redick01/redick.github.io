# 分治、回溯

- 分治
- 回溯

## 分治

### 分治代码模版

```java
public void divide(Object... params) {
    // 终止条件
    if (null == param) {
        process_result;
        return;
    }
    //  处理数据
    data = prepare_date(param, . . .);
    // 分割子问题
    subproblems = split_problem(param, data);
    // 处理子问题
    subproplem1 = divide(subproblems[0], param,. . .);
    subproplem1 = divide(subproblems[1], param,. . .);
    subproplem1 = divide(subproblems[2], param,. . .);
    ...

    // 处理并合并结果
    result = process_result(subproplem1, subproplem2, subproplem3,...);
}
```

## 回溯

  回溯法采用试错的思想，它尝试分部的解决一个问题，在分部解决问题的过程中，当它通过尝试发现现有的分部答案不能得到有效的正确的答案的时候，它将取消上一步甚至上几步的计算，再通过其他可能的分部解答再次尝试寻找问题的答案。
  回溯法通常用最简单的递归来实现，在反复重复上述的步骤后可能出现两种情况：
  > 找到可能正确的答案
  > 在尝试了所有的分步方法后宣告该问题没有答案

  在最坏的情况下，回溯会导致一次复杂度为指数时间的计算