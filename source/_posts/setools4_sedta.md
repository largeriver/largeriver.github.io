---
title: setools4 sedta 转换图的构建分析
date: 2016-02-26 11:52:33
categories:
- security
- python

tags:
- SELinux
- setools4
---

sedta.py
# 算法总结

sedta命令用来分析动态域转换，其基本原理是首先构建域转换图，然后使用路径分析查找 源域和目标域 之间的路径。在apol中，还可以调用networkx等工具来绘图。

本文分析了域转换有向图的构建。



## 域转换的类型

### Standard transitions a->b:

    # allow a b:process transition;
    # allow a b_exec:file execute;
    # allow b b_exec:file entrypoint;
    #
    # and at least one of:
    # allow a self:process setexec;
    # type_transition a b_exec:process b;
    #

标准的域转换有两种途径：
* type_transition: a 在执行b的可执行文件时，自动切换到b域。在SEAndroid中，宏domain_auto_trans 就是属于该种转换。
* process setexec: a 本身是selinux-aware的进程，因此主动调用setexec将进程域切换到b。在SEAndroid中，init进程就使用该方式来将子进程切换到新的域。

### Dynamic transition x->y:
    # allow x y:process dyntransition;
    # allow x self:process setcurrent;

动态域转换，在SEAndroid中，最常使用该转换的就是zygote进程了。因为zygote启动的apk，不是一个完整的可执行程序，不带有main入口，因此实际的切换过程是：zygote fork一个子进程，调用setcurrent切换到untrusted_app域，然后加载并执行apk里对应的可执行代码入口。

## 算法描述

1. 迭代所有的规则
    1. 跳过所有非allow/type_transition规则
    2. 如果是process transition/dyntransition规则,那么增加一条边，初始化规则列表，增加此规则
    3. 如果是process setexec或者setcurrent，增加到该subject对应的字典中。
    4. 如果是file exec/entryppint 或者是type_transition:process，那么将其增加到以（subject，object）为主键的字典中。
2. 遍历所有的边
    1. 如果是transition规则（否则丢到**无效转换表**中）
        1. 使用set intersection来查找匹配的exec和entrypointer规则。如果没有，那么将该边丢到无效转换表中
        2. 对每一个有效的entrypoint规则，如果存在相应的type+transition，或者其源域有着setexec权限，那么讲规则增加到边列表。
        3. 否则，丢到无效转换表中
    2. 如果是dyntransition规则（否则丢到**无效动态转换表中**）
        1. 如果源域存在对应的setcurrent规则，将其增加到边列表；否则，丢到无效动态转换表中
3. 遍历所有的边
    1. 如果一条边存在一条无效转换和无效动态转换，删除该边
    2. 如果一条边存在无效转换，清除该边上相应的列表
    3. 如果一条边存在无效动态转换，清除该边上相应的列表


## 一些package
### defaultdict
代码中，defaultdict取代了常用的dict，defaultdict类的初始化函数接受一个类型作为参数，当所访问的键不存在的时候，可以实例化一个值作为默认值。
官方文档的这个例子就能看到这种写法的简洁了

```python
>>> s = [('yellow', 1), ('blue', 2), ('yellow', 3), ('blue', 4), ('red', 1)]
>>> d = defaultdict(list)
>>> for k, v in s:
... d[k].append(v)
...
>>> d.items()
[('blue', [2, 4]), ('red', [1]), ('yellow', [1, 3])]
'''
```

