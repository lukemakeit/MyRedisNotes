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

