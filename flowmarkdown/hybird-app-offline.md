```flow
st=>start: 开始
e=>end: 升级结束
success=>operation: 升级成功
清空modulezip/[模块]
appStart=>operation: App 启动
appKill=>condition: 是否App
崩溃或被
杀掉进程
appKill1=>condition: 是否App
崩溃或被
杀掉进程
offlineRequest=>condition: 是否offline
ResourceInfo
接口返回数据
downloadZip=>operation: 下载模块Zip资源包
rollbackModule=>condition: 回滚模块
clearModule=>operation: 清空模块资源
clearModule1=>operation: 清空模块资源
setUpgradingTrue=>operation: 正在升级Flag=Yes
正在备份Flag=Yes
setUpgradingFalse=>operation: 正在升级Flag=No
setBackupingTrue=>operation: 正在备份Flag=Yes
setBackupingFalse=>operation: 正在备份Flag=No
upgradingIsTrue=>condition: 正在升级
Flag==Yes
backupingIsTrue=>condition: 正在备份
Flag==Yes
setUpgradingFalse1=>operation: 正在升级Flag=No
正在备份Flag=No
setUpgradingFalse2=>operation: 正在升级Flag=No
正在备份Flag=No

validateZipMd5Cond=>condition: 校验模块Zip
的Md5
unZipCond=>condition: 清除temp目录
并解压模块Zip
到temp目录
validateFilesMd5Cond=>condition: 校验模块
子资源的Md5
backupTargetModuleCond=>condition: 清空备份
并备份目标
模块资源
upgradeTargetModuleCond=>condition: 升级目标
模块资源


st->appStart->upgradingIsTrue
upgradingIsTrue(no)->offlineRequest
upgradingIsTrue(yes)->backupingIsTrue
backupingIsTrue(no)->clearModule->rollbackModule
backupingIsTrue(yes)->setUpgradingFalse1->offlineRequest
rollbackModule(no)->clearModule1->appKill(right)
rollbackModule(yes,right)->setUpgradingFalse1->offlineRequest
offlineRequest(no)->e
offlineRequest(yes)->downloadZip->setUpgradingTrue->validateZipMd5Cond
validateZipMd5Cond(no)->appKill(right)
validateZipMd5Cond(yes)->unZipCond
unZipCond(no)->appKill(right)
unZipCond(yes)->validateFilesMd5Cond
validateFilesMd5Cond(no)->appKill(right)
validateFilesMd5Cond(yes)->backupTargetModuleCond
backupTargetModuleCond(no)->appKill(right)
backupTargetModuleCond(yes)->setBackupingFalse->upgradeTargetModuleCond
upgradeTargetModuleCond(no)->appKill1
upgradeTargetModuleCond(yes)->setUpgradingFalse->success->e
appKill(no)->setUpgradingFalse2->e
appKill(yes,right)->appStart(right)
appKill1(no)->e
appKill1(yes,right)->appStart(right)

```



