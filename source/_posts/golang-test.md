title: golang多个pkg的单元测试覆盖率
date: 2017-06-11 17:24:56
categories: Go
tags:

---
在go test命令后面添加 -cover参数开启测试覆盖率统计，其结果如下：
`ok      models  0.012s  coverage: 71.4% of statements in models`
-coverpkg 标记来指定要被统计的代码包之后，未被指定的代码则肯定不会被统计，即使是被直接测试的那个代码包。`go test -coverpkg=./... pkg2`可以跑pkg2下的所有单元测试及pkg2所用到的其他包的覆盖率情况。
但是由于go不支持`go test -coverpkg=./... ./...`如果我们有多个pkg，则无法一次性统计出所有的测试覆盖率和跑完全部单元测试。所以需要挨个跑完单元测试然自己来合并覆盖率的结果：

```golang
src
├── pkg1
│   ├── pkg11
│   └── pkg12
└── pkg2
    ├── pkg21
    └── pkg22
    
go test -coverprofile=pkg1.cover.out -coverpkg=./... pkg1
go test -coverprofile=pkg1.cover.out -coverpkg=./... pkg2
```
-coverprofile用来指定统计测试覆盖率信息的输出路径，其内容如下：

```
mode: set
models/bot.go:32.40,46.18 12 1
models/bot.go:49.2,57.35 5 1
...
```
第一行是测试覆盖的mode，可取值为：set,count,atomic。剩下的行遵循以下的格式：
name.go:line.column,line.column numberOfStatements count
所以对每个pkg跑完单元测试后可以用以下的命令来合并：
```
echo "mode: set" > coverage.out && cat *.cover.out | grep -v mode: | sort -r | \
awk '{if($1 != last) {print $0;last=$1}}' >> coverage.out
```
然后用go自带的工具来将其输出为HTML：
`go tool cover -html=coverage.out -o cover.html`
把整个流程串起来写成脚本如下：
```
#!/bin/bash

set -e

profile="cover.out"
htmlfile="cover.html"
mergecover="merge_cover"
mode="set"

for package in $(go list ./...|grep -v src); do
    coverfile="$(echo $package | tr / -).cover"
    go test -covermode="$mode" -coverprofile="$coverfile" -coverpkg="$package" "$package"
done

# merge all profiles
grep -h -v "^mode:" *.cover | sort > $mergecover

# aggregate duplicated code-block data
echo "mode: $mode" > $profile
current=""
count=0
while read line; do
    block=$(echo $line | cut -d ' ' -f1-2)
    num=$(echo $line | cut -d ' ' -f3)
    if [ "$current" == "" ]; then
        current=$block
        count=$num
    elif [ "$block" == "$current" ]; then
        count=$(($count + $num))
    else
        echo $current $count >> $profile
        current=$block
        count=$num
    fi
done < $mergecover

if [ "$current" != "" ]; then
    echo $current $count >> $profile
fi

# save result
go tool cover -html=$profile -o $htmlfile
go tool cover -func=$profile

```