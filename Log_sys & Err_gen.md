# 基于ulog系统的Log_sys
核心知识点：预处理操作符 #  ##  ;宏__VA_ARGS__ 
> 参考： 
>   https://club.rt-thread.org/ask/question/e61a0c2e6a55be1c.html
>   https://murphy.tech/posts/40137057.html    

## 预处理操作符 #  ## 
> 基本用法： https://murphy.tech/posts/40137057.html
> 
> 注意事项： https://blog.csdn.net/qq_16135205/article/details/86770372

范例：
```C
#define ENUM_TO_STR(enum_name) (  #enum_name)
#define ENUM_TO_STR_PLUS_6(enum_name) (  #  enum_name"66"#  enum_name)
//#define ENUM_TO_STR_PLUS(enum_name) (  #  enum_name## # enum_name)
/**
 * @brief
 * 1. ( #  enum_name##"66")不合法，gcc编译中，##后必须使用“被预定义”的符号，例如宏定义__VA_ARGS__;
 * 2. 可以直接(  #  enum_name"66"),字符串间自动相互拼接；
 * 3. #操作的是“宏参数”!
 */

typedef enum {
    ONE,
    TWO,
    THREE,
    MAX = 10
} number_enum_t;

const char *test_str = ENUM_TO_STR(ONE);
const char *test_strs[] = {
    ENUM_TO_STR(ONE),
    ENUM_TO_STR(TWO),
    ENUM_TO_STR(THREE),
    ENUM_TO_STR(MAX)
};
const char *test_str_plus6 = ENUM_TO_STR_PLUS_6(ONE);

```
输出：
```C
    LOG_I("str=%s\n", test_str);
    LOG_I("strs1=%s strs2=%s\n", *test_strs, *(test_strs + 1));
    LOG_I("strs_plus=%s\n", test_str_plus6);
```
结果：
[16836] I/NO_TAG: str=ONE 
[16836] I/NO_TAG: strs1=ONE strs2=TWO 
[16836] I/NO_TAG: strs_plus=ONE66ONE 

**本质是宏展开时 拼接token：**
> http://blog.chinaunix.net/uid-27666459-id-3772549.html
> https://blog.csdn.net/acs713/article/details/6891837
> https://www.cnblogs.com/qrxqrx/articles/6501153.html

```C
/**
 * @brief
 * ##本质是用于“预处理阶段”“宏展开”时拼接token；不像部分资料中所写是拼接string；
 * 例：生成变量(未必可以提升效率，但是减少犯错几率)；
 *
 */
#define AUTO_VAR_GRN(n)  var##n
#define AUTO_PRT(n)  rt_kprintf("var"#n" = %d\n", var##n);

    int AUTO_VAR_GRN(1) = 1;
    int AUTO_VAR_GRN(2) = 2;
    AUTO_PRT(1);
    AUTO_PRT(2);
    rt_kprintf("\n");
```
var1 = 1
var2 = 2


## 结合宏__VA_ARGS__ 使用
**不同编译下 可变参数宏的不同写法：**
> https://blog.csdn.net/lijian2017/article/details/104836447
```C
/**
 * @brief
 * 此处有：#define LOG_I(...)      ulog_i(LOG_TAG, __VA_ARGS__)  参数透传；
 * 1._MY_LOG的写法好理解，唯一问题就是纯字符串无参数的输出会有问题，出现printf("xxx",);此类结构
 * 2.（当前C99编译）即使嵌套宏，TEST_MMACRO,__func__,##__VA_ARGS__，这种写法依然会在展开后形成一个单独‘，’
 * 3.还是需要借助##消除无参下的‘，’
 */
#define TEST_MMACRO "[func_name]:"
#define _MY_LOG(BASE,...) rt_kprintf(BASE,__VA_ARGS__)
#define MY_LOG(BASE,...) _MY_LOG("[head]%s %s"BASE,TEST_MMACRO,__func__,##__VA_ARGS__)

#define MY_STR_LOG(BASE_MSG,...) LOG_I(BASE_MSG, ##__VA_ARGS__)

```
有参数&无参数:
```C
    MY_STR_LOG("[STLOG]:%s %s", ENUM_TO_STR(ONE), ENUM_TO_STR(TWO));
    MY_STR_LOG("[STLOG]:%d %d", 1, 2);
    MY_STR_LOG("[STLOG]: para empty\n");

    MY_LOG("[LOG]:%d %d\n", 1, 2);
    MY_LOG("[LOG]:para empty\n\n");

```
[head][func_name]: test_init[LOG]:1 2
[head][func_name]: test_init[LOG]:para empty

[16838] I/NO_TAG: [STLOG]:ONE TWO
[16838] I/NO_TAG: [STLOG]:1 2
[16838] I/NO_TAG: [STLOG]: para empty

## ULOG
> 参考 ulog.c


# 错误码规划 & 生成
基本定义，划分32bits
```C
    #define ERRCODE_OK  0
    /* 类型都为enum */
    err_level_t level = ERR_LEVEL_DBG;
    subsys_id_t subsys = SUBSYS_NULL;
    module_t module = MODULE_DEFAULT;
    errcode_t err = ERRCODE_OK;
```
基于划分生成ERRCODE：
```C
#define GEN_ERRCODE(level, subsys, module, err) (((((uint32_t)level)&0x07) << 28) | ((((uint32_t)subsys)&0x3F) << 22) | ((((uint32_t)module)&0x3F) << 16) | (((uint32_t)err)&0xFFFF))
```
