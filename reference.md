###深入理解PHP7之REFERENCE

#####版权申明:
````
本文是原创作品，包括文字、资料、图片、网页格式，转载时请标注作者与来源。非经允许，不得用于赢利目的。
````

####REFERENCE
   上一站章说过引用(REFERENCE)在PHP5的时候是一个标志位, 而在PHP7以后我们把它变成了一种新的类型:`IS_REFERNCE`, 然而引用在PHP中还是比较常用的, 所以在一些事情的处理上, 这个变化会让内核开发者, 扩展开发者产生不少的疑惑.

   最简单的情况, 就是在处理各种类型的时候, 从此以后我们要多考虑这种新的类型, 比如在PHP7中, 这样的代码形式就变得很常见了:
````c

try_again:
swtich (Z_TYPE_P(zv)) {
	case IS_TRING:
	break;
	case IS_ARRAY:
	break;
    ...
	case IS_REFERNCE:
	zv = Z_REFVAL_P(zv); //解引用
	goto try_again;
	break;	
}
````
   
    除了这些, 还有不少变化, 要了解这些我们需要认真完全的理解下这个新的引用类型.

![IS_REFERNCE](/img/reference.png)
	
	如图所示, 引用是一种需要引用计数的类型:`zend_reference`, 所以`zval.value.ref`是一个指向`zend_reference`的指针, 它包含了引用计数, 和一个zval, 所以具体的zval的值是存在`zval.value.ref->val`中的.

####Change On Write
	PHP采用引用计数来做简单的垃圾回收, 考虑如下的代码:
````php
<?php
$val = "laruence";
$ref = &$val;
$copy = $val;
?>
````
	$ref和$val是指向同一个zval的引用, 在PHP5的时候, 我们是通过一个引用计数为2, 并且引用标志位为1来表示这种情况, 当把$val复制给$copy的时候, 我们发现$val是一个计数大于1的引用, 所以要产生Change on write, 也就是分离. 所以我们需要复制这个zval. 

	而在PHP7中, 情况就变得简单了很多, 首先在引用赋值给$ref的时候, 生成一个`IS_REFERNCE`类型, 然后因为此时有俩个变量引用它所以`zend_reference`这个结构的引用计数`zval.value.ref->gc.refcount`为2.

    再随后的赋值给$copy的时候, 发现$val是一个引用, 于是让$copy指向的是`zval.value.ref->val`, 也就是字符串值为`laruence`的zval, 然后把的的引用计数+1, 也就是`zval.value.ref->val.value.str.gc.refcount`为2. 并没有产生复制.

	从而这就很好的解决了上一章所说的PHP5的那个经典的问题, 比如我们在PHP7下运行上一章的那个问题, 我们得到的结果是:
````
$ php-7.0/sapi/cli/php /tmp/1.php
Used 0.00021380008539
Used 0.00020173048281
````
 
    可见确实没有发生复制, 从而不会产生任何的性能问题.

待续....
