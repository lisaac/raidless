# raidless

A script to manage btrfs subvolume snapshot and snapraid.

# usage

有一个 snapraid-btrfs 脚本，使用 btrfs 快照进行 snapraid，保证数据可靠性。
使用前提：
    - 所有 data 和 parity 必须使用 btrfs 或 subvolume.
    - 所有 content 全部保存在 data 或 pariyt 内。

### raidless init

为所有 data 和 parity 创建 名为 ".snapraidshots" 的 subvolume ，用于存放所有的 snapshots。

### raidless sync

  执行步骤：
  0. 执行 snapraid diff 确定是否需要 sync，如果存在 -f 选项，强制 sync

1. 对 data 创建可写 snapraidshots 快照
2. 修改 snapraid.conf 中的 data 和 `content` 为 snapraidshots 路径
3. 执行 snapraid sync 操作
4. 完成之后，对 parity 创建 只读 snapraidshots 快照
5. 对之前可些的 data snapraidshots 设置成 read-only
6. 复制 snapraidshots 中的 content 到 data

### readless info

  打印 snapraidshots 信息，第一列为 SN

### raidless fix

- 默认直接使用 snapraidshots 中的快照直接进行fix，也就是简单的 reflink cp 操作，可一使用 -n 指定 SN 编号：
- 可以使用 --snapraid 使用 snapraid array 来进行回复，一般用于 快照（磁盘） 损毁的时候，也可以使用 -n 来指定 SN 编号。
- 支持 -m 只恢复被删除的文件和目录

  raidless fix -f aa/bb/c -n 2 # 从 SN 编号为 2 的 snapraidshots 恢复（reflink cp）aa/bb/c
  raidless fix -d disk1 -n 3 # 从 SN 编号为 3 的 snapraidshots 恢复（reflink cp）disk

  raidless fix --snapraid -f aa/bb/c -n 2 # 先用 snapraid 恢复 SN 编号为 2 的 snapraidshots 中的 aa/bb/c，再回复到 aa/bb/c

### raidless del

  raidless del -n 3 # 删除所有 sn 为 3 的 snapraidshots
  raidless del -d disk1 -n 4 # 删除名字为disk1的 sn 为 4 的 snapraidshots

### raidless flush

  raidless del -n 1,2,7-9 # 删除所有 sn 为 2,3,7,8,9 的 snapraidshots
