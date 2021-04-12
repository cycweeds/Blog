# **Block __block原理**

参数未添加以及添加 *__block*，前后之前有什么区别？
## **编译前**
先来看看两个方法

```
@implementation BlockTest

- (void)test {
    int number = 0;
    void (^block)(void) = ^{
        NSLog(@"block %d", number); // 0
    };
    number ++;
    NSLog(@"%d", number); // 1
    block();
}

- (void)test2 {
    __block int number = 0;
    void (^block)(void) = ^{
        NSLog(@"block %d", number); // 1
    };
    number ++;
    NSLog(@"%d", number); // 1
    block();
}

@end
```
加了*__block* 的参数在 *block* 内部打印是自增之后，未添加则是输出一开始初始化的值。  

这是为什么呢？
## **编译后**
让我们来看看编译之后的关键信息（其他信息请参考 [Block 本质](https://github.com/cycweeds/blog/blob/main/2021-4-11%20Block%E6%9C%AC%E8%B4%A8.md)）：

test 的实现方法：
```
// test方法
static void _I_BlockTest_test(BlockTest * self, SEL _cmd) {
    int number = 0;
    // 可以看到这边是属于值传递
    void (*block)(void) = ((void (*)())&__BlockTest__test_block_impl_0((void *)__BlockTest__test_block_func_0, &__BlockTest__test_block_desc_0_DATA, number));
    number ++;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_l1_l7y9kxnx6mx5fy6nsslkh5gr0000gn_T_BlockTest_1e32d8_mi_1, number);
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}

// block结构体 （没用添加 __block）
struct __BlockTest__test_block_impl_0 {
  struct __block_impl impl;
  struct __BlockTest__test_block_desc_0* Desc;
  int number;
  __BlockTest__test_block_impl_0(void *fp, struct __BlockTest__test_block_desc_0 *desc, int _number, int flags=0) : number(_number) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __BlockTest__test_block_func_0(struct __BlockTest__test_block_impl_0 *__cself) {
  int number = __cself->number; // bound by copy

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_l1_l7y9kxnx6mx5fy6nsslkh5gr0000gn_T_BlockTest_a33ede_mi_0, number);
        number ;
    }
```

test2 的实现方法：
```
// test2 实现方法
static void _I_BlockTest_test2(BlockTest * self, SEL _cmd) {
  // number 编译后会生成一个结构体 __Block_byref_number_0
    __attribute__((__blocks__(byref))) __Block_byref_number_0 number = {(void*)0,(__Block_byref_number_0 *)&number, 0, sizeof(__Block_byref_number_0), 0};
    // 地址传递
    void (*block)(void) = ((void (*)())&__BlockTest__test2_block_impl_0((void *)__BlockTest__test2_block_func_0, &__BlockTest__test2_block_desc_0_DATA, (__Block_byref_number_0 *)&number, 570425344));
    // 原来的number++ 会通过__forwarding 去改变值
    (number.__forwarding->number) ++;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_l1_l7y9kxnx6mx5fy6nsslkh5gr0000gn_T_BlockTest_a7abec_mi_3, (number.__forwarding->number));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}

// block结构体 （添加 __block的）
struct __BlockTest__test2_block_impl_0 {
  struct __block_impl impl;
  struct __BlockTest__test2_block_desc_0* Desc;
  __Block_byref_number_0 *number; // by ref
  __BlockTest__test2_block_impl_0(void *fp, struct __BlockTest__test2_block_desc_0 *desc, __Block_byref_number_0 *_number, int flags=0) : number(_number->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// block里面的实现方式
static void __BlockTest__test2_block_func_0(struct __BlockTest__test2_block_impl_0 *__cself) {
  __Block_byref_number_0 *number = __cself->number; // bound by ref

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_l1_l7y9kxnx6mx5fy6nsslkh5gr0000gn_T_BlockTest_a33ede_mi_2, (number->__forwarding->number));
    }

// 我们发现number 在编译期间会生成一个新的结构体
struct __Block_byref_number_0 {
  void *__isa;
__Block_byref_number_0 *__forwarding;  // 指向本身，修改参数是通过修改  __forwarding.number 的形式去修改
 int __flags;
 int __size;
 int number; // 和参数名一致
};
```


我们可以发现用 **__block** 修饰，在编译期间会生成一个 **__Block_byref_number_0** 的一个结构体。

>  __Block_byref_number_0 的命名规范为  "__Block_byref_" + name + "_" + 当前是第几个外部参数

```
int number = 0;

会被编译成一下代码
__attribute__((__blocks__(byref))) __Block_byref_number_0 number = {(void*)0,(__Block_byref_number_0 *)&number, 0, sizeof(__Block_byref_number_0), 0};
```




对比没有用 *__block* ，我们会发现一个主要用值传递，另个一个用地址传递。所以使用了 *__block* 的参数在 *block* 内部改变值，外部也会改变。

> 在这里需要注意的是，如果外部变量没有使用 *__block* 修饰，*block* 内部实现也不能赋值。不然会编译不通过，如图：
![](https://github.com/cycweeds/Assets/raw/master/WeChat35237db5dbdbb1a81da2e1b564a7c4e6.png)
