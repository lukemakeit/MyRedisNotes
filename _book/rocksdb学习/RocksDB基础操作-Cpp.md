## RocksDB基础操作——CPP

#### Open一个DB

```cpp
  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  //options.error_if_exists = true; //如果文件夹已经存在则报错
  rocksdb::Status status =
      rocksdb::DB::Open(options, "/data/rocksdb/test01", &db);
  if (!status.ok()) {
    cout << "db open err:" << status.ToString() << endl;
    return -1;
  }
  std::shared_ptr<rocksdb::DB> spDb(db, [](rocksdb::DB* db) {
    rocksdb::Status status = db->Close();
    if (!status.ok()) {
      std::cout << "rocksdb db close fail,err:" << status.ToString()
                << std::endl;
      return;
    }
    std::cout << "rocksdb db close success" << std::endl;
    delete db;
  });
```

#### 基本的读写

```cpp
  // 基本的读写
  rocksdb::Slice k01("hello");
  rocksdb::Slice v01("value");
  rocksdb::Status s = spDb->Put(rocksdb::WriteOptions(), k01, v01);
  if (!s.ok()) {
    cout << "db put err:" << s.ToString() << endl;
    return -1;
  }
  string v1;
  s = spDb->Get(rocksdb::ReadOptions(), "hello", &v1);
  if (!s.ok()) {
    cout << "db get err:" << s.ToString() << endl;
    return -1;
  }
  cout << "v1:" << v1 << endl;
```

**rocksdb::Slice**

- `rocksdb::Slice` 自身由一个长度字段以及一个指向外部内存区域的指针[`const char* data_`]构成，返回slice比返回一个string廉价,且不存在内存拷贝问题;

- `rocksdb::Slice`和`string`之间的转换:

  ```cpp
  rocksdb::Slice s1 = “hello”;
  std::string str(“world”);
  rocksdb::Slice s2 = str;
  
  OR:
  
  std::string str = s1.ToString();
  assert(str == std::string(“hello”));
  ```

- <mark style="color:red">**注意rocksdb::Slice的安全性**</mark>。**当退出if语句块之后，`rocksdb::Slice`内部指针指向的内存区域已经不存在，此时再使用程序会有问题**。

  ```cpp
  rocksdb::Slice slice;
  if (…) {
   std::string str = …;
   slice = str;
  }
  Use(slice);
  ```


**rocksdb::PinnableSlice**

- `rocksdb::PinnableSlice`和`Slice`一样，可以减少内存拷贝，提高读性能。不过`rocksdb::PinnableSlice`中增加了一个引用计数功能，可以实现数据内存的延迟释放，延长相关数据的生命周期;

