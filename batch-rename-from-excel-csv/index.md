# 用 Excel 或 CSV 批量重命名文件：三种能落地的做法


# 用 Excel 或 CSV 批量重命名文件：三种能落地的做法

整理照片、扫描件、课程视频、发票的时候，最常见的处境是：**你手上已经有一张表了**——A 列是现在的文件名，B 列是你想要的文件名，可能还有 C 列是拍摄日期、D 列是客户编号。真正的问题不是"怎么想出新名字"，而是"这张表怎么变成磁盘上的实际改名"。

下面三种做法按"能自己控制的程度"从高到低排，都实测可用。

## 前提：先把对照表做对

不管用哪种方法，先在一张表里做三件事：

1. **A 列放旧文件名，必须和磁盘上完全一致**，包括扩展名。差一个空格就匹配不上。
2. **B 列放新文件名，也带扩展名。** 只写"IMG_001"不写".jpg"，改完文件就打不开了。
3. **C 列放一个"是否冲突"的检查**，比如在 Excel 里用 `=COUNTIF(B:B,B2)>1` 找出重复的新名字。两个文件改成同一个名字，第二个会直接覆盖第一个，而且**没有回收站**。

导出成 CSV 时选 **UTF-8** 编码。Windows 的 Excel 默认导出 GBK，中文文件名到脚本里就变成乱码——这是第一个大坑。

## 做法一：PowerShell（Windows，最稳）

假设 `rename.csv` 和要改的文件在同一个目录：

```powershell
Import-Csv .\rename.csv | ForEach-Object {
    $old = Join-Path $PWD $_.old
    $new = Join-Path $PWD $_.new
    if (Test-Path -LiteralPath $old) {
        Rename-Item -LiteralPath $old -NewName $_.new
    } else {
        Write-Warning "找不到: $old"
    }
}
```

几个必须注意的点：

- `-LiteralPath` 不能省。文件名里的 `[` `]` 在 PowerShell 里是通配符，用 `-Path` 会匹配失败。
- **先跑一遍空转**：把 `Rename-Item` 换成 `Write-Output "$old -> $new"`，确认每一条都对得上，再真跑。
- CSV 的列名要和 `$_.old` / `$_.new` 对上。用 `Import-Csv | Get-Member` 看一眼实际列名。

## 做法二：shell 循环（macOS / Linux）

同样的 CSV，用 `while read` 逐行处理：

```bash
while IFS=, read -r old new; do
  if [ -f "$old" ]; then
    mv -- "$old" "$new"
  else
    echo "missing: $old" >&2
  fi
done < rename.csv
```

坑在这里：

- `IFS=,` 只适合"文件名里没有逗号"的表。文件名带逗号的话，老老实实用带引号的 CSV 解析器，或者干脆把分隔符换成 Tab。
- `mv --` 的 `--` 是必须的：文件名以 `-` 开头时，不加它 `mv` 会把它当成参数。
- **`read` 会吃掉行尾空格**，如果你的文件名真有尾随空格（从 Windows 拷过来的文件经常有），得改用 `read -r` 加处理。
- macOS 自带的 bash 3.2 没有 `mapfile`，别抄 Linux 上的写法。

## 做法三：不写脚本

如果只是偶尔改一次，或者规则本身很复杂（按拍摄日期重排、按 EXIF 时间、按内容 hash 排序），写脚本的成本可能比改一次文件还高。这种情况用图形工具更划算，代价是失去"脚本可复用"这一点。

我自己做 [massrename.com](https://massrename.com) 就是因为卡在这个缝里：会写脚本的人不需要工具，不会写的人写不出来。它的做法是把 CSV 当成规则源——[从 Excel 列表批量改名](https://massrename.com/guides/rename-files-from-excel-list) 和 [按 CSV 的列组合出新文件名](https://massrename.com/guides/rename-multiple-files-from-csv) 这两条路径都支持，改之前能实时看到全部结果。

## 三个一定会踩的坑

**① 覆盖是不可逆的。** 不管哪种方法，先备份，或者至少先做一次"只输出不改"的预演。改名脚本写错一个字符，几百个文件的名字就全乱了。

**② 排序依赖会让结果变成随机的。** 如果你的新名字里有 `001` `002` 这种序号，序号是按什么排的？文件名排序？修改时间？EXIF？在 Excel 里用 `=ROW()-1` 生成的序号是按**表格行序**排的，而表格行序是导入时的顺序——不是你以为的时间顺序。先在表里按正确的列排一次序，再生成序号。

**③ 跨盘、跨网络盘的操作不是改名。** 同一个磁盘内改名只是改目录项，瞬间完成；跨盘"改名"实际是复制 + 删除，几百 GB 的文件会跑很久，中途失败会留下半个文件。先在本地盘改完再拷。

---

如果这三条里你只记住一条：**先预演，再执行。** 所有批量操作的事故都发生在"我以为它只会改这一个"的那一刻。

