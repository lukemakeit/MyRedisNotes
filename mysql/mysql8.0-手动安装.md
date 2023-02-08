#### MySQL 8.0 手动安装

1. 准备`my.conf.20000`

   ```shell
   [client]
   port=20000
   socket=/data/mysqldata/20000/mysql.sock
   [mysqld]
   bind-address=10.211.55.7
   default-storage-engine=innodb
   skip-name-resolve
   performance_schema=OFF
   log_slow_admin_statements=ON
   slow_query_log=1
   binlog_format=ROW
   datadir=/data/mysqldata/20000/data
   character-set-server=utf8mb4
   innodb_buffer_pool_instances=2
   innodb_buffer_pool_size=64M
   innodb_data_file_path=ibdata1:128M:autoextend
   innodb_data_home_dir=/data/mysqldata/20000/innodb/data
   innodb_file_per_table=1
   innodb_io_capacity=200
   innodb_lock_wait_timeout=50
   innodb_log_buffer_size=16M
   innodb_log_file_size=64M
   innodb_log_files_in_group=2
   innodb_log_group_home_dir=/data/mysqldata/20000/innodb/log
   innodb_read_io_threads=1
   innodb_thread_concurrency=1
   innodb_write_io_threads=1
   interactive_timeout=86400
   key_buffer_size=8M
   log_bin=/data/mysqllog/20000/binlog/binlog20000.bin
   log_bin_trust_function_creators=1
   log_slave_updates=1
   slow_query_log_file=/data/mysqllog/20000/slow-query.log
   log_error_verbosity=1
   long_query_time=1
   max_allowed_packet=64M
   max_binlog_size=16M
   max_connect_errors=99999999
   max_connections=6000
   myisam_sort_buffer_size=64M
   port=20000
   read_buffer_size=2M
   relay-log=/data/mysqldata/20000/relay-log/relay-log.bin
   relay_log_recovery=1
   server_id=76702824
   skip-external-locking
   sync_binlog=0
   secure_file_priv=''
   innodb_strict_mode=off
   log_timestamps=SYSTEM
   avoid_temporal_upgrade=true
   skip-symbolic-links
   slave_compressed_protocol=1
   slave_exec_mode=idempotent
   socket=/data/mysqldata/20000/mysql.sock
   sort_buffer_size=2M
   stored_program_cache=1024
   table_open_cache=5120
   thread_cache_size=8
   tmpdir=/data/mysqldata/20000/tmp
   wait_timeout=86400
   sql_mode=''
   [mysql]
   default-character-set=utf8mb4
   no-auto-rehash
   port=20000
   socket=/data/mysqldata/20000/mysql.sock
   [mysqldump]
   max_allowed_packet=64M
   quick
   ```

2. 目录创建:

   ```shell
   #创建命令
   mkdir -p /data/mysqldata/20000/relay-log
   mkdir -p /data/mysqldata/20000/tmp
   mkdir -p /data/mysqldata/20000/data
   mkdir -p /data/mysqldata/20000/innodb
   mkdir -p /data/mysqldata/20000/innodb/log
   mkdir -p /data/mysqldata/20000/innodb/data
   mkdir -p /data/mysqlog/20000
   mkdir -p /data/mysqlog/20000/binlog
   
   #清理命令
   rm -rf /data/mysqldata/20000/data/*
   rm -rf /data/mysqldata/20000/innodb/data/*
   rm -rf /data/mysqldata/20000/innodb/log/*
   ```

3. 初始化与启动:

   ```shell
   echo "export MYSQL_HOME=/usr/local/mysql" >> /etc/profile
   echo "export PATH=\$MYSQL_HOME/bin:\$PATH" >> /etc/profile
   
   cd /usr/local/mysql/bin/
   ./mysqld  --defaults-file=/data/mysqldata/20000/my.cnf.20000 --initialize --user=root
   
   ./mysqld_safe --defaults-file=/data/mysqldata/20000/my.cnf.20000 --user=root &
   ```

4. 本地连接与修改密码:

   ```shell
   mysql -uroot -p --socket=/data/mysqldata/20000/mysql.sock
   
   > alter user 'root'@'localhost' identified with mysql_native_password by 'xxxx';
   ```

5. 本地创建用户: