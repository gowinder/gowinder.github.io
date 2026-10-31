# Batch renaming files from an Excel or CSV list: three approaches that actually work


# Batch renaming files from an Excel or CSV list

If you are cleaning up photos, scans, lecture videos or invoices, the situation is usually the same: **you already have a table**. Column A is the current filename, column B is what you want it to be, maybe column C is a capture date and D is a client code. The hard part is not inventing new names. It is turning that table into actual renames on disk.

Three approaches, ordered from most control to least. All of them work.

## First, get the table right

Whatever method you use, do three things first:

1. **Column A must match the disk exactly**, extension included. One extra space and nothing will match.
2. **Column B must include the extension too.** Renaming `IMG_001.jpg` to `IMG_001` leaves you with a file no program can open.
3. **Add a collision check.** In Excel, `=COUNTIF(B:B,B2)>1` finds duplicate target names. Two files renamed to the same name means the second silently overwrites the first — and there is **no recycle bin** for that.

When exporting CSV, choose **UTF-8**. Excel on Windows defaults to the system codepage, and any non-ASCII filename turns into mojibake inside your script. That is trap number one.

## Approach 1: PowerShell (Windows, most reliable)

Assume `rename.csv` sits in the same folder as the files:

```powershell
Import-Csv .\rename.csv | ForEach-Object {
    $old = Join-Path $PWD $_.old
    $new = Join-Path $PWD $_.new
    if (Test-Path -LiteralPath $old) {
        Rename-Item -LiteralPath $old -NewName $_.new
    } else {
        Write-Warning "missing: $old"
    }
}
```

Details that matter:

- **`-LiteralPath` is not optional.** Brackets in a filename are wildcards to PowerShell; `-Path` will fail to match.
- **Dry run first.** Replace `Rename-Item` with `Write-Output "$old -> $new"` and check every line before the real pass.
- Column names must match `$_.old` and `$_.new`. Run `Import-Csv | Get-Member` to see what you actually got.

## Approach 2: a shell loop (macOS / Linux)

Same CSV, read line by line:

```bash
while IFS=, read -r old new; do
  if [ -f "$old" ]; then
    mv -- "$old" "$new"
  else
    echo "missing: $old" >&2
  fi
done < rename.csv
```

Traps:

- `IFS=,` only works if no filename contains a comma. If they do, use a real CSV parser or switch the delimiter to a tab.
- The `--` in `mv --` is required: a filename starting with `-` is otherwise read as a flag.
- `read` strips trailing whitespace. Files copied from Windows often have trailing spaces in their names.
- macOS ships bash 3.2, so no `mapfile`. Do not paste Linux recipes.

## Approach 3: no script at all

If this is a one-off, or the rule is genuinely complex — sort by capture date, by EXIF timestamp, by content hash — writing a script can cost more than doing it once by hand. A GUI tool wins there, at the price of losing reusability.

This gap is why I built [massrename.com](https://massrename.com): people who can write the script do not need a tool, and people who cannot, cannot. It treats a CSV as the rule source — both [renaming from an Excel list](https://massrename.com/guides/rename-files-from-excel-list) and [composing new names from CSV columns](https://massrename.com/guides/rename-multiple-files-from-csv) are supported, with a live preview of every result before anything moves.

## Three traps you will hit

**① Overwrites are irreversible.** Back up, or at minimum do a preview pass that writes nothing. One wrong character in a rename script can scramble hundreds of filenames.

**② Sorting assumptions make results random.** If your new names contain `001`, `002`, what determines that order? Filename sort? Modified time? EXIF? `=ROW()-1` in Excel numbers rows in **sheet order**, which is the import order — not the chronological order you assume. Sort the sheet by the right column *before* generating numbers.

**③ Renaming across volumes is not a rename.** Within one disk it is a directory entry change and is instant. Across disks it is copy + delete: hundreds of gigabytes take a long time, and a failure midway leaves half a file. Rename locally first, copy after.

---

If you remember one thing: **preview, then execute.** Every batch-operation disaster happens in the moment you were sure it would only touch that one file.

