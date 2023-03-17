---
title: dp
date: 2023-03-17 17:39:30
tags: c++
---

# DP(动态规划)-背包

## 01背包

* 题目

  有 N 件物品和一个容量是 V 的背包。每件物品只能使用一次。第 i 件物品的体积是 v~i~，价值是 w~i~。

  求解将哪些物品装入背包，可使这些物品的总体积不超过背包容量，且总价值最大。输出最大价值。

* 为什么称作01背包，因为对于每个物品，只对应选或者不选两种状态，因此称为01背包

**主要思想：**

* 状态表示: 我们用`f[i][j]`来表示，前`i`个物品，最大容量是`j`，可以得到的最大价值
* 状态计算:
  * 对于`f[i][j]`，第 i 个物品，我们有两种选择
    * 不选他: `f[i][j]=f[i-1][j]`，将第 i 个物品剔除然后选最大
    * 选他: `f[i][j]=f[i-1][j-v[i]]+w[i]`，先将第 i 个物品放入背包，然后在前 i-1 个物品，容量为 j -v[i]的状态下找最大
  * 因此得出状态方程: ` f[i][j] = max(f[i-1][j],f[i-1][j-v[i]]+w[i])`

<img src="https://img-blog.csdnimg.cn/119dcd0622ed424783892acdc976a3f5.png" alt="img" style="zoom: 50%;" />

### 二维朴素版

```C++
#include<iostream>
using namespace std;

const int N = 1010;
int v[N],w[N];
int f[N][N];
//对于f[i][j]，当i或者j为0时，f[i][j] = 0

int main()
{
    int n,m;
    cin>>n>>m;
    for(int i = 1 ; i <= n ; i++) cin>>v[i]>>w[i];
    for(int i = 1 ; i <= n; i++ ){
        for(int j = 0 ; j <= m ;j++){
            // 当前背包容量装不进第i个物品，则价值等于前i-1个物品
            if(j < v[i]) f[i][j] = f[i-1][j];
            // 可以装第i个物品，决策是否选他，用max来比较
            else f[i][j] = max(f[i-1][j],f[i-1][j-v[i]]+w[i]);
        }
    }
    cout<<f[n][m]<<endl;
}
```

### 一维终极版

可以看做我们将 i-1 层的拷贝下来





01背包从大到小

完全背包从小到大
