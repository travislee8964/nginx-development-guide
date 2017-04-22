Development guide
===========

* [Introduction](#introduction)
    * [Code layout](#code-layout)
    * [Include files](#include-files)
    * [Integers](#integers)
    * [Common return codes](#common-return-codes)
    * [Error handling](#error-handling)
* [Strings](#strings)
    * [Overview](#overview)
    * [Formatting](#formatting)
    * [Numeric conversion](#numeric-conversion)
    * [Regular expressions](#regular-expressions)
* [Containers](#containers)
    * [Array](#array)
    * [List](#list)
    * [Queue](#queue)
    * [Red Black tree](#red-Black-tree)
    * [Hash](#hash)
* [Memory management](#memory-management)
    * [Heap](#heap)
    * [Pool](#pool)
    * [Shared memory](#shared-memory)
* [Logging](#logging)
* [Cycle](#cycle)
* [Buffer](#buffer)
* [Networking](#networking)
    * [Connection](#connection)
* [Events](#events)
    * [Event](#event)
    * [I/O events](#i/O-events)
    * [Timer events](#timer-events)
    * [Posted events](#posted-events)
    * [Event loop](#event-loop)
* [Processes](#processes)
* [Modules](#modules)
    * [Adding new modules](#adding-new-modules)
    * [Core modules](#core-modules)
    * [Configuration directives](#configuration-directives)
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
    * [Load balancing](#load-balancing)
    
Introduction
============

Code layout
-----------
* auto — build scripts
* src
    * core — basic types and functions — string, array, log, pool etc
* event — event core
    * modules — event notification modules: epoll, kqueue, select etc
* http — core HTTP module and common code
    * modules — other HTTP modules
    * v2 — HTTPv2
* mail — mail modules
* os — platform-specific code
    * unix
    * win32
* stream — stream modules

Include files
-------------
Each nginx file should start with including the following two files:

```
#include <ngx_config.h>
#include <ngx_core.h>
```

In addition to that, HTTP code should include

```
#include <ngx_http.h>
```

Mail code should include

```
#include <ngx_mail.h>
```

Stream code should include

```
#include <ngx_stream.h>
```

Integers
--------
For general purpose, nginx code uses the following two integer types ngx_int_t and ngx_uint_t which are typedefs for intptr_t and uintptr_t.

Common return codes
-------------------
Most functions in nginx return the following codes:

* NGX_OK — operation succeeded
* NGX_ERROR — operation failed
* NGX_AGAIN — operation incomplete, function should be called again
* NGX_DECLINED — operation rejected, for example, if disabled in configuration. This is never an error
* NGX_BUSY — resource is not available
* NGX_DONE — operation done or continued elsewhere. Also used as an alternative success code
* NGX_ABORT — function was aborted. Also used as an alternative error code

Error handling
--------------
For getting the last system error code, the ngx_errno macro is available. It's mapped to errno on POSIX platforms and to GetLastError() call in Windows. For getting the last socket error number, the ngx_socket_errno macro is available. It's mapped to errno on POSIX systems as well, and to WSAGetLastError() call on Windows. For performance reasons the values of ngx_errno or ngx_socket_errno should not be accessed more than once in a row. The error value should be stored in a local variable of type ngx_err_t for using multiple times, if required. For setting errors, ngx_set_errno(errno) and ngx_set_socket_errno(errno) macros are available.

The values of ngx_errno or ngx_socket_errno can be passed to logging functions ngx_log_error() and ngx_log_debugX(), in which case system error text is added to the log message.

Example using ngx_errno:

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

Strings
=======

Overview
--------
For C strings, nginx code uses unsigned character type pointer u_char *.

The nginx string type ngx_str_t is defined as follows:

```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

The len field holds the string length, data holds the string data. The string, held in ngx_str_t, may or may not be null-terminated after the len bytes. In most cases it’s not. However, in certain parts of code (for example, when parsing configuration), ngx_str_t objects are known to be null-terminated, and that knowledge is used to simplify string comparison and makes it easier to pass those strings to syscalls.

A number of string operations are provided in nginx. They are declared in src/core/ngx_string.h. Some of them are wrappers around standard C functions:

* ngx_strcmp()
* ngx_strncmp()
* ngx_strstr()
* ngx_strlen()
* ngx_strchr()
* ngx_memcmp()
* ngx_memset()
* ngx_memcpy()
* ngx_memmove()

Some nginx-specific string functions:

* ngx_memzero() fills memory with zeroes
* ngx_cpymem() does the same as ngx_memcpy(), but returns the final destination address This one is handy for appending multiple strings in a row
* ngx_movemem() does the same as ngx_memmove(), but returns the final destination address.
* ngx_strlchr() searches for a character in a string, delimited by two pointers

Some case conversion and comparison functions:

* ngx_tolower()
* ngx_toupper()
* ngx_strlow()
* ngx_strcasecmp()
* ngx_strncasecmp()

Formatting
----------
A number of formatting functions are provided by nginx. These functions support nginx-specific types:
* ngx_sprintf(buf, fmt, ...)
* ngx_snprintf(buf, max, fmt, ...)
* ngx_slrintf(buf, last, fmt, ...)
* ngx_vslprint(buf, last, fmt, args)
* ngx_vsnprint(buf, max, fmt, args)

The full list of formatting options, supported by these functions, can be found in src/core/ngx_string.c. Some of them are:
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

The ‘u’ modifier makes most types unsigned, ‘X’/‘x’ convert output to hex.

Example:
```
u_char     buf[NGX_INT_T_LEN];
size_t     len;
ngx_int_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```


Numeric conversion
------------------
Several functions for numeric conversion are implemented in nginx:
* ngx_atoi(line, n) — converts a string of given length to a positive integer of type ngx_int_t. Returns NGX_ERROR on error
* ngx_atosz(line, n) — same for ssize_t type
* ngx_atoof(line, n) — same for off_t type
* ngx_atotm(line, n) — same for time_t type
* ngx_atofp(line, n, point) — converts a fixed point floating number of given length to a positive integer of type ngx_int_t. The result is shifted left by points decimal positions. The string representation of the number is expected to have no more than points fractional digits. Returns NGX_ERROR on error. For example, ngx_atofp("10.5", 4, 2) returns 1050
* ngx_hextoi(line, n) — converts hexadecimal representation of a positive integer to ngx_int_t. Returns NGX_ERROR on error

Regular expressions
-------------------
The regular expressions interface in nginx is a wrapper around the PCRE library. The corresponding header file is src/core/ngx_regex.h.

To use a regular expression for string matching, first, it needs to be compiled, this is usually done at configuration phase. Note that since PCRE support is optional, all code using the interface must be protected by the surrounding NGX_PCRE macro:

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

After successful compilation, ngx_regex_compile_t structure fields captures and named_captures are filled with count of all and named captures respectively found in the regular expression.

Later, the compiled regular expression may be used to match strings against it:

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

The arguments of ngx_regex_exec() are: the compiled regular expression re, string to match s, optional array of integers to hold found captures and its size. The captures array size must be a multiple of three, per requirements of the PCRE API. In the example, its size is calculated from a total number of captures plus one for the matched string itself.

Now, if there are matches, captures may be accessed:

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

The ngx_regex_exec_array() function accepts the array of ngx_regex_elt_t elements (which are just compiled regular expressions with associated names), a string to match and a log. The function will apply expressions from the array to the string until the match is found or no more expressions are left. The return value is NGX_OK in case of match and NGX_DECLINED otherwise, or NGX_ERROR in case of error.

Containers
==========

Array
-----
The nginx array type ngx_array_t is defined as follows

```
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

The elements of array are available through the elts field. The number of elements is held in the nelts field. The size field holds the size of a single element and is set when initializing the array.

An array can be created in a pool with the ngx_array_create(pool, n, size) call. An already allocated array object can be initialized with the ngx_array_init(array, pool, n, size) call.

```
ngx_array_t  *a, b;

/* create an array of strings with preallocated memory for 10 elements */
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

Adding elements to array are done with the following functions:

* ngx_array_push(a) adds one tail element and returns pointer to it
* ngx_array_push_n(a, n) adds n tail elements and returns pointer to the first one

If currently allocated memory is not enough for new elements, a new memory for elements is allocated and existing elements are copied to that memory. The new memory block is normally twice as large, as the existing one.

```
s = ngx_array_push(a);
ss = ngx_array_push_n(&b, 3);
```

List
----
List in nginx is a sequence of arrays, optimized for inserting a potentially large number of items. The list type is defined as follows:

```
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

The actual items are stored in list parts, defined as follows:

```
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

Initially, a list must be initialized by calling ngx_list_init(list, pool, n, size) or created by calling ngx_list_create(pool, n, size). Both functions receive the size of a single item and a number of items per list part. The ngx_list_push(list) function is used to add an item to the list. Iterating over the items is done by direct accessing the list fields, as seen in the example:

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

The primary use for the list in nginx is HTTP input and output headers.

The list does not support item removal. However, when needed, items can internally be marked as missing without actual removing from the list. For example, HTTP output headers which are stored as ngx_table_elt_t objects, are marked as missing by setting the hash field of ngx_table_elt_t to zero. Such items are explicitly skipped, when iterating over the headers.

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
* ngx_queue_data(q, type, link) — get reference to the beginning of a queue node data structure, considering the queue field offset in it

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

The src/core/ngx_rbtree.h header file provides access to the effective implementation of red-black trees.

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

To deal with a tree as a whole, you need two nodes: root and sentinel. Typically, they are added to some custom structure, thus allowing to organize your data into a tree which leaves contain a link to or embed your data.

To initialize a tree:

```
my_tree_t  root;

ngx_rbtree_init(&root.rbtree, &root.sentinel, insert_value_function);
```

The insert_value_function is a function that is responsible for traversing the tree and inserting new values into correct place. For example, the ngx_str_rbtree_insert_value functions is designed to deal with ngx_str_t type.

```
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                 ngx_rbtree_node_t *node,
                                 ngx_rbtree_node_t *sentinel)
```

Its arguments are pointers to a root node of an insertion, newly created node to be added, and a tree sentinel.

The traversal is pretty straightforward and can be demonstrated with the following lookup function pattern:

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

The compare() is a classic comparator function returning value less, equal or greater than zero. To speed up lookups and avoid comparing user objects that can be big, integer hash field is used.

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

Hash table functions are declared in src/core/ngx_hash.h. Exact and wildcard matching is supported. The latter requires extra setup and is described in a separate section below.

To initialize a hash, one needs to know the number of elements in advance, so that nginx can build the hash optimally. Two parameters that need to be configured are max_size and bucket_size. The details of setting up these are provided in a separate document. Usually, these two parameters are configurable by user. Hash initialization settings are stored as the ngx_hash_init_t type, and the hash itself is ngx_hash_t:

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

The key is a pointer to a function that creates hash integer key from a string. Two generic functions are provided: ngx_hash_key(data, len) and ngx_hash_key_lc(data, len). The latter converts a string to lowercase and thus requires the passed string to be writable. If this is not true, NGX_HASH_READONLY_KEY flag may be passed to the function, initializing array keys (see below).

The hash keys are stored in ngx_hash_keys_arrays_t and are initialized with ngx_hash_keys_array_init(arr, type):

```
ngx_hash_keys_arrays_t  foo_keys;

foo_keys.pool = cf->pool;
foo_keys.temp_pool = cf->temp_pool;

ngx_hash_keys_array_init(&foo_keys, NGX_HASH_SMALL);
```

The second parameter can be either NGX_HASH_SMALL or NGX_HASH_LARGE and controls the amount of preallocated resources for the hash. If you expect the hash to contain thousands elements, use NGX_HASH_LARGE.

The ngx_hash_add_key(keys_array, key, value, flags) function is used to insert keys into hash keys array;

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

This may fail, if max_size or bucket_size parameters are not big enough. When the hash is built, ngx_hash_find(hash, key, name, len) function may be used to look up elements:

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

To create a hash that works with wildcards, ngx_hash_combined_t type is used. It includes the hash type described above and has two additional keys arrays: dns_wc_head and dns_wc_tail. The initialization of basic properties is done similarly to a usual hash:

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

The function recognizes wildcards and adds keys into corresponding arrays. Please refer to the map module documentation for the description of the wildcard syntax and matching algorithm.

Depending on the contents of added keys, you may need to initialize up to three keys arrays: one for exact matching (described above), and two for matching starting from head or tail of a string:

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

The keys array needs to be sorted, and initialization results must be added to the combined hash. The initialization of dns_wc_tail array is done similarly.

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

* ngx_alloc(size, log) — allocate memory from system heap. This is a wrapper around malloc() with logging support. Allocation error and debugging information is logged to log
ngx_calloc(size, log) — same as ngx_alloc(), but memory is filled with zeroes after allocation
* ngx_memalign(alignment, size, log) — allocate aligned memory from system heap. This is a wrapper around posix_memalign() on those platforms which provide it. Otherwise implementation falls back to ngx_alloc() which provides maximum alignment
ngx_free(p) — free allocated memory. This is a wrapper around free()

Pool
----

Most nginx allocations are done in pools. Memory allocated in an nginx pool is freed automatically when the pool in destroyed. This provides good allocation performance and makes memory control easy.

A pool internally allocates objects in continuous blocks of memory. Once a block is full, a new one is allocated and added to the pool memory block list. When a large allocation is requested which does not fit into a block, such allocation is forwarded to the system allocator and the returned pointer is stored in the pool for further deallocation.

Nginx pool has the type ngx_pool_t. The following operations are supported:

* ngx_create_pool(size, log) — create a pool with given block size. The pool object returned is allocated in the pool as well.
* ngx_destroy_pool(pool) — free all pool memory, including the pool object itself.
* ngx_palloc(pool, size) — allocate aligned memory from pool
* ngx_pcalloc(pool, size) — allocated aligned memory from pool and fill it with zeroes
* ngx_pnalloc(pool, size) — allocate unaligned memory from pool. Mostly used for allocating strings
* ngx_pfree(pool, p) — free memory, previously allocated in the pool. Only allocations, forwarded to the system allocator, can be freed.

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

Since chain links ngx_chain_t are actively used in nginx, nginx pool provides a way to reuse them. The chain field of ngx_pool_t keeps a list of previously allocated links ready for reuse. For efficient allocation of a chain link in a pool, the function ngx_alloc_chain_link(pool) should be used. This function looks up a free chain link in the pool list and only if it's empty allocates a new one. To free a link ngx_free_chain(pool, cl) should be called.

Cleanup handlers can be registered in a pool. Cleanup handler is a callback with an argument which is called when pool is destroyed. Pool is usually tied with a specific nginx object (like HTTP request) and destroyed in the end of that object’s lifetime, releasing the object itself. Registering a pool cleanup is a convenient way to release resources, close file descriptors or make final adjustments to shared data, associated with the main object.

A pool cleanup is registered by calling ngx_pool_cleanup_add(pool, size) which returns ngx_pool_cleanup_t pointer to be filled by the caller. The size argument allows allocating context for the cleanup handler.

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

Shared memory is used by nginx to share common data between processes. Function ngx_shared_memory_add(cf, name, size, tag) adds a new shared memory entry ngx_shm_zone_t to the cycle. The function receives name and size of the zone. Each shared zone must have a unique name. If a shared zone entry with the provided name exists, the old zone entry is reused, if its tag value matches too. Mismatched tag is considered an error. Usually, the address of the module structure is passed as tag, making it possible to reuse shared zones by name within one nginx module.

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
    * exists — flag, showing that shared memory was inherited from the master process (Windows-specific)

Shared zone entries are mapped to actual memory in ngx_init_cycle() after configuration is parsed. On POSIX systems, mmap() syscall is used to create shared anonymous mapping. On Windows, CreateFileMapping()/MapViewOfFileEx() pair is used.

For allocating in shared memory, nginx provides slab pool ngx_slab_pool_t. In each nginx shared zone, a slab pool is automatically created for allocating memory in that zone. The pool is located in the beginning of the shared zone and can be accessed by the expression (ngx_slab_pool_t *) shm_zone->shm.addr. Allocation in shared zone is done by calling one of the functions ngx_slab_alloc(pool, size)/ngx_slab_calloc(pool, size). Memory is freed by calling ngx_slab_free(pool, p).

Slab pool divides all shared zone into pages. Each page is used for allocating objects of the same size. Only the sizes which are powers of 2, and not less than 8, are considered. Other sizes are rounded up to one of these values. For each page, a bitmask is kept, showing which blocks within that page are in use and which are free for allocation. For sizes greater than half-page (usually, 2048 bytes), allocation is done by entire pages.

To protect data in shared memory from concurrent access, mutex is available in the mutex field of ngx_slab_pool_t. The mutex is used by the slab pool while allocating and freeing memory. However, it can be used to protect any other user data structures, allocated in the shared zone. Locking is done by calling ngx_shmtx_lock(&shpool->mutex), unlocking is done by calling ngx_shmtx_unlock(&shpool->mutex).

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

Logging
=======

For logging nginx code uses ngx_log_t objects. Nginx logger provides support for several types of output:

* stderr — logging to standard error output
* file — logging to file
* syslog — logging to syslog
* memory — logging to internal memory storage for development purposes. The memory could be accessed later with debugger

A logger instance may actually be a chain of loggers, linked to each other with the next field. Each message is written to all loggers in chain.

Each logger has an error level which limits the messages written to that log. The following error levels are supported by nginx:

* NGX_LOG_EMERG
* NGX_LOG_ALERT
* NGX_LOG_CRIT
* NGX_LOG_ERR
* NGX_LOG_WARN
* NGX_LOG_NOTICE
* NGX_LOG_INFO
* NGX_LOG_DEBUG

For debug logging, debug mask is checked as well. The following debug masks exist:

* NGX_LOG_DEBUG_CORE
* NGX_LOG_DEBUG_ALLOC
* NGX_LOG_DEBUG_MUTEX
* NGX_LOG_DEBUG_EVENT
* NGX_LOG_DEBUG_HTTP
* NGX_LOG_DEBUG_MAIL
* NGX_LOG_DEBUG_STREAM

Normally, loggers are created by existing nginx code from error_log directives and are available at nearly every stage of processing in cycle, configuration, client connection and other objects.

Nginx provides the following logging macros:

* ngx_log_error(level, log, err, fmt, ...) — error logging
* ngx_log_debug0(level, log, err, fmt), ngx_log_debug1(level, log, err, fmt, arg1) etc — debug logging, up to 8 formatting arguments are supported

A log message is formatted in a buffer of size NGX_MAX_ERROR_STR (currently, 2048 bytes) on stack. The message is prepended with error level, process PID, connection id (stored in log->connection) and system error text. For non-debug messages log->handler is called as well to prepend more specific information to the log message. HTTP module sets ngx_http_log_error() function as log handler to log client and server addresses, current action (stored in log->action), client request line, server name etc.

Example:

```
/* specify what is currently done */
log->action = "sending mp4 to client”;

/* error and debug log */
ngx_log_error(NGX_LOG_INFO, c->log, 0, "client prematurely
              closed connection”);

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, mp4->file.log, 0,
               "mp4 start:%ui, length:%ui”, mp4->start, mp4->length);
```

Logging result:

```
2016/09/16 22:08:52 [info] 17445#0: *1 client prematurely closed connection while
sending mp4 to client, client: 127.0.0.1, server: , request: "GET /file.mp4 HTTP/1.1”
2016/09/16 23:28:33 [debug] 22140#0: *1 mp4 start:0, length:10000
```

Cycle
=====

Cycle object keeps nginx runtime context, created from a specific configuration. The type of the cycle is ngx_cycle_t. Upon configuration reload a new cycle is created from the new version of nginx configuration. The old cycle is usually deleted after a new one is successfully created. Currently active cycle is held in the ngx_cycle global variable and is inherited by newly started nginx workers.

A cycle is created by the function ngx_init_cycle(). The function receives the old cycle as the argument. It's used to locate the configuration file and inherit as much resources as possible from the old cycle to keep nginx running smoothly. When nginx starts, a fake cycle called “init cycle” is created and is then replaced by a normal cycle, built from configuration.

Some members of the cycle:

* pool — cycle pool. Created for each new cycle
* log — cycle log. Initially, this log is inherited from the old cycle. After reading configuration, this member is set to point to new_log
* new_log — cycle log, created by the configuration. It's affected by the root scope error_log directive
* connections, connections_n — per-worker array of connections of type ngx_connection_t, created by the event module while initializing each nginx worker. The number of connections is set by the worker_connections directive
* free_connections, free_connections_n — the and number of currently available connections. If no connections are available, nginx worker refuses to accept new clients
* files, files_n — array for mapping file descriptors to nginx connections. This mapping is used by the event modules, having the NGX_USE_FD_EVENT flag (currently, it's poll and devpoll)
* conf_ctx — array of core module configurations. The configurations are created and filled while reading nginx configuration files
modules, modules_n — array of modules ngx_module_t, both static and dynamic, loaded by current configuration
* listening — array of listening objects ngx_listening_t. Listening objects are normally added by the the listen directive of different modules which call the ngx_create_listening() function. Based on listening objects, listen sockets are created by nginx
* paths — array of paths ngx_path_t. Paths are added by calling the function ngx_add_path() from modules which are going to operate on certain directories. These directories are created by nginx after reading configuration, if missing. Moreover, two handlers can be added for each path:
    * path loader — executed only once in 60 seconds after starting or reloading nginx. Normally, reads the directory and stores data in nginx shared memory. The handler is called from a dedicated nginx process “nginx cache loader”
    * path manager — executed periodically. Normally, removes old files from the directory and reflects these changes in nginx memory. The handler is called from a dedicated nginx process “nginx cache manager”
* open_files — list of ngx_open_file_t objects. An open file object is created by calling the function ngx_conf_open_file(). After reading configuration nginx opens all files from the open_files list and stores file descriptors in the fd field of each open file object. The files are opened in append mode and created if missing. The files from this list are reopened by nginx workers upon receiving the reopen signal (usually it's USR1). In this case the fd fields are changed to new descriptors. The open files are currently used for logging
* shared_memory — list of shared memory zones, each added by calling the ngx_shared_memory_add() function. Shared zones are mapped to the same address range in all nginx processes and are used to share common data, for example HTTP cache in-memory tree

Buffer
======

For input/output operations, nginx provides the buffer type ngx_buf_t. Normally, it's used to hold data to be written to a destination or read from a source. Buffer can reference data in memory and in file. Technically it's possible that a buffer references both at the same time. Memory for the buffer is allocated separately and is not related to the buffer structure ngx_buf_t.

The structure ngx_buf_t has the following fields:

* start, end — the boundaries of memory block, allocated for the buffer
* pos, last — memory buffer boundaries, normally a subrange of start .. end
* file_pos, file_last — file buffer boundaries, these are offsets from the beginning of the file
* tag — unique value, used to distinguish buffers, created by different nginx module, usually, for the purpose of buffer reuse
* file — file object
* temporary — flag, meaning that the buffer references writable memory
* memory — flag, meaning that the buffer references read-only memory
* in_file — flag, meaning that current buffer references data in a file
* flush — flag, meaning that all data prior to this buffer should be flushed
* recycled — flag, meaning that the buffer can be reused and should be consumed as soon as possible
* sync — flag, meaning that the buffer carries no data or special signal like flush or last_buf. Normally, such buffers are considered an error by nginx. This flags allows skipping the error checks
* last_buf — flag, meaning that current buffer is the last in output
* last_in_chain — flag, meaning that there's no more data buffers in a (sub)request
* shadow — reference to another buffer, related to the current buffer. Usually current buffer uses data from the shadow buffer. Once current buffer is consumed, the shadow buffer should normally also be marked as consumed
* last_shadow — flag, meaning that current buffer is the last buffer, referencing a particular shadow buffer
* temp_file — flag, meaning that the buffer is in a temporary file

For input and output buffers are linked in chains. Chain is a sequence of chain links ngx_chain_t, defined as follows:

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

Processes
=========

There are several types of processes in nginx. The type of current process is kept in the ngx_process global variable:

* NGX_PROCESS_MASTER — the master process runs the ngx_master_process_cycle() function. Master process does not have any I/O and responds only to signals. It reads configuration, creates cycles, starts and controls child processes

* NGX_PROCESS_WORKER — the worker process runs the ngx_worker_process_cycle() function. Worker processes are started by master and handle client connections. They also respond to signals and channel commands, sent from master

* NGX_PROCESS_SINGLE — single process is the only type of processes which exist in the master_process off mode. The cycle function for this process is ngx_single_process_cycle(). This process creates cycles and handles client connections

* NGX_PROCESS_HELPER — currently, there are two types of helper processes: cache manager and cache loader. Both of them share the same cycle function ngx_cache_manager_process_cycle().

All nginx processes handle the following signals:

* NGX_SHUTDOWN_SIGNAL (SIGQUIT) — graceful shutdown. Upon receiving this signal master process sends shutdown signal to all child processes. When no child processes are left, master destroys cycle pool and exits. A worker process which received this signal, closes all listening sockets and waits until timeout tree becomes empty, then destroys cycle pool and exits. A cache manager process exits right after receiving this signal. The variable ngx_quit is set to one after receiving this signal and immediately reset after being processed. The variable ngx_exiting is set to one when worker process is in shutdown state

* NGX_TERMINATE_SIGNAL (SIGTERM) - terminate. Upon receiving this signal master process sends terminate signal to all child processes. If child processes do not exit in 1 second, they are killed with the SIGKILL signal. When no child processes are left, master process destroys cycle pool and exits. A worker or cache manager process which received this signal destroys cycle pool and exits. The variable ngx_terminate is set to one after receiving this signal

* NGX_NOACCEPT_SIGNAL (SIGWINCH) - gracefully shut down worker processes

* NGX_RECONFIGURE_SIGNAL (SIGHUP) - reconfigure. Upon receiving this signal master process creates a new cycle from configuration file. If the new cycle was created successfully, the old cycle is deleted and new child processes are started. Meanwhile, the old processes receive the shutdown signal. In single-process mode, nginx creates a new cycle as well, but keeps the old one until all clients, tied to the old cycle, are gone. Worker and helper processes ignore this signal

* NGX_REOPEN_SIGNAL (SIGUSR1) — reopen files. Master process passes this signal to workers. Worker processes reopen all open_files from the cycle

* NGX_CHANGEBIN_SIGNAL (SIGUSR2) — change nginx binary. Master process starts a new nginx binary and passes there a list of all listen sockets. The list is passed in the environment variable “NGINX” in text format, where descriptor numbers separated with semicolons. A new nginx instance reads that variable and adds the sockets to its init cycle. Other processes ignore this signal

While all nginx worker processes are able to receive and properly handle POSIX signals, master process normally does not pass any signals to workers and helpers with the standard kill() syscall. Instead, nginx uses inter-process channels which allow sending messages between all nginx processes. Currently, however, messages are only sent from master to its children. Those messages carry the same signals. The channels are socketpairs with their ends in different processes.

When running nginx binary, several values can be specified next to -s parameter. Those values are stop, quit, reopen, reload. They are converted to signals NGX_TERMINATE_SIGNAL, NGX_SHUTDOWN_SIGNAL, NGX_REOPEN_SIGNAL and NGX_RECONFIGURE_SIGNAL and sent to the nginx master process, whose pid is read from nginx pid file.

Modules
=======

Adding new modules
------------------

The standalone nginx module resides in a separate directory that contains at least two files: config and a file with the module source. The first file contains all information needed for nginx to integrate the module, for example:

```
ngx_module_type=CORE
ngx_module_name=ngx_foo_module
ngx_module_srcs="$ngx_addon_dir/ngx_foo_module.c"

. auto/module

ngx_addon_name=$ngx_module_name
```

The file is a POSIX shell script and it can set (or access) the following variables:

* ngx_module_type — the type of module to build. Possible options are CORE, HTTP, HTTP_FILTER, HTTP_INIT_FILTER, HTTP_AUX_FILTER, MAIL, STREAM, or MISC
* ngx_module_name — the name of the module. A whitespace separated values list is accepted and may be used to build multiple modules from a single set of source files. The first name indicates the name of the output binary for a dynamic module. The names in this list should match the names used in the module.
* ngx_addon_name — supplies the name of the module in the console output text of the configure script.
* ngx_module_srcs — a whitespace separated list of source files used to compile the module. The $ngx_addon_dir variable can be used as a placeholder for the path of the module source.
* ngx_module_incs — include paths required to build the module
* ngx_module_deps — a list of module's header files.
* ngx_module_libs — a list of libraries to link with the module. For example, libpthread would be linked using ngx_module_libs=-lpthread. The following macros can be used to link against the same libraries as nginx: LIBXSLT, LIBGD, GEOIP, PCRE, OPENSSL, MD5, SHA1, ZLIB, and PERL
* ngx_module_link — set by the build system to DYNAMIC for a dynamic module or ADDON for a static module and used to perform different actions depending on linking type.
ngx_module_order — sets the load order for the module which is useful for HTTP_FILTER and HTTP_AUX_FILTER module types. The order is stored in a reverse list.

    The ngx_http_copy_filter_module is near the bottom of the list so is one of the first to be executed. This reads the data for other filters. Near the top of the list is ngx_http_write_filter_module which writes the data out and is one of the last to be executed.

    The format for this option is typically the current module’s name followed by a whitespace separated list of modules to insert before, and therefore execute after. The module will be inserted before the last module in the list that is found to be currently loaded.

    By default for filter modules this is set to “ngx_http_copy_filter” which will insert the module before the copy filter in the list and therefore will execute after the copy filter. For other module types the default is empty.

A module can be added to nginx by means of the configure script using --add-module=/path/to/module for static compilation and --add-dynamic-module=/path/to/module for dynamic compilation.

Modules are building blocks of nginx, and most of its functionality is implemented as modules. The module source file must contain a global variable of ngx_module_t type which is defined as follows:

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

The omitted private part includes module version, signature and is filled using the predefined macro NGX_MODULE_V1.

Each module keeps its private data in the ctx field, recognizes specific configuration directives, specified in the commands array, and may be invoked at certain stages of nginx lifecycle. The module lifecycle consists of the following events:

* Configuration directive handlers are called as they appear in configuration files in the context of the master process
* The init_module handler is called in the context of the master process after the configuration is parsed successfully
* The master process creates worker process(es) and init_process handler is called in each of them
* When a worker process receives the shutdown command from master, it invokes the exit_process handler
* The master process calls the exit_master handler before exiting.

init_module handler may be called multiple times in the master process if the configuration reload is requested.

The init_master, init_thread and exit_thread handlers are not implemented at the moment; Threads in nginx are only used as supplementary I/O facility with its own API and init_master handler looks unnecessary.

The module type defines what exactly is stored in the ctx field. There are several types of modules:

* NGX_CORE_MODULE
* NGX_EVENT_MODULE
* NGX_HTTP_MODULE
* NGX_MAIL_MODULE
* NGX_STREAM_MODULE

The NGX_CORE_MODULE is the most basic and thus the most generic and most low-level type of module. Other module types are implemented on top of it and provide more convenient way to deal with corresponding problem domains, like handling events or http requests.

The examples of core modules are ngx_core_module, ngx_errlog_module, ngx_regex_module, ngx_thread_pool_module, ngx_openssl_module modules and, of course, http, stream, mail and event modules itself. The context of a core module is defined as:

```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

where the name is a string with a module name for convenience, create_conf and init_conf are pointers to functions that create and initialize module configuration correspondingly. For core modules, nginx will call create_conf before parsing a new configuration and init_conf after all configuration was parsed successfully. The typical create_conf function allocates memory for the configuration and sets default values. The init_conf deals with known configuration and thus may perform sanity checks and complete initialization.

For example, the simplistic ngx_foo_module can look like this:

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


Core modules
------------

The ngx_command_t describes single configuration directive. Each module, supporting configuration, provides an array of such specifications that describe how to process arguments and what handlers to call:

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

The array should be terminated by a special value “ngx_null_command”. The name is the literal name of a directive, as it appears in configuration file, for example “worker_processes” or “listen”. The type is a bitfield that controls number of arguments, command type and other properties using corresponding flags. Arguments flags:

* NGX_CONF_NOARGS — directive without arguments
* NGX_CONF_1MORE — one required argument
* NGX_CONF_2MORE — two required arguments
* NGX_CONF_TAKE1..7 — exactly 1..7 arguments
* NGX_CONF_TAKE12, 13, 23, 123, 1234 — one or two arguments, or other combinations

Directive types:

* NGX_CONF_BLOCK — the directive is a block, i.e. it may contain other directives in curly braces, or even implement its own parser to handle contents inside.
* NGX_CONF_FLAG — the directive value is a flag, a boolean value represented by “on” or “off” strings.

Context of a directive defines where in the configuration it may appear and how to access module context to store corresponding values:

* NGX_MAIN_CONF — top level configuration
* NGX_HTTP_MAIN_CONF — in the http block
* NGX_HTTP_SRV_CONF — in the http server block
* NGX_HTTP_LOC_CONF — in the http location
* NGX_HTTP_UPS_CONF — in the http upstream block
* NGX_HTTP_SIF_CONF — in the http server “if”
* NGX_HTTP_LIF_CONF — in the http location “if”
* NGX_HTTP_LMT_CONF — in the http “limit_except”
* NGX_STREAM_MAIN_CONF — in the stream block
* NGX_STREAM_SRV_CONF — in the stream server block
* NGX_STREAM_UPS_CONF — in the stream upstream block
* NGX_MAIL_MAIN_CONF — in the the mail block
* NGX_MAIL_SRV_CONF — in the mail server block
* NGX_EVENT_CONF — in the event block
* NGX_DIRECT_CONF — used by modules that don't create a hierarchy of contexts and store module configuration directly in ctx

The configuration parser uses this flags to throw an error in case of a misplaced directive and calls directive handlers supplied with a proper configuration pointer, so that same directives in different locations could store their values in distinct places.

The set field defines a handler that processes a directive and stores parsed values into corresponding configuration. Nginx offers a convenient set of functions that perform common conversions:

* ngx_conf_set_flag_slot — converts literal “on” or “off” strings into ngx_flag_t type with values 1 or 0
* ngx_conf_set_str_slot — stores string as a value of the ngx_str_t type
* ngx_conf_set_str_array_slot — appends ngx_array_t of ngx_str_t with a new value. The array is created if not yet exists
* ngx_conf_set_keyval_slot — appends ngx_array_t of ngx_keyval_t with a new value, where key is the first string and value is second. The array is created if not yet exists
* ngx_conf_set_num_slot — converts directive argument to a ngx_int_t value
* ngx_conf_set_size_slot — converts size to size_t value in bytes
* ngx_conf_set_off_slot — converts offset to off_t value in bytes
* ngx_conf_set_msec_slot — converts time to ngx_msec_t value in milliseconds
* ngx_conf_set_sec_slot — converts time to time_t value in seconds
* ngx_conf_set_bufs_slot — converts two arguments into ngx_bufs_t that holds ngx_int_t number and size of buffers
* ngx_conf_set_enum_slot — converts argument into ngx_uint_t value. The null-terminated array of ngx_conf_enum_t passed in the post field defines acceptable strings and corresponding integer values
* ngx_conf_set_bitmask_slot — arguments are converted to ngx_uint_t value and OR'ed with the resulting value, forming a bitmask. The null-terminated array of ngx_conf_bitmask_t passed in the post field defines acceptable strings and corresponding mask values
* set_path_slot — converts arguments to ngx_path_t type and performs all required initializations. See the proxy_temp_path directive description for details
* set_access_slot — converts arguments to file permissions mask. See the proxy_store_access directive description for details

The conf field defines which context is used to store the value of the directive, or zero if contexts are not used. Only simple core modules use configuration without context and set NGX_DIRECT_CONF flag. In real life, such modules like http or stream require more sophisticated configuration that can be applied per-server or per-location, or even more precisely, in the context of the “if” directive or some limit. In this modules, configuration structure is more complex. Please refer to corresponding modules description to understand how they manage their configuration.

* NGX_HTTP_MAIN_CONF_OFFSET — http block configuration
* NGX_HTTP_SRV_CONF_OFFSET — http server configuration
* NGX_HTTP_LOC_CONF_OFFSET — http location configuration
* NGX_STREAM_MAIN_CONF_OFFSET — stream block configuration
* NGX_STREAM_SRV_CONF_OFFSET — stream server configuration
* NGX_MAIL_MAIN_CONF_OFFSET — mail block configuration
* NGX_MAIL_SRV_CONF_OFFSET — mail server configuration

The offset defines an offset of a field in a module configuration structure that holds values of this particular directive. The typical use is to employ offsetof() macro.

he post is a twofold field: it may be used to define a handler to be called after main handler completed or to pass additional data to the main handler. In the first case, ngx_conf_post_t structure needs to be initialized with a pointer to handler, for example:

```
static char *ngx_do_foo(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_post = { ngx_do_foo };
```

The post argument is the ngx_conf_post_t object itself, and the data is a pointer to value, converted from arguments by the main handler with the appropriate type.


Configuration directives
------------------------
TODO

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

Load balancing
--------------
The ngx_http_upstream_module provides basic functionality to pass requests to remote servers. This functionality is used by modules that implement specific protocols, such as HTTP or FastCGI. The module also provides an interface for creating custom load balancing modules and implements a default round-robin balancing method.

Examples of modules that implement alternative load balancing methods are least_conn and hash. Note that these modules are actually implemented as extensions of the upstream module and share a lot of code, such as representation of a server group. The keepalive module is an example of an independent module, extending upstream functionality.

The ngx_http_upstream_module may be configured explicitly by placing the corresponding upstream block into the configuration file, or implicitly by using directives that accept a URL evaluated at some point to the list of servers, for example, proxy_pass. Only explicit configurations may use an alternative load balancing method. The upstream module configuration has its own directive context NGX_HTTP_UPS_CONF. The structure is defined as follows:

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

* srv_conf — configuration context of upstream modules
* servers — array of ngx_http_upstream_server_t, the result of parsing a set of server directives in the upstream block
* flags — flags that mostly mark which features (configured as parameters of the server directive) are supported by the particular load balancing method.
   * NGX_HTTP_UPSTREAM_CREATE — used to distinguish explicitly defined upstreams from automatically created by proxy_pass and “friends” (FastCGI, SCGI, etc.)
   * NGX_HTTP_UPSTREAM_WEIGHT — “weight” is supported
   * NGX_HTTP_UPSTREAM_MAX_FAILS — “max_fails” is supported
   * NGX_HTTP_UPSTREAM_FAIL_TIMEOUT — “fail_timeout” is supported
   * NGX_HTTP_UPSTREAM_DOWN — “down” is supported
   * NGX_HTTP_UPSTREAM_BACKUP — “backup” is supported
   * NGX_HTTP_UPSTREAM_MAX_CONNS — “max_conns” is supported
* host — the name of an upstream
* file_name, line — the name of the configuration file and the line where the upstream block is located
* port and no_port — unused by explicit upstreams
* shm_zone — a shared memory zone used by this upstream, if any
* peer — an object that holds generic methods for initializing upstream configuration:

```
typedef struct {
    ngx_http_upstream_init_pt        init_upstream;
    ngx_http_upstream_init_peer_pt   init;
    void                            *data;
} ngx_http_upstream_peer_t;
```

A module that implements a load balancing algorithm must set these methods and initialize private data. If init_upstream was not initialized during configuration parsing, ngx_http_upstream_module sets it to default ngx_http_upstream_init_round_robin.
   * init_upstream(cf, us) — configuration-time method responsible for initializing a group of servers and initializing the init() method in case of success. A typical load balancing module uses a list of servers in the upstream block to create some efficient data structure that it uses and saves own configuration to the data field.
   * init(r, us) — initializes per-request ngx_http_upstream_peer_t.peer (not to be confused with the ngx_http_upstream_srv_conf_t.peer described above which is per-upstream) structure that is used for load balancing. It will be passed as data argument to all callbacks that deal with server selection.
   
When nginx has to pass a request to another host for processing, it uses a configured load balancing method to obtain an address to connect to. The method is taken from the ngx_http_upstream_peer_t.peer object of type ngx_peer_connection_t:

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

The structure has the following fields:

* sockaddr, socklen, name — address of an upstream server to connect to; this is the output parameter of a load balancing method
* data — per-request load balancing method data; keeps the state of selection algorithm and usually includes the link to upstream configuration. It will be passed as an argument to all methods that deal with server selection (see below)
* tries — allowed number of attempts to connect to an upstream.
* get, free, notify, set_session, and save_session - methods of the load balancing module, see description below

All methods accept at least two arguments: peer connection object pc and the data created by ngx_http_upstream_srv_conf_t.peer.init(). Note that in general case it may differ from pc.data due to “chaining” of load balancing modules.

* get(pc, data) — the method is called when the upstream module is ready to pass a request to an upstream server and needs to know its address. The method is responsible to fill in the sockaddr, socklen, and name fields of ngx_peer_connection_t structure. The return value may be one of:
   * NGX_OK — server was selected
   * NGX_ERROR — internal error occurred
   * NGX_BUSY — there are no available servers at the moment. This can happen due to many reasons, such as: dynamic server group is empty, all servers in the group are in the failed state, all servers in the group are already handling the maximum number of connections or similar.
   * NGX_DONE — this is set by the keepalive module to indicate that the underlying connection was reused and there is no need to create a new connection to the upstream server.
* free(pc, data, state) — the method is called when an upstream module has finished work with a particular server. The state argument is the status of upstream connection completion. This is a bitmask, the following values may be set: NGX_PEER_FAILED — this attempt is considered unsuccessful, NGX_PEER_NEXT — a special case with codes 403 and 404 (see link above), which are not considered a failure. NGX_PEER_KEEPALIVE. Also, tries counter is decremented by this method.
* notify(pc, data, type) — currently unused in the OSS version.
* set_session(pc, data) and save_session(pc, data) — SSL-specific methods that allow to cache sessions to upstream servers. The implementation is provided by the round-robin balancing method.