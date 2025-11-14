# 最短路树

最短路树，~~就是和最短路有关的树。~~（废话）在$Dijkstra$单源最短路中，每个节点（除了源点）的$dis$值都会被图上的另外一个点更新，我们将使得目标点的$dis$值更新到最小时的当前点作为目标点在最短路树上的父节点，这样我们就构建了一棵最短路树。

如下左图，我们求以1为源点的单源最短路，得到右图所示的最短路树。
<table>
    <tr>
    <td><img src="https://i.ibb.co/xq1mzSV6/2025-11-13-185226.png" alt="原图" width="300"></td>
    <td><img src="https://i.ibb.co/5hT3fZYL/2025-11-13-190331.png" alt="最短路树" width="300"></td>
    </tr>
</table>

那么这棵树有什么用呢，首先很明显，它显示了源点到不同点最短路径的相交关系：如点4和点5，他们在最短路树中的$lca$为点2，我们来看原图，显然点4和点5的最短路径也正是最后相交于点2。所以我们可以使用比如倍增的方法快速求出两路径交点。

所以其实最短路树是原图的一棵生成树，但注意其不一定是最小生成树。

## 例题1（模板）：[CF545E Paths and Trees](https://www.luogu.com.cn/problem/CF545E)

题目描述：给出一个带权无向连通图，求边权和最小的最短路树。

最短路树好求，边权和最小怎么做呢。考虑我们在Dij的过程中，一个点要能被松弛才更新，为了使得边权和最小，我们在松弛操作的$w$和现在的$dis$相等且当前的边权更小时，也单独去更新一下目标点的父节点为当前节点。


```cpp
void solve() {
    int n, m;
    cin >> n >> m;
    vector<int> weight(m + 1);
    vector<vector<array<int, 3>>> graph(n + 1);
    rep(i, 1, m) {
        int u, v, w;
        cin >> u >> v >> w;
        weight[i] = w;
        graph[u].push_back({v, w, i});
        graph[v].push_back({u, w, i});
    }
    int s;
    cin >> s;
    vector<bool> chosen(m + 1, false), vis(n + 1);
    vector<int> fa(n + 1), line(n + 1, 0);
    vector<ll> dis(n + 1, LLONG_MAX);
    priority_queue<pll, vector<pll>, greater<>> pq;
    dis[s] = 0;
    pq.push({0, s});
    while(!pq.empty()) {
        auto [cw, cur] = pq.top();
        pq.pop();
        if(vis[cur]) {
            continue;
        }
        vis[cur] = true;
        for(auto [v, w, t]: graph[cur]) {
            if(!vis[v] && dis[v] > cw + w) {
                dis[v] = cw + w;
                pq.push({dis[v], v});
                fa[v] = cur;
                chosen[line[v]] = false;
                chosen[t] = true;
                line[v] = t;
            } else if(dis[v] == cw + w && weight[line[v]] > w) {
                fa[v] = cur;
                chosen[line[v]] = false;
                chosen[t] = true;
                line[v] = t;
            }
        }
    }
    ll ans = 0;
    rep(i, 1, m) {
        if(chosen[i]) {
            ans += weight[i];
        }
    }
    cout << ans << '\n';
    rep(i, 1, m) {
        if(chosen[i]) {
            cout << i << ' ';
        }
    }
}
```



## 例题2：[P1186 玛丽卡](https://www.luogu.com.cn/problem/P1186)

题目描述：给出一个带权无向连通图，求将任意一条边删去之后，能得到的n到1的最短路长度最大值。

首先我们可以分析发现，删去的边一定是原来的最短路上的边，否则最短路长度不变，一定不优。
于是一个想法是尝试枚举每一条删的边，跑最短路。但这样时间复杂度炸了，无法通过。于是我们转换一下角度，我们来枚举每一条不在原最短路上的边。

<div align="center">
  <img src="https://i.ibb.co/VprQf09k/2025-11-13-232947.png" width="500">
</div>

如图，我们假设n到1的最短路是中间那条，那么我们现在想要求出删去x后的最短路，其必然经过绿色所在的路径或者红色边所在的路径，我们可以使用这两个边去更新x的信息，我们发现x的答案实际上就是经过红绿两条的n到1的路径的长度中较小的那一个。

