### 深入理解PHP7之REFERENCE

##### 版权申明:
````
本文是原创作品，包括文字、资料、图片、网页格式，转载时请标注作者与来源。非经允许，不得用于赢利目的。
````

#### REFERENCE
 上一章说过引用(REFERENCE)在PHP5的时候是一个标志位, 而在PHP7以后我们把它变成了一种新的类型:`IS_REFERNCE`. 然而引用是一种很常见的应用, 所以这个变化带来了很多的变化, 也给我们在做PHP7开发的时候, 因为有的时候疏忽忘了处理这个类型, 而带来不少的bug.

 最简单的情况, 就是在处理各种类型的时候, 从此以后我们要多考虑这种新的类型, 比如在PHP7中, 这样的代码形式就变得很常见了:
````c
try_again:
swtich (Z_TYPE_P(zv)) {
	case IS_TRING:
	break;
	case IS_ARRAY:
	break;
    ...
	case IS_REFERENCE:
	zv = Z_REFVAL_P(zv); //解引用
	goto try_again;
	break;
}
````

 如果大家自己写的扩展, 如果忘了考虑这种新的类型, 那么就会导致问题.

#### 为什么?
 那么既然这种新类型会带来这么多问题, 那么当时为什么要用把引用变成一种类型呢? 为什么不还是使用一个标志位呢?

 一句话来说, 就是我们不得不这么做. -_#

 前面说到, Hashtable直接存储的是zval, 这样在符号表中, 俩个zval如何共用一个数值呢? 对于字符串等复杂类型来说还好, 我们貌似可以在`zend_refcounted`结构中加入一个标志位来表明是引用来解决, 然而这个也会遇到Change On Write带来的复制, 但是我们知道在PHP7中, 一些类型是直接存储在zval中的, 比如`IS_LONG`, 但是引用类型是需要引用计数的, 那么对于一个是`IS_LONG`并且又是`IS_REFERNCE`的zval该如何表示呢?

 为此, 我们创造了这个新的类型:

 ![IS_REFERNCE](/img/reference.png)

 如图所示, 引用是一种新的类型:`zend_reference`, 对于`IS_REFERNCE`类型的zval, `zval.value.ref`是一个指向`zend_reference`的指针, 它包含了引用计数和一个zval, 具体的zval的值是存在`zval.value.ref->val`中的.

 所以对于`IS_LONG`的引用来说, 就用一个类型是`IS_REFERNCE`的zval, 它指向一个`zend_reference`, 而这个`zend_reference->val`中是一个类型为`IS_LONG`的zval.

#### Change On Write
 PHP采用引用计数来做简单的垃圾回收, 考虑如下的代码:
````php
<?php
1. $val = "laruence";
2. $ref = &$val;
3. $copy = $val;
?>
````
 `$ref`和`$val`是指向同一个zval的引用, 在PHP5的时候, 我们是通过一个引用计数为2, 并且引用标志位为1来表示这种情况, 当把`$val`复制给`$copy`(line 3)的时候, 我们发现$val是一个计数大于1的引用, 所以要产生Change on write, 也就是分离. 所以我们需要复制这个zval.

 而在PHP7中, 情况就变得简单了很多, 首先在引用赋值给`$ref`(line 2)的时候, 生成一个`IS_REFERNCE`类型, 然后因为此时有俩个变量引用它所以`zend_reference`这个结构的引用计数`zval.value.ref->gc.refcount`为2.

 再随后的赋值给`$copy`(line 3)的时候, 发现`$val`是一个引用, 于是让`$copy`指向的是`zval.value.ref->val`, 也就是字符串值为`laruence`的zval, 然后把zval的引用计数+1, 也就是`zval.value.ref->val.value.str.gc.refcount`为2. 并没有产生复制.

 从而这就很好的解决了上一章所说的PHP5的那个经典的问题, 比如我们在PHP7下运行上一章的那个问题, 我们得到的结果是:
````
$ php-7.0/sapi/cli/php /tmp/1.php
Used 0.00021380008539
Used 0.00020173048281
````

 可见确实没有发生复制, 从而不会产生任何的性能问题.

####

待续....
