## [返回](/)

# 剑指offer合集

## 总结

- 二分查找类：[11](#_11)
- 回溯法：[12](#_12)，[13](#_13)
- 动态规划：[14- I](#_14- I)
- 贪心：[14- II](#_14- II)
- 位运算：[15](#_15)
- 快速幂：[16](#_16)
- 大数运算：[17](#_17)

> 小问题：对公司的万名员工的年龄实现时间效率O(N)的排序
>
> 解答：使用一个100的数组，遍历一遍统计每个年龄的人数，就相当于排序了

## 11

[剑指 Offer 11. 旋转数组的最小数字](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)

![image-20210422094849212](imgs\jz\jz11.png)

首先想到的是O(N)级别的方法，即遍历一次，如果出现一个元素小于前一个，那么这就是最小元素；如果遍历到末尾也没有这种元素，说明没有旋转，那么第一个就是最小元素

```java
class Solution {
    public int minArray(int[] numbers) {
        for(int i = 0; i < numbers.length - 1; i++){
            if(numbers[i + 1] >= numbers[i]) continue;
            return numbers[i + 1];
        }
        return numbers[0];
    }
}
```

有没有比O(N)更好的方法？首先应当想到O(logN)的二分查找。寻找这个数组的规律，我们发现，对于旋转了的数组，可以划分为两个数组，左边的递增部分总是大于右边的递增部分，且最小值为两个数组的分界处

令start和end指针分别指向左边数组的最小值和右边数组的最大值，也就是第一个元素和最后一个元素，如果mid处的元素大于start，说明最小值在mid的右边，则令start=mid；反之若mid处的元素值小于start，则最小值在mid的左边，则令end=mid，当start+1==end，两个指针到达各自数组的分界处，返回end处的值。那么，**如果mid处的元素与start或end处相等呢？**，以mid处的元素等于start来分析，对于[1, 1, 1, 0, 1]，最小值是在右边的，对于[1, 0, 1, 1, 1]，最小值是在左边的，也就是此时无法判断mid落在了那边，这时候只能切换回顺序查找了

特殊情况是数组未旋转，即start处的元素小于end，那么就可以直接返回start处的元素值了

```java
class Solution {
    public int minArray(int[] numbers) {
        int start = 0, end = numbers.length - 1;
        if(numbers[start] < numbers[end]) return numbers[start];
        while(start + 1 < end){
            int mid = ((end - start) >> 1) + start;
            if(numbers[mid] > numbers[start]){
                start = mid;
            }else if(numbers[mid] < numbers[start]){
                end = mid;
            }else{
                // 切换顺序查找
                return minArray(numbers, start, end);
            }
        }
        return numbers[end];
    }
    public int minArray(int[] numbers, int start, int end) {
        for(int i = start; i < end; i++){
            if(numbers[i + 1] >= numbers[i]) continue;
            return numbers[i + 1];
        }
        return numbers[0];
    }
}
```

[力扣官方题解](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/solution/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-by-leetcode-s/)

```java
class Solution {
    public int minArray(int[] numbers) {
        int start = 0, end = numbers.length - 1;
        if(numbers[start] < numbers[end]) return numbers[start];
        while(start < end){
            int mid = (start + end) >> 1;
            if(numbers[mid] < numbers[end]){
                end = mid;
            }else if(numbers[mid] > numbers[end]){
                start = mid + 1;
            }else{
                end--;
            }
        }
        return numbers[start];
    }
}
```

## 12

[剑指 Offer 12. 矩阵中的路径](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/)

![image-20210425101820395](imgs\jz\jz12.png)

首先的想法就是回溯法，对每个节点不断试四个方向上所有选项，某次选项若不符合，就回退到上一个选项重新选，注意访问过的元素不要再次访问，用一个二维数组存放访问记录

```java
class Solution {
    public boolean exist(char[][] board, String word) {
        char begin = word.charAt(0);
        int row = board.length, col = board[0].length;
        boolean[][] hasVisited = new boolean[row][col];
        for(int i = 0; i < board.length; i++){
            for(int j = 0; j < board[0].length; j++){
                if(begin == board[i][j]){
                    if(exist(board, word, hasVisited, i, j, 1)){
                        return true;
                    }
                }
            }
        }
        return false;
    }
    public boolean exist(char[][] board, String word, boolean[][] hasVisited, int x, int y, int index){
        // 当前节点已被访问
        hasVisited[x][y] = true;
        if(index >= word.length()) return true;
        int row = board.length, col = board[0].length;
        char target = word.charAt(index);
        if(x > 0 && target == board[x - 1][y] 
           && !hasVisited[x - 1][y]) {
            if(exist(board, word, hasVisited, x - 1, y, index + 1)){
                return true;
            }
        }
        if(x < row - 1 && target == board[x + 1][y] 
           && !hasVisited[x + 1][y]) {
            if (exist(board, word, hasVisited, x + 1, y, index + 1)){
                return true;
            }
        }
        if(y > 0 && target == board[x][y - 1] 
           && !hasVisited[x][y - 1]) {
            if (exist(board, word, hasVisited, x, y - 1, index + 1)){
                return true;
            }
        }
        if(y < col - 1 && target == board[x][y + 1] 
           && !hasVisited[x][y + 1]) {
            if (exist(board, word, hasVisited, x, y + 1, index + 1)){
                return true;
            }
        }
        // 当前节点的四个方向都找不到合适选项，访问记录重置
        hasVisited[x][y] = false;
        return false;
    }
}
```

这是我第一遍的写法，写的不好，很不优美，按照官方题解改进了一下

```java
class Solution {
    public boolean exist(char[][] board, String word) {
        char begin = word.charAt(0);
        int row = board.length, col = board[0].length;
        boolean[][] hasVisited = new boolean[row][col];
        for(int i = 0; i < board.length; i++){
            for(int j = 0; j < board[0].length; j++){
                if(begin == board[i][j]){
                    if(exist(board, word, hasVisited, i, j, 0)){
                        return true;
                    }
                }
            }
        }
        return false;
    }
    public boolean exist(char[][] board, String word, boolean[][] hasVisited, int x, int y, int index){
        if(x >= board.length || x < 0 
                || y >= board[0].length || y < 0 
                || hasVisited[x][y] 
                || board[x][y] != word.charAt(index))
            return false;
        // 当前节点访问记录置为已访问
        hasVisited[x][y] = true;
        // 成功到达字符串末尾
        if(index == word.length() - 1) return true;
        // 只要有一个选项满足就可
        if(exist(board, word, hasVisited, x - 1, y, index + 1)
                || exist(board, word, hasVisited, x + 1, y, index + 1)
                || exist(board, word, hasVisited, x, y - 1, index + 1)
                || exist(board, word, hasVisited, x, y + 1, index + 1)){
            return true;
        }
        // 当前节点的四个方向都找不到合适选项，访问记录重置
        hasVisited[x][y] = false;
        return false;
    }
}
```

## 13

[剑指 Offer 13. 机器人的运动范围](https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/)

![image-20210425114250987](imgs\jz\jz13.png)

跟[12](#_12)题基本一样，dfs实现的回溯

```java
class Solution {
    public int movingCount(int m, int n, int k) {
        boolean hasVisited[][] = new boolean[m][n];
        return dfs(m, n, k, 0, 0, hasVisited);
    }
    public int dfs(int m, int n, int k, int x, int y, boolean[][] hasVisited){
        if(x >= m || x < 0 || y >= n || y < 0 
           || !canGoInto(k, x, y) 
           || hasVisited[x][y]) 
            return 0;
        hasVisited[x][y] = true;
        return 1 
            + dfs(m, n, k, x - 1, y, hasVisited) + dfs(m, n, k, x + 1, y, hasVisited) 
            + dfs(m, n, k, x, y - 1, hasVisited) + dfs(m, n, k, x, y + 1, hasVisited);
    }
    public boolean canGoInto(int k, int x, int y){
        int sum = 0;
        while(x != 0){
            sum += x % 10;
            x /= 10;
        }
        while(y != 0){
            sum += y % 10;
            y /= 10;
        }
        return sum <= k;
    }
}
```

## 14- I

[剑指 Offer 14- I. 剪绳子](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/)

![image-20210427095614092](imgs\jz\jz14.png)

首先很容易想到要用动态规划，先写状态转移方程，设长度为n的绳子，最大乘积是f(n)

**f(n) = max( f(n), max( r * f(n - r), r * (n - r) ) )，1 <= r < n - 1**

**f(2) = 1**

> 为什么r < n - 1？因为如果r=n-1，则为(n-1)*f(1)，f(1)不满足题意
>
> 为什么要r * (n - r)？因为f(n - r)是切分了的结果，当然也要考虑不切分的情况

```java
class Solution {
    public int cuttingRope(int n) {
        int[] dp = new int[n - 1];
        dp[0] = 1;  // f(2)
        for(int i = 3; i <= n; i++){
            for(int r = 1; r < i - 1; r++){
                dp[i - 2] = Math.max(dp[i - 2], r * dp[i - 2 - r]);
                dp[i - 2] = Math.max(dp[i - 2], r * (i - r)); 
            }
        }
        return dp[n - 2];
    }
}
```

对状态方程做优化，完全可以写成如下，只需遍历到一半

**left = max(r, f(r))**

**right = max(n-r, f(n-r))**

**f(n) = max( f(n), left * right )，1 <= r <= n/2**

**f(2) = 1**

```java
class Solution {
    public int cuttingRope(int n) {
        int[] dp = new int[n - 1];
        dp[0] = 1;  // f(2)
        for(int i = 3; i <= n; i++){
            for(int r = 1; r <= i / 2; r++){
                int left = Math.max(r, (r < 2) ? r : dp[r - 2]);
                int right = Math.max(dp[i - 2 - r], i - r);
                dp[i - 2] = Math.max(dp[i - 2], left * right);
            }
        }
        return dp[n - 2];
    }
}
```

[力扣官方题解](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/solution/mian-shi-ti-14-i-jian-sheng-zi-tan-xin-si-xiang-by/)

> 贪心算法： 尽可能将绳子以长度3等分为多段时，乘积最大，证明过程见上面官方题解

```java
class Solution {
    public int cuttingRope(int n) {
        if(n <= 4) return (int)Math.pow(2, n - 2);
        int ans = 1;
        for(; n >= 3; n-=3){
            ans *= 3;
        }
        if(n == 2) ans *= 2;
        if(n == 1) ans = ans / 3 * 4;
        return ans;
    }
}
```

优化一下

```java
class Solution {
    public int cuttingRope(int n) {
        if(n <= 4) return (int)Math.pow(2, n - 2);
        int ans = 1;
        for(; n > 4; n-=3){
            ans *= 3;
        }
        return ans * n;
    }
}
```

## 14- II

[剑指 Offer 14- II. 剪绳子 II](https://leetcode-cn.com/problems/jian-sheng-zi-ii-lcof/)

![image-20210511110443889](imgs\jz\jz14-2.png)

贪心算法直接添加个取余就可以

```java
class Solution {
    public int cuttingRope(int n) {
        if(n <= 4) return (int)Math.pow(2, n - 2);
        long ans = 1;
        for(; n > 4; n-=3){
            ans *= 3;
            ans %= 1000000007;
        }
        return (int)((ans * n) % 1000000007);
    }
}
```

dp呢？

```java
import java.math.BigInteger;
class Solution {
    public int cuttingRope(int n) {
        BigInteger[] dp = new BigInteger[n - 1];
        Arrays.fill(dp, BigInteger.valueOf(0));
        dp[0] = BigInteger.valueOf(1);
        for(int i = 3; i <= n; i++){
            for(int r = 1; r < i - 1; r++){
                BigInteger temp = dp[i - 2 - r].multiply(BigInteger.valueOf(r));
                dp[i - 2] = dp[i - 2].compareTo(temp) > 0 ? dp[i - 2] : temp;
                temp = BigInteger.valueOf(r * (i - r));
                dp[i - 2] = dp[i - 2].compareTo(temp) > 0 ? dp[i - 2] : temp;
            }
        }
        return dp[n - 2].mod(BigInteger.valueOf(1000000007)).intValue();
    }
}
```

使用BigInteger

## 15

[剑指 Offer 15. 二进制中1的个数](https://leetcode-cn.com/problems/er-jin-zhi-zhong-1de-ge-shu-lcof/)

![image-20210512092003317](imgs\jz\jz15.png)

这里肯定需要用到移位运算

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int count = 0;
        for(int i = 0; i < 32; i++){
            if(n % 2 != 0) count++;
            n = n >>> 1;
        }
        return count;
    }
}
```

我第一次是通过n%2来判断最后一位是0还是1，最后一位是1，%2肯定不为0，然而还有很大优化空间

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int count = 0;
        while(n != 0){
            count += n & 1;
            n >>>= 1;
        }
        return count;
    }
}
```

直接用n&1判断，把for换成while，少一个局部变量，然而还是得一个一个参与循环，仍有优化空间

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int count = 0;
        while(n != 0){
            n &= (n - 1);
            count++;
        }
        return count;
    }
}
```

这个方法非常巧妙，对于n &= (n - 1)这一步，末尾是1的话，消去一个1；末尾是0的话，直接消去了若干个0和1个1

> 例如：
>
> ![image-20210512100811738](imgs\jz\jz15a.png)
>
> 如图，红框的若干个0被一起消去了，绿框的1个1被消去了

## 16

[剑指 Offer 16. 数值的整数次方](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/)

![image-20210513091011408](imgs\jz\jz16.png)

首先想到的肯定是循环一个个乘，但效率太低了，这时候想到，对于稍大一点的n，可以直接令x乘x，这样就是指数级增长而不是一个个增长，例如：x ^ 13 == ((x ^ 2) ^ 2) ^ 2  *  x ^ 5，x ^ 5可以作为余下的rest部分，一个一个循环来计算

```java
class Solution {
    public double myPow(double x, int n) {
        // 各种边界条件
        if(x == 1 || n == 0) return 1;
        if(x == -1) return n % 2 == 0 ? 1 : -1;
        if(n == 1) return x;
        if(n == Integer.MIN_VALUE || (n == Integer.MAX_VALUE && x < 1)) return 0;
        // n小于0时的处理
        if(n < 0){
            x = 1 / x;
            n = -n;
        }
        double ans = x;
        int rest = 0;
        for(int i = 2; i <= n; i <<= 1){
            rest = n - i;
            ans *= ans;
        }
        for(int i = 0; i < rest; i++){
            ans *= x;
        }
        return ans;
    }
}
```

仍然有很大的优化空间，例如对于x ^ 13 = x ^ 8 * x ^ 5，又可以分解成x ^ 13 = x ^ 8 * x ^ 4 * x ^ 1，而x ^ 4和x ^ 1在计算x ^ 8的过程中都计算过，这里就是快速幂算法

对于x ^ n，n的二进制为bm...b4b3b2b1，则x ^ n == x ^ (1 * b1 + 2 * b2 + 4 * b3……+ (2 ^ m−1) * bm)

例如：x ^ 13 == x ^ 1101 == x ^ (1 * 1) * x ^ (0 * 2) * x ^ (1 * 4) * x ^ (1 * 8)

依此改进：

```java
class Solution {
    public double myPow(double x, int n) {
        //-Integer.MIN_VALUE会溢出，所以用一个long来处理
        long b = n;
        if(b < 0){
            x = 1 / x;
            b = -b;
        }
        double ans = 1;
        while(b != 0){
            if((b & 1) == 1)
                ans *= x;
            x *= x;
            b >>= 1;
        }
        return ans;
    }
}
```

还有一种递归的思路，时间复杂度等同于快速幂，但空间复杂度高一些

```java
class Solution {
    public double myPow(double x, int n) {
        // 各种边界条件
        if(x == 1 || n == 0) return 1;
        if(x == -1) return n % 2 == 0 ? 1 : -1;
        if(n == 1) return x;
        if(n == Integer.MIN_VALUE || (n == Integer.MAX_VALUE && x < 1)) return 0;
        // 递归
        if(n < 0){
            x = 1 / (x * myPow(x, -1 - n));
        }else if(n % 2 == 0){
            x = myPow(x, n / 2) * myPow(x, n / 2);
        }else{
            x = x * myPow(x, n - 1);
        }
        return x;
    }
}
```

即n为偶数的话，x ^ n = (x ^ (n / 2)) * (x ^ (n / 2)) 

n为奇数的话，x ^ n = x * (x ^ (n - 1))

n < 0，x ^ n = 1 / x ^ -n，其中-n可能溢出，x ^ -n要改成x ^ (-1 - n) * x

## 17

[剑指 Offer 17. 打印从1到最大的n位数](https://leetcode-cn.com/problems/da-yin-cong-1dao-zui-da-de-nwei-shu-lcof/)

![image-20210513103835384](imgs\jz\jz17.png)

实际上，这一题需要考虑到大数，即超过MAX_VALUE的部分，需要用字符串来处理

1. 字符串模拟大数运算

2. 全排列：将每个位上从0-9的排列全部按顺序列出来

```java
class Solution {
    // sb，单个数字；ans：结果集
    StringBuilder sb = new StringBuilder(), ans = new StringBuilder();
    int n;
    public void printNumbers(int n) {
        this.n = n - 1;
        print(0);
        ans.deleteCharAt(ans.length() - 1);  // 去掉最后一个逗号
        System.out.println(ans);  // 打印结果
    }
    public void print(int index){
        for(int i = 0; i < 10; i++){
            sb.append(i);  // 添加当前位的数字
            if(index == n){  // 如果当前位是最后一位了，则不再往下递归了，直接将结果加入结果集
                ans.append(sb);
                ans.append(",");
            }else print(index + 1);
            sb.deleteCharAt(sb.length() - 1);  // 删除当前位的数字，下一轮循环中添加新的
        }
        // 删除递归上一层的最后一位数字，也就是删除上一位的数字
        if(sb.length() > 0)
            sb.deleteCharAt(sb.length() - 1);
    }
}
```

此时有两个特殊值没有处理，1. 第一个数字也就是0；2. 其他数字前面的0，比如说001，002，...，099，前面的0是需要去掉的

对于1，直接去掉第一个数字就好；对于2，需要在添加到结果集之前判断一下；可以都放在添加到结果集之前判断

```java
class Solution {
    StringBuilder sb = new StringBuilder(), ans = new StringBuilder();
    int n;
    public void printNumbers(int n) {
        this.n = n - 1;
        print(0);
        ans.deleteCharAt(ans.length() - 1);
        System.out.println(ans);
    }
    public void print(int index){
        for(int i = 0; i < 10; i++){
            sb.append(i);
            if(index == n){
                for(int j = 0; j < sb.length();){
                    // 如果是0就删除，遇到非0就跳出循环
                    if(sb.charAt(j) == '0') sb.deleteCharAt(j);
                    else break;
                }
                // 第一个数字全是0，因此length为0，不会被添加到结果集
                if(sb.length() != 0) {
                    ans.append(sb);
                    ans.append(",");
                }
            }else print(index + 1);
            if(sb.length() > 0) sb.deleteCharAt(sb.length() - 1);
        }
    }
}
```

写到这儿已经差不多了，然而力扣里面原题要求返回int数组，那就再改一下

```java
class Solution {
    StringBuilder sb = new StringBuilder();
    int[] ans;
    int n, count = 0;
    public int[] printNumbers(int n) {
        this.n = n - 1;
        ans = new int[(int) (Math.pow(10, n) - 1)];
        print(0);
        return ans;
    }
    public void print(int index){
        for(int i = 0; i < 10; i++){
            sb.append(i);
            if(index == n){
                for(int j = 0; j < sb.length();){
                    if(sb.charAt(j) == '0') sb.deleteCharAt(j);
                    else break;
                }
                if(sb.length() != 0) {
                    ans[count++] = Integer.parseInt(sb.toString());
                }
            }else print(index + 1);
            if(sb.length() > 0) sb.deleteCharAt(sb.length() - 1);
        }
    }
}
```

over！

## 18

[剑指 Offer 18. 删除链表的节点](https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof/)

![image-20210514091917723](imgs\jz\jz18.png)

```java
class Solution {
    public ListNode deleteNode(ListNode head, int val) {
        if(head == null) return head;
        if(head.val == val) return head.next;
        ListNode temp = head;
        while(temp.next != null){
            if(temp.next.val == val){
                temp.next = temp.next.next;
                break;
            }
            temp = temp.next;
        }
        return head;
    }
}
```

原题是这样的：给定头节点和某个节点，如何在O(1)的时间内删除它

一个很好的方法是，将要删除的节点的下一个节点的值复制给该节点，然后删除下一个节点

注意链表操作时要考虑的各种情况：头为空、要删除的节点在头部、要删除的节点在尾部、只有一个节点等等

```java
class Solution {
    public ListNode deleteNode(ListNode head, ListNode del) {
        if(del.next == null){
            del = null;
        }
        del.val = del.next.val;
        del.next = del.next.next;
        return head;
    }
}
```



## 参考

[1]《剑指offer》