现在我们就可以依次枚举每一条不在原最短路上的边来更新信息了，可以看到可以更新的信息就是两条路径交的中间部分，是一个连续区间，那么我们可以用线段树来维护区间min操作，最后信息中最大值就是答案。

那么怎么找到$(u, v)$这条边要更新的区间呢，这里就到了最短路树登场的时候了。区间左端点其实就是n到u的最短路与n到1的最短路的第一个交点,区间右端点其实就是1到v的最短路与1到n的最短路的第一个交点，这与我们上面说的最短路树的应用方法一致。

首先别忘了这题要的是n到1的最短路别写反了。实现细节较多，我自己踩了非常多坑，下面我尽力的回忆一下。

- 我写了一个$shortestPathTree$结构体来封装，因为我这个人喜欢写封装，调用方便而且不污染命名空间，还利于多测嘻嘻。
```cpp
struct shortestPathTree {
    int dis[MAXN];
    int st[MAXN][MAXP + 1]{};
    int depth[MAXN]{};

    void build_st() {
        rep(p, 1, MAXP) {
            rep(i, 1, n) {
                st[i][p] = st[st[i][p - 1]][p - 1];
            }
        }
    }

    int lca(int u, int v) {
        if(depth[u] < depth[v]) {
            swap(u, v);
        }
        srep(p, MAXP, 0) {
            if(int fu = st[u][p]; depth[fu] >= depth[v]) {
                u = fu;
            }
        }
        if(u == v) {
            return u;
        }
        srep(p, MAXP, 0) {
            int fu = st[u][p], fv = st[v][p];
            if(fu != fv) {
                u = fu;
                v = fv;
            }
        }
        return st[u][0];
    }
};
```


- 一定要注意边的处理，我们因为要去掉在最短路路径上的边，用剩下的边来更新答案，而本题中是可能有重边的。我用的multiset来处理。
```cpp
int cur = 1;
inpath.insert(1);
while(cur != n) {
    int fa = rfa[cur];
    int w = dis[cur] - dis[fa];

    auto it = ls.find({min(cur, fa), max(cur, fa), w});
    ls.erase(it);
    cur = fa;
    inpath.insert(cur);
}
```


- 首先在分别构建1到n的最短路树和n到1的最短路树时，一定要保证1和n之间的最短路路径保持不变。我这里的实现方法是先进行一次单纯的单源最短路，并记录好1到n路径上的点有哪些。在构建最短路树时如果当前的点不在路径上，就一定不能去更新原路径上的点；如果在最短路路径上，就优先更新也在路径上的邻居，实现如下。
```cpp
void build_path(int s, shortestPathTree& t) {//建立s到每个点的最短路径长度以及最短路树
    int* dis = t.dis, *depth = t.depth;
    fill(dis, dis + n + 1, INT_MAX);
    dis[s] = 0, depth[s] = 1;
    vector<bool> vis(n + 1, false);
    priority_queue<pii, vector<pii>, greater<>> pq;
    pq.push({0, s});
    while(!pq.empty()) {
        auto [cw, cur] = pq.top();
        pq.pop();
        if(vis[cur]) {
            continue;
        }
        vis[cur] = true;
        for(auto [v, w]: graph[cur]) if(inpath.count(cur) && inpath.count(v)) {
            if(!vis[v] && dis[v] > cw + w) {
                dis[v] = cw + w;
                pq.push({cw + w, v});
                t.st[v][0] = cur;
                depth[v] = depth[cur] + 1;
            }
        }
        for(auto [v, w]: graph[cur]) {
            if(!vis[v] && dis[v] > cw + w) if(!inpath.count(v)) {
                dis[v] = cw + w;
                pq.push({cw + w, v});
                t.st[v][0] = cur;
                depth[v] = depth[cur] + 1;
            }
        }
    }
    t.build_st();
}
```


- 边更新的时候要两个方向都更新一遍。
```cpp
auto update = [&](int u, int v, int w) {
    int L = Tn.lca(u, 1);
    int R = T1.lca(v, n);
    int jobv = Tn.dis[u] + T1.dis[v] + w;
    // cout << u << ' ' << v << " : " << L << ' ' << R << ' ' << jobv << '\n';
    st.update(rk[L], rk[R] - 1, jobv, 1, p - 1, 1);
};
for(auto [u, v, w]: ls) {
    update(u, v, w);
    update(v, u, w);
}
```


