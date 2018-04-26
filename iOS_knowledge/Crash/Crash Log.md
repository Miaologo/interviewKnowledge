Crash Log

# 符号化命令

## atos
使用`atos`单行符号化
```
atos [-p pid] [-o executable] [-f file] [-s slide | -l loadAddress] [-arch architecture] [-printHeader] [address ...]
 
atos -arch arm64 -o +-ﾅK.app.dSYM/MeituanMovie.app.dSYM/Contents/Resources/DWARF/MeituanMovie -l 0x100070000 0x0000000101c6768c
or 
atos -arch arm64 -o +-ﾅK.app.dSYM/MeituanMovie.app.dSYM/Contents/Resources/DWARF/MeituanMovie -l 0x100070000
再输入对应的偏移量查询 0x0000000101c6768c
```

```
dwarfdump -–lookup 0x000036d2 -–arch armv6 xxx.app.dSYM
```

## symbolicatecrash
 
```
 ./symbolicatecrash ./MeituanMovie\ \ 2017-3-13\ 上午11-15.crash  ./imovie910.app.dSYM/MeituanMovie.app.dSYM >symbol.crash
```
如果出现`Error: "DEVELOPER_DIR" is not defined at ./symbolicatecrash line`错误，输入以下命令
`export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`

使用命令查找 symbolicatecrash 
`find /Applications/Xcode.app -name symbolicatecrash -type f`
`/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash`
 
 
## 查看UUID
dwarfdump --uuid xxx.app/xxx (xxx工程名)
dwarfdump --uuid xxx.app.dSYM (xxx工程名)
```
 dwarfdump --uuid +-ﾅK.app.dSYM/MeituanMovie.app.dSYM/Contents/Resources/DWARF/MeituanMovie 
```
 



