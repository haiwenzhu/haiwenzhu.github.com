---
layout: post
title: "关于PHP字符串比较的一次探究"
categories:
  - PHP
---

昨天无意间看到有人抱怨PHP的`==`的判断，下面的这行代码的输出结果回事TRUE :
	
	php -r "var_dump('10' == '1E1');"

觉得挺奇怪的，就想看看PHP到底是怎么执行`==`操作的，通过vld输出opcode

	php -dvld.active=1 -r "var_dump('10' == '1E2');"

可以看到`==`对应的opcode是IS_EQUAL，于是找到IS_EQUAL对应的VM handler，所有的opcode的VM handler定义都在PHP源文件下的Zend/zend_vm_def.h中

	ZVAL_BOOL(result, fast_equal_function(result,
             GET_OP1_ZVAL_PTR(BP_VAR_R),
             GET_OP2_ZVAL_PTR(BP_VAR_R) TSRMLS_CC));

可以看到比较是通过调用`fast_equal_function`这个函数来完成的，`fast_equal_function`又调用了`compare_function`，一直跟下去，发现最终调用了`zend_strtod`这个函数将字符串转成数字再做比较，`zend_strtod`的定义如下

	ZEND_API double zend_strtod (CONST char *s00, CONST char **se)

不过很囧的是，这个函数太长了，看了半天都没看懂，不过`"1E1"`对应的最终调用是`zend_strtod("1E1", "E1")`，于是写了一个扩展看这个函数调用到底输出多少

	PHP_FUNCTION(sample_strtod)
	{
    	char s[4];
    	char se[3];
    	sprintf(s, "1E1");
    	sprintf(se, "E1");
    	double ret;
    	ret = zend_strtod(s, se);
    	RETURN_DOUBLE(ret);
	}

函数的最终返回是10。

开始看了半天都搞不懂`1E1`为什么会被转成数字10，后面突然发现mia的这不就是所谓的科学计数法嘛，1E1 = 1*10^1...