- 区间取min的实现我用的Segment Tree Beats，也就是吉司机线段树，但看题解区都说用标记永久化，我不知道这是啥……
```cpp

void modify(int i, int k) {
    if(maxn[i] > k) {
        maxn[i] = k;
        tag[i] = k;
    }
}

void update(int jobl, int jobr, int jobv, int l, int r, int i) {
    if(jobl > jobr) {
        return;
    }
    if(maxn[i] <= jobv) {
        return;
    }
    if(jobl <= l && jobr >= r && seg[i] > jobv) {
        modify(i, jobv);
    } else {
        int mid = (l + r) / 2;
        down(i);
        if(jobl <= mid) {
            update(jobl, jobr, jobv, l, mid, i << 1);
        }
        if(jobr > mid) {
            update(jobl, jobr, jobv, mid + 1, r, i << 1 | 1);
        }
        up(i);
    }
}
```

**双倍经验：[P2685 [TJOI2012] 桥](https://www.luogu.com.cn/problem/P2685)，这题记得加一些特判。**

## 例题3：[CF1163F Indecisive Taxi Fee](https://www.luogu.com.cn/problem/CF1163F)

<small>我的第一道3000分难度的题……真不愧是3000，写起来比看起来难飞起来。</small>


其实这题原本可以算是上题三倍经验的，但是实现起来远比想象中难，而且有很多值得一提的技巧，所以单独拎出来记录一下。

在这里首先感谢一下[MaxBlazeRes_Fire](https://codeforces.com/profile/MaxBlazeRes_Fire)的倾囊相授，在写这几道题时我踩了非常多坑，多亏了茵神帮助才能顺利通过。

题目描述：给出一个带权无向连通图。有多次询问，每次询问给出一个序号和新权值，求将序号对应的边的边权修改为新权值后1到n的最短路长度，询问之间互相独立。

首先记原图最短路长度为$d$我们可以分析，他询问的边是否在原图1到n的最短路路径上以及边权的变化情况：
- 不在最短路上，且边权变小：答案就是1到u的距离加新边权再加v到n的距离，再和&d&取一个最小值
- 不在最短路上，且边权变大：那么答案就是$d$
- 在最短路上，且边权变小：答案就是$d$减去原边权和新边权的差值
- 在最短路上，且边权最大：那么我们考虑走不走这条边，答案为不走这条边时的最短路长度和$d$减去原边权和新边权的差值的最小值

我们发现1，2情况之间和3，4情况之间可以合并在一起，而在上一题中我们已经知道了怎么获取不走一条边的最短路长度，问题就解决了……吗？

确实思路没啥问题了，但这道题数据量很大，加上还要回答同等数量级的询问，但时限只有2s。所以首先这道题需要一点卡常，注意多使用数组，快读。还有一个很重要的，上述所说的3和4情况中，线段树上询问是$O(logn)$的，在这题里首先预处理就非常耗时，基本上询问要做到$O(1)$才能通过。我们注意到，询问之间独立，也就是预处理完后我们并不需要再修改线段树了，并且单次询问的还是单点，所以我们只需要预先遍历一下整棵线段树，将懒标记全部下发，然后把叶节点的信息单独拿出来用于查询就好啦。
```cpp
O(1)查询处理
void query(int l, int r, int i) {
    if(l == r) {
        final_ans[l] = maxn[i];
    } else {
        int mid = (l + r) / 2;
        down(i);
        query(l, mid, i << 1);
        query(mid + 1, r, i << 1 | 1);
    }
}
ll query(int jobl) {
    return final_ans[jobl];
}
```
```cpp
查询
st.query(1, p - 1, 1);
while(Q--) {
    int t, x;
    t = read(), x = read();
    t--;
    if(ls.count({tls[t]})) {
        auto [u, v, w] = tls[t];
        ll ans = min(T1.dis[u] + x + Tn.dis[v], T1.dis[v] + x + Tn.dis[u]);
        printf("%lld\n", min(ans, T1.dis[n]));
    } else {
        auto [u, v, w] = tls[t];
        ll tans = st.query(min(rk[u], rk[v]));
        printf("%lld\n", min(tans, T1.dis[n] - w + x));
    }
}
```