当我们做这种带有统计性质的数据操作时候，这个对象很好用。
### networkx
networkX.DiGraph用来存储有向图。 [NetworkX (NX)](https://networkx.lanl.gov/) is a Python package for the creation, manipulation, and   study of the structure, dynamics, and functions of complex networks.   

```python
    >>> import networkx as nx
    >>> G=nx.Graph()
    >>> G.add_edge(1,2)
    >>> G.add_node(42)
    >>> print(sorted(G.nodes()))
    [1, 2, 42]
    >>> print(sorted(G.edges()))
    [(1, 2)]
```
## 代码分析





```python
class DomainTransitionAnalysis(object):

    def _build_graph(self):
        #初始化图，self.G = nx.DiGraph()
        self.G.clear()
        self.G.name = "Domain transition graph for {0}.".format(self.policy)

        self.log.info("Building graph from {0}...".format(self.policy))

        # hash tables keyed on domain type
        #   allow a self:process setexec;  type_transition a b_exec:process b;
        #   allow x y:process dyntransition; allow x self:process setcurrent;
        # setexec和setcurrent 记录的是source domian能够转换到的target domains 列表
        setexec = defaultdict(list)
        setcurrent = defaultdict(list)

        # hash tables keyed on (domain, entrypoint file type)
        # the parameter for defaultdict has to be callable
        # hence the lambda for the nested defaultdict
        #
        execute = defaultdict(lambda: defaultdict(list))
        entrypoint = defaultdict(lambda: defaultdict(list))

        # hash table keyed on (domain, entrypoint, target domain)
        type_trans = defaultdict(lambda: defaultdict(lambda: defaultdict(list)))

        # rule的组成：ruletype source target tclass:perms
        for rule in self.policy.terules():
            if rule.ruletype == "allow":
                if rule.tclass not in ["process", "file"]:
                    continue

                perms = rule.perms

                if rule.tclass == "process":
                    if "transition" in perms:
                        for s, t in itertools.product(rule.source.expand(), rule.target.expand()):
                            # only add edges if they actually
                            # transition to a new type
                            if s != t:
                                edge = Edge(self.G, s, t, create=True)
                                edge.transition.append(rule)

                    if "dyntransition" in perms:
                        for s, t in itertools.product(rule.source.expand(), rule.target.expand()):
                            # only add edges if they actually
                            # transition to a new type
                            if s != t:
                                e = Edge(self.G, s, t, create=True)
                                e.dyntransition.append(rule)

                    if "setexec" in perms:
                        for s in rule.source.expand():
                            setexec[s].append(rule)

                    if "setcurrent" in perms:
                        for s in rule.source.expand():
                            setcurrent[s].append(rule)

                else:
                    if "execute" in perms:
                        for s, t in itertools.product(
                                rule.source.expand(),
                                rule.target.expand()):
                            execute[s][t].append(rule)

                    if "entrypoint" in perms:
                        for s, t in itertools.product(rule.source.expand(), rule.target.expand()):
                            entrypoint[s][t].append(rule)

            elif rule.ruletype == "type_transition":
                if rule.tclass != "process":
                    continue

                d = rule.default
                for s, t in itertools.product(rule.source.expand(), rule.target.expand()):
                    type_trans[s][t][d].append(rule)

        invalid_edge = []
        clear_transition = []
        clear_dyntransition = []

        for s, t in self.G.edges_iter():
            edge = Edge(self.G, s, t)
            invalid_trans = False
            invalid_dyntrans = False

            if edge.transition:
                # get matching domain exec w/entrypoint type
                entry = set(entrypoint[t].keys())
                exe = set(execute[s].keys())
                match = entry.intersection(exe)

                if not match:
                    # there are no valid entrypoints
                    invalid_trans = True
                else:
                    # TODO try to improve the
                    # efficiency in this loop
                    for m in match:
                        if s in setexec or type_trans[s][m]:
                            # add key for each entrypoint
                            edge.entrypoint[m] += entrypoint[t][m]
                            edge.execute[m] += execute[s][m]

                        if type_trans[s][m][t]:
                            edge.type_transition[m] += type_trans[s][m][t]

                    if s in setexec:
                        edge.setexec.extend(setexec[s])

                    if not edge.setexec and not edge.type_transition:
                        invalid_trans = True
            else:
                invalid_trans = True

            if edge.dyntransition:
                if s in setcurrent:
                    edge.setcurrent.extend(setcurrent[s])
                else:
                    invalid_dyntrans = True
            else:
                invalid_dyntrans = True

            # cannot change the edges while iterating over them,
            # so keep appropriate lists
            if invalid_trans and invalid_dyntrans:
                invalid_edge.append(edge)
            elif invalid_trans:
                clear_transition.append(edge)
            elif invalid_dyntrans:
                clear_dyntransition.append(edge)

        # Remove invalid transitions
        self.G.remove_edges_from(invalid_edge)
        for edge in clear_transition:
            # if only the regular transition is invalid,
            # clear the relevant lists
            del edge.transition
            del edge.execute
            del edge.entrypoint
            del edge.type_transition
            del edge.setexec
        for edge in clear_dyntransition:
            # if only the dynamic transition is invalid,
            # clear the relevant lists
            del edge.dyntransition
            del edge.setcurrent

        self.rebuildgraph = False
        self.rebuildsubgraph = True
        self.log.info("Completed building graph.")
```        
