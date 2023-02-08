## labuladong 算法经验

### 归并排序

- **归并排序其实就是二叉树的后续遍历，先排序好左边部分、再排序好右边部分，最后合并**

- 算法框架

  ```cpp
  // 定义：排序 nums[lo..hi]
  void sort(int[] nums, int lo, int hi) {
      if (lo == hi) {
          return;
      }
      int mid = (lo + hi) / 2;
      // 利用定义，排序 nums[lo..mid]
      sort(nums, lo, mid);
      // 利用定义，排序 nums[mid+1..hi]
      sort(nums, mid + 1, hi);
      /****** 后序位置 ******/
      // 此时两部分子数组已经被排好序
      // 合并两个有序数组，使 nums[lo..hi] 有序
      merge(nums, lo, mid, hi);
  }
  
  // 将有序数组 nums[lo..mid] 和有序数组 nums[mid+1..hi]
  // 合并为有序数组 nums[lo..hi]
  void merge(int[] nums, int lo, int mid, int hi);
  ```

### 快速排序

- **归并排序是 先把左半边数组排好序，再把右半边数组排好序，然后把两半数组合并。**
- **快速排序是先将一个元素排好序，然后再将剩下的元素排好序**。
- **快速排序的框架 是二叉树的 前序遍历。**

```cpp
void sort(int[] nums, int lo, int hi) {
    if (lo >= hi) {
        return;
    }
    // 对 nums[lo..hi] 进行切分
    // 使得 nums[lo..p-1] <= nums[p] < nums[p+1..hi]
    int p = partition(nums, lo, hi);
    // 去左右子数组进行切分
    sort(nums, lo, p - 1);
    sort(nums, p + 1, hi);
}
```

所以 `partition` 函数干的事情，其实就是把 `nums[p]` 这个元素排好序了，把这个元素放到正确的位置。

- 快速排序，对于已经排好序的数组，快速排序时间复杂度会变得非常高。所以一般会引入**洗牌算法**将其打乱；

  