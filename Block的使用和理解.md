Block in Objective-C。

1. Block本质上是一个代码块(匿名函数，闭包),可以使用其上问中定义的变量，block内的代码块可以在任意时间被调用。
	```
	@implementation ViewController

	typedef int(^IntBlock)(int);
	
	- (void)viewDidLoad {
	    
	    IntBlock block = [self makeCounter:10];
	    int a = block(20);
	    a = block(20);
	    a = block(30);
	    
	}
	
	- (IntBlock)makeCounter:(int)base {
	    __block int result = base;
	    return ^(int a) {
	        int ret = result;
	        result += a;
	        return ret;
	    };
	}
	
	@end
	``` 	
2. 苹果设计block的一个目的是使GCD更容易处理，现在很多SDK中已经使用，比如数组enumerate，UIView动画，网络相关错误/完成处理，通知等
3. 通过`__weak`来避免循环引用问题
4. 声明和定义。

	```
	typedef void(^MyBlock)(); // returnType(^blockName)(params)
	
	@implementation ViewController
	
	- (void)viewDidLoad {
	    
	    ^{ NSLog(@""); }();
	    
	    void(^blk)() = ^ {
	        NSLog(@"");
	    };
	    blk();
	    
	    [self doTask:^void() { // void返回值可以省略,无参数时()可以省略
	        NSLog(@"");
	    }];
	}
	
	- (void)doTask:(void(^)())task {
	    
	}

	@end	
	```
	
5. 默认情况下，block对外部的属性是readonly的，在定义block的时候会拷贝一份，之后你无法在block内修改该属性，或者该属性变更后，block内的还是定义时的状态(引用类型除外)。可以通过`__block`来解决问题。
	
	```
	// __block用法1
	double value = 2;
	double (^times)(double) = ^(double num) {
		return num * value;
	}
	doubleValue(10); // 20
	
	value = 3;
	doubleValue(10); // 还是20，因为value没有用__block声明，所有block内的value还是2


	// __block用法2
	__block BOOL result = NO;
	[self.values  enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        result = YES;
        *stop = YES;
    }];	
	```
	---

下面看一下block是如何设计的

### part1

有这样一段代码：
```
#include <stdio.h>

int main() {
  void(^blk)(void) = ^{ 
	printf("hello world!");
  };
  blk();
  return 0;
}
```
通过`clang -rewrite-objc Main.c`可以将前端OC编译成C++的实现代码,主要部分如下：
```
struct __block_impl {
  void *isa; // NSObject对象，_NSConcreteStackBlock、_NSConcreteGlobalBlock、_NSConcreteMallocBlock
  int Flags;
  int Reserved;
  void *FuncPtr; // 函数指针
};

struct __main_block_impl_0 { // 当前函数体main和序号0来构成结构提
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) { // 构造函数
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// 函数体
static void __main_block_func_0(struct __main_block_impl_0 *__cself) { // 参数是__main_block_impl_0对象，该例子没什么作用
 	printf("hello world!");
 }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main() {
  void(*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA)); // 创建__main_block_impl_0对象并赋值给blk
  ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk); // 通过类型转化并调用函数。 这样应该是效率更高
  return 0;
}
```

大概顺序就是：main -> 创建__main_block_impl_0对象 -> 通过 FuncPtr执行函数

### part2
这次，我们传递一个参数到block内部：
```
#include <stdio.h>

int main() {
	int num = 10;
  void(^blk)(void) = ^{ 
	printf("%d",num);
  };
  blk();
  return 0;
}
```

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int num; // 增加的参数
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _num, int flags=0) : num(_num) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
// 获取num
  int num = __cself->num; // bound by copy  

  printf("%d",num);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main() {
 int num = 10;
  void(*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, num));
  ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
  return 0;
}
```
这次生成的代码稍微复杂一点，主要是多了个参数，这也说明了只是值拷贝而已，所以验证了之前的说法，在block之后修改值在block内并无法体现，以及block中无法修改num。如果试图修改num，编译时会报错：
```
#include <stdio.h>

int main() {
	int num = 10;
  void(^blk)(void) = ^{ 
	num = 0;
	printf("%d",num);
  };
  blk();
  return 0;
}
```

```
/var/folders/xz/md8w90c92s769k5ks9vvgs5c0000gn/T/Main-60bf42.i:400:6: error: variable is not assignable (missing __block type specifier)
 num = 0;
 ~~~ ^
1 error generated.
```
下面通过`__block`来实现传递指针达到读写的可能。

### part3
```
#include <stdio.h>

int main() {
	__block int num = 10;
  void(^blk)(void) = ^{ 
	num = 0;
	printf("%d",num);
  };
  blk();
  return 0;
}
```

```
struct __Block_byref_num_0 { // 引用
  void *__isa;
__Block_byref_num_0 *__forwarding;
 int __flags;
 int __size;
 int num;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_num_0 *num; // by ref 传递指针
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_num_0 *_num, int flags=0) : num(_num->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_num_0 *num = __cself->num; // bound by ref

 (num->__forwarding->num) = 0;
 printf("%d",(num->__forwarding->num));
  }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->num, (void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->num, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main() {
 __attribute__((__blocks__(byref))) __Block_byref_num_0 num = {(void*)0,(__Block_byref_num_0 *)&num, 0, sizeof(__Block_byref_num_0), 10};
  void(*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_num_0 *)&num, 570425344));
  ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
  return 0;
}
```

新增了类型__Block_byref_num_0，用来封装参数，以及desc多了两个成员变量copy和dispose，用来在栈到堆的拷贝和释放时dispose

---

参考：
[onevcat blog](https://onevcat.com/2011/11/objc-block/)
[iOS中block实现的探究](http://blog.csdn.net/jasonblog/article/details/7756763?reload)
[关于Block和C语言闭包](https://www.evernote.com/shard/s269/sh/23b61393-c6dd-4fa2-b7ae-306e9e7c9639/131de66a3257122ba903b0799d36c04c?noteKey=131de66a3257122ba903b0799d36c04c&noteGuid=23b61393-c6dd-4fa2-b7ae-306e9e7c9639)
[谈谈Objective-C block的实现](http://blog.devtang.com/2013/07/28/a-look-inside-blocks/)

