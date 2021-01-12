# 排序算法



## 一.计数排序

### 计数排序概念

​    计数排序不是一个比较排序算法，该算法于1954年由 Harold H. Seward提出，通过计数将时间复杂度降到了O(N)。

### 基础版算法步骤

```
第一步：找出原数组中元素值最大的，记为max。

第二步：创建一个新数组count，其长度是max加1，其元素默认值都为0。

第三步：遍历原数组中的元素，以原数组中的元素作为count数组的索引，以原数组中的元素出现次数作为count数组的元素值。

第四步：创建结果数组result，起始索引index。

第五步：遍历count数组，找出其中元素值大于0的元素，将其对应的索引作为元素值填充到result数组中去，每处理一次，count中的该元素值减1，直到该元素值不大于0，依次处理count中剩下的元素。

第六步：返回结果数组result。
```

### 基础版代码实现

```
public static int[] countingSort(int[] n) {
        // 找到待排序数组中最大值
        int max = Integer.MIN_VALUE;
        for (int a : n) {
            max = Math.max(max, a);
        }
        // 创建中间数组，数组长度最大值+1
        int[] count = new int[max + 1];
        for (int a : n) {
            count[a]++;
        }
        // 创建结果数组
        int[] result = new int[n.length];
        // 创建结果数组的起始索引
        int index = 0;
        // 遍历计数数组，将计数数组的索引填充到结果数组中
        for (int i=0; i<count.length; i++) {
            while (count[i]>0) {
                result[index++] = i;
                count[i]--;
            }
        }
        // 返回结果数组
        return result;
}
```

### 优化版

​    基础版能够解决一般的情况，但是它有一个缺陷，那就是存在空间浪费的问题。比如一组数据{101,109,108,102,110,107,103}，其中最大值为110，按照基础版的思路，我们需要创建一个长度为111的计数数组，但是我们可以发现，它前面的[0,100]的空间完全浪费了，那怎样优化呢？将数组长度定为max-min+1，即不仅要找出最大值，还要找出最小值，根据两者的差来确定计数数组的长度。

```
public static int[] countingSort2(int[] n) {
        // 找到待排序数组中最大值
        int max = Integer.MIN_VALUE;
        int min = Integer.MAX_VALUE;
        for (int a : n) {
            max = Math.max(max, a);
            min = Math.min(min, a);
        }
        // 创建中间数组，数组长度最大值+1
        int[] count = new int[max - min + 1];
        for (int a : n) {
            count[a - min]++;
        }
        // 创建结果数组
        int[] result = new int[n.length];
        // 创建结果数组的起始索引
        int index = 0;
        // 遍历计数数组，将计数数组的索引填充到结果数组中
        for (int i=0; i<count.length; i++) {
            while (count[i]>0) {
                result[index++] = i + min;
                count[i]--;
            }
        }
        // 返回结果数组
        return result;
}
```

## 二.快速排序

### 快速排序概念
    快速排序的基本思想（分治）：通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。
### 算法步骤
    快速排序使用分治法来把一个串（list）分为两个子串（sub-lists）。具体算法描述如下：

    从数列中挑出一个元素，称为 “基准”（pivot）；
    重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；
    递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序。
### 代码实现
```
function quickSort(arr, left, right) {
    varlen = arr.length,
        partitionIndex,
        left = typeofleft != 'number'? 0 : left,
        right = typeofright != 'number'? len - 1 : right;
 
    if(left < right) {
        partitionIndex = partition(arr, left, right);
        quickSort(arr, left, partitionIndex-1);
        quickSort(arr, partitionIndex+1, right);
    }
    returnarr;
}
 
function partition(arr, left ,right) {     // 分区操作
    varpivot = left,                      // 设定基准值（pivot）
        index = pivot + 1;
    for(vari = index; i <= right; i++) {
        if(arr[i] < arr[pivot]) {
            swap(arr, i, index);
            index++;
        }       
    }
    swap(arr, pivot, index - 1);
    returnindex-1;
}
 
function swap(arr, i, j) {
    vartemp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}
```
