###深入理解PHP7之zval

PHP7已经发布, 如承诺, 我也要开始这个系列的文章的编写, 今天我想先和大家聊聊zval的变化. 在讲zval变化的之前我们先来看看zval在PHP5下面是什么样子

####PHP5
#####zval回顾
在PHP5的时候, zval的定义如下:
````c
struct _zval_struct {
	union {
		long lval;
		double dval;
		struct {
			char *val;
			int len;
		} str;
		HashTable *ht;
		zend_object_value obj;
		zend_ast *ast;
	} value;
	zend_uint refcount__gc;
	zend_uchar type;
	zend_uchar is_ref__gc;
};

````

对PHP5内核有了解的同学应该对这个结构比较熟悉, 因为zval可以表示一切PHP中的数据类型, 所以它包含了一个type字段, 表示这个zval存储的是什么类型的值, 常见的可能选项是IS_NULL, IS_LONG, IS_STRING, IS_ARRAY, IS_OBJECT等等.

根据type字段的值不同, 我们就要用不同的方式解读value的值, 这个value是个联合体, 比如对于type是IS_STRING, 那么我们应该用value.str来解读zval.value字段, 而如果type是IS_LONG, 那么我们就要用value.lval来解读.

另外, 我们知道PHP是用引用计数来做基本的垃圾回收的, 所以zval中有一个refcount__gc字段, 表示这个zval的引用数目, 但这里有一个要说明的, 在5.3以前, 这个字段的名字还叫做refcount, 5.3以后, 在引入新的垃圾回收算法来对付循环引用计数的时候, 作者加入了大量的宏来操作refcount, 为了能让错误更快的显现, 所以改名为refcount__gc, 迫使大家都使用宏来操作refcount.

类似的, 还有is_ref, 这个值表示了PHP中的一个类型是否是引用, 这里我们可以看到是不是引用是一个标志位.

这就是PHP5时代的zval, 在2013年我们做PHP5的opcache JIT的时候, 因为JIT在实际项目中表现不佳, 我们转而意识到这个结构体的很多问题. 而PHPNG项目就是从改写这个结构体而开始的.

#####存在的问题

PHP5的zval定义是随着Zend Engine 2诞生的, 随着时间的推移, 当时设计的局限性也越来越明显:

首先这个结构体的大小是(在64位系统)24个字节, 我们仔细看这个zval.value联合体, 其中zend_object_value是最大的长板, 它导致整个value需要16个字节, 这个应该是很容易可以优化掉的, 比如把它挪出来, 用个指针代替,因为毕竟IS_OBJECT也不是最最常用的类型.

第二, 这个结构体的每一个字段都有明确的含义定义, 没有预留任何的自定义字段, 导致在PHP5时代做很多的优化的时候, 需要存储一些和zval相关的信息的时候, 不得不采用其他结构体映射, 或者外部包装后打补丁的方式来扩充zval, 比如5.3的时候新引入的GC, 它不得采用如下的比较hack的做法:
````c
/* The following macroses override macroses from zend_alloc.h */
#undef  ALLOC_ZVAL
#define ALLOC_ZVAL(z)                                   \
    do {                                                \
        (z) = (zval*)emalloc(sizeof(zval_gc_info));     \
        GC_ZVAL_INIT(z);                                \
    } while (0)
````

它用zval_gc_info劫持了zval的分配:
````c
typedef struct _zval_gc_info {
    zval z;
    union {
        gc_root_buffer       *buffered;
        struct _zval_gc_info *next;
    } u;
} zval_gc_info;
````

然后用zval_gc_info来扩充了zval, 所以实际上来说我们在PHP5时代申请一个zval其实真正的是分配了32个字节, 但其实GC只需要关心IS_ARRAY和IS_OBJECT类型, 这样就导致了大量的内存浪费.

还比如我之前做的Taint扩展, 我需要对于给一些字符串存储一些标记, zval里没有任何地方可以使用, 所以我不得不采用非常手段: 
````c
Z_STRVAL_PP(ppzval) = erealloc(Z_STRVAL_PP(ppzval), Z_STRLEN_PP(ppzval) + 1 + PHP_TAINT_MAGIC_LENGTH);
PHP_TAINT_MARK(*ppzval, PHP_TAINT_MAGIC_POSSIBLE);
````
就是把字符串的长度扩充一个int, 然后用magic number做标记写到后面去, 这样的做法安全性和稳定性在技术上都是没有保障的

还比如, PHP中大量的结构体都是基于Hashtable实现的, 增删改查Hashtable的操作占据了大量的CPU时间, 而字符串要查找首先要求它的Hash值, 理论上我们完全可以把一个字符串的Hash值计算好以后, 就存下来, 避免再次计算等等

第三, PHP的zval大部分都是按值传递, 写时拷贝的值, 但是有俩个例外, 就是对象和资源, 他们永远都是按引用传递, 这样就造成一个问题, 对象和资源在除了zval中的引用计数以外, 还需要一个全局的引用计数, 这样才能保证内存可以回收. 所以在PHP5的时代, 以对象为例, 它有俩套引用计数, 一个是zval中的, 另外一个是obj自身的计数:
````c
typedef struct _zend_object_store_bucket {
    zend_bool destructor_called;
    zend_bool valid;
    union _store_bucket {
        struct _store_object {
            void *object;
            zend_objects_store_dtor_t dtor;
            zend_objects_free_object_storage_t free_storage;
            zend_objects_store_clone_t clone;
            const zend_object_handlers *handlers;
            zend_uint refcount;
            gc_root_buffer *buffered;
        } obj;
        struct {
            int next;
        } free_list;
    } bucket;
} zend_object_store_bucket;
````

