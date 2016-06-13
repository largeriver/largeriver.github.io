---
title: 人名匹配问题
date: 20150508 15:22:33
categories:

tags:
- 算法
- c++
---

# 问题

腾讯的现场招聘会题目：在一篇英文文章中查找指定的人名，人名使用二十六个英文字母（可以是大写或小写）、空格以及两个通配符组成（*、?），通配符“*”表示零个或多个任意字母，通配符“?”表示一个任意字母。如：“J* Smi??” 可以匹配“John Smith” .
void scan(const char* pszText, const char* pszName);
注：pszText为整个文章字符，pszName为要求匹配的英文名。
请完成函数实现输出所有匹配的英文名，除了printf外，不能用第三方的库函数等。
<!-- more -->
# 分析

有人是这样分析的：这 个是kmp子串搜索（匹配），稍加改造，如 abcabd*?abe**??de这个窜，我们可以分成abcabd,?,abe,?,?,并按顺序先匹配abcabd，当匹配后，将匹配的文章中地址 及匹配的是何子串放到栈里记录下来，这样，每次匹配都入栈保存当前子串匹配信息，当一次完整的全部子串都匹配完后，就输出一个匹配结果，然后回溯一下，开 始对栈顶的子串的进行文中下一个起始位置的匹配。

我的看法是这样的，有一个细节被提及的比较少：要求匹配的是人名，以空格分隔的连续字符串。
标准的实现其实是一种状态机的模式：

* 如果是“\*”,那么需要注意其匹配边界是当前单词。否则，启动递归匹配模式。
* 如果是空格，那么需要注意，1）连续空格应该等同于一个空格2）tab字符应该等同于空格，这里做了简化，没处理。
* 如果是？，匹配单个字母即可


# 代码

```C++

#include <stdio.h>

static bool _isalpha(char c){
    return  (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z');
}

static bool _isalphanum(char c){
    return  _isalpha(c) || (c >= '0' && c <= '9');
}

static bool _isspace(char c){
    return c == ' ' || c == '\t';
}

bool _match(const char* s, const char* m, const char** _end = NULL){

    while (*s && *m){
        switch (*m)
        {
        case '*':
            //如果*是最后一个，那么匹配
            m++;

            //下一个模式字符是空白
            if (_isspace(*m)){
                //匹配完当前单词
                while (_isalpha(*s)){ s++; }
                continue;
            }

            //*已经是模式字符串的最后一个字符了，over
            if (*m == 0){
                while (!_isalpha(*s)){ s++; }
                if (_end) *_end = s;
                return true;
            }

            /*加持递归模式，注意只需要匹配当前word*/
            for (s; *s && _isalpha(*s); s++){
                if (_match(s, m, _end))
                    return true;
            }
            return false;
            break;
        case '?':
            if (!_isalpha(*s))//匹配任意字母
                return false;
            break;
        case ' ':
        case '\t'://匹配空白
            if (!_isspace(*s))
                return false;
            //跳过连续空白
            while (_isspace(*(m + 1))) { m++; }
            while (_isspace(*(s + 1))) { s++; }
            break;
        default://alpha and space
            //模式中不应该出现标点和数字
            if (*m != *s)
                return false;
            break;
        }
        s++, m++;

    }

    if (!*m && !_isalphanum(*s)){
        if (_end) *_end = s;
        return true;
    }

    return false;
}

const char* _strdump(const char* start, const char* end){
    char name[1024];
    char * t = name;
    while (start < end) { *t++ = *start++; }
    *t = 0;
    return name;

}

int _scan(const char* pszText, const char* pszName){
    int i = 0;

    for (const char* s = pszText; *s;){
        const char * _start = NULL, *_end = NULL;

        //find a start
        while (*s && !_isalpha(*s)){ s++; }
        _start = s;
        if (_match(s, pszName, &_end)){
            printf("%s\n", _strdump(_start, _end));
            s = _end;
            i++;
        }
        else{
            while (*s && _isalpha(*s)){ s++; }
        }

    }
    return i;
}


#include "gtest/gtest.h"
TEST(MatchNameTest, dummy) {
    //    EXPECT_TRUE(match_token("John", 4, "J*", 2));
    //EXPECT_TRUE(match_token("Smith",5, "Smi??",5));
    EXPECT_TRUE(_match("John", "J*"));
    EXPECT_TRUE(_match("Smith", "Smi??"));
    EXPECT_FALSE(_match("John Smith", "j* Smi??"));
    EXPECT_TRUE(_match("John Smith", "J* Smi??"));
    EXPECT_FALSE(_match("John Smith1", "J* Smi??"));

    EXPECT_FALSE(_match("John Jack Smith", "J* Smi??"));
    EXPECT_FALSE(_match("John John1 Smith", "J* Smi??"));
    EXPECT_FALSE(_match(",John Zhang Smith.", "J*h* S*i??"));
    EXPECT_FALSE(_match(",JohnSmith.", "J*h* S*i??"));

    EXPECT_TRUE(_match("John Smith Tom", "J* Smi?? Tom"));
    EXPECT_TRUE(_match("John    Smith Tom", "J* Smi?? Tom"));
    EXPECT_TRUE(_match("John Smith Tom", "J* Smi??     Tom"));
    EXPECT_TRUE(_match("John       Smith         Tom", "J* Smi??     Tom"));

    EXPECT_EQ(_scan("John Smith", "J* Smi??"), 1);

    EXPECT_EQ(_scan(", John Smith", "J* Smi??"), 1);
    EXPECT_EQ(_scan(", John Smith,c,John Smith.", "J* Smi??"), 2);

    EXPECT_EQ(_scan(", John Smith,c,John Smith.", "J*h* S*i??"), 2);

    EXPECT_EQ(_scan(", John Smith,John Zhang,John Smith.", "J*h* S*i??"), 2);

    EXPECT_EQ(_scan(", John Smith,John Zhang John John Smith,John Smith.", "J*h* S*i??"), 3);
    EXPECT_EQ(_scan(", John Smith,John Zhang John1 John Smith,John Smith.", "J*h* S*i??"), 3);
    EXPECT_EQ(_scan(", John Smith,John Zhang Smith,John Smith.", "J*h* S*i??"), 2);
}
```
