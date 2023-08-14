# 

Go 读取超长的单行非常有病，看这个链接吧：https://aimuke.github.io/go/2020/06/18/go-readline/

**输入：**

建议一旦开始用`sc`，之后所有的就都用`sc.ReadString`读取，如果中间你又用`fmt.Scanf()`这种方式，有可能会出错！

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	// 当用 sc.ReadString('\n') 读取输入的一行数据时，长度其实是输入数据的len+2，这是因为末尾有\r和\n，所以要去除末尾的\r和\n

	// 1. 获取字符串
	str, _ := sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n") // 去掉回车+换行 非常非常重要的一步!!!
	//此时, str 才是真正的输入字符
	fmt.Println(len(str))
	fmt.Println(str + " world")

	// 2. 获取单个数字
	// 方法一
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	num, _ := strconv.Atoi(str)
	fmt.Println(num)
    // 方法二(建议，g)
	var n, m int64
	fmt.Scanf("%v %v\n", &n, &m)
	fmt.Println(n, m)
	var x, y int
	fmt.Scanf("%v %v\n", &x, &y)
	fmt.Println(x, y)

	// 3. 获取数组
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	nums := []int64{}
	strs := strings.Split(str, " ")
	for _, str := range strs {
		num, _ := strconv.ParseInt(str, 10, 64)
		nums = append(nums, num)
	}
	fmt.Println(nums)
}
```

注意：Unix 和 Windows 的行结束符是不同的！Unix 似乎是 `\n`，Windows 是 `\r\n`。

**字符串转换：**

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	str, _ := sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	// 1. string 转 int  int 转 string
	numInt, _ := strconv.Atoi(str)
	numStr := strconv.Itoa(numInt)
	fmt.Println(numStr)
	// 2. string 转 int64  int64 转 string
	numInt64, _ := strconv.ParseInt(str, 10, 64)
	numStr = strconv.FormatInt(numInt64, 10)
	fmt.Println(numStr)
	// 3. string 转 float64  float64 转 string
	numFloat64, _ := strconv.ParseFloat(str, 64)
	numStr = strconv.FormatFloat(numFloat64, 'f', -1, 64)
	fmt.Println(numStr)
}
```

## 美团

1. **滑动窗口**

二维前缀和

> **题目：**小美在玩一项游戏，该游戏的目标是尽可能抓获敌人。敌人的位置将被一个二维坐标(x, y) 所描述。小美有一个全屏技，该技能能一次性将若干敌人一次性捕获。捕获的敌人之间的横坐标的最大差值不能大于A，纵坐标的最大差值不能大于B。
> 	现在给出所有敌人的坐标，你的任务是计算小美一次性最多能使用技能捕获多少敌人。
>
> **输入描述：**
>
> 第一行三个整数N,A,B，表示共有N个敌人，小人的全屏技能的参数A和参数B。接下来N行，每行两个数字x,y，描述一个敌人所在的坐标1 <= N <= 500，1 <= A, B <= 1000，1 <= x, y <= 1000.

```go
func count(coords [][2]int, A, B int) int {
	// 1. 把这些点放到矩阵中
	matrix := make([][]int, 1001)
	for i := 0; i < len(matrix); i++ {
		matrix[i] = make([]int, 1001)
	}
	for _, coord := range coords {
		matrix[coord[0]][coord[1]] = 1
	}
	// 2. 计算前缀和
	for i := 1; i < len(matrix); i++ {
		for j := 1; j < len(matrix[0]); j++ {
			matrix[i][j] += matrix[i-1][j] + matrix[i][j-1] - matrix[i-1][j-1]
		}
	}
	// 3. 计算子矩阵
	ans := 0
	for i := A + 1; i < len(matrix); i++ {
		for j := B + 1; j < len(matrix[i]); j++ {
			ans = max(ans, matrix[i][j]-matrix[i-A-1][j]-matrix[i][j-B-1]+matrix[i-A-1][j-B-1])
		}
	}
	return ans
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func main() {
	coords := [][2]int{{1, 1}, {2, 2}, {3, 3}, {1, 3}, {1, 4}}
	fmt.Println(count(coords, 1, 2))
}
```

2. **滑动窗口**

> **题目：**塔子哥是一名设计师，最近接到了一项重要任务：设计一个新款的生日蜡烛，以庆祝公司成立20周年。为了使蜡烛看起来更加炫目，塔子哥计划在蜡烛上缠绕彩带。他买了一串长度为 *N* 的非常漂亮的彩带，每一厘米的彩带上都是一种色彩，但是当他拿到彩带后才发现，彩带上的颜色太多了，超出了他设计中所需要的颜色种类数量。
>
> 于是，塔子哥决定从彩带上截取一段，使得这段彩带上的颜色种类不超过*K*种。但是，他希望这段彩带尽量长，这样才能在蜡烛上缠绕出更加炫目的效果。为了尽快完成设计，他来找你求助，希望你能帮他设计出一种截取方法，使得截取出来的彩带尽可能长，并且颜色种类不超过*K*种。
>
> **输入描述**
>
> 第一行两个整数�,�*N*,*K*，以空格分开，分别表示彩带有�*N*厘米长，你截取的一段连续的彩带不能超过�*K*种颜色。接下来一行�*N*个整数，每个整数表示一种色彩，相同的整数表示相同的色彩。 1≤�,�≤50001≤*N*,*K*≤5000，彩带上的颜色数字介于[1,2000][1,2000]之间
>
> **输出描述**
>
> 一行，一个整数，表示选取的彩带的最大长度。
>
> **样例1**
>
> **输入**
>
> ```none
> 8 3
> 1 2 3 2 1 4 5 1
> ```
>
> **输出**
>
> ```none
> 5
> ```
>
> **说明：** 最长的一段彩带是[1，2，3，2，1][1，2，3，2，1]，共55厘米。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	var n, k int
	fmt.Fscanln(sc, &n, &k)
	nums := []int{}
	str, _ := sc.ReadString('\n')
	strs := strings.Split(strings.TrimRight(str, "\r\n"), " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		nums = append(nums, num)
	}
	fmt.Println(test(n, k, nums))
}