- 参考文档: [PinnableSlice](https://rocksdb.org/blog/2017/08/24/pinnableslice.html)

- `PinnableSlice`只在析构 或 调用`::Reset`之后，内存数据才会释放;

- 用`PinnableSlice`替换前面的`Get`接口:

  ```cpp
  Status Get(const ReadOptions& options,
                       ColumnFamilyHandle* column_family, const Slice& key,
                       std::string* value)
  
  virtual Status Get(const ReadOptions& options,
                       ColumnFamilyHandle* column_family, const Slice& key,
                       PinnableSlice* value)
  ```

#### 原子批量更新

原子更新,将大量修改合并到一个批处理中,加速批量更新。

```cpp
rocksdb::WriteBatch batch;
batch.Delete("hello");
batch.Put("luke", v1);
s = spDb->Write(rocksdb::WriteOptions(), &batch);
if (!s.ok()) {
  cout << "db WriteBatch err:" << s.ToString() << endl;
  return -1;
}
```

#### 批量获取值

```cpp
vector<rocksdb::Slice> keys;
vector<string> values;
vector<rocksdb::Status> statuses;

keys.emplace_back("luke");
keys.emplace_back("good");

values.resize(keys.size());
statuses.resize(keys.size());

statuses = spDb->MultiGet(rocksdb::ReadOptions(), keys, &values);
for (auto& item : values) {
  cout << "value=>" << item.data() << endl;
}
```

#### SnapShot

```cpp
// 写入分数
int score01 = 97;
rocksdb::Slice vv1((char*)&score01, sizeof(int)); //如何向rocksdb中写入int类型
spDb->Put(write_options, "luke01_math", vv1);

int score02 = 88;
rocksdb::Slice vv2((char*)&score02, sizeof(int));
spDb->Put(write_options, "luke01_chinese", vv2);

rocksdb::ReadOptions ropts;
ropts.snapshot = spDb->GetSnapshot(); //获取一个snapShot
batch.Clear();
int newScore01 = 100;
int newScore02 = 101;
rocksdb::Slice vv11((char*)&newScore01, sizeof(int));
rocksdb::Slice vv22((char*)&newScore02, sizeof(int));

batch.Put("luke01_math", vv11);
batch.Put("luke01_chinese", vv22);
spDb->Write(rocksdb::WriteOptions(), &batch);
// 打印修改后的数据
it = spDb->NewIterator(rocksdb::ReadOptions());
for (it->Seek("luke01_*"); it->Valid(); it->Next()) {
  rocksdb::Slice tmpv = it->value();
  int score = *((int*)tmpv.data());
  cout << "After snapShot for: k=>" << it->key().ToString() << " v=>" << score
       << endl;
}
assert(it->status().ok());
delete it;

// 打印snapshot中的数据
it = spDb->NewIterator(ropts);
for (it->Seek("luke01_*"); it->Valid(); it->Next()) {
  rocksdb::Slice tmpv = it->value();
  int score = *((int*)tmpv.data());
  cout << "in snapShot for: k=>" << it->key().ToString() << " v=>" << score
       << endl;
}
assert(it->status().ok());
delete it;

//释放snapshot
spDb->ReleaseSnapshot(ropts.snapshot);
```

#### ColumnFamily 基本操作

```cpp
// open DB
rocksdb::DB* db;
rocksdb::Options options;
options.create_if_missing = true;

//添加已经存在的columnFamily
std::vector<rocksdb::ColumnFamilyDescriptor> columnFamies;
columnFamies.push_back(rocksdb::ColumnFamilyDescriptor(
    rocksdb::kDefaultColumnFamilyName, rocksdb::ColumnFamilyOptions()));
std::vector<rocksdb::ColumnFamilyHandle*> handles;

rocksdb::Status s = rocksdb::DB::Open(options, "/data/rocksdb/test02",
                                      columnFamies, &handles, &db);
if (!s.ok()) {
  std::cout << "db open err:" << s.ToString() << std::endl;
  return -1;
}
//创建新的columnFamily
rocksdb::ColumnFamilyHandle* cfh;
s = db->CreateColumnFamily(rocksdb::ColumnFamilyOptions(), "new_cf", &cfh);
if (!s.ok()) {
  std::cout << "db create columnFamily fail,err:" << s.ToString()
            << std::endl;
  return -1;
}
handles.push_back(cfh);  //保存columnFamilyHandle

std::shared_ptr<rocksdb::DB> spDb(db, [&](rocksdb::DB* db) {
  rocksdb::Status ss;
  for (auto handle : handles) {
    ss = db->DestroyColumnFamilyHandle(handle);
    if (!ss.ok()) {
      std::cout << "rocksdb destroyColumnFamilyHandle fail,err:"
                << ss.ToString() << std::endl;
      return;
    }
  }
  rocksdb::Status status = db->Close();
  if (!status.ok()) {
    std::cout << "rocksdb db close fail,err:" << status.ToString()
              << std::endl;
    return;
  }
  std::cout << "rocksdb db close success" << std::endl;
  delete db;
});

//朝handles[1]中写入 key=>value
s = spDb->Put(rocksdb::WriteOptions(), handles[1], rocksdb::Slice("key"),
              rocksdb::Slice("value"));
if (!s.ok()) {
  std::cout << "db put handles[1] fail,err:" << s.ToString() << std::endl;
  return -1;
}
rocksdb::PinnableSlice pVal;
s = spDb->Get(rocksdb::ReadOptions(), handles[1], rocksdb::Slice("key"),
              &pVal);
if (!s.ok()) {
  std::cout << "db put handles[1] fail,err:" << s.ToString() << std::endl;
  return -1;
}
std::cout << "pVal:" << pVal.ToString() << std::endl;

//跨column family的原子批量写入
rocksdb::WriteBatch batch;
batch.Put(handles[0], rocksdb::Slice("key2"), rocksdb::Slice("value2"));
batch.Put(handles[1], rocksdb::Slice("key3"), rocksdb::Slice("value3"));
batch.Delete(handles[1], rocksdb::Slice("key"));
s = spDb->Write(rocksdb::WriteOptions(), &batch);
if (!s.ok()) {
  std::cout << "db columnFamily writeBatch fail,err:" << s.ToString()
            << std::endl;
  return -1;
}

//朝多个columnFamily中写入一些数据
batch.Put(handles[0], rocksdb::Slice("col0-key1"), rocksdb::Slice("value1"));
batch.Put(handles[0], rocksdb::Slice("col0-key2"), rocksdb::Slice("value2"));
batch.Put(handles[1], rocksdb::Slice("col1-key1"), rocksdb::Slice("value1"));
batch.Put(handles[1], rocksdb::Slice("col1-key2"), rocksdb::Slice("value2"));
s = spDb->Write(rocksdb::WriteOptions(), &batch);
if (!s.ok()) {
  std::cout << "db columnFamily writeBatch fail,err:" << s.ToString()
            << std::endl;
  return -1;
}

//单个columnFamily的MGET
std::vector<rocksdb::Slice> keys = {"col0-key1", "col0-key2"};
std::vector<rocksdb::PinnableSlice> values;
std::vector<rocksdb::Status> slist;
std::vector<std::string> timelist;
values.resize(keys.size());
slist.resize(keys.size());

spDb->MultiGet(rocksdb::ReadOptions(), handles[0], keys.size(), keys.data(),
               values.data(), slist.data());
for (auto& val : values) {
  std::cout << "mget from single columnFamily,value:" << val.ToString()
            << std::endl;
}
//多个columnFamily中执行MGET
keys.clear();
values.clear();
slist.clear();

std::vector<std::string> values02;
keys.emplace_back("col0-key1");
keys.emplace_back("col1-key1");
values02.resize(keys.size());

slist = spDb->MultiGet(rocksdb::ReadOptions(), handles, keys, &values02,
                       &timelist);
for (auto& val : values02) {
  std::cout << "mget from multi columnFamily,value:" << val << std::endl;
}

s = spDb->DropColumnFamily(handles[1]);
if (!s.ok()) {
  std::cout << "db drop columnFamily fail,err:" << s.ToString() << std::endl;
  return -1;
}
```

#### Merge operators

Merge operators 对read-modify-write 这种操作，提供了效率更高的支持。具体参考。

