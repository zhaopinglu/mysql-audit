# A Quick Note on bulding mysql-audit for MySQL 8.0.33


### Compile Ref
https://github.com/trellix-enterprise/mysql-audit/blob/master/compiling.txt
https://github.com/trellix-enterprise/mysql-audit/issues/246


### Download mysql audit source code
https://github.com/trellix-enterprise/mysql-audit


### Download mysql source code
https://downloads.mysql.com/archives/get/p/23/file/mysql-boost-8.0.33.tar.gz


### Prepare
```
yum -y install  autoconf automake readline-devel gcc gcc-c++ boost make cmake cmake3 bison bison-devel ncurses-devel libaio-devel perl git libtirpc libtirpc-devel bzip2-devel python-devel curl-devel devtoolset-11-gcc devtoolset-11-gcc-c++ devtoolset-11-binutils
```


### Clone mysql audit source
```
git clone https://github.com/trellix-enterprise/mysql-audit.git
```

### Extract mysql 8.0.33 source code into mysql-audit code folder
```
cd mysql-audit
tar zxvf ../mysql-boost-8.0.33.tar.gz
```

### Build MySQL 8.0.33
```
cd mysql-audit/mysql-8.0.33
mkdir brelease
cd brelease
cmake3 .. -DWITH_BOOST=../boost 
```

### Modify mysql-audit source code

## Replace: "TABLE_LIST" -> "Table_ref"
```
cd mysql-audit
perl -pi -e 's/TABLE_LIST/Table_ref/g' `grep -rlw TABLE_LIST include/ src/ offset-extract/`
```

Ref: https://github.com/trellix-enterprise/mysql-audit/issues/266

## Replace: "my_charset_utf8_general_ci" To "my_charset_utf8mb3_general_ci" And "my_charset_utf8_bin" To "my_charset_utf8mb3_bin"
src/audit_handler.cc:
```
	836-#else	
	837-  /*
	838-    TODO: Migrate the data itself to UTF8MB4,
	839-    this is still UTF8MB3 printed in a UTF8MB4 column.
	840-  */
	841-  const char *well_formed_error_pos = NULL, *cannot_convert_error_pos = NULL,
	842-             *from_end_pos = NULL;
	843-  copy_length = well_formed_copy_nchars(
	844:      //&my_charset_utf8_bin
	845-      &my_charset_utf8mb3_bin
```
```
	1090-#if defined(MARIADB_BASE_VERSION) && MYSQL_VERSION_ID >= 100504	
	1091-					&my_charset_utf8mb3_general_ci,
	1092-#else
	1093:					//&my_charset_utf8_general_ci,
	1094-					&my_charset_utf8mb3_general_ci,
```

Ref: https://github.com/trellix-enterprise/mysql-audit/issues/261


### Build mysql-audit
```
cd mysql-audit
cp -rap ./mysql-8.0.33/brelease/include/* ./include/.
chmod +x bootstrap.sh
./bootstrap.sh
CXX='gcc -static-libgcc' CC='gcc -static-libgcc' ./configure --with-mysql=mysql-8.0.33/brelease --with-mysql-libservices=mysql-8.0.33/brelease/libservices/libmysqlservices.a
make -j16
```

### Install
```
cp src/.lib/libaudit_plugin.* /usr/lib64/mysql/plugin/.
chmod +x /usr/lib64/mysql/plugin/libaudit*
```

### Check
ls -l /usr/lib64/mysql/plugin/
```
-rwxr-xr-x 1 root root     991 1月  20 17:10:33 libaudit_plugin.lai
-rwxr-xr-x 1 root root     990 1月  20 17:10:33 libaudit_plugin.la
-rwxr-xr-x 1 root root 4661288 1月  20 17:10:33 libaudit_plugin.a
-rwxr-xr-x 1 root root 2222384 1月  20 17:10:36 libaudit_plugin.so
-rwxr-xr-x 1 root root 2222384 1月  20 17:10:36 libaudit_plugin.so.0
-rwxr-xr-x 1 root root 2222384 1月  20 17:10:36 libaudit_plugin.so.0.0.0
```

### Install mysql debug symbol file
```
yum install https://downloads.mysql.com/archives/get/p/23/file/mysql-community-debuginfo-8.0.33-1.el7.x86_64.rpm
```

### Collect mysqld offsets
```
cd offset-extract
./offset-extract.sh `which mysqld` /usr/lib/debug/usr/sbin/mysqld.debug

//offsets for: /usr/sbin/mysqld (8.0.33)
{"8.0.33","5ef11079d4a42b8d9e23ddd8af825ea3", 9504, 9544, 4960, 6444, 1288, 0, 0, 32, 64, 160, 1376, 9644, 6064, 4248, 4256, 4260, 7728, 1576, 32, 8688, 8728, 8712, 12568, 140, 664, 320},
```

Note: record the numbers. 

### Config mysqld
/etc/my.cnf:
```
plugin-load=AUDIT=libaudit_plugin.so
audit_json_file=on
audit_json_log_file=/var/lib/mysql/mysql-audit.json
audit_record_cmds='insert,delete,update,create,drop,alter,grant,truncate'
# For Oracle MySQL 8.0.33 EL7X64
audit_offsets=9504, 9544, 4960, 6444, 1288, 0, 0, 32, 64, 160, 1376, 9644, 6064, 4248, 4256, 4260, 7728, 1576, 32, 8688, 8728, 8712, 12568, 140, 664, 320
```

### Restart mysqld service
```
systemctl restart mysqld
```


### Test
```
mysql -uroot -p
use test;
create table test_log(id int, name varchar(30));
insert into test_log values(1,'asdf');
```

/var/lib/mysql/mysql-audit.json:
```
{"msg-type":"activity","date":"1705746443488","thread-id":"8","query-id":"13","user":"root","priv_user":"root","ip":"","host":"localhost","_pid":"918566","_platform":"x86_64","_os":"Linux","_client_name":"libmysql","os_user":"root","_client_version":"8.0.33","program_name":"mysql","pid":"918566","os_user":"root","appname":"mysql","rows":"1","status":"0","cmd":"insert","objects":[{"db":"test","name":"test_log","obj_type":"TABLE"}],"query":"insert into test_log values(1,'asdf')"}
```



# Issues

## Issue 1 WARNING: 'aclocal-1.15' is missing on your system.

### Symptom:
Hit error when making mysql-audit:
```
..
WARNING: 'aclocal-1.15' is missing on your system.
```

### Reason:
```
The install automake version is too old:
automake --version
1.13
```

### Solution:
Install automake 1.15 & build it
```
wget https://ftp.gnu.org/gnu/automake/automake-1.15.tar.gz
tar zxvf automake-1.15.tar.gz
cd automake-1.15
./configure
make -j4
make install
export PATH=/usr/local/bin:$PATH
```
Ref: https://blog.csdn.net/weixin_33724059/article/details/92385468

Verify:
```
aclocal --version
aclocal (GNU automake) 1.15
```


## Issue 2 fatal error: string_view: No such file or directory

### Symptom:
Hit error when making mysql-audit:
```
/root/tools/mysql-audit/mysql-8.0.33/include/lex_string.h:31:23: fatal error: string_view: No such file or directory
```

### Reason:
You'll need to update your compiler. std::string_view is not available in GCC until version 7.

Ref: https://stackoverflow.com/questions/64141078/compile-c-file-in-linux-terminal-string-view-no-such-file-or-directory


### Solution:
Install newer version of gcc:
```
wget https://ftp.tsukuba.wide.ad.jp/software/gcc/releases/gcc-13.2.0/gcc-13.2.0.tar.xz
tar xvf gcc-13.2.0.tar.xz
cd gcc-13.2.0
./contrib/download_prerequisites
./configure --disable-multilib --enable-languages=c,c++
make -j16
make install
export PATH=/usr/local/bin:$PATH
```

### Verify:
```
gcc --version
```

Ref: https://linuxhostsupport.com/blog/how-to-install-gcc-on-centos-7/


## Issue 3 crc32_z was not declared in this scope

### Solution:
Download the zlib 1.2.13 from https://github.com/madler/zlib/releases, then build & install it:
```
tar …
cd …
./configure
make 
make install
```

## Issue 4 error when loading audit plugin: Plugin 'AUDIT' registration as a AUDIT failed

### Symptom:
```
2024-01-20T09:20:42.552079Z 8 [ERROR] [MY-010202] [Server] Plugin 'AUDIT' init function returned error.
2024-01-20T09:20:42.552126Z 8 [ERROR] [MY-010734] [Server] Plugin 'AUDIT' registration as a AUDIT failed.
```

### Solution:
Install the corresponding version of mysql debuginfo rpm.

Collect mysqld offsets then set audit_offsets:
```
cd offset-extract
./offset-extract.sh `which mysqld` /usr/lib/debug/usr/sbin/mysqld.debug
```

/etc/my.cnf:
```
audit_offsets=9504, 9544, 4960, 6444, 1288, 0, 0, 32, 64, 160, 1376, 9644, 6064, 4248, 4256, 4260, 7728, 1576, 32, 8688, 8728, 8712, 12568, 140, 664, 320
```

Ref:
https://blog.csdn.net/omaidb/article/details/130387489![image](https://github.com/zhaopinglu/mysql-audit/assets/6966161/65de2060-fcc1-45e2-aeaa-09595d57b23a)
