---
layout: post
title: 组合与全排列
categories: CPP
---

[TOC]  

### 组合  
用`DFS`求解  
{% highlight c++ %}
//求组合C(n,k),用DFS,从集合[1....n-1]中任选k个数
vector<vector<int>> combine(int n, int k)
{
    vector<vector<int>> result;
    if (n < k) return result;
    vector<int> path;
    combine_help(result, n, 1, 0, path, k);
    return result;
}
//dfs
void combine_help(vector<vector<int>> &result, int n, int k, int length, vector<int> &path, int target)
{
    if (k <= n+1 && length == target)
    {
        result.push_back(path);
        return;
    }
    if (k > n) return;
    //选
    path.push_back(k);
    combine_help(result, n, k + 1, length + 1, path, target);
    path.pop_back();
    //不选
    combine_help(result, n, k + 1, length, path, target);
}
{% endhighlight %}

### 全排列  
用`STL`之`next_permutation`或者`prev_permutation`求解  
{% highlight c++ %}
vector<vector<int>> permutation(vector<int> nums)
{
    vector<vector<int>> ret;
    sort(nums.begin(),nums.end());
    ret.push_back(nums);
    while(next_permutation(nums.begin(),nums.end());
    {
        ret.push_back(nums);
    }
    return ret;
}
{% endhighlight %}