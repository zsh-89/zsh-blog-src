---
title: '实现权限分配遇到的算法问题: 简化的 set cover problem'
date: 2017-07-17 22:20:55
tags: 
 - algorithm
 - set cover problem

---


## 背景
小组项目遇到的一个 "有点算法性质" 的问题, 最后用了我想的方法, 比较开心.
最后列出的 code 是 [zhengyang 同学](https://github.com/feizhengyang) 的实现方式的剪裁版 (已得到授权 ;D).
跟公司相关的信息都已经去掉.


## 问题
每一个 `HighLevelRole` 拥有大于等于 1 个的, 能合法执行的 `Activity`;
每一个 `RawRole` 也拥有大于等于 1 个的, 能合法执行的 `Activity`;

现在, 需要把一个 `HighLevelRole` 的集合 (记为 A) 转换成一个 `RawRole` 的集合 (记为 B); 
要求:
1. B 的所有元素的并集 == A 的所有元素的并集; 即 A 和 B 拥有的 `Activity` 不变多也不变少
2. B 不能包含冗余的 `RawRole` 元素; 
   即结果集合中每一个 `RawRole` 都至少应该拥有一个集合中其他 `RawRole` 没有的 `Activity`


## 分析
正如标题所说的, 这个问题有那么一点像 [set cover problem](https://en.wikipedia.org/wiki/Set_cover_problem);
如果问题要求结果的集合 B 的元素个数 `最小`, 那么该问题就是 set cover problem.
好在原始问题只要求 `无冗余`, 而不要求 `最小`.

思路:
1. 求出 A 的所有元素的并集, 结果是 Activity 的集合, 记为 `actSet`
2. 用 `actSet` 的每一个元素对全部的 RawRole 进行染色; 
   + RawRole 为黑色 = RawRole 包含的所有 Activity 都在 `actSet` 中出现;
   + RawRole 为灰色 = RawRole 包含的部分 Activity 在 `actSet` 中出现, 但并非全都在 `actSet` 中出现;
   + RawRole 为白色 = RawRole 包含的所有 Activity 都不在 `actSet` 中出现
3. 将所有黑色的 RawRole 留下, 得到一个都是黑色的 RawRole 集合, 记为 `roleList`
   (这时候可以验证一下 `roleList` 包含的 Activity 集合是不是等于 `actSet`; 若不等, 说明无法找到满足要求的解)
4. 用一个 dictionary (记为 `actTable`), 记录 `roleList` 包含的每个 Activity 出现的次数 (遍历 `roleList` 即可算出);
   dictionary 的 Key 是 Activity, Value 是 "出现的次数" 的 counter.
5. 逐个尝试从 `roleList` 中去掉一个 RawRole. 
   如果某一个 RawRole 的每一个 Activity 在 `actTable` 中的 counter 都 > 1, 说明这个 RawRole 是冗余的;
   去掉这个 RawRole, 并在 `actTable` 中把它包含的 Activity 的 counter 都减 1.

这样得到的结果, 它包含的元素个数不会是最少的, 但是已经符合需求.


## Code

```cs
// 仅供参考, 省去了一些 edge case 判断.
//
// rawRoleActMappings: 定义了每个 RawRole 和它对应的 Acttivity 集合
// actSet: 就是前面提到 actSet. 假设已经这项计算完成.
public static List<string> CalcRawRoles(
    Dictionary<string, HashSet<string>> rawRoleActMappings, 
    HashSet<string> actSet
)
{   
    List<string> roleList = new List<string>();
    Dictionary<string, int> actTable = actSet.ToDictionary(
        act => act, act => 0
    );

    // C# 大法好: IsSuperSetOf 就迅速完成 '染色' 过程... 
    foreach(var roleActMap in rawRoleActMappings.Where(
        roleActMap => actSet.IsSuperSetOf(roleActMap))
    ) 
    {
        roleList.Add(roleActMap.Key);
        roleActMap.Value.ForEach(act => actTable[act]++);
    }

    List<string> resultRoles = new List<string>();
    foreach(var role in roleList.OrderBy(
        roleName => rawRoleActMappings[roleName].Count).ToList()
    )
    {
        if (rawRoleActMappings[role].All(act => actTable[act]>1))
            rawRoleActMappings[role].ForEach(act => actTable[act]--);
        else 
            resultRoles.Add(role);
    }

    return resultRoles;
}
```
