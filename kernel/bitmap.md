# bitmap定义
先做草稿，后续补充完善；
在include/linux/bitops.h头文件中有如下定义：
```
#define BITS_PER_TYPE(type)	(sizeof(type) * BITS_PER_BYTE)
#define BITS_TO_LONGS(nr)	__KERNEL_DIV_ROUND_UP(nr, BITS_PER_TYPE(long))
#define BITS_TO_U64(nr)		__KERNEL_DIV_ROUND_UP(nr, BITS_PER_TYPE(u64))
#define BITS_TO_U32(nr)		__KERNEL_DIV_ROUND_UP(nr, BITS_PER_TYPE(u32))
#define BITS_TO_BYTES(nr)	__KERNEL_DIV_ROUND_UP(nr, BITS_PER_TYPE(char)
```
可见这些宏都依赖两个宏：__KERNEL_DIV_ROUND_UP和BITS_PER_TYPE；
## BITS_PER_TYPE
- BITS_PER_BYTE在include/linux/bits.h中定义很简单，就是8，说人话就是一个字节8个bit；
```
#define BIT_MASK(nr)		(UL(1) << ((nr) % BITS_PER_LONG))
#define BIT_WORD(nr)		((nr) / BITS_PER_LONG)
#define BIT_ULL_MASK(nr)	(ULL(1) << ((nr) % BITS_PER_LONG_LONG))
#define BIT_ULL_WORD(nr)	((nr) / BITS_PER_LONG_LONG)
#define BITS_PER_BYTE		8
```
## __KERNEL_DIV_ROUND_UP
- 在include/uapi/linux/const.h文件中定义如下，
```
#define __KERNEL_DIV_ROUND_UP(n, d) (((n) + (d) - 1) / (d))
```
从这个宏定义的名字带ROUND_UP后缀也能看出来，是向上round了的。简单说就是要保证空间足够，宁愿多浪费一个d的空间，不能短缺，不然会造成踩内存；

## BITS_TO_LONGS
回到最开始的BITS_TO_LONGS，就可以看出来是保存这么多个bit，需要多少个long类型的空间了。
```
