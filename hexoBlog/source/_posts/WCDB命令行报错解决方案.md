---
title: WCDB命令行编译报错解决方案
date: 2018-01-16 14:13:20
tags:
categories:
---


# 目录
> 1. 问题
> 2. 尝试 xcodebuild
> 3. 查看 WCDB 是怎么编译
> 4. 尝试自动静态库
> 5. 尝试手动静态库
> 6. 总结

## 1. 问题
由于项目需求，需要使用一款数据库，直接使用 sqlite 会手动写很多 sql 代码，也容易出错。使用苹果官方的 core data，core data 不是线程安全的，需要严格区分在不同的线程使用不同的 manage context，使用上也增加了代码复杂度，也会更容易出现 bug。于是考虑下基于 sqlite 的开源方案。

> FMDB: 其实就是 sqlite 的语法糖，多线程访问需要使用专门的 queue，不支持 ORM。
> Realm: 是为速度而生的数据库，很多配套设施并不齐全。
> WCDB: 性能高，支持线程安全，支持 ORM。缺点：1. 基于objective c++，所以只要使用到了 WCDB 的地方，都需要以 mm 为文件后缀。2. 增大包体积。


想找到一款易使用有安全的数据库，最后还是考虑使用了 WCDB， 虽然它也有自己的不足，但是还是可以接受的。

