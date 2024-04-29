# raidless

A script to manage btrfs subvolume snapshot and snapraid.

又一个 `snapraid-btrfs` 脚本，使用 `btrfs` 快照进行 `snapraid`，保证数据可靠性。

`snapraid` 在默认情况下会对多个 `data` 进行校验，计算出校验值，保存到 `parity` 中。想法非常好，但是存在一个问题，例如：你在上一次 `snapraid sync` 修改了一些文件，同时磁盘又不争气的坏掉了，我们可以使用 `snapraid fix` 来进行修复，但是修复会存在一些问题，你在上次 `snapraid sync` 之后已经修改过一些文件，有几率会导致恢复的时候校验出错，所以在这里引入 `btrfs snapshots` 非常有必要。从 snapshot 中恢复文件时，可以使用非常快速且节省空间的 reflink 拷贝方式。

# Depends
- btrfs-progs
- snapraid
- grep

# usage

使用前提：

  - snapraid 配置文件中的所有 `data` 和 `parity` 必须使用 `btrfs` 或 `subvolume`
  - snapraid 配置文件中的所有 `content` 全部保存在 `data` 或 `pariyt` 内

### raidless init

为所有 `data` 和 `parity` 创建 名为 `.snapraidshots` 的 `subvolume` 容器，用于存放所有的 `snapshots`

### raidless sync

执行后，raidless 将会执行以下步骤：
0. 执行 snapraid diff 确定是否需要 sync，如果存在 -F 选项，强制 sync
1. 对 data 创建可写 snapraidshots 快照
2. 修改 snapraid.conf 中的 data 和 `content` 为 snapraidshots 路径
3. 执行 snapraid sync 操作
4. 完成之后，对 parity 创建 只读 snapraidshots 快照
5. 对之前可些的 data snapraidshots 设置成 read-only
6. 复制 snapraidshots 中的 content 到 data

简言之，对数据区创建`快照` 后再进行 `snapraid sync`，同步完成后，再创建 `parity` `快照`，同一个sync 之内的所有快照，都有同一个`SN`，以便于今后恢复。

### readless info

打印 snapraidshots 信息，第一列为 SN
```
raidless info
disk1   /media/mc1t:
SN      ID      gen     cgen    top level       otime   uuid    path
-       --      ---     ----    ---------       -----   ----    ----
5       263     420858  420856  256             2024-04-22 18:41:55     1c5328f3-2820-0c46-9305-eb1711697e8f    .snapraidshots/5
7       265     420964  420963  256             2024-04-22 20:02:25     c0281da0-c027-1841-a76b-f06e89e7e58b    .snapraidshots/7
disk2   /media/mc2t:
SN      ID      gen     cgen    top level       otime   uuid    path
-       --      ---     ----    ---------       -----   ----    ----
5       263     1080    1079    256             2024-04-22 18:41:55     79db0116-eb93-7040-85a0-86576397cdd4    .snapraidshots/5
7       265     1086    1085    256             2024-04-22 20:02:25     b00db419-4ecb-b14d-9ffd-468f1772e518    .snapraidshots/7
parity  /media/wd4t/parity:
SN      ID      gen     cgen    top level       otime   uuid    path
-       --      ---     ----    ---------       -----   ----    ----
5       645     2707    2707    390             2024-04-22 18:42:36     142318de-2424-5944-b3dc-df01f98ca538    parity/.snapraidshots/5
7       647     2730    2730    390             2024-04-22 20:02:51     ac239a53-9992-ce4e-b4e8-b99bb3fb4eac    parity/.snapraidshots/7
  ```

### raidless fix

- 默认直接使用 snapraidshots 中的快照直接进行fix，也就是简单的 reflink cp 操作，可一使用 -n 指定 SN 编号：
- 可以使用 --snapraid 使用 snapraid array 来进行回复，一般用于 快照（磁盘） 损毁的时候，也可以使用 -n 来指定 SN 编号。
- 支持 -m 只恢复被删除的文件和目录
```
# 从 SN 编号为 2 的 snapraidshots 恢复（reflink cp）aa/bb/c
raidless fix -f aa/bb/c -n 2

# 从 SN 编号为 3 的 snapraidshots 恢复（reflink cp）disk
raidless fix -d disk1 -n 3

# 先用 snapraid 恢复 SN 编号为 2 的 snapraidshots 中的 aa/bb/c，再恢复到data中的 aa/bb/c
raidless fix --snapraid -f aa/bb/c -n 2
```
### raidless del
```
# 删除所有 sn 为 3 的 snapraidshots
raidless del -n 3

# 删除名字为disk1的 sn 为 4 的 snapraidshots
raidless del -d disk1 -n 4

# 删除所有 sn 为 1,2,7,8,9 的 snapraidshots
raidless del -n 1,2,7-9

# 删除名为disk2，sn 为 1,2,7,8,9 的 snapraidshots
raidless del -d disk2 -n 1,2,7-9 
```
### 其他 snapriad 命令

由于 raidless 实际是一个 snapraid 的透明包装，所以除了上面这些命令之外，其他 snapraid 命令也是支持的。

同时，还可以使用 -n 来指定 SN 来操作某个<SN>的，值得注意的是，snapraidshots 中的内容默认都是只读的，如果有需要，可以关掉只读，你需要明确知道你在做什么，再进行相关操作。

# btrfs 压缩

在我自己使用情况来看，启用压缩还是挺有必要的，可以较大情况的节约空间
```
# 用 zstd 方式给整个 subvolume 重新压缩，建议在操作此步骤之前备份
btrfs filesystem defragment -r -v -czstd /path/to/subvolume

# 将 subvolume 设置为 zstd 压缩
btrfs property set /path/to/subvolume compression zstd
```