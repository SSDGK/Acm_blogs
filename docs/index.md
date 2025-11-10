##早上好
# 动态规划

_Last updated: {{ page.git_revision_date }}_  

## 问题描述
给定一个序列，求最大子段和。

## 思路
- 定义 dp[i] 为以 i 结尾的最大子段和
- 状态转移：dp[i] = max(dp[i-1]+a[i], a[i])

``` mermaid
graph LR
  A[Start] --> B{Error?};
  B -->|Yes| C[Hmm...];
  C --> D[Debug];
  D --> B;
  B ---->|No| E[Yay!];
```