于是通过 pod 在项目中使用了 WCDB，并写了小 [demo](https://github.com/karosLi/WCDBDemo) 熟悉下了使用方式。

```
pod 'WCDB'
```

在熟悉了使用方式之后，就可以在项目里使用了，在开发阶段都是通过模拟器或者真机测试，以为都很好。但是在搭建 Bamboo 的 CICD 就会报以下错。


```
fts5_storage.c:305:9: error: 'sqlite3_api_routines' has no member named '__builtin___snprintf_chk'

The following build commands failed:
	CompileC /Users/karosli/Library/Developer/Xcode/DerivedData/xxx-dleyffpkgrddqzfvrhnoakpqgdvz/Build/Intermediates.noindex/ArchiveIntermediates/LefitCoach/IntermediateBuildFilesPath/Pods.build/Debug-iphoneos/WCDB.build/Objects-normal/armv7/fts5.o WCDB/sqlcipher/fts5.c normal armv7 c com.apple.compilers.llvm.clang.1_0.compiler
(1 failure)
```

<!--more-->
## 2. 尝试 xcodebuild
由于 CICD 使用的是基于 flastlane gym 的 [shell](https://github.com/karosLi/jenkins-fastlane/blob/master/build.sh) 脚本，因而怀疑是不是 fastlane 的原因呢，因为我在 Xcode 上直接运行是好的。所以就尝试使用 xcodebuild 去编译打包。


```
# fastlane gym --silent --workspace ${app_workspace} --scheme ${app_schema} --clean --xcargs 'GCC_PREPROCESSOR_DEFINITIONS="$GCC_PREPROCESSOR_DEFINITIONS DEBUG=1 COCOAPODS=1"' --export_method development --output_directory ${outputDir} --output_name ${app_ipa_file}

# 把上面替换成下面的命令
# archive
  xcodebuild archive -sdk iphoneos -workspace ${app_workspace} -scheme ${app_schema} -configuration Debug -archivePath ${outputDir} GCC_PREPROCESSOR_DEFINITIONS="$GCC_PREPROCESSOR_DEFINITIONS DEBUG=1 COCOAPODS=1"
# export ipa
xcodebuild -exportArchive -exportFormat IPA -archivePath ${outputDir} -exportPath ${app_ipa_file} -exportOptionsPlist exportOptions_dev.plist
```

## 3. 查看 WCDB 是怎么编译
由于 WCDB 通过 pod 的方式是可以再 Xcode 上运行的，猜想下是不是 WCDB.podspec 里面做了额外的事情。

可以看到文件里面多了一个预处理命令 `wcdb.prepare_command`。

```
# pod lib lint --verbose --skip-import-validation WCDB.podspec
# pod trunk push WCDB.podspec --verbose --skip-import-validation
Pod::Spec.new do |wcdb|
  wcdb.name         = "WCDB"
  wcdb.version      = "1.0.6"
  wcdb.summary      = "WCDB is a cross-platform database framework developed by WeChat."
  wcdb.description  = <<-DESC
                      The WeChat Database, for Objective-C. (If you want to use WCDB from Objective-C, see the "WCDBSwift" pod.)

                      WCDB is an efficient, complete, easy-to-use mobile database framework used in the WeChat application.
                      It can be a replacement for Core Data, SQLite & FMDB.
                      DESC
  wcdb.homepage     = "https://github.com/Tencent/wcdb"
  wcdb.license      = { :type => "BSD", :file => "LICENSE"}
  wcdb.author             = { "sanhuazhang" => "sanhuazhang@tencent.com" }
  wcdb.ios.deployment_target = "7.0"
  wcdb.osx.deployment_target = "10.9"
  wcdb.watchos.deployment_target = "2.0"
  wcdb.tvos.deployment_target = "9.0"
  wcdb.source       = { :git => "https://github.com/Tencent/wcdb.git", :tag => "v#{wcdb.version}" }
  wcdb.public_header_files = "objc/WCDB/WCDB.h", "objc/WCDB/**/*.{h,hpp}"
  wcdb.source_files  = "objc/WCDB/WCDB.h", "objc/WCDB/**/*.{h,m,hpp,cpp,mm}", "repair"
  wcdb.frameworks = "CoreFoundation", "Security", "Foundation"
  wcdb.ios.frameworks = "UIKit"
  wcdb.libraries = "z", "c++"
  wcdb.requires_arc = true
  wcdb.prepare_command = "git submodule update --init sqlcipher; \
                          cd tools/templates; sh install.sh; cd ../..; \
                          cd sqlcipher; make -f Makefile.preprocessed; cd ..; \
                          cp sqlcipher/ext/fts3/fts3_tokenizer.h sqlcipher/"
  wcdb.xcconfig = {
    "GCC_PREPROCESSOR_DEFINITIONS" => "$(inherited) WCDB_BUILTIN_COLUMN_CODING WCDB_COCOAPODS",
    "HEADER_SEARCH_PATHS" => "$(inherited) ${PODS_ROOT}/WCDB",
    "LIBRARY_SEARCH_PATHS[sdk=macosx*]" => "$(inherited) $(SDKROOT)/usr/lib/system",
    "CLANG_CXX_LANGUAGE_STANDARD" => "gnu++0x",
    "CLANG_CXX_LIBRARY" => "libc++",
  }

  wcdb.subspec 'sqlcipher' do |sqlcipher|
    sqlcipher.public_header_files = "sqlcipher/sqlite3.h", "sqlcipher/fts3_tokenizer.h"
    sqlcipher.source_files = "sqlcipher/src/callback.c", "sqlcipher/src/loadext.c", "sqlcipher/src/rowset.c", "sqlcipher/src/treeview.c", "sqlcipher/ext/userauth.c", "sqlcipher/src/vtab.c", "sqlcipher/src/btmutex.c", "sqlcipher/src/btree.c", "sqlcipher/src/btreeInt.h", "sqlcipher/src/btree.h", "sqlcipher/fts5.c", "sqlcipher/fts5.h", "sqlcipher/ext/fts3/fts3_aux.c", "sqlcipher/ext/fts3/fts3_expr.c", "sqlcipher/ext/fts3/fts3_hash.c", "sqlcipher/ext/fts3/fts3_hash.h", "sqlcipher/ext/fts3/fts3_icu.c", "sqlcipher/ext/fts3/fts3_porter.c", "sqlcipher/ext/fts3/fts3_snippet.c", "sqlcipher/ext/fts3/fts3_tokenize_vtab.c", "sqlcipher/ext/fts3/fts3_tokenizer.c", "sqlcipher/ext/fts3/fts3_tokenizer1.c", "sqlcipher/ext/fts3/fts3_unicode.c", "sqlcipher/ext/fts3/fts3_unicode2.c", "sqlcipher/ext/fts3/fts3_write.c", "sqlcipher/ext/fts3/fts3.c", "sqlcipher/ext/fts3/fts3.h", "sqlcipher/ext/fts3/fts3Int.h", "sqlcipher/src/backup.c", "sqlcipher/src/legacy.c", "sqlcipher/src/main.c", "sqlcipher/src/notify.c", "sqlcipher/src/vdbeapi.c", "sqlcipher/src/table.c", "sqlcipher/src/wal.c", "sqlcipher/src/wal.h", "sqlcipher/src/status.c", "sqlcipher/src/prepare.c", "sqlcipher/src/malloc.c", "sqlcipher/src/mem0.c", "sqlcipher/src/mem1.c", "sqlcipher/src/mem2.c", "sqlcipher/src/mem3.c", "sqlcipher/src/mem5.c", "sqlcipher/src/memjournal.c", "sqlcipher/src/mutex_unix.c", "sqlcipher/src/mutex_noop.c", "sqlcipher/src/mutex.c", "sqlcipher/src/mutex.h", "sqlcipher/src/os_common.h", "sqlcipher/src/os_setup.h", "sqlcipher/src/os_unix.c", "sqlcipher/src/queue.c", "sqlcipher/src/queue.h", "sqlcipher/src/os_wcdb.c", "sqlcipher/src/os_wcdb.h", "sqlcipher/src/mutex_wcdb.c", "sqlcipher/src/mutex_wcdb.h", "sqlcipher/src/os.c", "sqlcipher/src/os.h", "sqlcipher/src/threads.c", "sqlcipher/src/bitvec.c", "sqlcipher/src/pager.c", "sqlcipher/src/pager.h", "sqlcipher/src/pcache.c", "sqlcipher/src/pcache.h", "sqlcipher/src/pcache1.c", "sqlcipher/ext/rtree/rtree.c", "sqlcipher/ext/rtree/rtree.h", "sqlcipher/ext/rtree/sqlite3rtree.h", "sqlcipher/src/complete.c", "sqlcipher/src/tokenize.c", "sqlcipher/src/resolve.c", "sqlcipher/parse.c", "sqlcipher/parse.h", "sqlcipher/src/analyze.c", "sqlcipher/src/func.c", "sqlcipher/src/wherecode.c", "sqlcipher/src/whereexpr.c", "sqlcipher/src/whereInt.h", "sqlcipher/src/alter.c", "sqlcipher/src/attach.c", "sqlcipher/src/auth.c", "sqlcipher/src/build.c", "sqlcipher/src/delete.c", "sqlcipher/src/expr.c", "sqlcipher/src/insert.c", "sqlcipher/src/pragma.c", "sqlcipher/src/pragma.h", "sqlcipher/src/select.c", "sqlcipher/src/trigger.c", "sqlcipher/src/update.c", "sqlcipher/src/vacuum.c", "sqlcipher/src/walker.c", "sqlcipher/src/where.c", "sqlcipher/opcodes.c", "sqlcipher/opcodes.h", "sqlcipher/src/sqlcipher.h", "sqlcipher/sqlite3.h", "sqlcipher/ext/rbu/sqlite3rbu.c", "sqlcipher/ext/rbu/sqlite3rbu.h", "sqlcipher/ext/userauth/sqlite3userauth.h", "sqlcipher/ext/misu/json1.c", "sqlcipher/ext/icu/icu.c", "sqlcipher/ext/icu/sqliteicu.h", "sqlcipher/src/global.c", "sqlcipher/src/ctime.c", "sqlcipher/src/hwtime.h", "sqlcipher/src/date.c", "sqlcipher/src/dbstat.c", "sqlcipher/src/fault.c", "sqlcipher/src/fkey.c", "sqlcipher/src/sqliteInt.h", "sqlcipher/src/sqliteLimit.h", "sqlcipher/src/sqlite3ext.h", "sqlcipher/src/hash.c", "sqlcipher/src/hash.h", "sqlcipher/src/printf.c", "sqlcipher/src/random.c", "sqlcipher/src/utf.c", "sqlcipher/src/util.c", "sqlcipher/src/crypto_cc.c", "sqlcipher/src/crypto_impl.c", "sqlcipher/src/crypto_libtomcrypt.c", "sqlcipher/src/crypto.c", "sqlcipher/src/crypto.h", "sqlcipher/src/vdbe.c", "sqlcipher/src/vdbe.h", "sqlcipher/src/vdbeaux.c", "sqlcipher/src/vdbeblob.c", "sqlcipher/src/vdbeInt.h", "sqlcipher/src/vdbemem.c", "sqlcipher/src/vdbesort.c", "sqlcipher/src/vdbetrace.c", "sqlcipher/src/msvc.h", "sqlcipher/src/vxworks.h", "sqlcipher/fts3_tokenizer.h", "sqlcipher/keywordhash.h"
    sqlcipher.ios.deployment_target = "8.0"
    sqlcipher.osx.deployment_target = "10.9"
    sqlcipher.watchos.deployment_target = "2.0"
    sqlcipher.tvos.deployment_target = "9.0"
    sqlcipher.xcconfig = {
      "GCC_PREPROCESSOR_DEFINITIONS" => "$(inherited) SQLITE_ENABLE_FTS3 SQLITE_ENABLE_FTS3_PARENTHESIS SQLITE_ENABLE_API_ARMOR SQLITE_OMIT_BUILTIN_TEST SQLITE_OMIT_AUTORESET SQLITE_ENABLE_UPDATE_DELETE_LIMIT SQLITE_ENABLE_RTREE SQLITE_ENABLE_LOCKING_STYLE=1 SQLITE_SYSTEM_MALLOC SQLITE_OMIT_LOAD_EXTENSION SQLITE_CORE SQLITE_THREADSAFE=2 SQLITE_DEFAULT_CACHE_SIZE=250 SQLITE_DEFAULT_CKPTFULLFSYNC=1 SQLITE_DEFAULT_PAGE_SIZE=4096 SQLITE_OMIT_SHARED_CACHE SQLITE_HAS_CODEC SQLCIPHER_CRYPTO_CC USE_PREAD=1 SQLITE_TEMP_STORE=2 SQLCIPHER_PREPROCESSED HAVE_USLEEP SQLITE_MALLOC_SOFT_LIMIT=0 SQLITE_WCDB_SIGNAL_RETRY=1 SQLITE_DEFAULT_MEMSTATUS=0 SQLITE_ENABLE_COLUMN_METADATA SQLITE_DEFAULT_WAL_SYNCHRONOUS=1 SQLITE_LIKE_DOESNT_MATCH_BLOBS SQLITE_MAX_EXPR_DEPTH=0 SQLITE_OMIT_DEPRECATED SQLITE_OMIT_PROGRESS_CALLBACK SQLITE_OMIT_SHARED_CACHE OMIT_CONSTTIME_MEM OMIT_MEMLOCK SQLITE_ENABLE_FTS3_TOKENIZER",
      "CLANG_WARN_CONSTANT_CONVERSION" => "YES",
      "GCC_WARN_64_TO_32_BIT_CONVERSION" => "NO",
      "CLANG_WARN_UNREACHABLE_CODE" => "NO",
      "GCC_WARN_UNUSED_FUNCTION" => "NO",
      "GCC_WARN_UNUSED_VARIABLE" => "NO",
    }
  end
end
```

Makefile.preprocessed


```
./configure --enable-tempstore=yes --with-crypto-lib=commoncrypto CFLAGS="-DSQLITE_HAS_CODEC -DSQLITE_TEMP_STORE=2 -DSQLITE_ENABLE_UPDATE_DELETE_LIMIT" --disable-amalgamation
	make opcodes.h opcodes.c keywordhash.h fts5.c fts5.h sqlite3.h parse.h parse.c

```

这里里面做了预编译处理。于是就想怎么在命令行里面加入这样的命令，尝试了很多了方式，发现没有那么简单，所以就搁浅在这里了，不要打我，还请大神能帮我解答下。


## 4. 尝试自动静态库
于是就另辟蹊径，想到了静态库，把 WCDB 做成静态库的形式，那么再命令行打包的时候，不会再次编译了。

于是想到了 cocoapod package，去自动打成静态库，需要建立一个壳工程，里面用 pod 安装 WCDB，然后用命令行打静态库。[教程在这里](https://www.jianshu.com/p/815fc21b9d0d)

```
sudo gem install cocoapods-packager
```

```
pod package XXDatabase.podspec --force
```

成功后，会生成一个带版本号的库目录，然后把里面的 framework 放到单独的测试工程师测试，最后发现编译是通过的，但是找不到 WCDB 的头文件（即使 podspec 文件配置了 `s.public_header_files = 'Pods/**/*.h'`），此路不通。


## 5. 尝试手动静态库
于是尝试用手动的方式去打静态库，手动方式就有两种方案了。
> 方案一: 使用 WCDB 官方的教程是打静态库。
> 方案二: 通过 lipo 命令合并模拟器和真机静态库。

使用方案一打出来的静态库，如果在主工程里使用没有问题，一旦放入到单独的 pod 里，检测的时候，就会上传失败。所以放弃了。

```
xcodebuild: error: The workspace named "App" does not contain a scheme named "xxxDatabase". The "-list" option can be used to find the names of the schemes in the workspace.
```

然后尝试方案二，打出的包可以引入工程，代码编译没有错，但是一运行就报错了。
![](http://olf3t4omk.bkt.clouddn.com/15160901483602.jpg)

真是懵逼，百思不得解，后来想想，我现在打开的工程是 WCDB 的 master 分支，而 pod repo 里面的是 1.0.6 的 tag，所以就切换到 1.0.6 上重新进行方案二。

果然通过这个 tag 打出来的静态库是没有问题的，放入到测试工程(需要加上 other linker flags: `$(inherited) -ObjC -all_load`)，模拟器和真机测试皆可通过。[静态库制作教程](https://www.jianshu.com/p/7f6a7e1b3235)

剩下就是用一个单独的 pod 基础组件去包含这个静态库，然后
让其他业务组件引入这个 pod 基础组件就可以避免这个静态库存在多份。


```
Pod::Spec.new do |s|
  s.name         = "XXXDatabase"
  s.version      = "1.0.0"
  s.summary      = "XXXDatabase."

  s.description  = <<-DESC
                    this is XXXDatabase
                   DESC

  s.homepage     = "http://XXX.com/XXXDatabase.git"
  
  s.license      = { :type => "MIT", :file => "FILE_LICENSE" }


  s.author             = { "Karos" => "likai@XXX.com" }
  

  s.platform     = :ios, "8.0"

  s.source       = { :git => "http://XXX.com/XXXDatabase.git", :tag => s.version.to_s }


  s.source_files  = "XXXDatabase/XXXDatabase/**/*.{h,m,mm}", "XXXDatabase/ThirdSDK/**/*.{h}"
  
  s.vendored_frameworks = "XXXDatabase/ThirdSDK/WCDB.framework"
  s.frameworks = "CoreFoundation", "Security", "Foundation", "UIKit"
  s.libraries = "z", "c++"

  s.pod_target_xcconfig = {
      'OTHER_LDFLAGS' => ['-lObjC','-all_load', '-lstdc++']
  }

  s.requires_arc = true
end

```

## 6. 总结
虽然最后是问题是解决了，但也说明自己在编译原理上的欠缺，所以后面还是要补充这块的姿势啊。

## 参考
https://github.com/Tencent/wcdb/wiki
http://www.sqlite.org/src/info/cd0471ca9f75e7c8
https://www.jianshu.com/p/815fc21b9d0d
https://www.jianshu.com/p/50e0efb66bdf
https://www.jianshu.com/p/283e67ba12a3
https://www.jianshu.com/p/7f6a7e1b3235


