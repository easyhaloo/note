```
RESIZE_STAMP_BITS = 16
```

```
Integer.numberOfLeadingZeros(n)  // 计算最高位非0位，左边0多个数
```

由于n 必须位2^n 的特性，Integer.numberOfLeadingZeros(n) <=27 < 2^5

2^14     + 2 ^5    =>  2^30 + 2 ^ 21

Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1)) << (32 - RESIZE_STAMP_BITS)

