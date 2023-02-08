### Redis Module

#### 加载module

`redis.conf`配置文件中加载module:

```c
loadmodule /path/to/mymodule.so
```

在一个running的实例中加载module:

```c
MODULE LOAD /path/to/mymodule.so
```

列出所有已加载的module:

```c
MODULE LIST
```

删除某个已加载module:

```c
MODULE UNLOAD mymodule
```

module用动态库加载是一个好习惯。

#### 一个最简单的module

```c
#include "redismodule.h"
#include <stdlib.h>

int HelloworldRand_RedisCommand(RedisModuleCtx *ctx,RedisModuleString **argv,int argc){
    RedisModule_ReplyWithLongLong(ctx,rand());
    return REDISMODULE_OK;
}

int RedisModule_OnLoad(RedisModuleCtx *ctx,RedisModuleString **argv,int argc){
    if(RedisModule_Init(ctx,"helloworld",1,REDISMODULE_APIVER_1) == REDISMODULE_ERR)
    return REDISMODULE_ERR;

    if(RedisModule_CreateCommand(ctx,"helloworld.rand",HelloworldRand_RedisCommand,"fast randown",0,0,0)== REDISMODULE_ERR)
        return REDISMODULE_ERR;
    
    return REDISMODULE_OK;
}
```

该示例中，有两个函数。一个实现了`helloworld.rand`命令。`RedisModule_OnLoad()`在redis的每个module中都是必须存在的，该函数是module的初始化命令，注册命令的入口，还会初始化其他数据结构。

command名字用 `module名.xxx`是一个好习惯。如果不同module之间有command冲突，那么他们将无法同时在Redis中工作，因为函数`RedisModule_CreateCommand()`将在某个模块中失败。

#### module初始化

`RedisModule_Init()`一般是`RedisModule_OnLoad()`中最开始调用的函数:

```c
int RedisModule_Init(RedisModuleCtx *ctx, const char *modulename, int module_version, int api_version);
```

该`Init`函数像redis core确定了module name、版本(`MODULE LIST`拿到)，以及那个版本的API。

如果API版本错误、module名以及被占用 或者其他类似错误，函数将返回`REDISMODULE_ERR`。

第二个函数:`RedisModule_CreateCommand()`，用于注册命令到Redis中。函数定义如下:

```c
int RedisModule_CreateCommand(RedisModuleCtx *ctx, const char *name,
                              RedisModuleCmdFunc cmdfunc, const char *strflags,
                              int firstkey, int lastkey, int keystep);
```

