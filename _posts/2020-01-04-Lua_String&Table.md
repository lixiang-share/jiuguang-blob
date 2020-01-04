---
layout:     post
title:      Lua数据类型
subtitle:   Lua源码分析——string
date:       2019-12-28
author:     jiuguang
catalog: true
tags:
    - lua源码分析
---

## 前言

string几乎是所有语言的基础建设，在lua中无byte类型&&需要高效地内存复用，基于这两点形成了lua <b>使用原始char*</b>+<b>string内存化</b>的设计方案。Lua的所有类型设计都是基于简单，高效的原则，不参杂复杂的数据类型，但是足够精巧，可以实现各种高级用法

### Lua string 结构定义

string核心实现在*lstring.lua*，它的定义如下：
```
/*
** Header for string value; string bytes follow the end of this structure
** (aligned according to 'UTString'; see next).
*/
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
} TString;

/*
** Ensures that address after this type is always fully aligned.
*/
typedef union UTString {
  L_Umaxalign dummy;  /* ensures maximum alignment for strings */
  TString tsv;
} UTString;
```

+ L_Umaxalign 仅是为了保证TString 按照最大8字节去对齐
+ CommonHeader 是lua引用类型通用定义头，维护了gc相关标记 [参考](https://lixiang-share.github.io/2019/12/28/Lua%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/)
+ string 处理上分为 long string/ short string 其中lngstr 在实现上做了特殊处理， 所以定义上 extra/shrlen/lnglen/hnext 做了一些区分
+ *hnext 是用来处理shrsring "internalized"(内存化或者字符串池化)

TString中并没有存储char*的字段，那如何存储呢？ 其实char\*是紧接着存储在TString对象之后的：

| TString{Mate 描述：length，hash...} | char*{内存char 数据} |
| ---- | ---- |

如果用[zero size of array](https://gcc.gnu.org/onlinedocs/gcc/Zero-Length.html), 也可以这么定义TString结构：

```
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
  buff char[0];
} TString;
```

buff 就可以作为char\*的存储字段，但是这总struct定义仅仅是编译器优化做的工作，并不是c标准一部分，如果lua这么定义，可能在一些应用环境下出现不兼容。lua就手动计算了一下char\*存储偏移，来实现set/get：

```
//malloc TString 时计算size方式，由于要实现 zero-terminated string(以便分割&&使用c原生库)， 所以size+1
#define sizelstring(l)  (sizeof(union UTString) + ((l) + 1) * sizeof(char))

//获得char*真正的存储位置
#define getstr(ts)  \
  check_exp(sizeof((ts)->extra), cast(char *, (ts)) + sizeof(UTString))
```

### Lua string 内存化

string 在实现上分为lngstring， shrstring分别对应数据类型：LUA_TLNGSTR/LUA_TSHRSTR，其中shrstring是“一直内存化”(可以复用)，lngstring则不会内存化(不可以复用)，区分两者类型的唯一标准就是size, 超过40个字符的就是lngstring，在创建string对象时会做区分：

```
/*
** Maximum length for short strings, that is, strings that are
** internalized. (Cannot be smaller than reserved words or tags for
** metamethods, as these strings must be internalized;
** #("function") = 8, #("__newindex") = 10.)
*/
#if !defined(LUAI_MAXSHORTLEN)
#define LUAI_MAXSHORTLEN	40
#endif

/*
** new string (with explicit length)
*/
TString *luaS_newlstr (lua_State *L, const char *str, size_t l) {
  if (l <= LUAI_MAXSHORTLEN)  /* short string? */
    return internshrstr(L, str, l);
  else {
    TString *ts;
    if (l >= (MAX_SIZE - sizeof(TString))/sizeof(char))
      luaM_toobig(L);
    ts = luaS_createlngstrobj(L, l);
    memcpy(getstr(ts), str, l * sizeof(char));
    return ts;
  }
}   

```

string的缓存复用逻辑主要分为两部分，有点类似cpu的多级缓存：

+ 有限容量的hash表缓存：STRCACHE_N * STRCACHE_M size的缓存，直接用ptraddr去hash建索引
+ 缓存shrstring的hash表，会触发resize，造比较大的内存分配和运算量

一级缓存比较简单，不过更加高效，基本是O(0)的效率：

```
/*
** Create or reuse a zero-terminated string, first checking in the
** cache (using the string address as a key). The cache can contain
** only zero-terminated strings, so it is safe to use 'strcmp' to
** check hits.
*/
TString *luaS_new (lua_State *L, const char *str) {
  unsigned int i = point2uint(str) % STRCACHE_N;  /* hash */
  int j;
  TString **p = G(L)->strcache[i];
  for (j = 0; j < STRCACHE_M; j++) {
    if (strcmp(str, getstr(p[j])) == 0)  /* hit? */
      return p[j];  /* that is it */
  }
  /* normal route */
  for (j = STRCACHE_M - 1; j > 0; j--)
    p[j] = p[j - 1];  /* move out last element */
  /* new element is first in the list */
  //每次都挂在第一个位置，淘汰最后一个位置，有点LRU做用
  p[0] = luaS_newlstr(L, str, strlen(str));
  return p[0];
}
```

luaS_newlstr 在处理lngstring时，会直接去new一个，并不会去缓存：

```
//new lngstring，直接去new一个， 此时u.字段会使用lngstr
TString *luaS_createlngstrobj (lua_State *L, size_t l) {
  TString *ts = createstrobj(L, l, LUA_TLNGSTR, G(L)->seed);
  ts->u.lnglen = l;
  return ts;
}

/*
** creates a new string object
*/
static TString *createstrobj (lua_State *L, size_t l, int tag, unsigned int h) {
  TString *ts;
  GCObject *o;
  size_t totalsize;  /* total size of TString object */
  totalsize = sizelstring(l);
  o = luaC_newobj(L, tag, totalsize);
  ts = gco2ts(o);
  ts->hash = h;
  ts->extra = 0;
  getstr(ts)[l] = '\0';  /* ending 0 */
  return ts;
}
```

shrstring 是用hash表管理的，充分去复用，避免重复string带来的内存开销

```
/*
* 先查有没有可复用的对象，没有则创建新的
*/
static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  TString *ts;
  global_State *g = G(L);
  unsigned int h = luaS_hash(str, l, g->seed);  //算hash，对应TString 中的hash字段
  TString **list = &g->strt.hash[lmod(h, g->strt.size)]; //算出mainPos
  lua_assert(str != NULL);  /* otherwise 'memcmp'/'memcpy' are undefined */
  for (ts = *list; ts != NULL; ts = ts->u.hnext) { //查可复用
    if (l == ts->shrlen &&
        (memcmp(str, getstr(ts), l * sizeof(char)) == 0)) {
      /* found! gc标记，如果已经标记dead，则需要转为white，确保不要被gc回收 */
      if (isdead(g, ts))  /* dead (but not collected yet)? */
        changewhite(ts);  /* resurrect it */
      return ts;
    }
  }
  //无可复用的情况下，检测是否超出了hash预设size，去resize
  if (g->strt.nuse >= g->strt.size && g->strt.size <= MAX_INT/2) {
    luaS_resize(L, g->strt.size * 2); //每次resize都是2的倍数
    list = &g->strt.hash[lmod(h, g->strt.size)];  /* recompute with new size */
  }
  ts = createstrobj(L, l, LUA_TSHRSTR, h);
  memcpy(getstr(ts), str, l * sizeof(char));
  ts->shrlen = cast_byte(l);
  ts->u.hnext = *list; //把ts挂在第一个
  *list = ts;
  g->strt.nuse++;
  return ts;
}
```
resize 是string比较费的一个操作，需要重新计算所有的string pos，并且扩大内存*2, 这个是内存化string无法避免的开销：

```
/*
** resizes the string table
*/
void luaS_resize (lua_State *L, int newsize) {
  int i;
  stringtable *tb = &G(L)->strt;
  if (newsize > tb->size) {  /* realloc去扩展内存，并把新内存指针值初始话null*/
    luaM_reallocvector(L, tb->hash, tb->size, newsize, TString *);
    for (i = tb->size; i < newsize; i++)
      tb->hash[i] = NULL;
  }
  for (i = 0; i < tb->size; i++) {  /* 重新计算所有string pos=> rehash */
    TString *p = tb->hash[i];
    tb->hash[i] = NULL;
    while (p) {  /* for each node in the list */
      TString *hnext = p->u.hnext;  /* save next： 本次仅仅调整p链上的对象 */
      unsigned int h = lmod(p->hash, newsize);  /* new position */
      p->u.hnext = tb->hash[h];  /* chain it： 如果还未h是rehash的pos，此时hash[h]的mainpos其实不对的，等到遍历到h时，会重新rehash调整*/
      tb->hash[h] = p;
      p = hnext;
    }
  }
  if (newsize < tb->size) {  /* shrink table if needed */
    /* vanishing slice should be empty */
    lua_assert(tb->hash[newsize] == NULL && tb->hash[tb->size - 1] == NULL);
    luaM_reallocvector(L, tb->hash, tb->size, newsize, TString *);
  }
  tb->size = newsize;
}
```

Lua的string复用+gc 机制可以很高效地复用又不至于撑爆内存，类似在一些高级语言如：C#/Java等均有类似机制，这么处理有一个非常必要的前提是：string不可变，简单说就是不能更改string中的char，所以任何对string做改变的逻辑最后都是生成另一个string对象


### Lua string 相关注意点

1. string 字节操作问题

string 基于c中的char*设计，对于string相关api来说，它操作的均是 char，如下面demo：
```
local str = "测试长度"
print(string.len(str))

--output:12
```
在lua中所有关于byte[] 的概念都是由string去完成的，如网络IO流，各种数据结构的serialize/unserialize 等等，其实就相当于在操作char*


2. 连接操作

如果有大量string需要连接操作，那么最好使用table.concat(), concat 函数实现会先把左右元素转char*，然后合到一个大的char* buff中，最后创建一个string对象，避免大量的查hash和创建string对象

```
local curTime = os.clock()
local str = ""
for i=1, 50000 do
  str = str .. i.. "b"
end
print("raw .. : ", os.clock() - curTime)

local tbl = {}
curTime = os.clock()
for i=1, 50000 do
  table.insert(tbl, i.. "b")
end
local s = table.concat(tbl, "")
print("concat .. : ", os.clock() - curTime)

--output:
--raw .. :        2.109375
--concat .. :     0.046875
```

3. lngstring操作每次都会大量产生lngstring对象，会频繁引起gc，如果有大量lngstring，可以考虑参考string中的"一级缓存" 或者扩大lngstring临界size
```
function TestMemory(preStr)
    collectgarbage("stop")
local beforeSize = collectgarbage("count")
print("befor mem: ", beforeSize)

local tbl = {}
for i=1,100000 do
    local str = preStr .. math.random(0, 9)
end
local endSize = collectgarbage("count")
print("end mem: ", endSize, "Diff: ", endSize - beforeSize)
collectgarbage("restart")
end

TestMemory("012345678901234567890123456789012345678") --39 size, 保证最终不会超过40
--output:
--befor mem:      25.05859375
--end mem:        28.69140625     Diff:   3.6328125

collectgarbage()
TestMemory("0123456789012345678901234567890123456789") --40 size, 保证最终超过40
--output:
--befor mem:      25.060546875
--end mem:        6471.3818359375 Diff:   6446.3212890625
```
