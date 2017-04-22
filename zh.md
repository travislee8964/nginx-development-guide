NGINX开发指南
===========

* [译者序](#译者序)
* [简介](#简介)
    * [代码结构](#代码结构)
    * [头文件](#头文件)
    * [整数](#整数)
    * [常用返回值](#常用返回值)
    * [错误处理](#错误处理)
* [字符串](#字符串)
    * [概述](#概述)
    * [格式化](#格式化)
    * [数值转换](#数值转换)
    * [正则表达式](#正则表达式)
* [容器](#容器)
    * [数组](#数组)
    * [列表](#列表)
    * [队列](#队列)
    * [红黑树](#红黑树)
    * [哈希](#哈希)
* [内存管理](#内存管理)
    * [堆](#堆)
    * [内存池](#内存池)
    * [共享内存](#共享内存)
* [日志](#日志)
* [周期](#周期)
* [Buffer](#buffer)
* [Networking](#networking)
    * [Connection](#connection)
* [Events](#events)
    * [Event](#event)
    * [I/O events](#i/O-events)
    * [Timer events](#timer-events)
    * [Posted events](#posted-events)
    * [Event loop](#event-loop)
* [进程](#进程)
* [模块](#模块)
    * [添加新模块](#添加新模块)
    * [核心模块](#核心模块)
    * [配置指令](#配置指令)
* [HTTP](#hTTP)
    * [Connection](#connection)
    * [Request](#request)
    * [Configuration](#configuration)
    * [Phases](#phases)
    * [Variables](#variables)
    * [Complex values](#complex-values)
    * [Request redirection](#request-redirection)
    * [Subrequests](#subrequests)
    * [Request finalization](#request-finalization)
    * [Request body](#request-body)
    * [Response](#response)
    * [Response body](#response-body)
    * [Body filters](#body-filters)
    * [Building filter modules](#building-filter-modules)
    * [Buffer reuse](#buffer-reuse)
    * [负载均衡](#负载均衡)
    
译者序
=====

本文档是nginx官方文档“Developer Guide”（[https://nginx.org/en/docs/dev/development_guide.html](https://nginx.org/en/docs/dev/development_guide.html)）的中文版本，由白山云（[http://www.baishancloud.com](http://www.baishancloud.com/zh/)）NGINX开发团队负责翻译。官方文档是HTML页面发布的，我们翻译的时候转成了Markdown，以方便编辑。同时也一并保留了英文的Markdown版本：[https://github.com/baishancloud/nginx-development-guide/blob/master/en.md](https://github.com/baishancloud/nginx-development-guide/blob/master/en.md)。希望此中文版文档能为广大的nginx以及开源爱好者提供入门指导，开发出优秀的nginx模块，回馈社区。

简介
===

代码结构
-------
* auto — 编译脚本
* src
    * core — 基础数据结构和函数 — 字符串，数组，日志，内存池等
* event — 事件机制核心模块
    * modules — 具体事件机制模块：epoll，kqueue，select等
* http — HTTP核心模块和公共代码
    * modules — 其他HTTP模块
    * v2 — HTTP/2模块
* mail — 邮件协议模块
* os — 平台相关代码
    * unix
    * win32
* stream — 流模块

头文件
-----
每个nginx文件都应该在开头包含如下两个头文件：

```
#include <ngx_config.h>
#include <ngx_core.h>
```

除此之外，HTTP相关的代码还要包含：

```
#include <ngx_http.h>
```

邮件模块的代码应该包含：

```
#include <ngx_mail.h>
```

Stream模块的代码应该包含：

```
#include <ngx_stream.h>
```

整数
----
一般情况下，nginx代码使用如下两个整数类型：ngx_int_t和ngx_uint_t，分别用typedef定义成了intptr_t和uintptr_t。

常用返回值
--------
nginx中的大多数函数使用如下类型的返回值：

* NGX_OK — 处理成功
* NGX_ERROR — 处理失败
* NGX_AGAIN — 处理未完成，函数需要被再次调用
* NGX_DECLINED — 处理被拒绝，例如相关功能在配置文件中被关闭。不要将此当成错误。
* NGX_BUSY — 资源不可用
* NGX_DONE — 处理完成或者在他处继续处理。也可以作为处理成功使用。
* NGX_ABORT — 函数终止。也可以作为处理出错的返回值。

错误处理
-------
为了获取最近一次系统错误码，nginx提供了ngx_errno宏。该宏被映射到了POSIX平台的errno变量上，而在Windows平台中，则变为对GetLastError()的函数调用。为了获取最近一次socket错误码，nginx提供了ngx_socket_errno宏。同样，在POSIX平台上该宏被映射为errno变量，而在Windows环境中则是对WSAGetLastError()进行调用。考虑到对性能的影响，ngx_errno和ngx_socket_errno不应该被连续访问。如果有连续、频繁访问的需要，则应该将错误码的值存储到类型为ngx_err_t的本地变量中，然后使用本地变量进行访问。如果需要设置错误码，可以使用ngx_set_errno(errno)和ngx_set_socket_errno(errno)这两个宏。

ngx_errno和ngx_socket_errno变量可以在调用日志相关函数ngx_log_error()和ngx_log_debugX()的时候使用，这样具体的错误文本就会被添加到日志输出中。

一个使用ngx_errno的例子：

```
void
ngx_my_kill(ngx_pid_t pid, ngx_log_t *log, int signo)
{
    ngx_err_t  err;

    if (kill(pid, signo) == -1) {
        err = ngx_errno;

        ngx_log_error(NGX_LOG_ALERT, log, err, "kill(%P, %d) failed", pid, signo);

        if (err == NGX_ESRCH) {
            return 2;
        }

        return 1;
    }

    return 0;
}
```

字符串
=====

概述
----
nginx使用无符号的char类型指针来表示C字符串：u_char *。

nginx字符串类型ngx_str_t的定义如下所示：

```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

结构体成员len存放字符串的长度，成员data指向字符串本身数据。在ngx_str_t中存放的字符串，对于超出len长度的部分可以是NULL结尾（'\0'——译者注），也可以不是。在大多数情况是不以NULL结尾的。然而，在nginx的某些代码中（例如解析配置的时候），ngx_str_t中的字符串是以NULL结尾的吗，这种情况会使得字符串比较变得更加简单，也使得使用系统调用的时候更加容易。

nginx提供了一系列关于字符串处理的函数。它们在src/core/ngx_string.h文件中定义。其中的一部分就是对C库中字符串函数的封装：

* ngx_strcmp()
* ngx_strncmp()
* ngx_strstr()
* ngx_strlen()
* ngx_strchr()
* ngx_memcmp()
* ngx_memset()
* ngx_memcpy()
* ngx_memmove()

还有一些nginx特有的字符串函数：

* ngx_memzero() 内存清0
* ngx_cpymem() 和ngx_memcpy()行为类似，不同的是该函数返回的是copy后的最终目的地址，这在需要连续拼接多个字符串的场景下很方便。
* ngx_movemem() 和ngx_memmove()的行为类似，不同的是该函数返回的是move后的最终目的地址。
* ngx_strlchr() 在字符串中查找一个特定字符，字符串由两个指针界定。

最后是一些大小写转换和字符串比较的函数：

* ngx_tolower()
* ngx_toupper()
* ngx_strlow()
* ngx_strcasecmp()
* ngx_strncasecmp()

格式化
-----
nginx提供了一些格式化字符串的函数。以下这些函数支持nginx特有的类型：

* ngx_sprintf(buf, fmt, ...)
* ngx_snprintf(buf, max, fmt, ...)
* ngx_slrintf(buf, last, fmt, ...)
* ngx_vslprint(buf, last, fmt, args)
* ngx_vsnprint(buf, max, fmt, args)

这些函数支持的全部格式化选项定义在src/core/ngx_string.c文件中，以下是其中的一部分：

```
%O — off_t
%T — time_t
%z — size_t
%i — ngx_int_t
%p — void *
%V — ngx_str_t *
%s — u_char * (null-terminated)
%*s — size_t + u_char *
```

'u'修饰符将类型指明为无符号，'X'和'x'则将输出转换为16禁止。

例如：

```
u_char     buf[NGX_INT_T_LEN];
size_t     len;
ngx_int_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```


数值转换
-------
nginx实现了若干用于数值转换的函数：

* ngx_atoi(line, n) — 将一个指定长度的字符串转换为一个正整数，类型为ngx_int_t。出错返回NGX_ERROR。
* ngx_atosz(line, n) — 同上，转换类型为ssize_t
* ngx_atoof(line, n) — 同上，转换类型为off_t
* ngx_atotm(line, n) — 同上，转换类型为time_t
* ngx_atofp(line, n, point) — 将一个固定长度的定点小数字符串转换为ngx_int_t类型的正整数。转换结果会左移point指定的10进制位数。字符串中的定点小数不能含有多过point参数指定的小数位。出错返回NGX_ERROR。举例：ngx_atofp("10.5", 4, 2) 返回1050
* ngx_hextoi(line, n) — 将表示16进制正整数的字符串转换为ngx_int_t类型的整数。出错返回NGX_ERROR。

正则表达式
--------
nginx中的正则表达式接口是对PCRE库的封装。相关的头文件是src/core/ngx_regex.h。

要使用正则表达式进行字符串匹配，首先需要对正则表达式进行编译，这通常是在配置解析阶段处理的。需要注意的是，因为PCRE的支持是可选的，因此所有使用正则相关接口的代码都需要用NGX_PCRE括起来：

```
#if (NGX_PCRE)
ngx_regex_t          *re;
ngx_regex_compile_t   rc;

u_char                errstr[NGX_MAX_CONF_ERRSTR];

ngx_str_t  value = ngx_string("message (\\d\\d\\d).*Codeword is '(?<cw>\\w+)'");

ngx_memzero(&rc, sizeof(ngx_regex_compile_t));

rc.pattern = value;
rc.pool = cf->pool;
rc.err.len = NGX_MAX_CONF_ERRSTR;
rc.err.data = errstr;
/* rc.options are passed as is to pcre_compile() */

if (ngx_regex_compile(&rc) != NGX_OK) {
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V", &rc.err);
    return NGX_CONF_ERROR;
}

re = rc.regex;
#endif
```

编译成功之后，结构体ngx_regex_compile_t的captures和named_captures成员分别会被填上正则表达式中全部以及命名捕获的数量。

然后，编译过的正则表达式就可以用来进行字符串匹配：

```
ngx_int_t  n;
int        captures[(1 + rc.captures) * 3];

ngx_str_t input = ngx_string("This is message 123. Codeword is 'foobar'.");

n = ngx_regex_exec(re, &input, captures, (1 + rc.captures) * 3);
if (n >= 0) {
    /* string matches expression */

} else if (n == NGX_REGEX_NO_MATCHED) {
    /* no match was found */

} else {
    /* some error */
    ngx_log_error(NGX_LOG_ALERT, log, 0, ngx_regex_exec_n " failed: %i", n);
}
```

ngx_regex_exec()的参数有：编译了的正则表达式re，待匹配的字符串s，可选的用于存放发现的捕获和其大小的整数数组。捕获数组的大小必须是3的倍数，这是PCRE库的API要求的。在上面例子中，该数组的大小是通过总捕获数加上字符串自身来计算得出的。

现在，如果成功匹配，则可以对捕获进行访问：

```
u_char     *p;
size_t      size;
ngx_str_t   name, value;

/* all captures */
for (i = 0; i < n * 2; i += 2) {
    value.data = input.data + captures[i];
    value.len = captures[i + 1] — captures[i];
}

/* accessing named captures */

size = rc.name_size;
p = rc.names;

for (i = 0; i < rc.named_captures; i++, p += size) {

    /* capture name */
    name.data = &p[2];
    name.len = ngx_strlen(name.data);

    n = 2 * ((p[0] << 8) + p[1]);

    /* captured value */
    value.data = &input.data[captures[n]];
    value.len = captures[n + 1] — captures[n];
}
```

ngx_regex_exec_array()函数接受ngx_regex_elt_t元素的数组（其实就是多个编译好的正则表达式以及对应的名字），一个待匹配字符串以及一个log。该函数会对待匹配字符串逐一应用数组中的正则表达式，直到匹配成功或者无一匹配。存在成功的匹配则返回NGX_OK，否则返回NGX_DECLINED，出错返回NGX_ERROR。

容器
====

数组
----
表示nginx数组（array）的结构体ngx_array_t定义如下：

```
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

数组的元素可以通过elts成员获取。元素的个数存放在nelts成员里。size成员记录单个元素的大小，size成员是在数组初始化的时候设置的。

数组可以使用调用ngx_array_create(pool, n, size)来创建，其所需内存在提供的pool中。一个已经分配过内存的数组对象，可以调用ngx_array_init(array, pool, n, size)进行初始化。


```
ngx_array_t  *a, b;

/* create an array of strings with preallocated memory for 10 elements */
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

使用下面的函数向数组添加元素：

* ngx_array_push(a) 向数组末尾添加一个元素并返回其指针
* ngx_array_push_n(a, n) 向数组末尾添加n个元素并返回指向其中第一个元素的指针

如果现有内存无法满足新元素的需要，数组会分配新的内存并将现有元素复制过去。新分配的内存一般是原有内存的2倍大。

```
s = ngx_array_push(a);
ss = ngx_array_push_n(&b, 3);
```

列表
----
nginx中的列表（List）由一系列的数组组成，并为可能插入大量item进行了优化。列表类型定义如下：

```
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

实际的item存放在列表部件结构中，定义如下：

```
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

Initially, a list must be initialized by calling ngx_list_init(list, pool, n, size) or created by calling ngx_list_create(pool, n, size). Both functions receive the size of a single item and a number of items per list part. The ngx_list_push(list) function is used to add an item to the list. Iterating over the items is done by direct accessing the list fields, as seen in the example:

使用之前，列表必须通过ngx_list_init(list, pool, n, size)初始化，或者通过ngx_list_create(pool, n, size)创建。两个方式都需要指定单一条目的大小以及每个列表部件中item的数量。ngx_list_push(list)函数用来向列表添加一个item。遍历item是通过直接访问列表成员实现的，参考以下示例：

```
ngx_str_t        *v;
ngx_uint_t        i;
ngx_list_t       *list;
ngx_list_part_t  *part;

list = ngx_list_create(pool, 100, sizeof(ngx_str_t));
if (list == NULL) { /* error */ }

/* add items to the list */

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "foo");

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "bar");

/* iterate over the list */

part = &list->part;
v = part->elts;

for (i = 0; /* void */; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }

        part = part->next;
        v = part->elts;
        i = 0;
    }

    ngx_do_smth(&v[i]);
}
```

nginx中列表的主要用途是处理HTTP中输入和输出的头部。

列表不支持删除item。然而，如果需要的话，可以将item标识成missing而不是真正的删除他们。例如，HTTP的输出头部——以ngx_table_elt_t对象存储——可以通过将ngx_table_elt_t结构的hash成员设置成0来将其标识为missing。这样一来，该HTTP头部就不会被遍历到。

Queue
-----

Queue in nginx is an intrusive doubly linked list, with each node defined as follows:

```
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

The head queue node is not linked with any data. Before using, the list head should be initialized with ngx_queue_init(q) call. Queues support the following operations:

* ngx_queue_insert_head(h, x), ngx_queue_insert_tail(h, x) — insert a new node
* ngx_queue_remove(x) — remove a queue node
* ngx_queue_split(h, q, n) — split a queue at a node, queue tail is returned in a separate queue
* ngx_queue_add(h, n) — add second queue to the first queue
* ngx_queue_head(h), ngx_queue_last(h) — get first or last queue node
* ngx_queue_sentinel(h) - get a queue sentinel object to end iteration at
* ngx_queue_data(q, type, link) — get reference to the beginning of a queue node data structure, consid ering the queue field offset in it

Example:

```
typedef struct {
    ngx_str_t    value;
    ngx_queue_t  queue;
} ngx_foo_t;

ngx_foo_t    *f;
ngx_queue_t   values;

ngx_queue_init(&values);

f = ngx_palloc(pool, sizeof(ngx_foo_t));
if (f == NULL) { /* error */ }
ngx_str_set(&f->value, "foo");

ngx_queue_insert_tail(&values, f);

/* insert more nodes here */

for (q = ngx_queue_head(&values);
     q != ngx_queue_sentinel(&values);
     q = ngx_queue_next(q))
{
    f = ngx_queue_data(q, ngx_foo_t, queue);

    ngx_do_smth(&f->value);
}
```

Red-Black tree
--------------

The src/core/ngx_rbtree.h header file provides access to the effective implementation of red-black tree
s.

```
typedef struct {
    ngx_rbtree_t       rbtree;
    ngx_rbtree_node_t  sentinel;

    /* custom per-tree data here */
} my_tree_t;

typedef struct {
    ngx_rbtree_node_t  rbnode;

    /* custom per-node data */
    foo_t              val;
} my_node_t;
```

To deal with a tree as a whole, you need two nodes: root and sentinel. Typically, they are added to som
e custom structure, thus allowing to organize your data into a tree which leaves contain a link to or e
mbed your data.

To initialize a tree:

```
my_tree_t  root;

ngx_rbtree_init(&root.rbtree, &root.sentinel, insert_value_function);
```

The insert_value_function is a function that is responsible for traversing the tree and inserting new v
alues into correct place. For example, the ngx_str_rbtree_insert_value functions is designed to deal wi
th ngx_str_t type.

```
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                 ngx_rbtree_node_t *node,
                                 ngx_rbtree_node_t *sentinel)
```

Its arguments are pointers to a root node of an insertion, newly created node to be added, and a tree s
entinel.

The traversal is pretty straightforward and can be demonstrated with the following lookup function patt
ern:

```
my_node_t *
my_rbtree_lookup(ngx_rbtree_t *rbtree, foo_t *val, uint32_t hash)
{
    ngx_int_t           rc;
    my_node_t          *n;
    ngx_rbtree_node_t  *node, *sentinel;

    node = rbtree->root;
    sentinel = rbtree->sentinel;

    while (node != sentinel) {

        n = (my_node_t *) node;

        if (hash != node->key) {
            node = (hash < node->key) ? node->left : node->right;
            continue;
        }

        rc = compare(val, node->val);

        if (rc < 0) {
            node = node->left;
            continue;
        }

        if (rc > 0) {
            node = node->right;
            continue;
        }

        return n;
    }

    return NULL;
}
```

The compare() is a classic comparator function returning value less, equal or greater than zero. To spe
ed up lookups and avoid comparing user objects that can be big, integer hash field is used.

To add a node to a tree, allocate a new node, initialize it and call ngx_rbtree_insert():

```
    my_node_t          *my_node;
    ngx_rbtree_node_t  *node;

    my_node = ngx_palloc(...);
    init_custom_data(&my_node->val);

    node = &my_node->rbnode;
    node->key = create_key(my_node->val);

    ngx_rbtree_insert(&root->rbtree, node);
```

to remove a node:

```
ngx_rbtree_delete(&root->rbtree, node);
```

Hash
----

Hash table functions are declared in src/core/ngx_hash.h. Exact and wildcard matching is supported. The
 latter requires extra setup and is described in a separate section below.

To initialize a hash, one needs to know the number of elements in advance, so that nginx can build the
hash optimally. Two parameters that need to be configured are max_size and bucket_size. The details of
setting up these are provided in a separate document. Usually, these two parameters are configurable by
 user. Hash initialization settings are stored as the ngx_hash_init_t type, and the hash itself is ngx_
hash_t:

```
ngx_hash_t       foo_hash;
ngx_hash_init_t  hash;

hash.hash = &foo_hash;
hash.key = ngx_hash_key;
hash.max_size = 512;
hash.bucket_size = ngx_align(64, ngx_cacheline_size);
hash.name = "foo_hash";
hash.pool = cf->pool;
hash.temp_pool = cf->temp_pool;
```

The key is a pointer to a function that creates hash integer key from a string. Two generic functions a
re provided: ngx_hash_key(data, len) and ngx_hash_key_lc(data, len). The latter converts a string to lo
wercase and thus requires the passed string to be writable. If this is not true, NGX_HASH_READONLY_KEY
flag may be passed to the function, initializing array keys (see below).

The hash keys are stored in ngx_hash_keys_arrays_t and are initialized with ngx_hash_keys_array_init(ar
r, type):

```
ngx_hash_keys_arrays_t  foo_keys;

foo_keys.pool = cf->pool;
foo_keys.temp_pool = cf->temp_pool;

ngx_hash_keys_array_init(&foo_keys, NGX_HASH_SMALL);
```

The second parameter can be either NGX_HASH_SMALL or NGX_HASH_LARGE and controls the amount of prealloc
ated resources for the hash. If you expect the hash to contain thousands elements, use NGX_HASH_LARGE.

The ngx_hash_add_key(keys_array, key, value, flags) function is used to insert keys into hash keys arra
y;

```
ngx_str_t k1 = ngx_string("key1");
ngx_str_t k2 = ngx_string("key2");

ngx_hash_add_key(&foo_keys, &k1, &my_data_ptr_1, NGX_HASH_READONLY_KEY);
ngx_hash_add_key(&foo_keys, &k2, &my_data_ptr_2, NGX_HASH_READONLY_KEY);
```

Now, the hash table may be built using the call to ngx_hash_init(hinit, key_names, nelts):

```
ngx_hash_init(&hash, foo_keys.keys.elts, foo_keys.keys.nelts);
```

This may fail, if max_size or bucket_size parameters are not big enough. When the hash is built, ngx_ha
sh_find(hash, key, name, len) function may be used to look up elements:

```
my_data_t   *data;
ngx_uint_t   key;

key = ngx_hash_key(k1.data, k1.len);

data = ngx_hash_find(&foo_hash, key, k1.data, k1.len);
if (data == NULL) {
    /* key not found */
}
```

Wildcard matching
-----------------

To create a hash that works with wildcards, ngx_hash_combined_t type is used. It includes the hash type
 described above and has two additional keys arrays: dns_wc_head and dns_wc_tail. The initialization of
 basic properties is done similarly to a usual hash:

```
ngx_hash_init_t      hash
ngx_hash_combined_t  foo_hash;

hash.hash = &foo_hash.hash;
hash.key = ...;
```

It is possible to add wildcard keys using the NGX_HASH_WILDCARD_KEY flag:

```
/* k1 = ".example.org"; */
/* k2 = "foo.*";        */
ngx_hash_add_key(&foo_keys, &k1, &data1, NGX_HASH_WILDCARD_KEY);
ngx_hash_add_key(&foo_keys, &k2, &data2, NGX_HASH_WILDCARD_KEY);
```

The function recognizes wildcards and adds keys into corresponding arrays. Please refer to the map modu
le documentation for the description of the wildcard syntax and matching algorithm.

Depending on the contents of added keys, you may need to initialize up to three keys arrays: one for ex
act matching (described above), and two for matching starting from head or tail of a string:

```
if (foo_keys.dns_wc_head.nelts) {

    ngx_qsort(foo_keys.dns_wc_head.elts,
              (size_t) foo_keys.dns_wc_head.nelts,
              sizeof(ngx_hash_key_t),
              cmp_dns_wildcards);

    hash.hash = NULL;
    hash.temp_pool = pool;

    if (ngx_hash_wildcard_init(&hash, foo_keys.dns_wc_head.elts,
                               foo_keys.dns_wc_head.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    foo_hash.wc_head = (ngx_hash_wildcard_t *) hash.hash;
}
```

The keys array needs to be sorted, and initialization results must be added to the combined hash. The i
nitialization of dns_wc_tail array is done similarly.

The lookup in a combined hash is handled by the ngx_hash_find_combined(chash, key, name, len):

```
/* key = "bar.example.org"; — will match ".example.org" */
/* key = "foo.example.com"; — will match "foo.*"        */

hkey = ngx_hash_key(key.data, key.len);
res = ngx_hash_find_combined(&foo_hash, hkey, key.data, key.len);
```

Memory management
=================

Heap
----

To allocate memory from system heap, the following functions are provided by nginx:

* ngx_alloc(size, log) — allocate memory from system heap. This is a wrapper around malloc() with loggi
ng support. Allocation error and debugging information is logged to log
ngx_calloc(size, log) — same as ngx_alloc(), but memory is filled with zeroes after allocation
* ngx_memalign(alignment, size, log) — allocate aligned memory from system heap. This is a wrapper arou
nd posix_memalign() on those platforms which provide it. Otherwise implementation falls back to ngx_all
oc() which provides maximum alignment
ngx_free(p) — free allocated memory. This is a wrapper around free()

Pool
----

Most nginx allocations are done in pools. Memory allocated in an nginx pool is freed automatically when
 the pool in destroyed. This provides good allocation performance and makes memory control easy.

A pool internally allocates objects in continuous blocks of memory. Once a block is full, a new one is
allocated and added to the pool memory block list. When a large allocation is requested which does not
fit into a block, such allocation is forwarded to the system allocator and the returned pointer is stor
ed in the pool for further deallocation.

Nginx pool has the type ngx_pool_t. The following operations are supported:

* ngx_create_pool(size, log) — create a pool with given block size. The pool object returned is allocat
ed in the pool as well.
* ngx_destroy_pool(pool) — free all pool memory, including the pool object itself.
* ngx_palloc(pool, size) — allocate aligned memory from pool
* ngx_pcalloc(pool, size) — allocated aligned memory from pool and fill it with zeroes
* ngx_pnalloc(pool, size) — allocate unaligned memory from pool. Mostly used for allocating strings
* ngx_pfree(pool, p) — free memory, previously allocated in the pool. Only allocations, forwarded to th
e system allocator, can be freed.

```
u_char      *p;
ngx_str_t   *s;
ngx_pool_t  *pool;

pool = ngx_create_pool(1024, log);
if (pool == NULL) { /* error */ }

s = ngx_palloc(pool, sizeof(ngx_str_t));
if (s == NULL) { /* error */ }
ngx_str_set(s, "foo");

p = ngx_pnalloc(pool, 3);
if (p == NULL) { /* error */ }
ngx_memcpy(p, "foo", 3);
```

Since chain links ngx_chain_t are actively used in nginx, nginx pool provides a way to reuse them. The
chain field of ngx_pool_t keeps a list of previously allocated links ready for reuse. For efficient all
ocation of a chain link in a pool, the function ngx_alloc_chain_link(pool) should be used. This functio
n looks up a free chain link in the pool list and only if it's empty allocates a new one. To free a lin
k ngx_free_chain(pool, cl) should be called.

Cleanup handlers can be registered in a pool. Cleanup handler is a callback with an argument which is c
alled when pool is destroyed. Pool is usually tied with a specific nginx object (like HTTP request) and
 destroyed in the end of that object’s lifetime, releasing the object itself. Registering a pool cleanu
p is a convenient way to release resources, close file descriptors or make final adjustments to shared
data, associated with the main object.

A pool cleanup is registered by calling ngx_pool_cleanup_add(pool, size) which returns ngx_pool_cleanup
_t pointer to be filled by the caller. The size argument allows allocating context for the cleanup hand
ler.

```
ngx_pool_cleanup_t  *cln;

cln = ngx_pool_cleanup_add(pool, 0);
if (cln == NULL) { /* error */ }

cln->handler = ngx_my_cleanup;
cln->data = "foo";

...

static void
ngx_my_cleanup(void *data)
{
    u_char  *msg = data;

    ngx_do_smth(msg);
}
```

Shared memory
-------------

Shared memory is used by nginx to share common data between processes. Function ngx_shared_memory_add(c
f, name, size, tag) adds a new shared memory entry ngx_shm_zone_t to the cycle. The function receives n
ame and size of the zone. Each shared zone must have a unique name. If a shared zone entry with the pro
vided name exists, the old zone entry is reused, if its tag value matches too. Mismatched tag is consid
ered an error. Usually, the address of the module structure is passed as tag, making it possible to reu
se shared zones by name within one nginx module.

The shared memory entry structure ngx_shm_zone_t has the following fields:

* init — initialization callback, called after shared zone is mapped to actual memory
* data — data context, used to pass arbitrary data to the init callback
* noreuse — flag, disabling shared zone reuse from the old cycle
* tag — shared zone tag
* shm — platform-specific object of type ngx_shm_t, having at least the following fields:
    * addr — mapped shared memory address, initially NULL
    * size — shared memory size
    * name — shared memory name
    * log — shared memory log
    * exists — flag, showing that shared memory was inherited from the master process (Windows-specific
)

Shared zone entries are mapped to actual memory in ngx_init_cycle() after configuration is parsed. On P
OSIX systems, mmap() syscall is used to create shared anonymous mapping. On Windows, CreateFileMapping(
)/MapViewOfFileEx() pair is used.

For allocating in shared memory, nginx provides slab pool ngx_slab_pool_t. In each nginx shared zone, a
 slab pool is automatically created for allocating memory in that zone. The pool is located in the begi
nning of the shared zone and can be accessed by the expression (ngx_slab_pool_t *) shm_zone->shm.addr.
Allocation in shared zone is done by calling one of the functions ngx_slab_alloc(pool, size)/ngx_slab_c
alloc(pool, size). Memory is freed by calling ngx_slab_free(pool, p).

Slab pool divides all shared zone into pages. Each page is used for allocating objects of the same size
. Only the sizes which are powers of 2, and not less than 8, are considered. Other sizes are rounded up
 to one of these values. For each page, a bitmask is kept, showing which blocks within that page are in
 use and which are free for allocation. For sizes greater than half-page (usually, 2048 bytes), allocat
ion is done by entire pages.

To protect data in shared memory from concurrent access, mutex is available in the mutex field of ngx_s
lab_pool_t. The mutex is used by the slab pool while allocating and freeing memory. However, it can be
used to protect any other user data structures, allocated in the shared zone. Locking is done by callin
g ngx_shmtx_lock(&shpool->mutex), unlocking is done by calling ngx_shmtx_unlock(&shpool->mutex).

```
ngx_str_t        name;
ngx_foo_ctx_t   *ctx;
ngx_shm_zone_t  *shm_zone;

ngx_str_set(&name, "foo");

/* allocate shared zone context */
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_foo_ctx_t));
if (ctx == NULL) {
    /* error */
}

/* add an entry for 65k shared zone */
shm_zone = ngx_shared_memory_add(cf, &name, 65536, &ngx_foo_module);
if (shm_zone == NULL) {
    /* error */
}

/* register init callback and context */
shm_zone->init = ngx_foo_init_zone;
shm_zone->data = ctx;


...

static ngx_int_t
ngx_foo_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_foo_ctx_t  *octx = data;

    size_t            len;
    ngx_foo_ctx_t    *ctx;
    ngx_slab_pool_t  *shpool;

    value = shm_zone->data;

    if (octx) {
        /* reusing a shared zone from old cycle */
        ctx->value = octx->value;
        return NGX_OK;
    }

    shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        /* initialize shared zone context in Windows nginx worker */
        ctx->value = shpool->data;
        return NGX_OK;
    }

    /* initialize shared zone */

    ctx->value = ngx_slab_alloc(shpool, sizeof(ngx_uint_t));
    if (ctx->value == NULL) {
        return NGX_ERROR;
    }

    shpool->data = ctx->value;

    return NGX_OK;
}
```

日志
=======

nginx用ngx_log_t对象记录日志。nginx的日志提供以下几种方式：

* stderr — 记录到标准错误输出
* file — 记录到文件
* syslog — 记录到syslog
* memory — 记录到内部内存用于开发的目的。这块内存可以在debugger时访问。

一个日志实例可以是一个日志对象链接，每个通过next连接起来。每个消息都被写到所有的日志对象。

每个日志对象有错误级别，用于限制消息写到它自己。以下是nginx提供的几种错误级别：

* NGX_LOG_EMERG
* NGX_LOG_ALERT
* NGX_LOG_CRIT
* NGX_LOG_ERR
* NGX_LOG_WARN
* NGX_LOG_NOTICE
* NGX_LOG_INFO
* NGX_LOG_DEBUG

对于调试日志，有以下几种选项：

* NGX_LOG_DEBUG_CORE
* NGX_LOG_DEBUG_ALLOC
* NGX_LOG_DEBUG_MUTEX
* NGX_LOG_DEBUG_EVENT
* NGX_LOG_DEBUG_HTTP
* NGX_LOG_DEBUG_MAIL
* NGX_LOG_DEBUG_STREAM

通常而言，日志是通过error_log指令创建的，并且在各个阶段都有效，cycle, 配置解析, 客户端连接和其它。

nginx提供以下的日志宏：

* ngx_log_error(level, log, err, fmt, ...) — 记录错误
* ngx_log_debug0(level, log, err, fmt), ngx_log_debug1(level, log, err, fmt, arg1) etc — 调试日志，提供最多8个可格式化的参数。

周期
=====

Buffer
======

For input/output operations, nginx provides the buffer type ngx_buf_t. Normally, it's used to hold data
 to be written to a destination or read from a source. Buffer can reference data in memory and in file.
 Technically it's possible that a buffer references both at the same time. Memory for the buffer is all
ocated separately and is not related to the buffer structure ngx_buf_t.

The structure ngx_buf_t has the following fields:

* start, end — the boundaries of memory block, allocated for the buffer
* pos, last — memory buffer boundaries, normally a subrange of start .. end
* file_pos, file_last — file buffer boundaries, these are offsets from the beginning of the file
* tag — unique value, used to distinguish buffers, created by different nginx module, usually, for the
purpose of buffer reuse
* file — file object
* temporary — flag, meaning that the buffer references writable memory
* memory — flag, meaning that the buffer references read-only memory
* in_file — flag, meaning that current buffer references data in a file
* flush — flag, meaning that all data prior to this buffer should be flushed
* recycled — flag, meaning that the buffer can be reused and should be consumed as soon as possible
* sync — flag, meaning that the buffer carries no data or special signal like flush or last_buf. Normal
ly, such buffers are considered an error by nginx. This flags allows skipping the error checks
* last_buf — flag, meaning that current buffer is the last in output
* last_in_chain — flag, meaning that there's no more data buffers in a (sub)request
* shadow — reference to another buffer, related to the current buffer. Usually current buffer uses data
 from the shadow buffer. Once current buffer is consumed, the shadow buffer should normally also be mar
ked as consumed
* last_shadow — flag, meaning that current buffer is the last buffer, referencing a particular shadow b
uffer
* temp_file — flag, meaning that the buffer is in a temporary file

For input and output buffers are linked in chains. Chain is a sequence of chain links ngx_chain_t, defi
ned as follows:

```
typedef struct ngx_chain_s  ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

Each chain link keeps a reference to its buffer and a reference to the next chain link.

Example of using buffers and chains:

```
ngx_chain_t *
ngx_get_my_chain(ngx_pool_t *pool)
{
    ngx_buf_t    *b;
    ngx_chain_t  *out, *cl, **ll;

    /* first buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_calloc_buf(pool);
    if (b == NULL) { /* error */ }

    b->start = (u_char *) "foo";
    b->pos = b->start;
    b->end = b->start + 3;
    b->last = b->end;
    b->memory = 1; /* read-only memory */

    cl->buf = b;
    out = cl;
    ll = &cl->next;

    /* second buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_create_temp_buf(pool, 3);
    if (b == NULL) { /* error */ }

    b->last = ngx_cpymem(b->last, "foo", 3);

    cl->buf = b;
    cl->next = NULL;
    *ll = cl;

    return out;
}
```

Networking
==========

Connection
----------

Connection type ngx_connection_t is a wrapper around a socket descriptor. Some of the structure fields are:

* fd — socket descriptor
* data — arbitrary connection context. Normally, a pointer to a higher level object, built on top of the connection, like HTTP request or Stream session
* read, write — read and write events for the connection
* recv, send, recv_chain, send_chain — I/O operations for the connection
* pool — connection pool
* log — connection log
* sockaddr, socklen, addr_text — remote socket address in binary and text forms
* local_sockaddr, local_socklen — local socket address in binary form. Initially, these fields are empty. Function ngx_connection_local_sockaddr() should be used to get socket local address
* proxy_protocol_addr, proxy_protocol_port - PROXY protocol client address and port, if PROXY protocol is enabled for the connection
* ssl — nginx connection SSL context
* reusable — flag, meaning, that the connection is at the state, when it can be reused
* close — flag, meaning, that the connection is being reused and should be closed

An nginx connection can transparently encapsulate SSL layer. In this case the connection ssl field holds a pointer to an ngx_ssl_connection_t structure, keeping all SSL-related data for the connection, including SSL_CTX and SSL. The handlers recv, send, recv_chain, send_chain are set as well to SSL functions.

The number of connections per nginx worker is limited by the worker_connections value. All connection structures are pre-created when a worker starts and stored in the connections field of the cycle object. To reach out for a connection structure, ngx_get_connection(s, log) function is used. The function receives a socket descriptor s which needs to be wrapped in a connection structure.

Since the number of connections per worker is limited, nginx provides a way to grab connections which are currently in use. To enable or disable reuse of a connection, function ngx_reusable_connection(c, reusable) is called. Calling ngx_reusable_connection(c, 1) sets the reuse flag of the connection structure and inserts the connection in the reusable_connections_queue of the cycle. Whenever ngx_get_connection() finds out there are no available connections in the free_connections list of the cycle, it calls ngx_drain_connections() to release a specific number of reusable connections. For each such connection, the close flag is set and its read handler is called which is supposed to free the connection by calling ngx_close_connection(c) and make it available for reuse. To exit the state when a connection can be reused ngx_reusable_connection(c, 0) is called. An example of reusable connections in nginx is HTTP client connections which are marked as reusable until some data is received from the client.


Events
======

Event
-----

Event object ngx_event_t in nginx provides a way to be notified of a specific event happening.

Some of the fields of the ngx_event_t are:

* data — arbitrary event context, used in event handler, usually, a pointer to a connection, tied with the event
* handler — callback function to be invoked when the event happens
* write — flag, meaning that this is the write event. Used to distinguish between read and write events
* active — flag, meaning that the event is registered for receiving I/O notifications, normally from notification mechanisms like epoll, kqueue, poll
* ready — flag, meaning that the event has received an I/O notification
* delayed — flag, meaning that I/O is delayed due to rate limiting
* timer — Red-Black tree node for inserting the event into the timer tree
* timer_set — flag, meaning that the event timer is set, but not yet expired
* timedout — flag, meaning that the event timer has expired
* eof — read event flag, meaning that the eof has happened while reading data
* pending_eof — flag, meaning that the eof is pending on the socket, even though there may be some data available before it. The flag is delivered via EPOLLRDHUP epoll event or EV_EOF kqueue flag
* error — flag, meaning that an error has happened while reading (for read event) or writing (for write event)
* cancelable — timer event flag, meaning that the event handler should be called while performing nginx worker graceful shutdown, event though event timeout has not yet expired. The flag provides a way to finalize certain activities, for example, flush log files
* posted — flag, meaning that the event is posted to queue
* queue — queue node for posting the event to a queue

I/O events
----------

Each connection, received with the ngx_get_connection() call, has two events attached to it: c->read and c->write. These events are used to receive notifications about the socket being ready for reading or writing. All such events operate in Edge-Triggered mode, meaning that they only trigger notifications when the state of the socket changes. For example, doing a partial read on a socket will not make nginx deliver a repeated read notification until more data arrive in the socket. Even when the underlying I/O notification mechanism is essentially Level-Triggered (poll, select etc), nginx will turn the notifications into Edge-Triggered. To make nginx event notifications consistent across all notifications systems on different platforms, it's required, that the functions ngx_handle_read_event(rev, flags) and ngx_handle_write_event(wev, lowat) are called after handling an I/O socket notification or calling any I/O functions on that socket. Normally, these functions are called once in the end of each read or write event handler.

Timer events
------------

An event can be set to notify a timeout expiration. The function ngx_add_timer(ev, timer) sets a timeout for an event, ngx_del_timer(ev) deletes a previously set timeout. Timeouts currently set for all existing events, are kept in a global timeout Red-Black tree ngx_event_timer_rbtree. The key in that tree has the type ngx_msec_t and is the time in milliseconds since the beginning of January 1, 1970 (modulus ngx_msec_t max value) at which the event should expire. The tree structure provides fast inserting and deleting operations, as well as accessing the nearest timeouts. The latter is used by nginx to find out for how long to wait for I/O events and for expiring timeout events afterwards.

Posted events
-------------

An event can be posted which means that its handler will be called at some point later within the current event loop iteration. Posting events is a good practice for simplifying code and escaping stack overflows. Posted events are held in a post queue. The macro ngx_post_event(ev, q) posts the event ev to the post queue q. Macro ngx_delete_posted_event(ev) deletes the event ev from whatever queue it's currently posted. Normally, events are posted to the ngx_posted_events queue. This queue is processed late in the event loop — after all I/O and timer events are already handled. The function ngx_event_process_posted() is called to process an event queue. This function calls event handlers until the queue is not empty. This means that a posted event handler can post more events to be processed within the current event loop iteration.

Example:

```
void
ngx_my_connection_read(ngx_connection_t *c)
{
    ngx_event_t  *rev;

    rev = c->read;

    ngx_add_timer(rev, 1000);

    rev->handler = ngx_my_read_handler;

    ngx_my_read(rev);
}


void
ngx_my_read_handler(ngx_event_t *rev)
{
    ssize_t            n;
    ngx_connection_t  *c;
    u_char             buf[256];

    if (rev->timedout) { /* timeout expired */ }

    c = rev->data;

    while (rev->ready) {
        n = c->recv(c, buf, sizeof(buf));

        if (n == NGX_AGAIN) {
            break;
        }

        if (n == NGX_ERROR) { /* error */ }

        /* process buf */
    }

    if (ngx_handle_read_event(rev, 0) != NGX_OK) { /* error */ }
}
```

Event loop
----------

All nginx processes which do I/O, have an event loop. The only type of process which does not have I/O, is nginx master process which spends most of its time in sigsuspend() call waiting for signals to arrive. Event loop is implemented in ngx_process_events_and_timers() function. This function is called repeatedly until the process exits. It has the following stages:

* find nearest timeout by calling ngx_event_find_timer(). This function finds the leftmost timer tree node and returns the number of milliseconds until that node expires
* process I/O events by calling a handler, specific to event notification mechanism, chosen by nginx configuration. This handler waits for at least one I/O event to happen, but no longer, than the nearest timeout. For each read or write event which has happened, the ready flag is set and its handler is called. For Linux, normally, the ngx_epoll_process_events() handler is used which calls epoll_wait() to wait for I/O events
* expire timers by calling ngx_event_expire_timers(). The timer tree is iterated from the leftmost element to the right until a not yet expired timeout is found. For each expired node the timedout event flag is set, timer_set flag is reset, and the event handler is called
* process posted events by calling ngx_event_process_posted(). The function repeatedly removes the first element from the posted events queue and calls its handler until the queue gets empty

All nginx processes handle signals as well. Signal handlers only set global variables which are checked after the ngx_process_events_and_timers() call.

进程
=========

nginx有好几种进程类型。当前进程的类型保存在ngx_process这个全局变量。

* NGX_PROCESS_MASTER — 主进程运行ngx_master_process_cycle()这个函数。主进程不能有任何的I/O，并且只对信号响应。它读取配置，创建cycle，启动和控制子进程。

* NGX_PROCESS_WORKER — 工作进程运行ngx_worker_process_cycle()函数。工作进程由子进程创建，处理客户端连接。他们同样也响应来自主进程的信号。

* NGX_PROCESS_SINGLE — 单进程只存在于master_process模式模式的情况下。生命周期函数是ngx_single_process_cycle()。这个进程创建生命周期并且处理客户端连接。

* NGX_PROCESS_HELPER — 目前只有两种help进程：cache manager 和 cache loader. 它们共用同样的生命周期函数ngx_cache_manager_process_cycle()。

所有的nginx处理如下信号：

* NGX_SHUTDOWN_SIGNAL (SIGQUIT) — 优雅结束。收到此信号后主进程发送 shutdown 信号给所有的子进程。当没有任何子进程时，主进程释放生命周期内存池然后结束。工作进程收到此信号后，关闭所有的监听端口然后一直等到超时树为空，最后释放生命周期内存池并且结束。cache 管理进程收到这个信号后立马退出。收到信号后 ngx_quit 设置为0，然后在处理完成后立马重置。ngx_exiting 在工作进程处理退出状态时设置为1。

* NGX_TERMINATE_SIGNAL (SIGTERM) - 终止。. 收到此信号后主进程发送 terminate 信号给所有的子进程。如果子进程1秒内没结束，它们会通过SIGKILL 信号被杀掉。当没有任何子进程时，主进程释放生命周期内存池然后结束。工作进程或cache管理进程释放生命周期内存池并且结束。ngx_terminate 在收到结信号后设置为1.

* NGX_NOACCEPT_SIGNAL (SIGWINCH) - 优雅结束工作进程。

* NGX_RECONFIGURE_SIGNAL (SIGHUP) - 配置热加载。 收到此信号后主进程根据配置文件创建新的cycle。如果这个新的cycle被成功的创建了，旧的cycle会被删除并且启动新的子进程。同时旧进程会被到 shutdown 信号。在单进程模式下，nginx 同样创建新的cycle，但是旧的会一直保留到所有跟它关联的连接都结束了。工作进程和helper进程忽略这种信号。

* NGX_REOPEN_SIGNAL (SIGUSR1) — 重新打开文件。主进程发送这个信号给工作进程。工作进程重新打开来自cycle的open_files。

* NGX_CHANGEBIN_SIGNAL (SIGUSR2) — 更新可执行程序。主进程启动新的可执行程序，将所有的监听文件描述符传给它。这些列表是通过环境变量“NGINX” 传递的，描述符值以分号分隔。新的nginx实例读这个变量然后将socket描述符添加到自己的初始cycle。其它进程忽略这种信号。

虽然nginx工作进程可以接受和处理POSIX信号，但是主进程却不通过调用标准kill()给工作进程和help进程发送信号。nginx通过内部进程间通道发送消息。即使这样，目前nginx也只是从主进程给工作进程发送消息。这些消息携带同样的信号。这些通过是socketpairs，其对端在不同的进程。

当运行可执行程序，可以通过-s参数指定几种值。分别是 stop, quit, reopen, reload。它们被转化成信号 NGX_TERMINATE_SIGNAL, NGX_SHUTDOWN_SIGNAL, NGX_REOPEN_SIGNAL 和 NGX_RECONFIGURE_SIGNAL 并且被发送给nginx主进程，通过从nginx pid文件获取进程id。

模块
=======

添加新模块
--------
标准nginx模块位于独立的目录，至少包含两个文件：config和包含模块源码的文件。config包含需要跟nginx整合的信息，比如：
```
ngx_module_type=CORE
ngx_module_name=ngx_foo_module
ngx_module_srcs="$ngx_addon_dir/ngx_foo_module.c"

. auto/module

ngx_addon_name=$ngx_module_name
```

这是个POSIX shell脚本，它能设置（或访问）以下变量：

* ngx_module_type — 模块类型。可选值包括 CORE, HTTP, HTTP_FILTER, HTTP_INIT_FILTER, HTTP_AUX_FILTER, MAIL, STREAM, or MISC
* ngx_module_name — 模块名称。可以用空格分隔并且单个源文件可以构造多个模块。如果是动态模块，第一个名称将作为二制进文件的名称。这些名称必须跟模块里面的能匹配。
* ngx_addon_name — 该模块在控制台的输出文本。
* ngx_module_srcs — 编译该模块时用到的源文件列表，用空格分隔。$ngx_addon_dir 变量可用作替代符，表示模块的当前路径。
* ngx_module_incs — 用于构建该模块的包含路径。
* ngx_module_deps — 模块依赖头文件列表。
* ngx_module_libs — 模块用到的链接库列表。 举个例子，libpthread 可以这样被链接 ngx_module_libs=-lpthread。这些宏可以直接在nginx里使用： LIBXSLT, LIBGD, GEOIP, PCRE, OPENSSL, MD5, SHA1, ZLIB, and PERL
* ngx_module_link — 模块链接形式，DYNAMIC表示动态模块，ADDON表示静态模块，其它根据不同的值会执行不同的操作。
* ngx_module_order — 模块顺序，设置模块的加载顺序在 HTTP_FILTER 和 HTTP_AUX_FILTER 类型的模块中是很有用的。模块按反序加载。
  在列表底部附近的 ngx_http_copy_filter_module 是最先被执行的。它读数据给其它的filter使用。在列表头部附近的ngx_http_write_filter_module 输出数据，并且是最后执行的。

  选项格式是这样的：当前模块名称紧接着用空格分隔的模块列表，这些列表位置靠前，但执行是靠后。这个模块将被插入在这个列表最后一个模块的前面。
  
  对filter模块默认是“ngx_http_copy_filter”，这样该模块被插入在copy filter之前，执行也就是copy filter的后面。对其它类型模块默认值为空。

模块通过使用 --add-module=/path/to/module 表示静态编译，--add-dynamic-module=/path/to/module 表示动态编译。

核心模块
-------

模块是nginx的构建方式，nginx的大部份功能也被实现成模块。模块源文件必须包含类型为 ngx_module_t 的全局变量，定义为：

```
struct ngx_module_s {

    /* private part is omitted */

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    /* stubs for future extensions are omitted */
};
```

省略私有部分包含模块版本，签名和预定义的宏 NGX_MODULE_V1。

每个模块包含有私有数据的上下文，特定的配置指令，命令数组，还有可能在特写被调用的函数钩子。模块的生命周期由下面这些组成：

* 配置指令处理函数在master进程解析配置文件时被调用。
* init_module 在master进程成功解析配置后调用。
* master进程创建了worker进程，然后调用这些worker进程各自的 init_process。
* 当一个工作进程收到来自master的shutdown命令后 exit_process 被调用。
* master进程在退出前调用 exit_master。

init_module 可能会被调用多次，如果master进程做了配置的reload。

init_master, init_thread and exit_thread 目前是没有实现的；线程在nginx里用于补充处理IO功能，而init_master看起来不是必须的。

type定义了模块类型，有以下几种：

* NGX_CORE_MODULE
* NGX_EVENT_MODULE
* NGX_HTTP_MODULE
* NGX_MAIL_MODULE
* NGX_STREAM_MODULE

NGX_CORE_MODULE 是最基础和通用的，处于最低层次的类型。其它类型都依赖在它上面，并且提供更方便的方式去处理各自领域的问题，比如事件和http请求。

核心模块有 ngx_core_module, ngx_errlog_module, ngx_regex_module, ngx_thread_pool_module, ngx_openssl_module，当然 http, stream, mail and event 也是。核心模块的上下文定义如下：

```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

name只是用于方便识别的模块字符串名称，create_conf 和 init_conf 指向创建和初始模块对应的配置结构体。对核心模块，create_conf在解析配置之前被调用， init_conf 在配置成功解析后调用。典型的 create_conf 函数分配空间用于配置，并且设置默认值。init_conf 处理已知配置，然后执行合理的校验和完成配置初始化。

举个例子，很简单的模块 ngx_foo_module 是这样的：

```
/*
 * Copyright (C) Author.
 */


#include <ngx_config.h>
#include <ngx_core.h>


typedef struct {
    ngx_flag_t  enable;
} ngx_foo_conf_t;


static void *ngx_foo_create_conf(ngx_cycle_t *cycle);
static char *ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf);

static char *ngx_foo_enable(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_enable_post = { ngx_foo_enable };


static ngx_command_t  ngx_foo_commands[] = {

    { ngx_string("foo_enabled"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_foo_conf_t, enable),
      &ngx_foo_enable_post },

      ngx_null_command
};


static ngx_core_module_t  ngx_foo_module_ctx = {
    ngx_string("foo"),
    ngx_foo_create_conf,
    ngx_foo_init_conf
};


ngx_module_t  ngx_foo_module = {
    NGX_MODULE_V1,
    &ngx_foo_module_ctx,                   /* module context */
    ngx_foo_commands,                      /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static void *
ngx_foo_create_conf(ngx_cycle_t *cycle)
{
    ngx_foo_conf_t  *fcf;

    fcf = ngx_pcalloc(cycle->pool, sizeof(ngx_foo_conf_t));
    if (fcf == NULL) {
        return NULL;
    }

    fcf->enable = NGX_CONF_UNSET;

    return fcf;
}


static char *
ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_foo_conf_t *fcf = conf;

    ngx_conf_init_value(fcf->enable, 0);

    return NGX_CONF_OK;
}


static char *
ngx_foo_enable(ngx_conf_t *cf, void *post, void *data)
{
    ngx_flag_t  *fp = data;

    if (*fp == 0) {
        return NGX_CONF_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cf->log, 0, "Foo Module is enabled");

    return NGX_CONF_OK;
}
```

配置指令
------------------------

ngx_command_t 表示一个配置指令。每个模块包含一组指令，每个指令的格式表示了如何处理参数和解析时调用的函数。

```
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

指令数组以 “ngx_null_command” 结束。name 是指令名称，体现在配置文件中，比如 “worker_processes” or “listen”。type 是bit组合，表示参数个数，指令类型和其它对应的属性。参数的标记为：

* NGX_CONF_NOARGS — 没有参数
* NGX_CONF_1MORE — 至少一个参数
* NGX_CONF_2MORE — 至少两个参数
* NGX_CONF_TAKE1..7 — 明确的1..7个参数
* NGX_CONF_TAKE12, 13, 23, 123, 1234 — 一个或两个参数，一个或参数，依此类推。

指令类型：

* NGX_CONF_BLOCK — 表示是一个块，比如它可能用 { } 包含其它指令，或自己实现的解析以处理包含的内容，比如 map 指领。
* NGX_CONF_FLAG — 表示是个boolean的标记，“on” 或者 “off”。

指令的上下文定义了配置的位置，并且关联到对应的存储配置的地方。

* NGX_MAIN_CONF — 上层配置
* NGX_HTTP_MAIN_CONF — http 块
* NGX_HTTP_SRV_CONF — http server 块
* NGX_HTTP_LOC_CONF — http location 块
* NGX_HTTP_UPS_CONF — http upstream 块
* NGX_HTTP_SIF_CONF — http server “if” 块
* NGX_HTTP_LIF_CONF — http location “if” 块
* NGX_HTTP_LMT_CONF — http “limit_except” 块
* NGX_STREAM_MAIN_CONF — stream 块
* NGX_STREAM_SRV_CONF — stream server 块
* NGX_STREAM_UPS_CONF — stream upstream 块
* NGX_MAIL_MAIN_CONF — mail 块
* NGX_MAIL_SRV_CONF — mail server 块
* NGX_EVENT_CONF — event 块
* NGX_DIRECT_CONF — 没有层级的上下文，直接存储在模块的ctx

配置解析时根据这些标记，要么对放错位置的指令抛出错误，要么调用指令handler，这样即使相同的配置在不同的location也能存储到能区分的位置。

set字段定义了解析配置时调用的handler，并且将解析的值存放到对应的配置结构体。Nginx提供了一些方便的公共函数集：

* ngx_conf_set_flag_slot — 将 “on” or “off” 转化成 ngx_flag_t 类型的值 1 or 0
* ngx_conf_set_str_slot — 存储类型为 ngx_str_t 的值
* ngx_conf_set_str_array_slot — 追加元素为ngx_str_t的ngx_array_t一个新的值。array会自动创建，如果不存在的话。
* ngx_conf_set_keyval_slot — 追加元素为ngx_keyval_t的ngx_array_t一个新的值。第一个作为键，第二个作为值，如果不存在的话。
* ngx_conf_set_num_slot — 转化参数为 ngx_int_t 类型的值
* ngx_conf_set_size_slot — 转化参数为 size_t 类型的值
* ngx_conf_set_off_slot — 转化参数为 off_t 类型的值
* ngx_conf_set_msec_slot — 转化参数为 ngx_msec_t 类型的值
* ngx_conf_set_sec_slot — 转化参数为 time_t 类型的值
* ngx_conf_set_bufs_slot — 转化两个参数为 ngx_bufs_t，包含了 ngx_int_t 类型的 number 和 buffers的size
* ngx_conf_set_enum_slot — 转化参数为 ngx_uint_t 类型的值。这是个类似枚举的功能，可以传以 null-terminated 结尾的 ngx_conf_enum_t 数组给post字段，以设置对应的值。
* ngx_conf_set_bitmask_slot — 转化参数为 ngx_uint_t 类型的值。这是个类似枚举的功能，可以传以 null-terminated ngx_conf_bitmask_t 数组给post字段，以设置对应的值。 
* set_path_slot — 转化参数为 ngx_path_t 类型并且做必须的初始化。详情请看 proxy_temp_path 指令
* set_access_slot — 转化参数为文件权限mask。详情请看 proxy_store_access 指令。

conf字段定义了哪个上下文用来存储指令的值，或者用NULL表示不使用上下文。简单的核心模块不用配置上下文并且设置 NGX_DIRECT_CONF 标识。 在真实场景里，像http或stream的模块往往更复杂，配置可以在pre-server或者pre-location里，还有甚至是在 "if" 里的。这样的模块里，配置结构会更复杂，请到一些模块里看他们是如何管理各自的配置的。

* NGX_HTTP_MAIN_CONF_OFFSET — http 块配置
* NGX_HTTP_SRV_CONF_OFFSET — http 块配置
* NGX_HTTP_LOC_CONF_OFFSET — http 块配置
* NGX_STREAM_MAIN_CONF_OFFSET — stream 块配置
* NGX_STREAM_SRV_CONF_OFFSET — stream server 块配置
* NGX_MAIL_MAIN_CONF_OFFSET — mail 块配置
* NGX_MAIL_SRV_CONF_OFFSET — mail server 块配置

offset字段定义了存储该指令值的位置在配置结构体的偏移大小。典型的使用是调用 offsetof() 宏。

post字段包含双重意思：它可能在主handler完成后调用，或者传额外的数据给主handler。第一种情况 ngx_conf_post_t 需要初始化handler，举个例子：

```
static char *ngx_do_foo(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_post = { ngx_do_foo };
```

post函数参数是：ngx_conf_post_t它自己, data 来自主handler的参数。


HTTP
====

Connection
----------
TODO

Request
-------
TODO

Configuration
-------------
TODO

Phases
------
Each HTTP request passes through a list of HTTP phases. Each phase is specialized in a particular type of processing. Most phases allow installing handlers. The phase handlers are called successively once the request reaches the phase. Many standard nginx modules install their phase handlers as a way to get called at a specific request processing stage. Following is the list of nginx HTTP phases.

* NGX_HTTP_POST_READ_PHASE is the earliest phase. The ngx_http_realip_module installs its handler at this phase. This allows to substitute client address before any other module is invoked
* NGX_HTTP_SERVER_REWRITE_PHASE is used to run rewrite script, defined at the server level, that is out of any location block. The ngx_http_rewrite_module installs its handler at this phase
* NGX_HTTP_FIND_CONFIG_PHASE — a special phase used to choose a location based on request URI. This phase does not allow installing any handlers. It only performs the default action of choosing a location. Before this phase, the server default location is assigned to the request. Any module requesting a location configuration, will receive the default server location configuration. After this phase a new location is assigned to the request
* NGX_HTTP_REWRITE_PHASE — same as NGX_HTTP_SERVER_REWRITE_PHASE, but for a new location, chosen at the prevous phase
* NGX_HTTP_POST_REWRITE_PHASE — a special phase, used to redirect the request to a new location, if the URI was changed during rewrite. The redirect is done by going back to NGX_HTTP_FIND_CONFIG_PHASE. No handlers are allowed at this phase
* NGX_HTTP_PREACCESS_PHASE — a common phase for different types of handlers, not associated with access check. Standard nginx modules ngx_http_limit_conn_module and ngx_http_limit_req_module register their handlers at this phase
* NGX_HTTP_ACCESS_PHASE — used to check access permissions for the request. Standard nginx modules such as ngx_http_access_module and ngx_http_auth_basic_module register their handlers at this phase. If configured so by the satisfy directive, only one of access phase handlers may allow access to the request in order to confinue processing
* NGX_HTTP_POST_ACCESS_PHASE — a special phase for the satisfy any case. If some access phase handlers denied the access and none of them allowed, the request is finalized. No handlers are supported at this phase
* NGX_HTTP_TRY_FILES_PHASE — a special phase, for the try_files feature. No handlers are allowed at this phase
* NGX_HTTP_CONTENT_PHASE — a phase, at which the response is supposed to be generated. Multiple nginx standard modules register their handers at this phase, for example ngx_http_index_module or ngx_http_static_module. All these handlers are called sequentially until one of them finally produces the output. It's also possible to set content handlers on a per-location basis. If the ngx_http_core_module's location configuration has handler set, this handler is called as the content handler and content phase handlers are ignored
* NGX_HTTP_LOG_PHASE is used to perform request logging. Currently, only the ngx_http_log_module registers its handler at this stage for access logging. Log phase handlers are called at the very end of request processing, right before freeing the request

Following is the example of a preaccess phase handler.

```
static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_foo_init,                     /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_str_t  *ua;

    ua = r->headers_in->user_agent;

    if (ua == NULL) {
        return NGX_DECLINED;
    }

    /* reject requests with "User-Agent: foo" */
    if (ua->value.len == 3 && ngx_strncmp(ua->value.data, "foo", 3) == 0) {
        return NGX_HTTP_FORBIDDEN;
    }

    return NGX_DECLINED;
}


static ngx_int_t
ngx_http_foo_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_foo_handler;

    return NGX_OK;
}
```

Phase handlers are expected to return specific codes:

* NGX_OK — proceed to the next phase
* NGX_DECLINED — proceed to the next handler of the current phase. If current handler is the last in current phase, move to the next phase
* NGX_AGAIN, NGX_DONE — suspend phase handling until some future event. This can be for example asynchronous I/O operation or just a delay. It is supposed, that phase handling will be resumed later by calling ngx_http_core_run_phases()
* Any other value returned by the phase handler is treated as a request finalization code, in particular, HTTP response code. The request is finalized with the code provided

Some phases treat return codes in a slightly different way. At content phase, any return code other that NGX_DECLINED is considered a finalization code. As for the location content handlers, any return from them is considered a finalization code. At access phase, in satisfy any mode, returning a code other than NGX_OK, NGX_DECLINED, NGX_AGAIN, NGX_DONE is considered a denial. If none of future access handlers allow access or deny with a new code, the denial code will become the finalization code.

Variables
---------
TODO

Complex values
--------------
TODO

Request redirection
-------------------
TODO

Subrequests
-----------
TODO

Request finalization
--------------------
TODO

Request body
------------
TODO

Response
--------
TODO

Response body
-------------
TODO

Body filters
------------
TODO

Building filter modules
-----------------------
TODO

Buffer reuse
------------
TODO

负载均衡
-------
ngx_http_upstream_module提供了向远程服务器发送HTTP请求的基本功能。其他具体的协议模块，例如HTTP或FastCDI，都会使用这个功能。该模块同时还提供了可以定制负载均衡算法的接口并默认实现了round-robin（轮询）算法

例如，提供其他的负载均衡算法的模块有least_conn和hash这些。需要注意的是，这些模块实际上是作为upstream模块的扩展而实现的，他们之间共享了大量的代码，比如对于服务器组的表示。keepalive模块是另外一个例子，这是一个独立的模块，扩展了upstream的功能。

ngx_http_upstream_module可以通过在配置文件中配置upstream块来显式配置，或者通过使用可以接受URL作为参数的指令来隐式开启，比如proxy_pass这种指令。只有显示的配置才能选择负载均衡算法。upstream模块有自己的指令上下文NGX_HTTP_UPS_CONF。相关结构体定义如下：

```
struct ngx_http_upstream_srv_conf_s {
    ngx_http_upstream_peer_t         peer;
    void                           **srv_conf;

    ngx_array_t                     *servers;  /* ngx_http_upstream_server_t */

    ngx_uint_t                       flags;
    ngx_str_t                        host;
    u_char                          *file_name;
    ngx_uint_t                       line;
    in_port_t                        port;
    ngx_uint_t                       no_port;  /* unsigned no_port:1 */

#if (NGX_HTTP_UPSTREAM_ZONE)
    ngx_shm_zone_t                  *shm_zone;
#endif
};
```

* srv_conf — upstream模块的配置上下文
* servers — ngx_http_upstream_server_t的数组，存放的是对upstream块中一组server指令解析的配置
* flags — 指定特定负载均衡算法支持哪些特性（通过server指令的参数配置）的标记位。
   * NGX_HTTP_UPSTREAM_CREATE — 用来区分显式定义的upstream和通过proxy_pass类型指令(FastCGI, SCGI等)隐式创建的upstream
   * NGX_HTTP_UPSTREAM_WEIGHT — 支持“weight”
   * NGX_HTTP_UPSTREAM_MAX_FAILS — 支持“max_fails”
   * NGX_HTTP_UPSTREAM_FAIL_TIMEOUT — 支持“fail_timeout”
   * NGX_HTTP_UPSTREAM_DOWN — 支持“down”
   * NGX_HTTP_UPSTREAM_BACKUP — 支持“backup”
   * NGX_HTTP_UPSTREAM_MAX_CONNS — 支持“max_conns”
* host — upstream的名字
* file_name, line — 配置文件名字以及upstream块所在行
* port and no_port — 显式upstream未使用
* shm_zone — 此upstream使用的共享内存
* peer — 存放用来初始化upstream配置通用方法的对象：

```
typedef struct {
    ngx_http_upstream_init_pt        init_upstream;
    ngx_http_upstream_init_peer_pt   init;
    void                            *data;
} ngx_http_upstream_peer_t;
```

实现负载均衡算法的模块必须设置这些方法并初始化私有数据。
如果init_upstream在配置阶段没有初始化，ngx_http_upstream_module会将其默认设置成ngx_http_upstream_init_round_robin。

   * init_upstream(cf, us) — 配置阶段方法，用于初始化一组服务器并初始化init()方法。一个典型的负载均衡模块使用upstream块中的一组服务器来创建某种有效的数据结构并在data成员中存放自身的配置。
   * init(r, us) — 初始化用于每个请求的ngx_http_upstream_peer_t.peer (不要和之前用于每个upstream的ngx_http_upstream_srv_conf_t.peer搞混了)结构，该结构用于进行负载均衡。该结构会作为所有处理服务器选择的回调函数的data参数传递。

当nginx需要将请求转给其他服务器进行处理时，它会调用配置好的负载均衡算法来选择一个地址，并发起连接。选择算法是从ngx_http_upstream_peer_t.peer对象中获取的，该对象的类型是ngx_peer_connection_t：

```
struct ngx_peer_connection_s {
    [...]

    struct sockaddr                 *sockaddr;
    socklen_t                        socklen;
    ngx_str_t                       *name;

    ngx_uint_t                       tries;

    ngx_event_get_peer_pt            get;
    ngx_event_free_peer_pt           free;
    ngx_event_notify_peer_pt         notify;
    void                            *data;

#if (NGX_SSL || NGX_COMPAT)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

    [..]
};
```

这个结构体有如下成员：

* sockaddr, socklen, name — 待连接的upstream服务器的地址；此为负载均衡算法的输出参数
* data — 每请求的负载均衡算法所需数据；记录选择算法的状态并且通常会含有指向upstream配置的指针。此data会被作为参数传递给所有处理服务器选择的函数（见下文）
* tries — 连接upstream服务器的重试次数
* get, free, notify, set_session, and save_session - 负载均衡算法模块的方法，详细见下文

所有的方法至少接受两个参数：peer连接对象pc以及由ngx_http_upstream_srv_conf_t.peer.init()创建的data参数。注意，一般来说，由于负载均衡算法的”chaining”，这个data和pc.data是不同的，

* get(pc, data) — 当upstream模块需要将请求发送给一个服务器而需要知道服务器地址的时候，该方法会被调用。该方法负责填写ngx_peer_connection_t结构的sockaddr，socklen和name成员。返回值有如下几种：
   * NGX_OK — 服务器已选择
   * NGX_ERROR — 发生了内部错误
   * NGX_BUSY — 当前没有可用服务器。有多种原因会导致这个情况的发生，例如：动态服务器组为空，全部服务器均为失败状态，全部服务器已经达到最大连接数或者其他类似情况。
   * NGX_DONE — keepalive模块用这个返回值来说明底层连接进行了复用，因此不需要和upstream服务器间创建一条新连接。
* free(pc, data, state) — 当upstream模块同某个upstream服务器通信结束后，调用此方法。state参数指示了upstream连接的完成状态，是一个bitmask，可以被设置成这些值：NGX_PEER_FAILED - 失败，NGX_PEER_NEXT - 403和404的特殊情况，不作为失败对待，NGX_PEER_KEEPALIVE。此外，尝试次数也在这个方法递减。
* notify(pc, data, type) — 开源版本中未使用。
* set_session(pc, data)和save_session(pc, data) — SSL相关方法，用于缓存同upstream服务器间的SSL会话，由round-robin负载均衡算法实现。
