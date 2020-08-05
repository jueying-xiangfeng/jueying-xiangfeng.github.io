---
title: block原理然就-3
date: 2019-01-14 19:49:18
tags:
---

测试

### block 底层数据结构

测试代码:

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) = ^{
            NSLog(@"block1234");
        };
        block();
    }
    return 0;
}
```

<!-- more -->