除了上面提到的两套引用以外, 如果我们要获取一个object, 则我们需要通过如下方式:

EG(objects_store).object_buckets[Z_OBJ_HANDLE_P(z)].bucket.obj

经过漫长的多次内存读取, 才能获取到真正的objec对象本身. 效率可想而知.

这一切都是因为Zend引擎最初设计的时候, 并没有考虑到后来的对象. 一个良好的设计, 一旦有了意外, 就会导致整个结构变得复杂, 维护性降低, 这是一个很好的例子.

第四, 我们知道PHP中, 大量的计算都是面向字符串的, 然而因为引用计数是作用在zval的, 那么就会导致如果要拷贝一个字符串类型的zval, 我们别无他法只能复制这个字符串. 当我们把一个zval的字符串作为key添加到一个数组里的时候, 我们别无他法只能复制这个字符串. 虽然在PHP5.4的时候, 我们引入了INTERNED STRING, 但是还是不能根本解决这个问题.

第五, 这个是关于引用的, PHP5的时代, 我们采用写时分离, 但是结合到引用这里就有了一个比较让人郁闷的例子:

````php
<?php

    $array = range(1, 100000);

    $b = &$array;

    function array_count($array) {
        return count($array);
    }

    array_count($b);
?>
````

当我们调用array_count的时候, 本来只是简单的一个传值就行的地方, 但是因为$b 是一个引用, 就必须发生分离, 导致数组复制, 从而极大的拖慢性能, 这里有一个简单的测试:

````php
<?php
$array = range(1, 100000);

function array_count($array) {
    return count($array);
}

$i = 0;
$start = microtime(true);
while($i++ < 100) {
    array_count($array);
}

printf("Used %sS\n", microtime(true) - $start);

$b = &$array; //注意这里, 假设我不小心把这个Array引用给了一个变量
$i = 0;
$start = microtime(true);
while($i++ < 100) {
    array_count($b);
}
printf("Used %sS\n", microtime(true) - $start);
?>
````

我们在5.6下运行这个例子, 得到如下结果:
````
$ php-5.6/sapi/cli/php /tmp/1.php
Used 0.00045204162597656S
Used 4.2051479816437S
````
相差1万倍之多. 这就造成, 如果在一大段代码中, 我不小心把一个变量变成了引用(比如foreach as &$v), 那么就有可能触发到这个问题, 造成严重的性能问题, 然而却又很难排查.

第六, 也是最重要的一个, 为什么说它重要呢? 因为这点促成了很大的性能提升, 我们习惯了在PHP5的时代调用MAKE_STD_ZVAL在堆内存上分配一个zval, 然后对他进行操作, 最后呢通过RETURN_ZVAL把这个zval的值"copy"给return_value, 然后又销毁了这个zval, 比如pathinfo这个函数:
````c
PHP_FUNCTION(pathinfo)
{
.....
	MAKE_STD_ZVAL(tmp);
	array_init(tmp);
....

    if (opt == PHP_PATHINFO_ALL) {
        RETURN_ZVAL(tmp, 0, 1);
    } else {
.....
}
````

这个tmp变量, 完全是一个临时变量的作用, 我们又何必在堆内存分配它呢? MAKE_STD_ZVAL/ALLOC_ZVAL在PHP5的时候, 到处都有, 是一个非常常见的用法, 如果我们能把这个变量用栈分配, 那无论是内存分配, 还是缓存友好, 都是非常有利的

还有很多, 我就不一一详细列举了, 但是我相信你们也有了和我们当时一样的想法, zval必须得改改了, 对吧? 

####PHP7

到了PHP7中, zval变成了如下的结构, 要说明的是, 这个是现在的结构, 已经和PHPNG时候有了一些不同了, 因为我们新增加了一些解释 (联合体的字段), 但是总体大小, 结构, 是和PHPNG的时候一致的:
````c
struct _zval_struct {
	union {
		zend_long         lval;             /* long value */
		double            dval;             /* double value */
		zend_refcounted  *counted;
		zend_string      *str;
		zend_array       *arr;
		zend_object      *obj;
		zend_resource    *res;
		zend_reference   *ref;
		zend_ast_ref     *ast;
		zval             *zv;
		void             *ptr;
		zend_class_entry *ce;
		zend_function    *func;
		struct {
			uint32_t w1;
			uint32_t w2;
		} ww;
	} value;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    type,         /* active type */
                zend_uchar    type_flags,
                zend_uchar    const_flags,
                zend_uchar    reserved)     /* call info for EX(This) */
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t     var_flags;
        uint32_t     next;                 /* hash collision chain */
        uint32_t     cache_slot;           /* literal cache slot */
        uint32_t     lineno;               /* line number (for ast nodes) */
        uint32_t     num_args;             /* arguments number for EX(This) */
        uint32_t     fe_pos;               /* foreach position */
        uint32_t     fe_iter_idx;          /* foreach iterator index */
    } u2;
};
````

虽然看起来变得好大, 但其实你仔细看, 全部都是联合体, 这个新的zval在64位环境下,现在只需要16个字节(2个指针size), 它主要分为俩个部分, value和扩充字段, 而扩充字段又分为u1和u2俩个部分, 其中u1是type info,  u2是各种辅助字段.

(开会去了, 待续吧)
