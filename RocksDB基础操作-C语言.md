## RocksDB基础操作——C语言

#### Open一个DB

```c
#include "rocksdb/c.h"
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

const char DBPath[] = "./data/c_simple_example";
rocksdb_t *db;
rocksdb_options_t *options = rocksdb_options_create();
//设置并发度
long cpus = sysconf(_SC_NPROCESSORS_ONLN);
rocksdb_options_increase_parallelism(options, cpus);
rocksdb_options_optimize_level_style_compaction(options, 0);
//如果DB不存在,则创建
rocksdb_options_set_create_if_missing(options, 1);

char *err = NULL;
db = rocksdb_open(options, DBPath, &err);
if (err)
{
    perror(err);
    exit(-1);
}
rocksdb_options_destroy(options);
rocksdb_close(db);
```

#### 写入k/v

```c
char *err = NULL;
//put key-value
rocksdb_writeoptions_t *writeopt = rocksdb_writeoptions_create();
const char key[] = "hello";
const char *val = "world";
rocksdb_put(db, writeopt, key, strlen(key), val, strlen(val), &err);
if (err)
{
    perror(err);
    exit(-1);
}
rocksdb_writeoptions_destroy(writeopt);
```

#### 读取k/v

```c
//get value
rocksdb_readoptions_t *readopt = rocksdb_readoptions_create();
size_t len;
char *ret_val = rocksdb_get(db, readopt, key, strlen(key), &len, &err);
assert(!err);
assert(strncmp(ret_val, val, len) == 0);
printf("ret_val=>%.*s val_len=>%d\n", len, ret_val, len);
free(ret_val);
rocksdb_readoptions_destroy(readopt);
```

#### 使用pinnableslice 进行读取，可减少一次内存拷贝

```c
rocksdb_readoptions_t *readopt = rocksdb_readoptions_create();
size_t len;
char *err = NULL;
const char key[] = "hello";
const char *val = "world";
rocksdb_pinnableslice_t *pinVal = rocksdb_get_pinned(db, readopt, key, strlen(key), &err);
if (err)
{
    printf("rocksdb_get_pinned fail,err:%s\n", err);
    return -1;
}
const char *pConstVal = rocksdb_pinnableslice_value(pinVal, &len); //从pinnableslice中读取值
assert(strncmp(pConstVal, val, len) == 0);
printf("pConstVal=>%.*s pConstValLen=>%d\n", len, pConstVal, len);

rocksdb_pinnableslice_destroy(pinVal);
```

#### 保存二进制key 同时 根据前缀遍历

```c
char *k01 = (char *)malloc(20);
char name[20] = {0};
char delimiter = ':';
size_t l01 = 0;
size_t prefixLen = 0;
int serverID = 666666; //如果将666666保存为字符串需要6个字节,保存为int则只需要4个字节
memcpy(k01 + l01, (char *)&serverID, sizeof(int)); //4个字节保存int
l01 += sizeof(int);
memcpy(k01 + l01, &delimiter, sizeof(char));
l01 += sizeof(char);
prefixLen = l01;

strcpy(name, "zhangsan");
memcpy(k01 + l01, name, strlen(name));
l01 += strlen(name);
rocksdb_put(db, writeopt, k01, l01, val, strlen(val), &err);
if (err)
{
    perror(err);
    exit(-1);
}

l01 = prefixLen;
strcpy(name, "lisi");
memcpy(k01 + l01, name, strlen(name));
l01 += strlen(name);
rocksdb_put(db, writeopt, k01, l01, val, strlen(val), &err);
if (err)
{
    perror(err);
    exit(-1);
}

l01 = prefixLen;
strcpy(name, "wangwu");
memcpy(k01 + l01, name, strlen(name));
l01 += strlen(name);
rocksdb_put(db, writeopt, k01, l01, val, strlen(val), &err);
if (err)
{
    perror(err);
    exit(-1);
}

//准备遍历
rocksdb_iterator_t *iter = rocksdb_create_iterator(db, readopt);
if (iter == NULL)
{
    printf("create iterator fail\n");
    return;
}
rocksdb_iter_seek(iter, (char *)&serverID, sizeof(int));
rocksdb_iter_get_error(iter, &err);
if (err != NULL)
{
    printf("rocksdb iterator failed...%s\n", err);
    return;
}
while (rocksdb_iter_valid(iter))
{
    size_t keyLen = 0;
    size_t valLen = 0;
    char *keyData = rocksdb_iter_key(iter, &keyLen);
    rocksdb_iter_get_error(iter, &err);
    if (err != NULL)
    {
        printf("rocksdb iterator key failed...%s\n", err);
        return;
    }
    char *valData = rocksdb_iter_value(iter, &valLen);
    rocksdb_iter_get_error(iter, &err);
    if (err != NULL)
    {
        printf("rocksdb iterator value failed...%s\n", err);
        return;
    }

    //解析key中的serverID
    int keySerID = *(int *)(keyData);
    char keyBuf[40] = {0};
    sprintf(keyBuf, "key=>%d", keySerID);
    //获取key中的剩余部分
    memcpy(keyBuf + strlen(keyBuf), keyData + sizeof(int), keyLen - sizeof(int));

    //获取value
    sprintf(keyBuf, "%s value=>", keyBuf);
    memcpy(keyBuf + strlen(keyBuf), valData, valLen);

    printf("[iter] %s\n", keyBuf);
    rocksdb_iter_next(iter);
    rocksdb_iter_get_error(iter, &err);
    if (err != NULL)
    {
        printf("rocksdb iterator next failed...%s\n", err);
        return;
    }
}

rocksdb_iter_destroy(iter);
free(k01);
rocksdb_writeoptions_destroy(writeopt);
rocksdb_readoptions_destroy(readopt);
rocksdb_options_destroy(options);
rocksdb_backup_engine_close(be);
rocksdb_close(db);

结果:
[iter] key=>666666:lisi value=>world
[iter] key=>666666:wangwu value=>world
[iter] key=>666666:zhangsan value=>world
```