func test(n, k int, nums []int) int {
	res := 0
	hashMap := map[int]int{}
	count := 0
	left, right := 0, 0
	for right < n {
		for left < right && count > k {
			hashMap[nums[left]]--
			if hashMap[nums[left]] == 0 {
				count--
			}
			left++
		}
		res = max(res, right-left)
		if val, exist := hashMap[nums[right]]; !exist || val == 0 {
			count++
		}
		hashMap[nums[right]]++
		right++
	}
	if count <= k {
		res = max(res, right-left)
	}
	return res
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

3. 贪心算法

> **题目：**现在小美获得了一个字符串。小美想要使得这个字符串是回文串。小美找到了你。你可以将字符串中至多两个位置改为任意小写英文字符’a’-‘z’。
>
> 你的任务是帮助小美在当前制约下，获得字典序最小的回文字符串。数据保证能在题目限制下形成回文字符串。
>
> 注：回文字符串：即一个字符串从前向后和从后向前是完全一致的字符串。
>
> 例如字符串abcba, aaaa, acca都是回文字符串。字符串abcd, acea都不是回文字符串。
>
> **输入描述：**一行，一个字符串。字符串中仅由小写英文字符构成。保证字符串不会是空字符串。字符串长度介于 [1, 100000] 之间。 

```go
func count(str string) string {
	strByte := []byte(str)
	left, right := 0, len(strByte)-1
	pos := [][]int{}
    // 统计不相同的位置对
	for left < right {
		if strByte[left] != strByte[right] {
			pos = append(pos, []int{left, right})
		}
		left++
		right--
	}
	if len(pos) == 0 {
		for i := 0; i < len(strByte); i++ {
			if strByte[i] != 'a' {
				strByte[i], strByte[len(strByte)-1-i] = 'a', 'a'
			}
		}
	} else if len(pos) == 1 {
		if strByte[pos[0][0]] == 'a' || strByte[pos[0][1]] == 'a' {
			if len(strByte)&1 == 1 {
				strByte[len(strByte)/2] = 'a'
			}
		}
		strByte[pos[0][0]], strByte[pos[0][1]] = 'a', 'a'
	} else if len(pos) == 2 {
		mb := min(strByte[pos[0][0]], strByte[pos[0][1]])
		strByte[pos[0][0]], strByte[pos[0][1]] = mb, mb
		mb = min(strByte[pos[1][0]], strByte[pos[1][1]])
		strByte[pos[1][0]], strByte[pos[1][1]] = mb, mb
	}
	return string(strByte)
}

func min(a, b byte) byte {
	if a < b {
		return a
	}
	return b
}
```

4. 贪心算法/动态规划

> **题目：**现在商店里有 N 个物品，每个物品有原价和折扣价小美想要购买商品。小美拥有 X 元，一共 Y 张折扣券。小美需要最大化购买商品的数量，并在所购商品数量尽量多的前提下，尽量减少花费。
>
> 你的任务是帮助小美求出最优情况下的商品购买数量和花费的钱数。
>
> **输入描述：**
>
> 第一行三个整数，以空格分开，分别表示 N，X，Y。
>
> 接下来 N 行，每行两个整数，以空格分开，表示一个的原价和折扣价。1 <= N <= 100，1 <= X <= 5000，1 <= Y <= 50，每个商品原价和折扣价均介于[1,50]之间。
>
> **输出描述：**
>
> 一行，两个整数，以空格分开。第一个数字表示最多买几个商品，第二个数字表示在满足商品尽量多的前提下所花费的最少的钱数。

```go
// 不会写，先写个dfs
func count(oriPrices, disPrices []int, money, k int) (int, int) {
    // 返回 使用的钱，买到物品的数量
	var backTrack func(index, resMoney, resK, cnt int) (int, int)
	backTrack = func(index, resMoney, resK, cnt int) (int, int) {
		if index == len(oriPrices) {
			return money - resMoney, cnt
		}
		// 共三个分支，即三个选择
		// 1. 不买
		minMoney, maxCnt := backTrack(index+1, resMoney, resK, cnt)
		// 2. 原价买
		if resMoney >= oriPrices[index] {
			tempMoney, tempCnt := backTrack(index+1, resMoney-oriPrices[index], resK, cnt+1)
			if tempCnt > maxCnt {
				maxCnt = tempCnt
				minMoney = tempMoney
			}
			if tempCnt == maxCnt {
				minMoney = min(minMoney, tempMoney)
			}
		}
		// 3. 折扣价买
		if resMoney >= disPrices[index] && resK > 0 {
			tempMoney, tempCnt := backTrack(index+1, resMoney-disPrices[index], resK-1, cnt+1)
			if tempCnt > maxCnt {
				maxCnt = tempCnt
				minMoney = tempMoney
			}
			if tempCnt == maxCnt {
				minMoney = min(minMoney, tempMoney)
			}
		}
		return minMoney, maxCnt
	}
	minMoney, maxCnt := backTrack(0, money, k, 0)
	return maxCnt, minMoney
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func main() {
	oriPrices := []int{2, 3, 2, 10, 6, 4, 2, 10, 5, 4}
	disPrices := []int{1, 2, 1, 8, 5, 3, 1, 9, 4, 2}
	fmt.Println(count(oriPrices, disPrices, 30, 3))
}
```

```go
// 动态规划
func count(oriPrices, disPrices []int, X, Y int) (int, int) {
	// 1. dp初始化
	dp := make([][][]int, len(oriPrices)+1)
	cost := make([][][]int, len(oriPrices)+1)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([][]int, X+1)
		cost[i] = make([][]int, X+1)
		for j := 0; j < len(dp[i]); j++ {
			dp[i][j] = make([]int, Y+1)
			cost[i][j] = make([]int, Y+1)
		}
	}
	// 2. 状态转移
	for i := 1; i < len(dp); i++ {
		for j := 1; j < len(dp[i]); j++ {
			for k := 1; k < len(dp[i][j]); k++ {
				dp[i][j][k], cost[i][j][k] = dp[i-1][j][k], cost[i-1][j][k]
				if j >= oriPrices[i-1] {
					if dp[i-1][j-oriPrices[i-1]][k]+1 > dp[i][j][k] {
						dp[i][j][k] = dp[i-1][j-oriPrices[i-1]][k] + 1
						cost[i][j][k] = cost[i-1][j-oriPrices[i-1]][k] + oriPrices[i-1]
					} else if dp[i-1][j-oriPrices[i-1]][k]+1 == dp[i][j][k] {
						cost[i][j][k] = min(cost[i][j][k], cost[i-1][j-oriPrices[i-1]][k]+oriPrices[i-1])
					}
				}
				if j >= disPrices[i-1] && k > 0 {
					if dp[i-1][j-disPrices[i-1]][k-1]+1 > dp[i][j][k] {
						dp[i][j][k] = dp[i-1][j-disPrices[i-1]][k-1] + 1
						cost[i][j][k] = cost[i-1][j-disPrices[i-1]][k-1] + disPrices[i-1]
					} else if dp[i-1][j-disPrices[i-1]][k-1]+1 == dp[i][j][k] {
						cost[i][j][k] = min(cost[i][j][k], cost[i-1][j-disPrices[i-1]][k-1]+disPrices[i-1])
					}
				}
			}
		}
	}
	// 3. 取结果
	resNum, resPrice := dp[len(dp)-1][X][Y], math.MaxInt
	for i := 1; i < len(dp); i++ {
		for j := 1; j < len(dp[i]); j++ {
			for k := 1; k < len(dp[i][j]); k++ {
				if dp[i][j][k] == resNum {
					resPrice = min(resPrice, cost[i][j][k])
				}
			}
		}
	}
	return resNum, resPrice
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func main() {
	oriPrices := []int{2, 3, 2, 10, 6, 4, 2, 10, 5, 4}
	disPrices := []int{1, 2, 1, 8, 5, 3, 1, 9, 4, 2}
	fmt.Println(count(oriPrices, disPrices, 30, 3))
}
```

5. 图

> **题目：**现在有若干节点。每个节点上有能量塔。所有节点构成一棵树。 某个节点 u 可以为和 u 距离不超过给定值的节点各提供一点能量。
>
> 此处距离的定义为两个节点之间经过的边的数量。特别的，节点 u 到本身的距离为零。 
>
> 现在给出每个节点上的能量塔可以为多远的距离内的点提供能量。 
>
> 小美想要探究每个节点上的能量值具体是多少。你的任务是帮助小美计算得到，并依次输出。
>
> **输入描述：**第一行一个整数 N，表示节点的数量。
>
> 接下来一行 N 个以空格分开的整数，依次表示节点1，节点2，…，节点 N 的能量塔所能提供能量的最远距离。
>
> 接下来 N-1 行，每行两个整数，表示两个点之间有一条边。
>
> 1 ≤ N ≤ 500，节点上能量塔所能到达的最远距离距离不会大于 500.

```go
func count(edges [][]int, distances []int, n int) []int {
	trees := buildTree(edges, n)
	sums := make([]int, n)
    // 从结点出发，能在dist之内到达的节点均+1
	var dfs func(dist int, node int)
	dfs = func(dist int, node int) {
		if dist < 0 {
			return
		}
		sums[node-1]++
		for _, next := range trees[node] {
			dfs(dist-1, next)
		}
	}
	for i := 1; i <= n; i++ {
		dfs(distances[i-1], i)
	}
	return sums
}

func buildTree(edges [][]int, n int) [][]int {
	trees := make([][]int, n+1)
	for _, edge := range edges {
		trees[edge[0]] = append(trees[edge[0]], edge[1])
		trees[edge[1]] = append(trees[edge[1]], edge[0])
	}
	return trees
}

func main() {
	edges := [][]int{{1, 2}, {2, 3}}
	distances := []int{1, 1, 1}
	fmt.Println(count(edges, distances, 3))
}
```

6. 双指针

> **题目：**
>
> 小美是一个火车迷。最近她在观察家附近火车站的火车驶入和驶出情况，发现火车驶入和驶出的顺序并不一致。经过小美调查发现，原来这个火车站里面有一个类似于栈的结构，如下图所示：
>
> <img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230326090359784.png" alt="image-20230326090359784" style="zoom:33%;" />
>
> 例如可能1号火车驶入了火车站中的休息区s，在驶出之前2号火车驶入了。那么在这种情况下，1号火车需要等待2号火车倒车出去后才能出去（显然被后面驶入的2号火车挡住了，这个休息区s只有一个出入口）。
>
> 出于好奇，小美统计了近些天的火车驶入驶出情况，开始统计和结束统计时休息区s中均是空的。由于中途疏忽，小美觉得自己好像弄错了几个驶入驶出顺序，想请你帮她验证一下。值得注意的是，小美虽然可能弄错了顺序，但对火车的记录是不重不漏的。
>
> 形式化地来形容休息区s，我们视其为一个容量无限大的空间，假设两列火车 i 和 j 同时处于休息区s中，驶入时刻Tin满足Tin(i)<Tin(j)，则驶出时间Tout必定满足Tout(i)>Tout(j)，即，先进后出。
>
> **输入描述**
>
> 第一行一个整数T表示数据组数。
>
> 对每组测试而言：
>
> 第一行一个整数n，表示观察到的火车数量。
>
> 第二行n个整数x1,x2,...,xn，表示小美记录的火车驶入休息区s的顺序。
>
> 第三行n个整数y1,y2,...,yn，表示小美记录的火车驶出休息区s的顺序。
>
> 1≤T≤10,1≤n≤50000,1≤xi,yi≤n, 且{xn} 、{yn} 均为{1,2,3,...,n}的一个排列，即1~n这n个数在其中不重不漏恰好出现一次。
>
> **输出描述**
>
> 对每组数据输出一行：如果小美记录的驶入和驶出顺序无法被满足则输出No，否则输出Yes。
>
> **样例输入**
>
> 3
>
> 3
>
> 1 2 3
>
> 1 2 3
>
> 3
>
> 1 2 3
>
> 3 2 1
>
> 3
>
> 1 2 3
>
> 3 1 2
>
> **样例输出**
>
> Yes
>
> Yes
>
> No

```go
func main() {
	fmt.Println(isValid("123", "123"))
	fmt.Println(isValid("123", "321"))
	fmt.Println(isValid("123", "312"))
}

func isValid(ori string, dst string) bool {
	left, right := 0, 0
	stack := []byte{}
	for left < len(ori) {
		if ori[left] == dst[right] {
			left++
			right++
		} else {
			stack = append(stack, ori[left])
			left++
		}
	}
	for i := len(stack) - 1; i >= 0; i-- {
		if stack[i] != dst[right] {
			return false
		}
		right++
	}
	return true
}
```

7. 前缀和+二分查找

> **题目：**
>
> 小美明天要去春游了。她非常喜欢吃巧克力，希望能够带尽可能多的巧克力在春游的路上吃。她现在有n个巧克力，很巧的是她所有的巧克力都是厚度一样的正方形的巧克力板，这n个巧克力板的边长分别为a_1,a_2,...,a_n。因为都是厚度一致的正方形巧克力板，我们认为第 i 个巧克力的重量$a_i^2$。小美现在准备挑选一个合适大小的包来装尽可能多的巧克力板，她十分需要你的帮助来在明天之前准备完成，请你帮帮她。
>
> **输入描述**
>
> 第一行两个整数n和m，表示小美的巧克力数量和小美的询问数量。
>
> 第二行n个整数a1,a2,...,an，表示n块正方形巧克力板的边长。注意你不能将巧克力板进行拆分。
>
> 第三行m个整数q1,q2,...,qm，第 i 个整数qi表示询问：如果小美选择一个能装qi重量的包，最多能装多少块巧克力板？（不考虑体积影响，我们认为只要质量满足要求，巧克力板总能塞进包里）
>
> 1≤n,m≤50000,1≤ai≤104,1≤qi≤1018
>
> **输出描述**
>
> 输出一行m个整数，分别表示每次询问的答案。
>
> **样例输入**
>
> 5 5
>
> 1 2 2 4 5
>
> 1 3 7 9 15
>
> **样例输出**
>
> 1 1 2 3 3
>
> **提示**
>
> 包最大重量为1，能装1^2^
>
> 包最大重量为3，也最多只能装1^2^重量（如果添加2^2^就超载了）
>
> 包最大重量为7，能装1^2^+2^2^
>
> 包最大重量为9，能装 1^2^+2^2^+2^2^（因为有两块巧克力板边长都为2）
>
> 包最大重量为15，也最多能装 1^2^+2^2^+2^2^（如果添加4^2^就超载了）

```go
func main() {
	nums := []int{1, 2, 2, 4, 5}
	ch := NewCh(nums)
	fmt.Println(ch.count(1))
	fmt.Println(ch.count(3))
	fmt.Println(ch.count(7))
	fmt.Println(ch.count(9))
	fmt.Println(ch.count(15))
}

type Chocolate struct {
	prefix []int
}

func NewCh(nums []int) Chocolate {
	sort.Ints(nums)
	prefix := []int{}
	for index, num := range nums {
		if index == 0 {
			prefix = append(prefix, num*num)
		} else {
			prefix = append(prefix, prefix[index-1]+num*num)
		}
	}
	return Chocolate{prefix: prefix}
}

func (ch Chocolate) count(pac int) int {
	index := sort.Search(len(ch.prefix), func(i int) bool {
		return ch.prefix[i] > pac
	})
	return index
}
```

8. 

> **题目：**
>
> 小美特别爱吃糖果。小美家楼下正好有一个糖果专卖店，每天供应不同种类的糖果。小美预先拿到了糖果专卖店接下来n天的进货计划表，并针对每天的糖果种类标注好了对小美而言的美味值。小美当然想每天都能去买糖果吃，不过由于零花钱限制（小美零花钱并不多！）以及健康考虑，小美决定原则上如果今天吃了，那么明天就不能吃。但小美认为凡事都有例外，所以她给了自己k次机会，在昨天已经吃了糖果的情况下，今天仍然连续吃糖果！**简单来说**，小美每天只能吃一次糖果，原则上如果昨天吃了糖果那么今天就不能吃，但有最多k次机会打破这一原则。
>
> 小美不想浪费每次吃糖果的机会，所以请你帮帮她规划一下她的吃糖果计划，使得她能吃到的糖果美味值最大。
>
> **输入描述**
>
> 第一行两个整数n和k，表示拿到的进货计划表的天数和最多打破原则的次数。
>
> 第二行n个整数a1,a2,...,an，其中ai表示接下来第 i 天糖果专卖店的糖果的美味值。
>
> 1≤n≤2000,1≤k≤1000,1≤ai≤10000
>
> **输出描述**
>
> 输出一行一个数，表示小美能吃到的糖果美味值之和最大值。
>
> **样例输入**
>
> 7 1
>
> 1 2 3 4 5 6 7
>
> **样例输出**
>
> 19
>
> **提示**
>
> 最优的方案是选择选择第2、4、6天吃糖果，并在第7天打破一次原则也吃糖果（因为第6天已经吃过，原则上不能继续吃，需要使用一次打破原则的机会）。

```go
func main() {
	fmt.Println(maxValue([]int{1, 2, 3, 4, 5, 6, 7}, 1))
}

func maxValue(nums []int, k int) int {
	dp := make([][]int, len(nums)+1)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([]int, k+1)
	}
	// 初始化
	for j := 0; j <= k; j++ {
		dp[1][j] = nums[0]
	}
	for i := 2; i < len(dp); i++ {
		for j := 0; j <= k; j++ {
			if j == 0 {
				dp[i][j] = max(dp[i-1][j], dp[i-2][j]+nums[i-1])
			} else {
				dp[i][j] = max(dp[i-1][j], nums[i-1]+max(dp[i-2][j], dp[i-1][j-1]))
			}
		}
	}
	return dp[len(nums)][k]
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

9. 2023.04.01-第一题-整理

简单排序题：http://101.43.147.120/p/P1137

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"sort"
	"strconv"
	"strings"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	str, _ := sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	_, _ = strconv.Atoi(str)

	nums := []int64{}
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	strs := strings.Split(str, " ")
	for _, str := range strs {
		num, _ := strconv.ParseInt(str, 10, 64)
		nums = append(nums, num)
	}
	fmt.Println(test9(nums))
}

func test9(nums []int64) int64 {
	var res int64
	sort.Slice(nums, func(i, j int) bool {
		return nums[i] < nums[j]
	})
	for i := 1; i < len(nums); i++ {
		res += nums[i] - nums[i-1]
	}
	return res
}
```

10. 2023.04.01-第四题-倒水魔法

> **题目：**塔子哥是一个年轻的魔法学徒，他一直在努力学习各种魔法技能。他听说倒水魔法是一种非常强大的魔法，但也是一种非常难掌握的魔法。他早早地来到了魔法训练室，希望能够掌握这种魔法。
>
> 魔法训练室里有 *n* 个神奇的杯子，有着不同的大小，假设第 *i* 个杯子已满，向其倒水，多余的水会正正好好流向第 i*+1 个杯子(如果 *i*=*n* 时没有下一个杯子，不会有杯子接住此时多余的水而溢出到魔法训练室的水池）。
>
> 这些杯子有着初始固定的水量，每次练习后杯子里的水都会还原到最初状态。每次练习时，魔法黑板会告诉塔子哥需要将哪一人杯子倒满水。因为每个杯子的材质和形状有所不同所以对其释放倒水魔法需要消耗的魔法值不同。塔子哥想尽可能多练习，所以需要最小化每次消耗的魔法值的总量。
>
> **输入描述**
>
> 第一行一个整数 *n* ，表示杯了数量。
>
> 第二行 *n* 个整数 1,2,...,x*1,*x*2,...,*x*n* ，依次表示第 *i* 个杯子能容纳水的量（单位为毫升）。
>
> 第三行 *n* 个整数 1,2,...,*y*1,*y*2,...,*y**n* ，依次表示第 *i* 个杯子初始有的水量（单位为毫升）。
>
> 第四行 *n* 个整数 1,2,...,*z*1,*z*2,...,*z**n* ，依次表示对第 *i* 个杯子每添加1毫升水需要消耗的法力值。
>
> 第五行一个整数 *m* ，表示练习的数量。
>
> 第六行 *m* 个整数 1,2,…,*q*1,*q*2,…,*q**n* ，依次表示第 i* 次练习时需要将第 *q**i* 个杯子倒满。(每次练习过后，杯子里的水量都会还原为初始状态，不会影响到下一次练习）
>
> 1≤�,�≤3000，1≤��≤��≤109，1≤��≤300，1≤��≤�1≤*n*,*m*≤3000，1≤*y**i*≤*x**x*≤109，1≤*z**i*≤300，1≤*q**i*≤*n*
>
> **输出描述**
>
> 输出第一行 *m* 个数，依次表示每次训练时需要消耗的最小法力值。
>
> 如果杯子初始本身就是满的，则需要消耗的法力值为 00 。
>
> **样例**
>
> **输入**
>
> ```none
> 3
> 1 2 3
> 1 1 2
> 1 2 5
> 2
> 3 1
> ```
>
> **输出**
>
> ```none
> 2 0
> ```

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

func main() {
	// 杯数
	var n int
	fmt.Scanf("%v\n", &n)
	sc := bufio.NewReader(os.Stdin)
	// 容量
	caps := []int{}
	str, _ := sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	strs := strings.Split(str, " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		caps = append(caps, num)
	}
	// 初始水量
	begins := []int{}
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	strs = strings.Split(str, " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		begins = append(begins, num)
	}
	// 法力值/ml
	prices := []int{}
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	strs = strings.Split(str, " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		prices = append(prices, num)
	}
	// 练习数
	//var m int
	//fmt.Scanf("%v\n", &m)	// 本地不会报错，但是系统内会报错，很。。。睿智
	str, _ = sc.ReadString('\n')	// 只是读取一下，根本没用到，所以就

	//  倒满哪个杯子
	str, _ = sc.ReadString('\n')
	str = strings.TrimRight(str, "\r\n")
	strs = strings.Split(str, " ")
	for _, str := range strs {
		target, _ := strconv.Atoi(str)
		fmt.Printf("%v ", test10(target, caps, begins, prices))
	}
}

func test10(target int, caps []int, begins []int, prices []int) int {
	// 保持原样, 创建副本
	tmp := make([]int, len(begins))
	copy(tmp, begins)
	suffix := 0
	for i := target - 1; i >= 0; i-- {
		suffix += caps[i] - begins[i]
	}
	res := (caps[target-1] - begins[target-1]) * prices[target-1]
	for i := 0; i < target; i++ {
		res = min(res, suffix*prices[i])
		suffix -= caps[i] - begins[i]
	}
	begins = tmp
	return res
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

#### 2023.04.01-第五题-染色の树

> **题目：**塔子哥是一名计算机科学家，他正在研究一种新的数据结构——树。树是一种无向无环联通图，它由若干个节点和若干条边组成。
>
> 每个节点都可以有零个或多个子节点，而每条边都连接两个节点。塔子哥现在有一棵树，树上的每个节点都有自己的价值。价值的计算规则如下所示：
>
> 1. 若某节点 *N* 没有儿子节点，那么节点 *N* 价值为 1 ；
> 2. 若某节点 *N* 有两个儿子节点，那么节点 *N* 价值为两个儿子节点的价值之和，或者价值之按位异或。这取决于节点 *N* 的颜色，若 *N* 的颜色为红色，那么节点 *N* 价值为两个儿子节点的价值之和；若 *N* 的颜色为绿色，那么节点 *N* 价值为两个儿子节点的价值之按位异或。
>
> **3. 若某节点 *N* 只有一个儿子节点，则它等于这个唯一的儿子的权值**
>
> 按位运算就是基于整数的二进制表示进行的运算。按位异或用符号 `⊕` 表示，两个对应位不同时为 1 ，否则为 0 。
>
> 如：
>
> 5=(101)_2_
>
> 6=(110)_2_
>
> 5⊕6=3，即(101)_2_⊕(110)_2_=(011)_2_
>
> **输入描述：**
>
> 第一行一个正整数 *n* 表示节点个数。
>
> 第二行 n*−1 个正整数 p*[*i*] （ 2≤*i*≤*n* ）表示第 *i* 个节点的父亲。 1 号节点是根节点。
>
> 第三行 *n* 个整数 c*[*i*] （ 1≤*i*≤*n )，当 c*[*i*]=1 时表示第 *i* 个节点是红色，c*[*i*]=2 则表示绿色。
>
> 数据保证形成合法的树。
>
> 对于所有的数据，*n*≤50000
>
> **输出描述：**
>
> 输出一行一个整数表示根节点的值。
>
> **输入**
>
> ```none
> 3
> 1 1
> 2 2 2
> ```
>
> **输出**
>
> ```none
> 0
> ```

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	p := []int{0, 1}
	str, _ := sc.ReadString('\n')
	strs := strings.Split(strings.TrimRight(str, "\r\n"), " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		p = append(p, num)
	}
	c := []int{-1}
	for i := 0; i < n; i++ {
		var num int
		fmt.Fscan(sc, &num)
		c = append(c, num)
	}
	fmt.Println(test15(n, p, c))
}

func test15(n int, p, c []int) int {
	graph := map[int][]int{}
	for i := 2; i < len(p); i++ {
		graph[p[i]] = append(graph[p[i]], i)
	}
	var sufOrder func(curNode int) int
	sufOrder = func(curNode int) int {
		val := 0
		if len(graph[curNode]) == 0 {
			val = 1
		} else if len(graph[curNode]) == 1 {
			val = sufOrder(graph[curNode][0])
		} else if len(graph[curNode]) == 2 {
			left, right := sufOrder(graph[curNode][0]), sufOrder(graph[curNode][1])
			if c[curNode] == 1 {
				val = left + right
			} else {
				val = left ^ right
			}
		}
		return val
	}

	return sufOrder(1)
}
```

#### 2023.3.18-第四题-提瓦特商店

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n, x, y int
	fmt.Fscanln(sc, &n, &x, &y)
	prices := [][]int{}
	for i := 0; i < n; i++ {
		var ori, dis int
		fmt.Fscanln(sc, &ori, &dis)
		prices = append(prices, []int{ori, dis})
	}
	test8(x, y, prices)
}

func test8(x, y int, prices [][]int) {
	dp := make([][]int, x+1)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([]int, y+1)
	}
	for _, price := range prices {
		for i := x; i >= price[1]; i-- {
			for j := y; j >= 0; j-- {
				if i >= price[0] {
					dp[i][j] = max(dp[i][j], dp[i-price[0]][j]+1)
				}
				if i >= price[1] && j >= 1 {
					dp[i][j] = max(dp[i][j], dp[i-price[1]][j-1]+1)
				}
			}
		}
	}
	count, cost := 0, 0
	for i := x; i >= 0; i-- {
		if count < dp[i][y] {
			count = dp[i][y]
			cost = i
		} else if count == dp[i][y] {
			cost = min(cost, i)
		}
	}
	fmt.Println(count, cost)
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

#### 2023.04.08-第二题-必经之路

> **题目：**塔子哥的班主任最近组织了一次户外拓展活动，让班里的同学们一起去爬山。在路上，塔子哥看到了一棵漂亮的树，他对这棵树产生了浓厚的兴趣，开始观察并记录这棵树的一些特征。
>
> 塔子哥发现这棵树有 *n* 个节点，其中有一条边被特别标记了出来。他开始思考这条特殊的边在树上起到了什么样的作用，于是他想知道，经过这条选定边的所有树上简单路径中，最长的那条路径有多长，以便更好地理解这棵树的结构。
>
> 一条简单的路径的长度指这条简单路径上的边的个数。
>
> **输入描述**
>
> 第一行一个整数 *n* ，表示树的节点个数。
>
> 第二行 *n*−1 个整数，第 *i* 个数 *pi* 表示节点 *i*+1 和 *pi* 之间有一条边相连。
>
> 第三行两个整数 *x* ， *y* ，表示这条选定的边。保证这条边一定是树上的一条边。
>
> 对于全部数据， 2≤*n*≤10^5^，1≤*pi*≤*n* ，1≤*x*,*y*≤*n* ， *x*!=*y* 。
>
> 保证输入数据正确描述一棵树，并且(*x*,*y*) 是树上的条边。
>
> **输出描述**
>
> 输出一行，一个整数，表示所有经过选定边的树上简单路径中，最长的那条的长度。
>
> **样例**
>
> **输入**
>
> ```none
> 7
> 1 2 3 4 5 3
> 3 7
> ```
>
> **输出**
>
> ```none
> 4
> ```

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	edges := [][]int{}
	for i := 1; i < n; i++ {
		var pi int
		fmt.Fscan(sc, &pi)
		edges = append(edges, []int{i + 1, pi})
	}
	fmt.Fscanln(sc)
	var x, y int
	fmt.Fscanln(sc, &x, &y)
	test14(n, edges, x, y)
}

func test14(n int, edges [][]int, x, y int) {
	// 建图
	graph := map[int][]int{}
	for _, edge := range edges {
		graph[edge[0]] = append(graph[edge[0]], edge[1])
		graph[edge[1]] = append(graph[edge[1]], edge[0])
	}
	// 分别从 x, y 向两边延伸, 分别得到最长的, 相加 + 1
	visitedMap := map[int]bool{}
	var dfs func(node int) int
	dfs = func(node int) int {
		if len(graph[node]) == 0 {
			return 1
		}
		res := 0
		for _, next := range graph[node] {
			if !visitedMap[next] {
				visitedMap[next] = true
				res = max(res, 1+dfs(next))
				visitedMap[next] = false
			}
		}
		return res
	}
	visitedMap[x] = true	// 这两步挺有意思的
	visitedMap[y] = true
	resX, resY := dfs(x), dfs(y)
	fmt.Println(resX + resY + 1)
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

#### 2023.04.08-第三题-玩具打包

> **题目：**塔子哥开的玩具店生意越来越好，每天都有很多客人前来选购玩具。有一天，他接到了一个大单，客户想购买 *n* 个玩具，并且要求打包成多个玩具袋。塔子哥精心为客户挑选了 *n* 个玩具，并且将它们编号为 1,2,…,*n*。
>
> 然而，塔子哥发现这个订购单还有一个要求：每个玩具袋最多只能装 *m* 个玩具，并且同一个玩具袋里装的玩具编号必须是连续的。玩具袋的成本与容积成线性关系。
>
> 为了解决这个问题，他决定采用样本中点估计的方法来计算玩具袋的容积。具体来说，如果一个玩具袋中装的最大的玩具容积是 *u*，最小的是 *v*，那么这个玩具袋的成本就是 *k*×*f**l**oor*((*u*+*v*)/2)+*s*，其中 *k* 是玩具袋中装入玩具的个数，*s* 是一个常数， *f**l**oor*(*x*) 是下取整函数，比如 *f**l**oor*(3.8)=3 ，*f**l**oor*(2)=2
>
> 客户并没有规定玩具袋的数量，但是希望玩具袋的成本越小越好，毕竟买玩具就很贵了。请求出塔子哥打包这 *n* 个玩具所用的最小花费。
>
> #### 输入描述
>
> 第一行三个正整数 *n*,*m*,*s* 。意义如题面所示
>
> 第二行 *n* 个正整数 1,2,...,*a*1,*a*2,...,*an* ，表示每个玩具的体积。
>
> 对于全部数据, 1≤*n*≤104 ，1≤*m*≤103 ，*m*≤*n* ，1≤*a**i*,*s*≤104 。
>
> #### 输出描述
>
> 输出一个整数，表示打包这 *n* 个玩具玩具袋的最小成本。
>
> #### 样例
>
> **输入**
>
> ```none
> 6 4 3
> 1 4 5 1 4 1
> ```
>
> **输出**
>
> ```none
> 21
> ```
>
> **样例解释**
>
> 前三个玩具装成一个玩具袋，后三个玩具装成一个玩具袋。

动态规划：

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n, m, s int
	fmt.Fscanln(sc, &n, &m, &s)
	nums := []int{}
	for i := 0; i < n; i++ {
		var ai int
		fmt.Fscan(sc, &ai)
		nums = append(nums, ai)
	}
	fmt.Fscanln(sc)
	test15(n, m, s, nums)
}

func test15(n, m, s int, nums []int) {
	dp := make([]int, n+1)
	for i := 1; i < len(dp); i++ {
		dp[i] = math.MaxInt
	}
	for i := 1; i < len(dp); i++ {
		u, v := nums[i-1], nums[i-1]
		for j := i; j >= 1 && j > i-m; j-- {	// 注意遍历范围，并非是 n^2 的时间复杂度
			u, v = max(u, nums[j-1]), min(v, nums[j-1])
			dp[i] = min(dp[i], dp[j-1]+(i-j+1)*((u+v)/2)+s)
		}
	}
	fmt.Println(dp[n])
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

#### 2023.04.08-第五题-RGB树

> **题目：**塔子哥是一位著名的冒险家，他经常在各种森林里探险。今天，他来到了道成林，这是一片美丽而神秘的森林。在探险途中，他遇到了一棵 *n* 个节点的树，树上每个节点都被涂上了红、绿、蓝三种颜色之一。
>
> 塔子哥发现，如果这棵树同时存在一个红色节点、一个绿色节点和一个蓝色节点，那么我们就称这棵树是多彩的。很幸运，他发现这棵树最初就是多彩的。
>
> 但是，在探险的过程中，塔子哥发现这棵树上有一条边非常重要，如果砍掉这条边，就可以把这棵树分成两个部分。他想知道，有多少种砍法可以砍掉这条边，使得砍完之后形成的两棵树都是多彩的。
>
> **输入描述**
>
> 第一行个整数 *n* ，表示节点个数
>
> 第二行 *n*−1 个整数 2，3,…,*p*2，*p*3,…,*pn*，*pi* 表示树上 *i* 和 *p* 两点之间有一条边。保证给出的定是一棵树。
>
> 第三行一个长度为 *n* 的字符串，第 *i* 个字符表示第 *i* 个节点的初始颜色。其中 *R* 表示红色，*G* 表示绿色，*B* 表示蓝色。
>
> 保证字符串只由这三种大写字母构成对于全部数据， 3≤*n*≤105 。
>
> **输出描述**
>
> 输出一行，一个正整数，表示答案。
>
> **样例**
>
> **输入**
>
> ```none
> 7
> 1 2 3 1 5 5
> GBGRRBB
> ```
>
> **输出**
>
> ```none
> 1
> ```

**树形 DP**：

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	ps := []int{0, 1}
	for i := 2; i <= n; i++ {
		var p int
		fmt.Fscan(sc, &p)
		ps = append(ps, p)
	}
	fmt.Fscanln(sc)
	var str string
	fmt.Fscanln(sc, &str)
	str = "#" + str
	test17(n, ps, str)
}

func test17(n int, ps []int, str string) {
	dp := make([][3]int, n+1) // [R, G, B]
	graph := make([][]int, n+1)
	for i := 2; i < len(ps); i++ {
		graph[ps[i]] = append(graph[ps[i]], i)
	}
	// 得到每个子树的色彩情况
	var dfsGetDp func(cur int)
	dfsGetDp = func(cur int) {
		if str[cur] == 'R' {
			dp[cur][0]++
		} else if str[cur] == 'G' {
			dp[cur][1]++
		} else if str[cur] == 'B' {
			dp[cur][2]++
		}

		for _, next := range graph[cur] {
			dfsGetDp(next)
			dp[cur][0] += dp[next][0]
			dp[cur][1] += dp[next][1]
			dp[cur][2] += dp[next][2]
		}
		return
	}
	// 寻找边
	res := 0
	var dfsGetEdge func(cur int)
	dfsGetEdge = func(cur int) {
		if isValid(dp[1], dp[cur]) {
			res++
		}
		for _, next := range graph[cur] {
			dfsGetEdge(next)
		}
	}
	dfsGetDp(1)
	dfsGetEdge(1)
	fmt.Println(res)
}

func isValid(sum, cur [3]int) bool {
	for i := 0; i < 3; i++ {
		if cur[i] == 0 || sum[i]-cur[i] == 0 {
			return false
		}
	}
	return true
}
```



## 米哈游

1. 图

> **题目：**米小游拿到了一个矩阵，矩阵上有一格有一个颜色，为红色( `R` )。绿色( `G` )和蓝色( `B` )这三种颜色的一种。
>
> 然而米小游是蓝绿色盲，她无法分游蓝色和绿色，所以在米小游眼里看来，这个矩阵只有两种颜色，因为蓝色和绿色在她眼里是一种颜色。
>
> 米小游会把相同颜色的部分看成是一个连通块。请注意，这里的连通划是上下左右四连通的。
>
> 由于色盲的原因，米小游自己看到的连通块数量可能比真实的连通块数量少。
>
> 你可以帮米小游计算连通块少了多少吗？
>
> **输入描述：**
>
> 第一行输入两个正整数 *n* 和 *m* ，代表矩阵的行数和列数。
>
> 接下来的 *n* 行，每行输入一个长度为 *m* 的，仅包含 `R` 、`G` 、`B` 三种颜色的字符串，代表米小游拿到的矩阵。
>
> 1 ≤ n, m ≤ 1000
>
> **输出描述：**
>
> 一个整数，代表米小游视角里比真实情况少的连通块数量。
>
> **输入**
>
> ```
> 2 6
> RRGGBB
> RGBGRR
> ```
>
> **输出**
>
> ```
> 3
> ```

**BFS：**不知道是否会超时

```go
package main

import (
	"fmt"
)

func main() {
	//var n, m int
	//fmt.Scanf("%v %v\n", &n, &m)
	//sc := bufio.NewScanner(os.Stdin)
	//matrix := []string{}
	//if sc.Scan() {
	//	str := sc.Text()
	//	matrix = append(matrix, str)
	//}
	matrix := []string{"RRGGBB", "RGBGRR"}
	fmt.Println(count(matrix))
}

func count(matrix []string) int {
	sig, both := 0, 0
	hashMap := map[byte]int{'G': 0, 'B': 1}
	visitedMap := make([][]bool, len(matrix))
	for i := 0; i < len(visitedMap); i++ {
		visitedMap[i] = make([]bool, len(matrix[0]))
	}
	for i := 0; i < len(matrix); i++ {
		for j := 0; j < len(matrix[i]); j++ {
			if !visitedMap[i][j] && (matrix[i][j] == 'G' || matrix[i][j] == 'B') {
				sig++
				bfs(matrix, hashMap[matrix[i][j]], visitedMap, i, j)
			}
		}
	}
	for i := 0; i < len(visitedMap); i++ {
		for j := 0; j < len(visitedMap[i]); j++ {
			visitedMap[i][j] = false
		}
	}
	for i := 0; i < len(matrix); i++ {
		for j := 0; j < len(matrix[i]); j++ {
			if !visitedMap[i][j] && (matrix[i][j] == 'G' || matrix[i][j] == 'B') {
				both++
				bfs(matrix, 2, visitedMap, i, j)
			}
		}
	}
	return sig - both
}

func bfs(matrix []string, k int, visitedMap [][]bool, i, j int) {
	// 0, 1, 2
	var color byte
	if k == 0 {
		color = 'G'
	} else if k == 1 {
		color = 'B'
	} else if k == 2 {
		color = 'A'
	}
	queue := [][]int{{i, j}}
	visitedMap[i][j] = true
	dirs := [][]int{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}
	for len(queue) > 0 {
		curNode := queue[0]
		queue = queue[1:]
		for _, dir := range dirs {
			newX, newY := curNode[0]+dir[0], curNode[1]+dir[1]
			if 0 <= newX && newX < len(matrix) && 0 <= newY && newY < len(matrix[0]) && !visitedMap[newX][newY] {
				if color != 'A' && matrix[newX][newY] == color {
					queue = append(queue, []int{newX, newY})
					visitedMap[newX][newY] = true
				}
				if color == 'A' && (matrix[newX][newY] == 'G' || matrix[newX][newY] == 'B') {
					queue = append(queue, []int{newX, newY})
					visitedMap[newX][newY] = true
				}
			}
		}
	}
}
```

DFS：

```go
func count(matrix []string) int {
	sig, both := 0, 0
	hashMap := map[byte]int{'G': 0, 'B': 1}
	visitedMap := make([][]bool, len(matrix))
	for i := 0; i < len(visitedMap); i++ {
		visitedMap[i] = make([]bool, len(matrix[0]))
	}
	for i := 0; i < len(matrix); i++ {
		for j := 0; j < len(matrix[i]); j++ {
			if !visitedMap[i][j] && (matrix[i][j] == 'G' || matrix[i][j] == 'B') {
				sig++
				dfs(matrix, hashMap[matrix[i][j]], visitedMap, i, j)
			}
		}
	}
	for i := 0; i < len(visitedMap); i++ {
		for j := 0; j < len(visitedMap[i]); j++ {
			visitedMap[i][j] = false
		}
	}
	for i := 0; i < len(matrix); i++ {
		for j := 0; j < len(matrix[i]); j++ {
			if !visitedMap[i][j] && (matrix[i][j] == 'G' || matrix[i][j] == 'B') {
				both++
				dfs(matrix, 2, visitedMap, i, j)
			}
		}
	}
	return sig - both
}

func dfs(matrix []string, k int, visitedMap [][]bool, i, j int) {
	var color byte
	if k == 0 {
		color = 'G'
	} else if k == 1 {
		color = 'B'
	} else {
		color = 'A'
	}
	dirs := [][]int{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}
	visitedMap[i][j] = true
	for _, dir := range dirs {
		newX, newY := i+dir[0], j+dir[1]
		if 0 <= newX && newX < len(matrix) && 0 <= newY && newY < len(matrix[0]) && !visitedMap[newX][newY] {
			if color != 'A' && matrix[newX][newY] == color {
				dfs(matrix, k, visitedMap, newX, newY)
			}
			if color == 'A' && (matrix[newX][newY] == 'G' || matrix[newX][newY] == 'B') {
				dfs(matrix, k, visitedMap, newX, newY)
			}
		}
	}
}
```

2. 

> **题目：**米小游拿到了一个字符串 `s` 。她可以进行任意次以下两种操作：
>
> - 删除 `s` 的一个 `"mhy"` 子序列。
> - 添加一个 `"mhy"` 子序列在 `s` 上。
>
> 例如，给定 `s` 为 `"mhbdy"` ，米小游进行一次操作后可以使 `s` 变成 `"bd"` ，或者变成 `"mhmbhdyy"` 。
>
> 米小游想知道，经过若干次操作后 `s` 是否可以变成 `t` ？
>
> 注：子序列在原串中的顺序也是从左到右，但可以不连续。
>
> **输入描述：**
>
> 第一行输入一个正整数 *q* ，代表询问的次数。
>
> 接下来每两行为一次询问：每行均为一个字符串，分别代表 `s` 和 `t` 。
>
> 1≤�≤1031≤*q*≤103
>
> 字符串的长度均不超过 103103 。
>
> **输出描述：**
>
> 输出 `q` 行，每行输入一行答案。若可以使 `s` 变成 `t` ，则输出 `"Yes"` 。否则输出 `"No"` 。
>
> **输入**
>
> ```
> 3
> mhbdy
> bd
> mhbdy
> mhmbhdyy
> mhy
> abc
> ```
>
> **输出**
>
> ```
> Yes
> Yes
> No
> ```

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

// 这题只能过 20% ！！！
func main() {
	//var row, col int
	//fmt.Scanf("%v %v\n", &row, &col)
	var q int
	fmt.Scanf("%v\n", &q)
	sc := bufio.NewScanner(os.Stdin)
	var s, t string
	for i := 0; i < 2*q; i++ {
		if sc.Scan() {
			if i&1 == 0 {
				s = sc.Text()
			}
			if i&1 == 1 {
				t = sc.Text()
			}
		}
		if i&1 == 1 {
			if isValid(s, t) {
				fmt.Println("Yes")
			} else {
				fmt.Println("No")
			}
		}
	}
}

func isValid(s, t string) bool {
	// 统一用 添加
	if len(s) > len(t) {
		s, t = t, s
	}
	count := []byte{}
	index1, index2 := 0, 0
	for index1 < len(s) && index2 < len(t) {
		for index2 < len(t) && s[index1] != t[index2] {
			count = append(count, t[index2])
			index2++
		}
		index1++
		index2++
	}
	//if index1 < len(s) {
	//	return false
	//}
	count = append(count, []byte(t[index2:])...)
	if len(count)%3 != 0 {
		return false
	}
	m, h, y := 0, 0, 0
	for i := 0; i < len(count); i++ {
		if count[i] == 'm' {
			m++
		} else if count[i] == 'h' {
			h++
			if m < h {
				return false
			}
		} else if count[i] == 'y' {
			y++
			if m < y || h < y {
				return false
			}
		} else {
			return false
		}
	}
	return m == h && h == y
}

```

3. 

> **题目：**米小游拿到了一个集合（集合中元素互不相等）。
>
> 她想知道，该集合有多少个元素数量大于 11 的子集，满足子集内的元素两两之间互为倍数关系？
>
> 由于数量可能过大，请对 1e9+7 取模。
>
> **输入描述：**
>
> 第一行输入一个正整数 n ，代表集合大小。
>
> 第二行输入 n 个正整数 $a_i$ ，代表集合的元素。
>
> 1 ≤ n ≤ $10^5$
>
> 1 ≤ $a_i$ ≤ $10^6$
>
> **输出描述：**
>
> 一个整数，代表满足题意的子集数量对 对 1e9+7 取模的值。
>
> **输入**
>
> ```
> 5
> 1 2 3 4 5
> ```
>
> **输出**
>
> ```
> 6
> ```
>
> **样例解释**
>
> 共有6个合法的子集：
>
> ```
> {1,2},{1,3},{1,4},{1,5},{1,2,4},{2,4}
> ```

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"sort"
	"strconv"
	"strings"
)

// DFS 只能过 20%
func main() {
	var n int
	fmt.Scanf("%v\n", &n)
	sc := bufio.NewScanner(os.Stdin)
	nums := []int{}
	if sc.Scan() {
		str := sc.Text()
		strs := strings.Split(str, " ")
		for _, str := range strs {
			num, _ := strconv.Atoi(str)
			nums = append(nums, num)
		}
	}
	fmt.Println(count(nums))
}

func count(nums []int) int {
	mod := 1000000007
	sort.Ints(nums)
	res := 0
	var dfs func(index int, lastNum int)
	dfs = func(index int, lastNum int) {
		if index == len(nums) {
			return
		}
		if nums[index]%lastNum == 0 {
			res %= mod
			res++
			dfs(index+1, nums[index])
		}
		dfs(index+1, lastNum)
	}
	for i := 1; i < len(nums); i++ {
		dfs(i, nums[i-1])
	}
	return res
}

```

```go
func main() {
	fmt.Println(count([]int{1, 2, 3, 4, 5}))
}

func count(nums []int) int {
	mod := 1000000007
	dp := make([]int, len(nums)+1)

	for i := 1; i < len(dp); i++ {
		for j := 1; j < i; j++ {
			if nums[i-1]%nums[j-1] == 0 {
				dp[i] += dp[j] + 1
			}
			dp[i] %= mod
		}
	}
	res := 0
	for i := 1; i < len(dp); i++ {
		res += dp[i]
	}
	return res
}
```

## 阿里

1. 

> **题目：**给定一个字符串，判断是否为“ali”型字符串。字符串满足以下条件：
> （1）字符串仅包含a、l、i三种字母， (包括大写和小写)
> （2）字符串的开头为仅包含a或者“A"的连续子串
> （3）在该子串后面，为仅包含l或者L的连续子申
> （4）在该子串后面，为仅包含i或者I的连续子甲。该子审结束后将直接到达字符串的结尾
> 输入AAaLlLLiili，输出Yse；
> 输入aAiIlL，输出No
> 输入alIali，输出No

```go
func isValid(str string) bool {
	flagA, flagB, flagC := false, false, false
	for i := 0; i < len(str); i++ {
		cur := str[i] | 32
		if cur == 'a' {
			if flagB || flagC {
				return false
			}
			flagA = true
		}
		if cur == 'l' {
			if !flagA || flagC {
				return false
			}
			flagB = true
		}
		if cur == 'i' {
			if !flagA || !flagB {
				return false
			}
			flagC = true
		}
	}
	return flagA && flagB && flagC
}

func main() {
	fmt.Println(isValid("AAaLlLLiiIi"))
	fmt.Println(isValid("ail"))
	fmt.Println(isValid("aliali"))
}
```

2. 



3. DFS

> **题目：**给定一个n层的满二叉树，一共2^n^ - 1个节点，编号从1到2^n^ -1。对于编号为i的节点，它的左儿子为2i，它的右儿子为2i+ 1.有q次操作，每次操作我们选择一个节点，将该节点的子树的所有节点全部染红。每次操作后，你需要输出当前二叉树红色节点的数量。我们定义一棵二叉树是满二叉树，当且仅当每一层的节点数量都达到了最大值(即无法在这一层添加新节点)。

```go
type TreeNode struct {
	Color int // 0: 未染色, 1: 染色
	Left  *TreeNode
	Right *TreeNode
}

func buildTree(height int) *TreeNode {
	if height == 0 {
		return nil
	}
	return &TreeNode{0, buildTree(height - 1), buildTree(height - 1)}
}

type Tree struct {
	count int // 保存已染色节点数量
	root  *TreeNode
}

func main() {
	var n, q int
	fmt.Scanf("%v %v\n", &n, &q)
	tree := &Tree{
		count: 0,
		root:  buildTree(n),
	}
	sc := bufio.NewScanner(os.Stdin)
	for i := 0; i < q; i++ {
		if sc.Scan() {
			str := sc.Text()
			num, _ := strconv.Atoi(str)
			fmt.Println(tree.RanSe(num))
		}
	}
}

// RanSe 返回染色后共有多少染色节点
func (t *Tree) RanSe(i int) int {
	node := getNode(i, t.root)
	t.count += dfs(node)
	return t.count
}

func getNode(i int, node *TreeNode) *TreeNode {
	if i == 1 {
		return node
	}
	stack := []int{}
	for i > 1 {
		stack = append(stack, i)
		i >>= 1
	}
	for len(stack) > 0 {
		num := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if num&1 == 0 {
			node = node.Left
		} else {
			node = node.Right
		}
	}
	return node
}

func dfs(node *TreeNode) int {
	if node == nil || node.Color == 1 {
		return 0
	}
	res := 1
	node.Color = 1
	res += dfs(node.Left)
	res += dfs(node.Right)
	return res
}
```

```java
import java.util.Scanner;
import java.util.Stack;

class TreeNode {
    int color;
    TreeNode left;
    TreeNode right;

    public TreeNode(int color, TreeNode left, TreeNode right) {
        this.color = color;
        this.left = left;
        this.right = right;
    }
}

class Tree {
    int count;
    TreeNode root;

    public Tree(int count, TreeNode root) {
        this.count = count;
        this.root = root;
    }

    public int ranSe(int i) {
        TreeNode node = getNode(i, root);
        count += dfs(node);
        return count;
    }

    private TreeNode getNode(int i, TreeNode node) {
        if (i == 1) {
            return node;
        }
        Stack<Integer> st = new Stack<>();
        while (i > 1) {
            st.push(i);
            i >>= 1;
        }
        while (!st.empty()) {
            int num = st.pop();
            if ((num & 1) == 0) {
                node = node.left;
            } else {
                node = node.right;
            }
        }
        return node;
    }

    private int dfs(TreeNode node) {
        if (node == null || node.color == 1) {
            return 0;
        }
        int res = 1;
        node.color = 1;
        res += dfs(node.left);
        res += dfs(node.right);
        return res;
    }
}

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int n = scanner.nextInt();
        int q = scanner.nextInt();
        Tree tree = new Tree(0, buildTree(n));
        for (int i = 0; i < q; i++) {
            int num = scanner.nextInt();
            System.out.println(tree.ranSe(num));
        }
    }

    private static TreeNode buildTree(int height) {
        if (height == 0) {
            return null;
        }
        return new TreeNode(0, buildTree(height - 1), buildTree(height - 1));
    }
}
```

#### 2023.3.15-第一题-满二叉树计数

> **题目：**在一个遥远的国度，有一个古老的神秘森林，被认为是森林之王的家园。传说森林之王是一只拥有巨大力量和智慧的生物，掌管着整个森林的命运。为了探索森林之王的秘密，许多勇敢的探险家一直在进入这片神秘的森林中。然而，进入森林之后，他们都没有回来过，因此这个秘密依然没有被解开。
>
> 有一天，一位名叫塔子哥的年轻探险家也进入了森林。他翻越了陡峭的山峰，穿过了茂密的丛林，终于到达了一处古老的废墟。在废墟的中心，塔子哥发现了一棵神奇的二叉树。这棵二叉树是如此的美丽，以至于塔子哥不禁驻足观赏，他想知道这棵二叉树有多少个节点满足以该节点为根的子树是满二叉树。于是，他开始了他的计算，希望能够揭开这个森林之王的秘密。
>
> 我们定义一棵树是满二叉树，当且仅当每一层的节点数量都达到了最大值(即无法在这一层添加新节点)。
>
> **输入描述**
>
> 第一行输入一个正整数*n*，代表节点的数量。
>
> 接下来的*n*行，第i行输入两个整数*l_i*和*r_i*，代表*i*号节点的左儿子和右儿子。请注意，如果一个节点没有左儿子/右儿子，则对应的*l_i*/*r_i*为-1。
>
> 1≤*n*≤10^5^
>
> **输出描述**
>
> 子树为满二又树的节点数量。
>
> **样例11**
>
> **输入**
>
> ```none
> 5
> 2 3
> 4 5
> -1 -1
> -1 -1
> -1 -1
> ```
>
> **输出**
>
> ```none
> 4
> ```
>
> **说明** 22、33、44、55号节点的子树都是满二叉树

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	nodes := make([][]int, n+1)
	for i := 1; i <= n; i++ {
		var l, r int
		fmt.Fscanln(sc, &l, &r)
		nodes[i] = []int{l, r}
	}
	test3(n, nodes)
}

func test3(n int, nodes [][]int) {
	deeps := make([]int, n+1)
	var dfs func(node int)
	dfs = func(node int) {
		left, right := nodes[node][0], nodes[node][1]
		// 当前节点为叶节点
		if left == -1 && right == -1 {
			deeps[node] = 1
			return
		}
		// 当前节点只有左子树或右子树
		if left == -1 {
			deeps[node] = -1
			dfs(right)
			return
		}
		if right == -1 {
			deeps[node] = -1
			dfs(left)
			return
		}
		// 当前节点的左、右子树还未搜索
		if deeps[left] == 0 {
			dfs(left)
		}
		if deeps[right] == 0 {
			dfs(right)
		}
		// 当前节点左右子树都搜索过了, 且左右子树不都为满二叉树
		if deeps[left] == -1 || deeps[right] == -1 {
			deeps[node] = -1
			return
		}
		// 当前节点的左右子树均为满二叉树, 且高度一致
		if deeps[left] == deeps[right] {
			deeps[node] = deeps[left] + 1
		} else {
			deeps[node] = -1
		}
		return
	}
	dfs(1)
	res := 0
	for i := 1; i < len(deeps); i++ {
		if deeps[i] > 0 {
			res++
		}
	}
	fmt.Println(res)
}
```

#### 2023.3.15-第二题-极差三元组计数

> **题目：**在一个小镇上，有一家古董店。这家古董店主人是一个极具收藏热情的老人，他一生都在收集和珍藏各种珍奇古董。他把他的珍藏分成了许多类别，其中包括了各种古代文物、稀有的艺术品和珍贵的文献资料等。
>
> 然而，这位老人在年轻时曾有一段失恋的经历，他的爱人因意外离世，让他深受打击。他渐渐地迷上了麻将这项运动，麻将成为了他减轻痛苦的一种方式。他甚至在自己的古董店里开了一间麻将室，邀请各位朋友前来打麻将。
>
> 一天，他在整理古董时发现了一组古董，这组古董是他年轻时收藏的。他把这些古董拿到麻将室，让朋友们来观赏和欣赏。这组古董是由 *n* 个不同的古董组成的，老人和他的朋友们都非常喜欢这些古董。
>
> 他在整理这些古董时，发现了一组数字序列 a*0,*a*1,⋯,*a*n−1* ，其中每个 a*i* 表示古董的编号。
>
> 他想让朋友们玩一下游戏，要求朋友们找出有多少组三元组 <*i*,*j*,*k*> 满足 0≤*i*<*j*<*k*<*n* 且 ma**x*(*a**i*,*a**j*,*a**k*)−*min*(*a*i*,*a*j*,*a*k*)=1。
>
> 这时，老人问你是否能够帮助他计算出答案。
>
> **输入描述**
>
> 第一行输入一个正整数 *n* 。
>
> 第二行输入*n*个正整数*ai* 。
>
> 3≤*n*≤200000
>
> 1≤*a*≤10^9^
>
> **输出描述**
>
> 一个整数，代表合法的三元组数量。
>
> **样例** 1
>
> **输入**
>
> ```none
> 3
> 1 2 3
> ```
>
> **输出**
>
> ```none
> 0
> ```
>
> **样例** 2
>
> **输入**
>
> ```none
> 4
> 2 1 2 2
> ```
>
> **输出**
>
> ```none
> 3
> ```

**滑动窗口：**

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	nums := make([]int, n)
	for i := 0; i < n; i++ {
		var a int
		fmt.Fscan(sc, &a)
		nums[i] = a
	}
	fmt.Fscanln(sc)
	test4(nums)
}

func test4(nums []int) {
	var res int64
	sort.Ints(nums)
	left, right := 0, 0
	for right < len(nums) {
		for left <= right && nums[right]-nums[left] > 1 {
			left++
		}
		if right-left >= 2 && nums[right]-nums[left] == 1 {
			// 统计所有以right结尾的合法三元组
			cur := left
			for right-cur >= 2 && nums[right]-nums[cur] == 1 {
				res += int64(right - cur - 1)
				cur++
			}
		}
		right++
	}
	fmt.Println(res)
}
```

#### 2023.04.12-研发-第一题

> **题目：**曾经有一个叫做塔子哥的年轻人，他是一个热衷于研究二叉树的数学家。他认为，一棵树只有当它的红色节点数量大于蓝色节点数量时才是好树。他的定义引起了人们的热议，有些人同意他的定义，有些人则持不同意见。但无论怎样，这个定义还是被广泛地接受了。
>
> 有一天，塔子哥发现了一棵好树，但他觉得它不够完美。于是他开始思考，如果能够将这棵好树分成两个好树，那它就更加完美了。
>
> 经过一番研究，他发现只需要删除一条边，就可以将这棵树分成两个好树。他想知道有多少种删除边的方案，于是他向你求助。
>
> **输入描述**
>
> 第一行输入一个正整数 *n* ，代表节点的数量。
>
> 第二行输入一个长度为 *n* 的，仅包含 `'R'` 和 `'B'` 两种字符的字符串，第 *i* 个字符为 `'R'` 代表第 *i* 个节点被染成红色，`'B'`代表被染成蓝色。
>
> 接下来的 *n*−1 行，每行输入两个正整数 *u* 和 *v* ，代表节点 *u* 和节点 *v* 有一条边连接。
>
> 1≤*n*≤10^5^
>
> **输出描述**
>
> 一个整数，代表删边的方案数。
>
> **样例**
>
> **输入**
>
> ```none
> 5
> RRBRB
> 1 2
> 2 3
> 1 4
> 4 5
> ```
>
> **输出**
>
> ```none
> 0
> ```
>
> **样例解释**
>
> 无论删除哪条边，都会导致有一个子树不是好树。

类似美团的RGB树，树形 DP

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	str, _ := sc.ReadString('\n')
	colors := strings.TrimRight(str, "\r\n")
	colors = "#" + colors
	edges := [][]int{}
	for i := 0; i < n-1; i++ {
		var u, v int
		fmt.Fscanln(sc, &u, &v)
		edges = append(edges, []int{u, v})
	}
	test5(n, colors, edges)
}

func test5(n int, colors string, edges [][]int) {
	// 1. 计算红、蓝的总数
	sumRed, sumBlue := 0, 0
	for i := 1; i < len(colors); i++ {
		if colors[i] == 'R' {
			sumRed++
		} else {
			sumBlue++
		}
	}
	// 2. 建图
	graph := map[int][]int{}
	for _, edge := range edges {
		graph[edge[0]] = append(graph[edge[0]], edge[1])
	}
	// 3. dfs
	res := 0
	var dfs func(curNode int) (int, int)
	dfs = func(curNode int) (int, int) {
		curRed, curBlue := 0, 0
		if len(graph[curNode]) > 0 {
			tempRed, tempBlue := 0, 0
			for _, next := range graph[curNode] {
				tempRed, tempBlue = dfs(next)
				curRed += tempRed
				curBlue += tempBlue
			}
		}
		if colors[curNode] == 'R' {
			curRed++
		} else {
			curBlue++
		}
		if sumRed-curRed > sumBlue-curBlue && curRed > curBlue {
			res++
		}
		return curRed, curBlue
	}
	dfs(1)
	fmt.Println(res)
}
```

#### 2023.04.12-研发-第三题

> **题目：**曾经有一个小镇叫做“数字王国”，这个小镇以数字相关的工艺和技术而闻名于世。其中最出名的数字工匠就是塔子哥。他被认为是数字领域中最聪明的人之一。他的天赋是发现数字之间的规律，并创造一些有趣的数字游戏。
>
> 其中一天，塔子哥想出了一个游戏：删除数字。这个游戏的规则是：给定一个正整数，你需要删除其中连续的一段数字，使得它变成15的倍数。他想知道有多少种不同的删除方式可以达到这个目标。
>
> 现在，他把这个问题交给了你。
>
> 注：删除的位置不同，即可记为两种不同的方式，并且允许删除后的数存在前导零。同时,**不能全删也不能不删**
>
> **输入描述**
>
> 输入一个正整数 *n* 。1≤*n*≤10^100000^
>
> **输出描述**
>
> 一个正整数，代表删除的方案数。
>
> **样例**
>
> **输入**
>
> ```none
> 12313565
> ```
>
> **输出**
>
> ```none
> 9
> ```

**思路：**

没全过，不太想看了。

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var str string
	fmt.Fscanln(sc, &str)
	test6(str)
}

func test6(str string) {
	nums := []int{}
	for i := 0; i < len(str); i++ {
		nums = append(nums, int(str[i]-'0'))
	}
	// 求前缀和
	prefixs := []int{}
	cur := 0
	for i := 0; i < len(nums); i++ {
		cur += nums[i]
		prefixs = append(prefixs, cur)
	}
	// 滑动窗口
	res := 0
	sum := prefixs[len(nums)-1]
	for i := 0; i < len(nums); i++ {
		lastNum := nums[len(nums)-1]
		if i != len(nums)-1 && lastNum%5 != 0 {
			continue
		}
		for j := 0; j <= i; j++ {
			if i == len(nums)-1 {
				if j == 0 {
					continue
				} else {
					lastNum = nums[j-1]
				}
			}
			curSum := 0
			if j == 0 {
				curSum = sum - prefixs[i]
			} else {
				curSum = sum - (prefixs[i] - prefixs[j-1])
			}
			// 判断是否有效
			if lastNum%5 == 0 && curSum%3 == 0 {
				res++
			}
		}
	}
	fmt.Println(res)
}
```

#### 2023.03.26-第一题-切割环形数组

> **题目：**塔子哥是一个游戏爱好者，他喜欢用数字和算法解决各种难题。有一天，他遇到了一个有趣的问题。他手头有一个环形数组 *a* ，数组中有 *n* 个数字。
>
> 他想玩一个游戏，用两刀把它切割成两段，使得两段的数字之和相等。他可以选择在任何两个数字之间切割，但是他必须确保切割的两段都不为空。
>
> 塔子哥非常聪明，很快就意识到这是一个非常有趣的问题。他开始思考如何解决这个问题，并且尝试了很多不同的方法，还是没有想出来如何解决。
>
> 于是，塔子哥想分享这个问题并且寻求你的帮助。他想知道有多少种不同的切割方案可以使得两段数字之和相等。
>
> **输入描述**
>
> 第一行输入一个正整数 *n* ，代表环形数组的元素数量。
>
> 第二行输入 *n* 个正整数 *ai*，代表环形数组的元素。
>
> 1≤*n*≤10^5^
>
> −10^9^≤*ai*≤10^9^
>
> **输出描述**
>
> 一个整数，代表切割的方案数。
>
> **样例**
>
> **输入**
>
> ```none
> 5
> 1 4 -2 5 2
> ```
>
> **输出**
>
> ```none
> 2
> ```

**思路：**前缀和+**哈希表**

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	nums := []int{}
	for i := 0; i < n; i++ {
		var ai int
		fmt.Fscan(sc, &ai)
		nums = append(nums, ai)
	}
	fmt.Fscanln(sc)
	test7(nums)
}

func test7(nums []int) {
	sum := 0
	for _, num := range nums {
		sum += num
	}
	if sum&1 == 1 {
		fmt.Println(0)
		return
	}
	target := sum / 2
	prefix := 0
	prefixs := []int{}
	for i := 0; i < len(nums); i++ {
		prefix += nums[i]
		prefixs = append(prefixs, prefix)
	}
	res := 0
	hashMap := map[int]int{}
	for i := 0; i < len(nums); i++ {
		pre := prefixs[i] - target
		if val, exist := hashMap[pre]; exist {
			res += val
		}
		hashMap[prefixs[i]]++
	}
	fmt.Println(res)
}
```



## 腾讯音乐

1. 贪心（不想写贪心！）

> 小红拿到了一个二叉树，二叉树共有n个节点。小红希望你将所有节点赋值为1到n的正整数，且没有两个节点的值相等。需要满足：奇数层的权值和与偶数层的权值和之差的绝对值不超过1。
>
> 如果有多种赋值方案，请返回任意—种方案。如果无解，请返回空树。数据范围: 1 < n ≤ 10^5^。给定的二叉树节点初始权值默认为-1。
>
> **示例1**
>
> **输入**
>
> {-1,-1,-1}
>
> **输出**
>
> {3,1,2}
>
> **示例2**
>
> **输入**
>
> {-1,-1,#,-1,-1}
>
> **输出**
>
> {}
>
> **示例3**
>
> **输入**
>
> {-1,-1,-1,#,-1,-1}
>
> **输出**
>
> {1,3,4,#,2,5}

2. 二分查找

这题能用二分真么想到啊，写了一个小时，DP只能过40%。

**动态规划：**

```go
```

**二分查找：**

```go
```

#### 2023.04.13-第二题-价值二叉树

> **题目：**塔子哥是一位著名的计算机科学家，在一次采集森林中的植物时，偶然发现了一棵非常特殊的树。这棵树是一棵二叉树，其节点上都标有不同的数字。
>
> 在细心观察后，塔子哥意识到这棵二叉树是由多个相同的子树组成的，每个子树的根节点都是同一个数字。他对这个发现感到非常兴奋，并且开始研究如何计算这些子树的价值。
>
> 他定义每个节点的价值为其子树节点乘积的末尾 0 的数量。因此，如果一个节点的子树中有 *k* 个数末尾带有 0，那么该节点的价值为 *k*。
>
> 塔子哥想编写一个程序来计算每个节点的价值，以便能够更好地研究这棵树的特性。他请求您的帮助来实现这个程序，您需要返回一棵二叉树，树的结构和给定的二叉树相同，将每个节点的权值替换为该节点的价值。
>
> 二又树节点数不超过 105 。
>
> 二又树每个节点的权值都是不超过 109 的正整数。
>
> **输入描述**
>
> 第一行为一个整数 *n* ，表示这棵树的节点个数。
>
> 第二行为 *n* 个整数，第 *i* 个整数为 *ai* ，表示这 *n* 个节点的取值。
>
> 接下来的 *n*−1 行，每行输入两个正整数 *u* 和 *v* ，代表节点 *u* 和节点 *v* 有一条边相连。 **根为1**
>
> 1≤*n*≤10^5^，1≤*ai*≤10^9^ 。
>
> **输出描述**
>
> 输出一行 *n* 个正整数，分别代表 1 号节点到 *n* 号节点，每个节点的子树权值乘积尾零的数量。
>
> **样例**
>
> **输入**
>
> ```none
> 5
> 2 5 10 4 5
> 1 2
> 1 3
> 3 4
> 3 5
> ```
>
> **输出**
>
> ```none
> 3 0 2 0 0
> ```

似乎可以用树形 DP。

这里用的二叉树的后序遍历。

```go
type TreeNode struct {
	Id          int
	Nums        []int // 存储 2 和 5 的个数, 直接存乘积可能会溢出
	Left, Right *TreeNode
}

func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	a := []int{0}
	for i := 1; i <= n; i++ {
		var ai int
		fmt.Fscan(sc, &ai)
		a = append(a, ai)
	}
	fmt.Fscanln(sc)
	edges := [][]int{}
	for i := 1; i < n; i++ {
		var u, v int
		fmt.Fscanln(sc, &u, &v)
		edges = append(edges, []int{u, v})
	}
	test2(a, edges)
}

func test2(a []int, edges [][]int) {
	// 1. 建树
	tree := make([]*TreeNode, len(a))
	for _, edge := range edges {
		if tree[edge[0]] == nil {
			tree[edge[0]] = &TreeNode{}
		}
		if tree[edge[1]] == nil {
			tree[edge[1]] = &TreeNode{}
		}
		tree[edge[0]].Id, tree[edge[1]].Id = edge[0], edge[1]
		tree[edge[0]].Nums = calZero(a[edge[0]])
		tree[edge[1]].Nums = calZero(a[edge[1]])
		if tree[edge[0]].Left == nil {
			tree[edge[0]].Left = tree[edge[1]]
		} else {
			tree[edge[0]].Right = tree[edge[1]]
		}
	}
	root := tree[1]
	// 2. 后序遍历
	res := make([]int, len(a))
	var sufOrder func(curNode *TreeNode)
	sufOrder = func(curNode *TreeNode) {
		if curNode.Left == nil && curNode.Right == nil {
			res[curNode.Id] = min(curNode.Nums[0], curNode.Nums[1])
			return
		}
		if curNode.Left != nil {
			sufOrder(curNode.Left)
			curNode.Nums[0] += curNode.Left.Nums[0]
			curNode.Nums[1] += curNode.Left.Nums[1]
		}
		if curNode.Right != nil {
			sufOrder(curNode.Right)
			curNode.Nums[0] += curNode.Right.Nums[0]
			curNode.Nums[1] += curNode.Right.Nums[1]
		}
		res[curNode.Id] = min(curNode.Nums[0], curNode.Nums[1])
	}
	sufOrder(root)
	for i := 1; i < len(res); i++ {
		fmt.Printf("%v ", res[i])
	}
}

func calZero(val int) []int {
	nums := make([]int, 2)
	for val > 0 && val%2 == 0 {
		nums[0]++
		val /= 2
	}
	for val > 0 && val%5 == 0 {
		nums[1]++
		val /= 5
	}
	return nums
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

## 恒生

2. **动态规划/DFS**

给定m初始金额，a 为每天的股价，k为一共可买卖多少次，求最大利润(主要要减去本金！)

```go
func getMaxProfit(m float64, n int, a []float64, k int) float64 {
	dp := make([][][2]float64, n)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([][2]float64, k+1)
	}
	// 初始化
	dp[0][0][0] = m
	dp[0][0][1] = m / a[0]
	for i := 1; i < len(dp); i++ {
		for j := 0; j <= k; j++ {
			if j == 0 {
				dp[i][j][0] = dp[i-1][j][0]
				dp[i][j][1] = dp[i-1][j][0] / a[i]
			} else {
				dp[i][j][0] = max(dp[i-1][j][0], dp[i-1][j-1][1]*a[i]) // 第j次 不持有
				dp[i][j][1] = max(dp[i-1][j][1], dp[i-1][j][0]/a[i])   // 第j次 持有
			}
		}
	}
	return dp[n-1][k][0] - m
}

func max(a, b float64) float64 {
	if a > b {
		return a
	}
	return b
}

func main() {
	historyPrices := []float64{1.0, 2.0, 1.0, 2.0, 2.0, 3.0, 2.0}
	fmt.Println(getMaxProfit(10000, 7, historyPrices, 2))
}
```

```java
public class Main {
    public static double get_max_profit(double M, int N, double[] historyPrices, int K) {
        return dfs(M, 0, 0, historyPrices, K) - M;
    }

    public static double dfs(double profit, int gu, int index, double[] historyPrices, int k) {
        if (index == historyPrices.length) {
            return profit;
        }
        double res = profit;
        // 持有股票，卖出
        if (gu > 0) {
            res = Math.max(res, dfs(profit + gu * historyPrices[index], 0, index + 1, historyPrices, k));
        }
        // 没持有股票，买入
        if (gu == 0 && k > 0) {
            int nextGu = (int) (profit / historyPrices[index]);
            res = Math.max(res, dfs(profit - nextGu * historyPrices[index], nextGu, index + 1, historyPrices, k - 1));
        }
        // 不买也不卖
        res = Math.max(res, dfs(profit, gu, index + 1, historyPrices, k));
        return res;
    }

    public static void main(String[] args) {
        double[] historyPrices = new double[]{1.0, 2.0, 1.0, 2.0, 2.0, 3.0, 2.0};
        System.out.println(get_max_profit(10000, 7, historyPrices, 2));   // 50000
    }
}
```

## 蚂蚁

1. 2022.10.27-括号匹配

> **题目：**给定个仅包含 `(` 、 `)` 和 `?` 三种字符构成的字符串，`?` 字符可以代替左括号或者右括号。请问该字符串可以代表多少种不同的合法括号序列?
>
> **输入描述**
>
> 一个仅包含`(` 、`)` 和 `?` 的字符串，长度不超过 20002000 。
>
> **输出描述**
>
> 合法序列的数量。由于数量可能过大，请对 109+7109+7 取模。
>
> **样例**
>
> **输入**
>
> ```none
> ????(?
> ```
>
> **输出**
>
> ```none
> 2
> ```
>
> **样例解释**
>
> 共有 22 种不同的合法括号序列， `() () ()` 或 `(()) ()`

假设字符串长度为 n，可以考虑使用动态规划求解。令 `dp[i][j]` 表示前 i 个字符中，左括号比右括号多 j 个的方案数。考虑第 i+1 个字符：

1. 如果是左括号，则 `dp[i+1][j+1] = dp[i][j]`。
2. 如果是右括号，则 `dp[i+1][j-1] = dp[i][j]`。
3. 如果是问号，则既可以代表左括号，也可以代表右括号，因此 `dp[i+1][j+1] = dp[i][j] + dp[i+1][j-1]`。

最终答案即为 `dp[n][0]`，表示左右括号数量相等的方案数。

需要注意的是，由于原始字符串中可能存在 ? 字符，因此需要初始化 `dp[0][0] = 1`，并将剩余的 `dp[0][j]` 和 `dp[i][j]` 的值都设为 0。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

var mod int = 1e9 + 7

func main() {
	sc := bufio.NewScanner(os.Stdin)
	sc.Scan()
	str := sc.Text()
	fmt.Println(test1(str))
}

func test1(str string) int {
	dp := make([][]int, len(str)+1)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([]int, i+1)
	}
	str = "#" + str
	dp[0][0] = 1
	for i := 1; i < len(dp); i++ {
		for j := 0; j <= i; j++ {
			if str[i] == '(' && j > 0 {
				dp[i][j] = dp[i-1][j-1] % mod
			}
			if str[i] == ')' && j < i-1 {
				dp[i][j] = dp[i-1][j+1] % mod
			}
			if str[i] == '?' {
				if j > 0 {
					dp[i][j] += dp[i-1][j-1] % mod
				}
				if j < i-1 {
					dp[i][j] += dp[i-1][j+1] % mod
				}
			}
		}
	}
	return dp[len(dp)-1][0]
}
```

可以简化一下for循环里的逻辑：

```go
func test1(str string) int {
	dp := make([][]int, len(str)+1)
	for i := 0; i < len(dp); i++ {
		dp[i] = make([]int, i+1)
	}
	str = "#" + str
	dp[0][0] = 1
	for i := 1; i < len(dp); i++ {
		for j := 0; j <= i; j++ {
			if (str[i] == '(' || str[i] == '?') && j > 0 {
				dp[i][j] = dp[i-1][j-1] % mod	// 这边是 = 
			}
			if (str[i] == ')' || str[i] == '?') && j < i-1 {
				dp[i][j] += dp[i-1][j+1] % mod	// 这边是 +=, 主要是应对 '?' d
			}
		}
	}
	return dp[len(dp)-1][0]
}
```

2. 2023.03.21-第二题-红白染色树

> **题目：**曾经有一个名叫塔子哥的年轻人，他喜欢探索和解决各种难题。有一天，他发现了一棵神奇的树。树上已经有一些点被染成了红色，另一些点被染成了白色。
>
> 但是塔子哥想让相邻的两个点不能够颜色相同，因此他想知道最少需要进行多少次操作才能让树上所有相邻两个点的颜色不同，每次操作塔子哥可以选择一个点改变它的染色状态（红色变白色或者白色变红色）。
>
> 这个问题困扰着他很长一段时间，但是他决定不放弃，你能帮塔子哥想一想怎么解决这个问题吗？
>
> **输入描述**
>
> 第一行输入一个正整数 *n* ，代表节点的数量。
>
> 第二行输入一个长度为 *n* 的、仅由 `'R'` 、 `'W'` 两种字符组成的字符串，第 *i* 个字符为 `'R'` 代表 *i* 号节点被染成红色， `'W'` 代表被染成白色。
>
> 接下来的 *n*−1 行，每行输入两个正整数 *u* 和 *v* ，代表节点 *u* 和节点 *v* 有一条路径相连。
>
> 1 ≤ *n* ≤ 10^5^
>
> 1 ≤ *u*,*v* ≤ *n*
>
> **输出描述**
>
> 一个整数，代表最小的操作次数。
>
> **样例**
>
> **输入**
>
> ```none
> 4
> RWWW
> 1 2
> 2 3
> 2 4
> ```
>
> **输出**
>
> ```none
> 2
> ```
>
> **样例解释**
>
> 对 11 号和 22 号节点各操作一次即可。

**思路：**

拓扑排序

```go
func main() {
	var n int
	fmt.Scanf("%v\n", &n)
	sc := bufio.NewReader(os.Stdin)
	str, _ := sc.ReadString('\n')
	colors := strings.TrimRight(str, "\r\n")

	edges := [][]int{}
	for i := 0; i < n-1; i++ {
		var u, v int
		fmt.Fscanln(sc, &u, &v)
		edges = append(edges, []int{u - 1, v - 1})
	}
	
	fmt.Println(test2(colors, edges))
}

func test2(colors string, edges [][]int) int {
	graph := map[int][]int{}
	inDegrees := make([]int, len(colors))
	for _, edge := range edges {
		graph[edge[0]] = append(graph[edge[0]], edge[1])
		inDegrees[edge[1]]++
	}

	return min(updateColor(graph, inDegrees, colors, 0), updateColor(graph, inDegrees, colors, 1))
}

func updateColor(graph map[int][]int, inDegrees []int, colors string, startColor int) int {
	res := 0
	colorMap := []byte{'R', 'W'}
	tmpInDegrees := make([]int, len(inDegrees))
	copy(tmpInDegrees, inDegrees)
	queue := []int{}
	nextQueue := []int{}
	for i, inDegree := range tmpInDegrees {
		if inDegree == 0 {
			queue = append(queue, i)
		}
	}
	curColor := startColor
	for len(queue) > 0 {
		cur := queue[0]
		queue = queue[1:]
		if colors[cur] != colorMap[curColor] {
			res++
		}
		for _, node := range graph[cur] {
			tmpInDegrees[node]--
			if tmpInDegrees[node] == 0 {
				nextQueue = append(nextQueue, node)
			}
		}
		if len(queue) == 0 {
			queue = nextQueue
			nextQueue = []int{}
			curColor ^= 1
		}
	}
	return res
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

#### 2023.04.11-蚂蚁-第一题

> **题目：**塔子哥是一位年轻的数学家，他一直对数学理论充满热情和好奇心。有一天，他在研究数学理论时发现了一个有趣的问题。他偶然间发现了一个古老的书卷，上面记载着一些神秘的数学定理和公式，其中有一个问题深深吸引了他的注意力。
>
> 这个问题涉及到一个长度为n的整数数组，其中每个元素都是一个正整数。
>
> 塔子哥很感兴趣的是，有多少个奇数出现了奇数次。
>
> 注：出现多次的元素也都要计算在内。
>
> **输入描述**
>
> 第一行输入一个正整数 *n* ，代表数组的大小。
>
> 第二行输入 *n* 个正整数 *ai* ，代表数组的元素。
>
> 1≤*n*≤200000
>
> 1≤*ai*≤109
>
> **输出描述**
>
> 一个整数，代表最终出现次数是奇数的奇数数量。
>
> **样例**
>
> **输入**
>
> ```none
> 5
> 1 2 3 3 5
> ```
>
> **输出**
>
> ```none
> 2
> ```
>
> **样例解释**
>
> 1 出现了一次， 2 出现了一次， 3 出现了两次， 5 出来了一次，符合条件的只有一个 1 和一个 5 。

**思路：**滑动窗口

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	a := []int{}
	for i := 0; i < n; i++ {
		var ai int
		fmt.Fscan(sc, &ai)
		a = append(a, ai)
	}
	fmt.Fscanln(sc)
	test4(a)
}

func test4(a []int) {
	sort.Ints(a)
	res := 0
	left, right := 0, 0
	for right < len(a) {
		for right < len(a) && a[right] == a[left] {
			right++
		}
		if a[left]&1 == 1 && (right-left)&1 == 1 {
			res += right - left
		}
		left = right
	}
	fmt.Println(res)
}
```



## 百度

#### 2023.3.13-第二题-构造回文串

> **题目：**有一个神话传说，说在很久以前，红色的守护者曾经来到人间，为了保护人们免受邪恶的侵害，他留下了一个强大的魔法。这个魔法被传说中的红宝石所承载，只有能够创造出特定数量回文子串的字符串才能够揭示出它的秘密。
>
> 为了获得这个神秘的魔法，有一个名叫塔子哥的人在神秘的传说中展开了他的冒险。他在一本古老的书籍中找到了一篇有关于这个魔法的描述：
>
> 在所有由‘r’,‘e’和‘d’这三种字符构成的字符串中，只有当回文子串的数量为特定数量时，才能揭示红宝石的魔法。这个特定数量由一个神秘的整数 *x* 所定义。
>
> 为了获得这个强大的魔法，塔子哥开始思考如何创造出刚好有 *x* 个回文子串的字符串。他发现，如果想要获得这个魔法，就必须要先找到这个特定的字符串，因此他开始了漫长的冒险之旅。
>
> 字符串的长度不得超过10^5^
>
> **输入描述**
>
> 一个正整数*x*.
>
> 1≤*x*≤10^9^
>
> **样例**
>
> **输入**
>
> ```none
> 3
> ```
>
> **输出**
>
> ```none
> red
> ```
>
> **说明**
>
> 输出"dd"也可以通过本题

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	var x int
	fmt.Scanf("%v\n", &x)
	fmt.Println(test4(x))
}

func test4(x int) string {
	res := []byte{}
	chars := []byte{'r', 'e', 'd'}
	cur := 0
	for x > 0 {
		num := sort.Search(x+1, func(i int) bool {
			return (i*i+i)/2 > x
		})
		num -= 1
		ch := make([]byte, num)
		for i := 0; i < len(ch); i++ {
			ch[i] = chars[cur]
		}
		cur = (cur + 1) % 3
		res = append(res, ch...)
		x -= (num*num + num) / 2
	}
	return string(res)
}
```

#### [2023.03.28-第三题-塔子的有根树](http://101.43.147.120/p/P1134)

> **题目：**在一个偏远的山区里，有一个叫做塔子哥的数学家。他热爱数学，对树这种数据结构也有着浓厚的兴趣。
>
> 有一天，他在森林中漫步时发现了一棵美丽的大树。这棵树非常漂亮，每个节点 *i* 都是独特的，有着它自己的权值 *vali* ，并且规定 1 号点为这棵树的根节点。
>
> 塔子哥对这棵树产生了浓厚的兴趣，他开始研究这棵树的性质，并思考一些问题。他想知道如果对于树上的某个节点 *t* ，以 *t* 为根的子树中所有节点的权值都乘上一个 *g* ，会对整棵树产生什么影响。
>
> 为了更好地研究这个问题，他进行了 *q* 次操作，每次选择了一个节点 *t* ，并将以 *t* 为根的子树中所有节点的权值都乘上了一个 *g* 。这个过程中，他记录了每个节点的最终权值，但他还想知道一个更有趣的问题：在 *q* 次操作结束以后，以节点 *i* 为根的子树中**所有节点的权值的乘积的末尾有多少个0**。
>
> 这个问题非常有趣，因为它不仅涉及到树的结构，还需要考虑数学中数字的性质。塔子哥非常期待你能帮他解决这个问题。
>
> **输入描述**
>
> 第一行输入一个正整数*n*，代表这颗树的节点的数量。
>
> 第二行输入*n*个正整数*vali*，代表每个节点的权值。
>
> 接下来的*n*−1行，每行输入两个正整数*u*和*v*，代表节点*u*和节点*v*有一条边相连。
>
> 接下来的一行输入一个正整数*q*，代表操作次数。
>
> 接下来的*q*行，每行输入两个正整数*t*和*g*，代表塔子哥的一次操作。
>
> 1⩽*n*,*q*⩽10^5^
>
> 1⩽*vi*,*g*⩽10^9^
>
> 1⩽*t*,*u*,*v*⩽*n*
>
> **输出描述**
>
> 输出一行*n*个正整数，分别代表1号节点到*n*号节点，每个节点的子树权值乘积尾零的数量。
>
> **样例**
>
> **输入**
>
> ```none
> 6
> 1 2 3 4 5 6
> 1 2
> 2 3
> 1 4
> 2 5
> 4 6
> 2
> 2 5
> 4 5
> ```
>
> **输出**
>
> ```none
> 4 1 0 2 0 1
> ```

跟腾讯音乐那个很像，只不过不是二叉树了，也带来了点启发：**不用建树，就按图写就行。直接对下一层的所有节点dfs，也可以实现先序、后序遍历**。

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	vals := []int{0}
	for i := 0; i < n; i++ {
		var v int
		fmt.Fscan(sc, &v)
		vals = append(vals, v)
	}
	fmt.Fscanln(sc)
	graph := map[int][]int{}
	for i := 0; i < n-1; i++ {
		var u, v int
		fmt.Fscanln(sc, &u, &v)
		graph[u] = append(graph[u], v)
	}
	var q int
	fmt.Fscanln(sc, &q)
	opts := [][]int{}
	for i := 0; i < q; i++ {
		var t, g int
		fmt.Fscanln(sc, &t, &g)
		opts = append(opts, []int{t, g})
	}

	test5(graph, vals, opts)
}

func test5(graph map[int][]int, vals []int, opts [][]int) {
	// 1. 得到初始化状态各节点的 [2, 5]
	dp := make([][]int, len(vals))
	for i := 1; i < len(dp); i++ {
		dp[i] = make([]int, 2)
		getTwoFive(vals[i], dp[i])
	}
	// 2. dfs 执行操作
	var dfsOpt func(cur int, val int)
	dfsOpt = func(cur int, val int) {
		getTwoFive(val, dp[cur])
		for _, next := range graph[cur] {
			dfsOpt(next, val)
		}
	}
	for _, opt := range opts {
		dfsOpt(opt[0], opt[1])
	}
	// 3. dfs 计算所有节点的0
	var dfsAdd func(cur int) (int, int)
	dfsAdd = func(cur int) (int, int) {
		for _, next := range graph[cur] {
			twoNext, fiveNext := dfsAdd(next)
			dp[cur][0] += twoNext
			dp[cur][1] += fiveNext
		}
		return dp[cur][0], dp[cur][1]
	}
	dfsAdd(1)
	// 4. 输出结果
	for i := 1; i < len(dp); i++ {
		if i != len(dp)-1 {
			fmt.Printf("%v ", min(dp[i][0], dp[i][1]))
		} else {
			fmt.Printf("%v", min(dp[i][0], dp[i][1]))
		}
	}
}

func getTwoFive(val int, nums []int) {
	two, five := 0, 0
	for val > 0 && val%2 == 0 {
		two++
		val /= 2
	}
	for val > 0 && val%5 == 0 {
		five++
		val /= 5
	}
	nums[0] += two
	nums[1] += five
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

## 小红书

#### 2023.04.09-第一题-拆分树

> **题目：**塔子哥是一个聪明勇敢的探险家。他听说有一棵神秘的树，据说在它的某个节点上藏有一个宝藏。为了寻找这个宝藏，塔子哥决定前往这棵树所在的深山探险。
>
> 经过漫长的征途，塔子哥终于到达了这棵树的附近。他仔细观察这棵树，发现它非常美丽，树干粗壮，树叶繁茂，枝干交错。但是，他也发现了一个问题：这棵树的某个节点上藏有宝藏，但是这个节点与其他节点之间的连接关系非常复杂，很难直接找到宝藏。
>
> 经过一番思考之后，塔子哥决定采用一种特殊的方法来寻找宝藏。他发现，如果他能够找到树上的某一条边，并删除它，那么这棵树就会被分成两个部分 *A* 和 *B*，而宝藏可能就在其中的某一部分中。
>
> 但是，塔子哥也知道，他不能随便删除一条边，因为他需要保证两个部分的节点数的差的绝对值 ∣∣*A*∣−∣*B*∣∣ 尽可能小，才能有更大的机会找到宝藏。因此，他需要找到最优的划分方案。
>
> 现在，他想请你输出最小的 ∣∣*A*∣−∣*B*∣∣ 和最优方案的数量，使得他有更大的机会找到宝藏。
>
> **输入描述**
>
> 第一行一个整数 *n* 表示节点的数量，节点从 1 到 *n* 编号。
>
> 接下来 n*−1 行每行两个正整数 *s* ， *t* ，表示 *s* 的父亲是 *t* 。
>
> 输入保证是一棵树。
>
> 对于所有数据 1≤*n*≤100000 。
>
> **输出描述**
>
> 输出一行两个整数，用空格分开，分别表示最优解和最优方案数。
>
> **样例**
>
> **输入**
>
> ```none
> 3
> 2 1
> 3 1
> ```
>
> **输出**
>
> ```none
> 1 2
> ```

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	edges := [][]int{}
	for i := 0; i < n-1; i++ {
		var s, t int
		fmt.Fscanln(sc, &s, &t)
		edges = append(edges, []int{s - 1, t - 1})
	}
	test5(n, edges)
}

func test5(n int, edges [][]int) {
	nodes := make([]int, n)
	graph := make([][]int, n)
	fathers := make([]int, n)
	inDegrees := make([]int, n)
	for _, edge := range edges {
		inDegrees[edge[1]]++
		graph[edge[1]] = append(graph[edge[1]], edge[0])
		fathers[edge[0]] = edge[1]
	}
	queue := []int{}
	for i, inDegree := range inDegrees {
		if inDegree == 0 {
			queue = append(queue, i)
		}
	}
	for len(queue) > 0 {
		cur := queue[0]
		queue = queue[1:]
		nodes[cur] = 1
		for _, i := range graph[cur] {
			nodes[cur] += nodes[i]
		}
		inDegrees[fathers[cur]]--
		if inDegrees[fathers[cur]] == 0 {
			queue = append(queue, fathers[cur])
		}
	}
	res, count := n, -1
	for _, node := range nodes {
		cur := abs(n-node, node)
		if res > cur {
			res = cur
			count = 1
		} else if res == cur {
			count++
		}
	}
	fmt.Println(res, count)
}

func abs(a, b int) int {
	if a > b {
		return a - b
	}
	return b - a
}
```

#### 2023.04.09-第二题-融合试剂

> **题目：**塔子哥是一位优秀的化学家，他的研究领域是配制各种化学试剂。今天，他的研究重点是一种特殊的化学溶液。这种溶液需要通过合并其他的多种溶液来制备，以达到理想的浓度和体积。
>
> 在实验室里，塔子哥看到了 *n* 种溶液，每一种都有无限多瓶，第 *i* 种的溶液体积为 *x_i* ，里面含有 *y_i* 单位的该物质。他想要用这些溶液来制备出一个体积恰好为 *C* 的溶液，且尽量浓，使得其中所含有的该物质数量尽可能多。
>
> 但是，这个过程并不容易。因为当两个瓶子的体积相等时，他们合并的过程中会发生化学反应，导致物质含量增加 *X* 单位。这也就意味着，如果选择了某两种体积相等的溶液进行合并，可能会获得更高的物质含量。因此，为了制备出更浓的溶液，塔子哥需要仔细考虑每一步的操作。
>
> 最终，经过反复的试验和计算，塔子哥终于成功地制备出了体积恰好为 *C* 的溶液，并且其中所含有的该物质数量也达到了最大值。他非常开心，因为他的努力得到了回报。现在，他想请你告诉他，这个溶液中所含有的该物质数量最多是多少。
>
> **输入描述**
>
> 第一行三个正整数 �*n* ， �*X* ， �*C* ；
>
> 第二行 �*n* 个正整数 �1,�2,...,��*x*1,*x*2,...,*x**n* ，中间用空格隔开；
>
> 第三行 �*n* 个正整数 �1,�2,...,��*y*1,*y*2,...,*y**n* ，中间用空格隔开。
>
> 对于所有数据， 1≤�,�,�,��≤10001≤*n*,*X*,*C*,*y**i*≤1000 ， 1≤��≤�1≤*x**i*≤*C* 数据保证至少存在一种方案能够配制溶液体积恰好等于 �*C* 的溶液。
>
> **输出描述**
>
> 输出一个整数，表示物质含量最多是多少。
>
> **样例**
>
> **输入**
>
> ```none
> 3 4 16
> 5 3 4
> 2 4 1
> ```
>
> **输出**
>
> ```none
> 29
> ```

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n, X, C int
	fmt.Fscanln(sc, &n, &X, &C)
	xs := make([]int, n)
	for i := 0; i < n; i++ {
		var xi int
		fmt.Fscan(sc, &xi)
		xs[i] = xi
	}
	fmt.Fscanln(sc)
	ys := make([]int, n)
	for i := 0; i < n; i++ {
		var yi int
		fmt.Fscan(sc, &yi)
		ys[i] = yi
	}
	fmt.Fscanln(sc)
	test5(X, C, xs, ys)
}

func test5(X, C int, xs, ys []int) {
	dp := make([]int, C+1)
	for j := 0; j < len(xs); j++ {
		dp[xs[j]] = max(dp[xs[j]], ys[j])
	}
	for i := 1; i < len(dp); i++ {
		for j := 1; j < i; j++ {
			// 体积为 j 和 i-j 的方案都得存在才能继续
			if dp[j] == 0 || dp[i-j] == 0 {
				continue
			}
			if i-j == j {
				dp[i] = max(dp[i], dp[j]+dp[i-j]+X)
			} else {
				dp[i] = max(dp[i], dp[j]+dp[i-j])
			}
		}
	}
	fmt.Println(dp[C])
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

```

#### 2023.04.09-第三题-神奇的盒子

没啥意思，简单记下：

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	nums := make([]int, n+1)
	for i := 1; i <= n; i++ {
		var temp int
		fmt.Fscan(sc, &temp)
		nums[i] = temp
	}
	sc.ReadString('\n')
	str, _ := sc.ReadString('\n')
	colors := "#" + strings.TrimRight(str, "\r\n")
	var m int
	fmt.Fscanln(sc, &m)
	times := make([]int, m+1)
	for i := 1; i <= m; i++ {
		var temp int
		fmt.Fscan(sc, &temp)
		times[i] = temp
	}
	opts := make([]int, m+1)
	for i := 1; i <= m; i++ {
		var temp int
		fmt.Fscan(sc, &temp)
		opts[i] = temp
	}
	test4(nums, times, opts, colors)
}

func test4(nums, times, opts []int, colors string) {
	res := []int64{}
	var cur int64
	flag := 0
	inTimes := map[int]int{} // 记录物品进入的时间
	for i := 1; i < len(opts); i++ {
		cur += int64(flag * (times[i] - times[i-1]))
		if opts[i] == 0 {
			res = append(res, cur)
		} else if opts[i] > 0 {
			inTimes[opts[i]] = times[i]
			if colors[opts[i]] == 'R' {
				flag++
			} else {
				flag--
			}
			cur += int64(nums[opts[i]])		// 加上
		} else if opts[i] < 0 {
			if colors[-opts[i]] == 'R' {
				nums[-opts[i]] += times[i] - inTimes[-opts[i]]	// 求它现在应该是多少
				flag--
			} else {
				nums[-opts[i]] -= times[i] - inTimes[-opts[i]]  // 求它现在应该是多少
				flag++
			}
			cur -= int64(nums[-opts[i]])	// 减去
		}
	}
	fmt.Printf("%v ", len(res))
	for i := 0; i < len(res); i++ {
		fmt.Printf("%v ", res[i])
	}
}
```

## 腾讯

#### 2023.03.26-第二题-重组字符串

> **题目：**
>
> 塔子哥有 N 个小写字母字符串，每个最长 8 个字母。他想玩一个游戏，从每个字符串里选一个字母，拼成一个重组字符串。重组字符串不能有重复的字母。问塔子哥能拼出多少种不同的重组字符串？
>
> **输入描述**
>
> 第一行输入整数为N
>
> 第二行到第N+1行输入N个字符串，全部由小写字母组成
>
> 2≤*N*≤6
>
> 1≤*l**e**n*(字符串)≤8
>
> **输出描述**
>
> 输出一个整数，代表总共能组成多少个重组字符串
>
> **样例**
>
> **输入**
>
> ```none
> 3
> ab
> ca
> ccb
> ```
>
> **输出**
>
> ```none
> 2
> ```
>
> **样例解释**
>
> 能有acb和bac这2个重组字符串

**回溯：**

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	strs := [][]byte{}
	for i := 0; i < n; i++ {
		str, _ := sc.ReadString('\n')
		str = strings.TrimRight(str, "\r\n")
		strs = append(strs, []byte(str))
	}
	test2(strs)
}

func test2(strs [][]byte) {
	res := 0
	for s := 0; s < len(strs); s++ {
		sort.Slice(strs[s], func(i, j int) bool {
			return strs[s][i] < strs[s][j]
		})
	}
	hashMap := map[byte]bool{}
	var backTrack func(height int)
	backTrack = func(height int) {
		if height == len(strs) {
			res++
			return
		}
		for i := 0; i < len(strs[height]); i++ {
			if !hashMap[strs[height][i]] && (i == 0 || strs[height][i-1] != strs[height][i]) {
				hashMap[strs[height][i]] = true
				backTrack(height + 1)
				delete(hashMap, strs[height][i])
			}
		}
	}
	backTrack(0)
	fmt.Println(res)
}
```

#### 2023.03.26-第三题-构造最小值数组

> **题目：**塔子哥有两个长度为 N 的整数数组 A 和 B。B 是一个权值数组，每个元素都是 0，1 或 2。
>
> 他想玩一个游戏，找一个 1 到 N 的排列 C，满足以下条件：
>
> 1. 若 b*i*>*bj* ，则 ci*>*cj 。
> 2. C 和 A 的每个元素之差的绝对值之和 *x* 要最小 。
>
> 问 *x* 的最小值为多少。
>
> **输入描述**
>
> 第一行输入一个整数N
>
> 第二行输入N个正整数，每个数代表数组A的元素
>
> 第三行输入N个整数，每个数代表数字B的元素，范围为[0，2]
>
> 1⩽*N*⩽2∗105
>
> 1⩽*A*[*i*]⩽109
>
> 0⩽*B*[*i*]⩽2
>
> **输出描述**
>
> 输出 *x* 的最小值。
>
> **样例**
>
> **样例** 1
>
> **输入**
>
> ```none
> 2
> 1 10
> 1 0
> ```
>
> **输出**
>
> ```none
> 10
> ```
>
> **样例解释**
>
> 当 *i*=0,*j*=1 时，*i*<*j*，*B*[*i*]=1>*B*[*j*]=0，所以只能为
>
> *C*[*i*]=2,*C*[*j*]=1，故∣1−2∣+∣10−1∣=10，*x*=10
>
> **样例** 2
>
> **输入**
>
> ```none
> 4
> 2 1 4 2
> 2 2 2 2
> ```
>
> **输出**
>
> ```none
> 1
> ```
>
> **样例解释**
>
> C 数组可以为 [2,1,4,3] ，故 *x*=1
>
> **样例** 3
>
> **输入**
>
> ```none
> 6
> 100 2 3 1 5 6
> 0 1 2 0 2 1
> ```
>
> **输出**
>
> ```none
> 104
> ```

贪心：

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	aNums, bNums := []int{}, []int{}
	for i := 0; i < n; i++ {
		var num int
		fmt.Fscan(sc, &num)
		aNums = append(aNums, num)
	}
	fmt.Fscanln(sc)
	for i := 0; i < n; i++ {
		var num int
		fmt.Fscan(sc, &num)
		bNums = append(bNums, num)
	}
	fmt.Fscanln(sc)
	test3(n, aNums, bNums)
}

func test3(n int, aNums, bNums []int) {
	v := [3][]int{} // 按 b 数组 0、1、2 分组
	for i := 0; i < n; i++ {
		v[bNums[i]] = append(v[bNums[i]], aNums[i])
	}
	res := 0
	c := 0
	for i := 0; i < 3; i++ {
		sort.Ints(v[i])
		for j := 0; j < len(v[i]); j++ {
			c++
			res += abs(v[i][j], c)
		}
	}
	fmt.Println(res)
}

func abs(a, b int) int {
	if a < b {
		return b - a
	}
	return a - b
}
```

#### 2023.03.26-第四题-子数组异或和

> **题目：**塔子哥有一个正整数数组 *A* ，他想玩一个游戏，找出数组中有多少个连续的子数组，满足以下条件： 子数组中的所有数字相乘的结果和相异或的结果相等。
>
> 每有一个满足条件的子数组即得一分，问塔子哥最多能得到多少分？
>
> 一个数组的子数组指数组中非空的一段连续数字。
>
> **输入描述**
>
> 第一行一个正整数 *n*，代表给出数组长度
>
> 第二行 *n* 个空格分隔的正整数 *A*;
>
> 1⩽*n*⩽10^5^
>
> 1⩽*Ai*⩽10^9^
>
> **输出描述**
>
> 输出一个正整数代表答案
>
> **样例**
>
> **输入**
>
> ```none
> 3
> 1 2 1
> ```
>
> **输出**
>
> ```none
> 4
> ```
>
> **样例解释**
>
> 数组 [1],[2],[1],[1,2,1],[2],[1],[1,2,1] 都是满足条件的。

**思路：**

乘法的增长要比异或快得多，`a * 1 = a`，`a ^ 1 = a + 1、a - 1`，这就要求这几个数字中只能**由 1 个数字**和**偶数个 1** 组成。

滚吧，不知道怎么写，下边的代码有问题，有时间看看吧。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	nums := []int64{}
	for i := 0; i < n; i++ {
		var num int64
		fmt.Fscan(sc, &num)
		nums = append(nums, num)
	}
	test4(nums)
}

func test4(nums []int64) {
	t := []int{}
	index := 0
	for index < len(nums) {
		count := 0
		if nums[index] == 1 {
			for index < len(nums) && nums[index] == 1 {
				count++
				index++
			}
		} else {
			index++
		}
		t = append(t, count) // 0: 表示该位置是 >1 的数字, >0 : 表示该位置是连续 1 的个数
	}
	res := 0
	for i := 0; i < len(t); i++ {
		temp := 0
		if t[i] == 0 {
			pre, suf := 0, 0 // 统计前后的 1 的个数
			if i > 0 {
				pre = t[i-1]
			}
			if i+1 < len(t) {
				suf = t[i+1]
			}
			temp = (pre/2+1)*(suf/2+1) + (pre+1)/2*(suf+1)/2 // 左边的偶数*右边的偶数 + 左边的奇数*右边的奇数
		} else {
			if t[i]&1 == 0 {
				temp = (t[i] + 2) / 2 * t[i] / 2
			} else {
				temp = (t[i] + 1) / 2 * (t[i] + 1) / 2
			}
		}
		res += temp
	}
	fmt.Println(res)
}
```

## 华为

#### [2022.11.9-华为-攻城战](http://101.43.147.120/p/P1025):star:

> **题目：**现在塔子哥有火药枪若干， 以及数量有限的火药，每种火药枪的威力不尽相同，且在每次开火之前都需要一定时间填充火药， 请你帮助塔子哥在给定的时间结束之前或者火药存量耗尽之前给予敌人最大的伤害。
>
> 限制:
>
> 1. 火药枪每次开火的威力一样；
> 2. 火药剩余量不小于火药枪的消耗量，该火药枪才能开火；
> 3. 填充火药之外的时间忽略不计；
> 4. 不同种火药枪可以同时开火。
>
> #### 输入描述
>
> 第一行，整数 *N* ， *M* ， *T* ， *N* 表示火药枪种类个数， *M* 表示火药数量， *T* 表示攻城时间，1≤*N*,*M*,*T*≤1000。
>
> 接下来 *N* 行，每一行三个整数 *A* ， *B* ， *C* 。分别表示火药枪的威力，火药枪每次攻击消耗的火药量，火药枪每次攻击填充火药的时间，0≤*A*,*B*,*C*≤100000。
>
> #### 输出描述
>
> 输出在给定的时间结束之前或者火药存量耗尽之前给予敌人最大的伤害。
>
> #### 样例
>
> #### 样例一：
>
> **输入**
>
> ```none
> 3 88 30
> 10 7 5
> 5 3 1
> 4 4 8
> ```
>
> **输出**
>
> ```none
> 145
> ```
>
> **样例解释**
>
> 总共有 33 种火药枪，火药存量 8888 ， 攻城时间 3030 ;
>
> 第 11 种火药枪威力 1010 ，每次攻击消耗火药 77 ，每次攻击填充火药时间 55 ；
>
> 第 22 种火药枪威力 55 ，每次攻击消耗火药 33 ，每次攻击填充火药时间 11 ；
>
> 第 33 种火药枪威力 44 ，每次攻击消耗火药 44 ，每次攻击填充火药时间 88 ；
>
> #### 样例二：
>
> **输入**
>
> ```none
> 2 10 15
> 2 2 2
> 3 3 3
> ```
>
> **输出**
>
> ```none
> 10
> ```
>
> **样例解释**
>
> 总共有 22 种火药枪，火药存量 1010 ，攻城时间 1515 ；
>
> 第 11 种火药枪威力 22 ，每次攻击消耗火药 22 ，每次攻击填充火药时间 22 ；
>
> 第 22 种火药枪威力 33 ，每次攻击消耗火药 33 ，每次攻击填充火药时间 33 ；

**思路：**

若不能同时开炮，就是**完全背包**；若能同时开炮，就是**多重背包**。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	var n, m, t int
	fmt.Fscanln(sc, &n, &m, &t)
	guns := [][3]int{}
	for i := 0; i < n; i++ {
		var a, b, c int
		fmt.Fscanln(sc, &a, &b, &c)
		guns = append(guns, [3]int{a, b, c})
	}
	test3(m, t, guns)
}

// 一个大炮可以发射若干枚，并且不同大炮可以同时发射，若该条件是不同大炮不能同时发射，转换为二维完全背包问题，时间和弹药是消耗量。
// 完全背包
//func test3(m, t int, guns [][3]int) {
//	dp := make([][]int, m+1)
//	for i := 0; i < len(dp); i++ {
//		dp[i] = make([]int, t+1)
//	}
//	for _, gun := range guns {
//		for i := gun[1]; i < len(dp); i++ {
//			for j := gun[2]; j < len(dp[i]); j++ {
//				dp[i][j] = max(dp[i][j], dp[i-gun[1]][j-gun[2]]+gun[0])
//			}
//		}
//	}
//	fmt.Println(dp[m][t])
//}

// 因为能同时开炮，时间就只能对单个炮进行约束了，进而转换成多重背包(01背包)
func test3(m, t int, guns [][3]int) {
	dp := make([]int, m+1)
	for _, gun := range guns {
		for i := m; i >= gun[1]; i-- { // 01背包从后往前，完全背包从前往后
			times := min(i/gun[1], t/gun[2]) // 得到该炮的最大发射次数
			for j := 1; j <= times; j++ {
				dp[i] = max(dp[i], dp[i-j*gun[1]]+j*gun[0])
			}
		}
	}
	fmt.Println(dp[m])
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

#### 2023.04.12-第一题-购物系统的降级策略

> **题目：**在一个购物APP中，有一个核心购物系统，它的接口被 *N* 个客户端调用。这些客户端负责处理来自不同渠道的交易请求，并将这些请求发送给核心购物系统。每个客户端有不同的调用量 *R*=[*R*1,*R*2,...,*RN*]，表示在一定时间内，这个客户端向核心购物系统发送的交易请求的数量。核心购物系统必须能够及时响应所有的请求，以确保交易顺利进行。
>
> 然而，最近核心购物系统出现了集群故障，导致交易请求的处理速度变慢。为了避免系统崩溃，必须临时降级并限制调用量。具体而言，核心购物系统能接受的最大调用量为 *c**n**t*，如果客户端发送的请求总量超过 *c**n**t*，则必须限制一些系统的请求数量，以确保核心购物系统不会超负荷工作。
>
> 现在需要一个降级规则，来限制客户端的请求数量。规则如下：
>
> - 如果 *s**u**m*(*R*1,*R*2,...,*RN*) 小于等于 *c**n**t* ，则全部可以正常调用，返回 −1；
> - 如果 *s**u**m*(*R*1,*R*2,...,*RN*) 大于 *c**n**t*，则必须设定一个阈值 *v**a**l**u**e*，如果某个客户端发起的调用量超过 *v**a**l**u**e*，则该客户端的请求数量必须限制为 *v**a**l**u**e*。其余未达到 *v**a**l**u**e* 的系统可以正常发起调用。要求求出最大的 *v**a**l**u**e*（*v**a**l**u**e* 可以为0）。
>
> 为了保证交易的顺利进行，必须保证客户端请求的数量不会超过核心购物系统的最大调用量，同时最大的 *v**a**l**u**e* 要尽可能的大。需要高效地解决这个问题，以确保购物系统的高效性。
>
> **输入描述**
>
> 第一行：每个客户端的调用量(整型数组)
>
> 第二行：核心购物系统的最大调用量
>
> 0<*R*.*l**e**n**g**th*≤10^5^，0≤*R*[*i*]≤10^5^，0≤*c**n**t*≤10^9^
>
> **输出描述**
>
> 调用量的阈值 *v**a**l**u**e*
>
> **样例**
>
> **样例一**
>
> **输入**
>
> ```none
> 1 4 2 5 5 1 6 
> 13
> ```
>
> **输出**
>
> ```none
> 2
> ```
>
> **样例解释**
>
> 因为 1+4+2+5+5+1+6>13 ，将 *v**a**l**u**e* 设置为 22 ，则 1+2+2+2+2+1+2=12<13 。所以 *v**a**l**u**e* 为 22 。
>
> **样例二**
>
> **输入**
>
> ```none
> 1 7 8 8 1 0 2 4 9
> 7
> ```
>
> **输出**
>
> ```none
> 0
> ```
>
> **样例解释**
>
> 因为即使 *v**a**l**u**e* 设置为 1 , 1+1+1+1+1+1+1+1=8>7 也不满足，所以 *v**a**l**u**e* 只能为 0 。

**二分查找：**

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	rs := []int{}
	str, _ := sc.ReadString('\n')
	strs := strings.Split(strings.TrimRight(str, "\r\n"), " ")
	for _, str := range strs {
		num, _ := strconv.Atoi(str)
		rs = append(rs, num)
	}
	var cnt int
	fmt.Fscanln(sc, &cnt)
	test1(cnt, rs)
}

func test1(cnt int, rs []int) {
	sort.Ints(rs)
	prefix := 0
	prefixs := []int{}
	for i := 0; i < len(rs); i++ {
		prefix += rs[i]
		prefixs = append(prefixs, prefix)
	}
	var isValid func(val int) bool
	isValid = func(val int) bool {
		index := sort.Search(len(rs), func(i int) bool {
			return val < rs[i]
		})
		sum := 0
		if index == 0 {
			sum = len(rs) * val
		} else {
			sum = prefixs[index-1] + (len(rs)-index)*val
		}
		return sum <= cnt
	}
	left, right := 0, cnt+1
	for left < right {
		mid := (right-left)/2 + left
		if isValid(mid) {
			left = mid + 1
		} else {
			right = mid
		}
	}
	if right == cnt+1 {
		right = 0
	}
	fmt.Println(right - 1)
}
```

#### 2023.04.13-第二题-获取最多食物

> **题目：**塔子哥设计的这个游戏是一个冒险类游戏，参与者需要在地图上寻找食物并获得尽可能多的食物，同时需要注意在游戏过程中所处的位置，因为不同的位置可以通过传送门到达其他位置，可能会影响食物获取的数量。
>
> 在游戏开始时，参与者会出发点选择一个方格作为起点，每个方格上至多 22 个传送门，通过传送门可将参与者传送至指定的其它方格。每个方格上都标注了三个数字：`id` 、 `parent-id` 和 `value` 。其中， `id` 代表方格的编号， `parent-id` 代表可以通过传送门到达该方格的方格编号， `value` 代表在该方格获取或失去的食物单位数。
>
> 参与者需要在地图上行进，到达每个方格并获取或失去对应的食物单位数，直到满足退出游戏的条件之一。参与者的最终得分是所获取食物单位数的总和，需要尽可能地高。
>
> 需要注意的是地图设计时保证了参与者不可能到达相同的方格两次，并且至少有一个方格的 �����*v**a**l**u**e* 是正整数，因此，参与者当前所处的方格无传送门，游戏将立即结束。另外，参与者在任意方格上都可以宣布退出游戏，同样会结束游戏。
>
> 请计算参与者退出游戏后，最多可以获得多少单位的食物。
>
> **输入描述**
>
> 第一行：方块个数 *N* ( *N*≤10000 )
>
> 接下来 *N* 行，每行三个整数 `id` ， `parent-id` ， `value` ，具体含义见题面。
>
> 0≤*id*, *p**a**re**n**t*−*id*<*N* ，−100≤*v**a**l**u**e*≤100
>
> 特殊的 `parent-id` 可以取 −1−1 则表示没有任何方格可以通过传送门传送到此方格，这样的方格在地图中有且仅有一个。
>
> **输出描述**
>
> 输出为一个整数，表示参与者退出游戏后最多可以获得多少单位的食物。
>
> **样例**
>
> **样例** 1
>
> **输入**
>
> ```none
> 7
> 0 1 8
> 1 -1 -2
> 2 1 9
> 4 0 -2
> 5 4 3
> 3 0 -3
> 6 2 -3
> ```
>
> **输出**
>
> ```none
> 9
> ```
>
> **样例解释**
>
> 参与者从方格 0 出发，通过传送门到达方格 4 ，再通过传送门到达方格 5 。一共获得 8+(−2)+3=9 个单位食物，得到食物最多或者参与者在游戏开始时处于方格 2 ，直接主动宣布退出游戏，也可以获得 9 个单位食物。
>
> **样例** 2
>
> **输入**
>
> ```none
> 3
> 0 -1 3
> 1 0 1
> 2 0 2
> ```
>
> **输出**
>
> ```none
> 5
> ```
>
> **样例解释**
>
> 参与者从方格 0 出发，通过传送门到达方格 2 ，一共可以获得 3+2=5 个单位食物，此时得到食物最多。

**简单动态规划**，就是有点烦，注意下标和节点的对应关系。

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	ids, parents, vals := []int{}, []int{}, []int{}
	for i := 0; i < n; i++ {
		var id, parent, val int
		fmt.Fscanln(sc, &id, &parent, &val)
		ids = append(ids, id)
		parents = append(parents, parent)
		vals = append(vals, val)
	}
	test2(ids, parents, vals)
}

func test2(ids, parents, vals []int) {
	start := -1
	nodeParent := make([]int, len(parents))
	nodeVals := make([]int, len(vals))
	graph := make([][]int, len(ids))
	for i := 0; i < len(ids); i++ {
		nodeVals[ids[i]] = vals[i]
		nodeParent[ids[i]] = parents[i]
		if parents[i] == -1 {
			start = ids[i]
			continue
		}
		graph[parents[i]] = append(graph[parents[i]], ids[i])
	}
	queue := []int{start}
	dp := make([]int, len(ids))
	for len(queue) > 0 {
		cur := queue[0]
		queue = queue[1:]
		if nodeParent[cur] == -1 {
			dp[cur] = nodeVals[cur]
		} else {
			dp[cur] = max(dp[nodeParent[cur]]+nodeVals[cur], nodeVals[cur])
		}
		for _, next := range graph[cur] {
			queue = append(queue, next)
		}
	}
	res := math.MinInt
	for i := 0; i < len(dp); i++ {
		res = max(res, dp[i])
	}
	fmt.Println(res)
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

#### 2023.04.19-第一题-塔子哥监考

> **题目：**塔子哥是一个强大严厉的监考机器人，有他监考的考场总能抓到很多不不听话的学生，塔子哥”手眼通天“，能够同时仔细的观察很多考场，每个学生的一举一动他都能尽收眼底。
>
> 很多学校都想使用塔子哥监督考试，而塔子哥的使用成本也非常昂贵，当只监督一个考场时，每监视一分钟收费3金币；同时监视两个以上的考场，每监视一分钟收费4金币；当处于两个监考任务之间的空隙之间时，塔子哥会进入待机状态，每分钟消耗1金币。也就是说，直到完成最后一个监考任务之前，塔子哥机器人都会持续消耗金币。
>
> 今天塔子哥又来监考，他今天一共要监视n个不同的考场，每个考场的考试开始时间和结束时间都不同，为简化表达，将时间简化为整数代表的时间单位，时间从0开始。
>
> 请你计算塔子哥今天的监考任务能够收取多少金币？
>
> **输入描述**
>
> 第一行一个整数 *n* 表示塔子哥今天的监考考场数量
>
> 接下来 *n* 行，给出由空格分开的两个整数*ai*,*bi*。表示 *n* 个考场考试的开始时间 *ai* 与结束时间 *bi* ，也就是塔子哥开始监考与结束监考的时间（闭区间），保证结束时间大于起始时间
>
> 1⩽*n*⩽10000
>
> 0⩽*ai*,*bi*⩽10^6^
>
> **输出描述**
>
> 一个整数，代表塔子哥今天监考所能赚取的金币
>
> **样例**
>
> **输入**
>
> ```none
> 3
> 1 5
> 4 6
> 6 6
> ```
>
> **输出**
>
> ```none
> 21
> ```

扫描线

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	tasks := make([][2]int, n)
	for i := 0; i < n; i++ {
		var start, end int
		fmt.Fscanln(sc, &start, &end)
		tasks[i] = [2]int{start, end}
	}
	test4(n, tasks)
}

func test4(n int, tasks [][2]int) {
	end := 0
	for _, task := range tasks {
		if task[1] > end {
			end = task[1]
		}
	}
	taskNums := make([]int, end+2) // [0, end+1]
	for _, task := range tasks {
		taskNums[task[0]]++
		taskNums[task[1]+1]--
	}
	res := 0
	curTaskNum := 0
	start := false
	for i := 0; i < end+1; i++ {
		curTaskNum += taskNums[i]
		if !start && curTaskNum > 0 {
			start = true
		}
		if start {
			if curTaskNum == 0 {
				res += 1
			} else if curTaskNum == 1 {
				res += 3
			} else if curTaskNum > 1 {
				res += 4
			}
		}
	}
	fmt.Println(res)
}
```

#### 2023-04-19-第二题-塔子哥出城

> **题目：**塔子哥居住在数据结构之城，如果将这个城市的路口看做点，两个路口之间的路看做边，那么该城市的道路能够构成一棵由市中心路口向城市四周生长的树，树的叶子节点即是出城口。
>
> 塔子哥今天想要出城办事，但不巧的是，有几个路口堵车了，塔子哥无法从一个正常的路口前往堵车的路口。假定塔子哥从一个正常的路口出发，请问塔子哥能否顺利出城（到达出城口）？如果可以，请帮塔子哥找到最省油的路径（经过路口最少的路径），否则请输出“NULL”。
>
> **输入描述**
>
> 第一行给出数字n，表示这个城市有n个路口，路口从0开始依次递增，0固定为根节点，1<=n<10000
>
> 第二行给出数字m，表示接下来有m行，每行是一条道路
>
> 接下来的m行是边: x，y，表示x和y路口有一条道路连接。保证是一颗树
>
> 道路信息结束后接下来的一行给出数d，表示接下菜有d行，每行是一个堵车的路口
>
> 接下来的d行是堵车路口k，表示路口k已堵车
>
> **输出描述**
>
> 如果塔子哥能够顺利出城，请输出塔子哥能够到达任意一个出城口的最短路径（通过路口最少），比如塔子哥从0经过1到达2 (出城口) ，那么输出“0->1->2”;否则输出“NULL”。注意如果存在多条最短路径，请按照节点序号排序输出，比如 0->1 和 0-> 3两条路径，第一个节点0一样，则比较第二个节点1和3，1比3小，因此输出0->1这条路径。再如0->5->2->3和 0->5->1->4，则输出 0->5->1->4。
>
> **样例**
>
> **输入**
>
> ```none
> 4
> 3
> 0 1
> 0 2
> 0 3
> 2
> 2
> 3
> ```
>
> **输出**
>
> ```none
> 0->1
> ```
>
> **说明**
>
> n=4, edge=[[0,1],[0,2], [0.3], block=[2, 3]] 表示一个有4个节点，3条边的树，其中节点2和节点3上有障碍物，小猴子都能从01到达叶子节点1（节点1只有一条边[0,1]和它连接，因此也是叶子节点），即可以跑出这个树，所以输出为0->1.

BFS求最短距离，DFS求该路径

```go
func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	var edgeNum int
	fmt.Fscanln(sc, &edgeNum)
	edges := [][]int{}
	for i := 0; i < edgeNum; i++ {
		var x, y int
		fmt.Fscanln(sc, &x, &y)
		edges = append(edges, []int{x, y})
	}
	var blockNum int
	fmt.Fscanln(sc, &blockNum)
	blocks := []int{}
	for i := 0; i < blockNum; i++ {
		var x int
		fmt.Fscanln(sc, &x)
		blocks = append(blocks, x)
	}
	test5(n, edges, blocks)
}

func test5(n int, edges [][]int, blocks []int) {
	// 1. 建图
	graph := map[int][]int{}
	for _, edge := range edges {
		graph[edge[0]] = append(graph[edge[0]], edge[1])
	}
	// 2. block
	blockMap := map[int]bool{}
	for _, block := range blocks {
		blockMap[block] = true
	}
	if blockMap[0] {
		fmt.Println("NULL")
		return
	}
	// 3. BFS 先得到最短长度
	queue := []int{0}
	nextQueue := []int{}
	flag := false
	length := 1
	for len(queue) > 0 {
		cur := queue[0]
		queue = queue[1:]
		// 到达叶子节点
		if len(graph[cur]) == 0 {
			flag = true
			break
		}
		for _, next := range graph[cur] {
			if _, ok := blockMap[next]; !ok {
				nextQueue = append(nextQueue, next)
			}
		}
		if len(queue) == 0 {
			queue = nextQueue
			nextQueue = []int{}
			length++
		}
	}
	if !flag {
		fmt.Println("NULL")
		return
	}
	// 4. dfs 寻找路径
	path := []int{}
	var dfs func(curNode int) bool
	dfs = func(curNode int) bool {
		path = append(path, curNode)
		if len(path) == length && len(graph[curNode]) == 0 {
			return true
		}
		// 剪枝
		if len(path) == length {
			path = path[:len(path)-1]
			return false
		}
		for _, next := range graph[curNode] {
			if _, ok := blockMap[next]; !ok {
				if dfs(next) {
					return true
				}
			}
		}
		path = path[:len(path)-1]
		return false
	}
	dfs(0)
	for i := 0; i < len(path); i++ {
		if i < len(path)-1 {
			fmt.Printf("%v->", path[i])
		} else {
			fmt.Printf("%v", path[i])
		}
	}
}
```

## 华为 od



## 字节跳动

#### [2023.03.24-第一题-罐装知识](http://101.43.147.120/p/P1112)

> **题目：**多莉的罐装知识自选零元购活动开始啦！未来的骑士艾琳想要趁机入手一波罐装知识来提升自己的战力。但是多莉做生意一向狡诈，只有按照她规定的方式才能零元带走罐装知识。 在多莉的地摊上，有n瓶罐装知识排成一列，每瓶罐装知识有两个属性，用 (*weighti*,*skipi*) 标识，weighti 表示第i瓶罐装知识的重量（单位坷垃），*skipi*表示拿完第i瓶罐装知识后，就必须跳过接下来的*skipi*瓶罐装知识（即 [*i*+1，*skipi*+*i*] 范围罐装知识），不能将它们收入口袋，但是你也可以选择不拿某瓶罐装知识。而且我们经过每瓶罐装知识的时候都必须做出拿走或者不拿的选择，不能回头。
>
> 举个例子，给你罐装知识＝［［2,2］，［1,1］，［1,1］］：如果罐装知识0被拿起了，那么你可以获得2坷垃的罐装知识，但是你不能拿走罐装知识1和2。
>
> 如果你不拿罐装知识0，把罐装知识1拿起了，你可以得到1坷垃罐装知识，但是你不能带走罐装知识2了。当然你可以跳过罐装知识0和1，直接拿走罐装知识2，那么你获得的也是1坷垃的罐装知识。
>
> 艾琳认为知识是有重量的，越重一定越有价值，她希望聪明的你帮她算一下她最多能带走多重的罐装知识呢？
>
> **输入描述**
>
> 首先输入一个(1⩽*n*⩽100000)代表罐装知识的数量
>
> 然后是 *n* 瓶罐装知识的(*weighti*,*skipi*)
>
> (1⩽*weighti*,*skipi*⩽100000)
>
> **输出描述**
>
> 输出艾琳能带走的罐装知识的最大重量
>
> **样例**
>
> **输入**
>
> ```none
> 3
> 2 2
> 1 1
> 1 1
> ```
>
> **输出**
>
> ```none
> 2
> ```

**思路**：

倒序 DP

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	var n int
	fmt.Fscanln(sc, &n)
	acks := [][]int{}
	for i := 0; i < n; i++ {
		var weight, skip int
		fmt.Fscanln(sc, &weight, &skip)
		acks = append(acks, []int{weight, skip})
	}
	test1(acks)
}

func test1(acks [][]int) {
	dp := make([]int, len(acks)+1)
	// dp[i] 表示 [i: ] 最大结果
	for i := len(dp) - 2; i >= 0; i-- {
		next := i + acks[i][1] + 1
		if next < len(dp) {
			dp[i] = max(dp[i+1], dp[next]+acks[i][0])
		} else {
			dp[i] = max(dp[i+1], acks[i][0])
		}
	}
	fmt.Println(dp[0])
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```

## 携程

#### 2023.04.15-第二题-最小公倍数

> **思路：**很久很久以前，有一个古老的国度，国王和王后十分仁慈，善于施行公正。然而，这个国度一直存在着一个问题：财政困难。于是，国王决定组织一场大规模的征税行动，以减轻财政压力。
>
> 为了避免对民众造成过大的负担，国王命令在每个家庭中只征收一次税款，而且税款数额应该尽可能的小，以不影响民生。因此，国王命令他的财务大臣塔子哥，设计一种算法，使得征税的税款尽可能小，但仍能够收取足够的财政收入。
>
> 塔子哥经过多次思考和试验，最终发现，如果每个家庭的税款数额为该家庭两个人的年龄之和，那么所得税款总和最小，这是因为家庭中年龄相近的人往往收入也相近。然而，由于不同家庭的人的年龄各不相同，为了使征税的税款尽可能小，塔子哥希望在每个家庭中选择两个年龄差距最小的人，作为该家庭的代表，计算出税款数额。
>
> 在进行计算的时候，塔子哥面临着一个问题：**如何选取两个数，使得这两个数的和为n，且它们的最小公倍数尽可能大**？
>
> **输入描述**
>
> 第一行输入一个正整数 *t* ，代表询问的次数。
>
> 对于每组询问，输入一行一个正整数 *n* 。
>
> 1≤*t*≤105，2≤*n*≤1013
>
> **输出描述**
>
> 共输出 *t* 行。对于每组询问，输出一行两个正整数 *a* 和 *b* ，用空格隔开。
>
> **样例**
>
> **输入**
>
> ```none
> 5
> 2
> 5
> 4
> 7
> 10
> ```
>
> **输出**
>
> ```none
> 1 1
> 2 3
> 1 3
> 3 4
> 3 7
> ```

**思路：**

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	sc := bufio.NewReader(os.Stdin)
	var t int
	fmt.Fscanln(sc, &t)
	for i := 0; i < t; i++ {
		var n int64
		fmt.Fscanln(sc, &n)
		test1(n)
	}
}

func test1(n int64) {
	left, right := n/2, (n+1)/2
	for 0 < left && right < n {
		if gcd(left, right) == 1 {
			break
		}
		left--
		right++
	}

	fmt.Println(left, right)
}

func gcd(a, b int64) int64 {
	if b == 0 {
		return a
	}
	return gcd(b, a%b)
}
```



## 杂项

#### [圣诞树](https://www.nowcoder.com/practice/9a03096ed8ab449e9b10b0466de29eb2)

```go
func main() {
	var n int
	fmt.Scanln(&n)
	test(n)
}

func test(n int) {
	// 打印 n 层树
	for i := 0; i < n; i++ {
		d := 3 * (n - i) // d 标记每行最开始的空格数
		// 打印树的第 1 层
		fmt.Println(strings.Repeat(" ", d-1) + strings.Repeat("*     ", i+1)) // 5 个空格
		// 打印树的第 2 层
		fmt.Println(strings.Repeat(" ", d-2) + strings.Repeat("* *   ", i+1)) // 3 个空格
		// 打印树的第 3 层
		fmt.Println(strings.Repeat(" ", d-3) + strings.Repeat("* * * ", i+1)) // 1 个空格
	}
	// 打印树根
	for i := 0; i < n; i++ {
		fmt.Println(strings.Repeat(" ", 3*n-1) + "*")
	}
}
```


