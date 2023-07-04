**_PROTIP_**: \'"%"\' allows xargs -I% to be used within single quotes

Regex all -> `^.*$ OR /^.*$/gm`
all that doesn't match "twitch.tv", "youtube.com", and "youtube-nocookie.com" -> `/^((?!twitch\.tv|youtube\.com|youtube-nocookie\.com).)*$/`

`cat file | tr ' ' _ | nl` -> `< file tr ' ' _ | nl`

---

# Exiftool
Copy metadata from 1 image to another<br>
`exiftool -TagsFromFile src.jpg -all:all dst.jpg`

Make a copy of an image, but without metadata<br>
`exiftool -all= dst.jpg -o output.jpg`

---

# Vim

1. `:w $HOME/_vimrc [Windows]` or `:w $HOME/.vimrc`
2. then `:e` same file (edit)
3. then `:w` / `:wq` to finish
4. `set guifont=*` -> `set guifont?`
5. press "I" for insert mode
6. Paste the following

```
if has ("gui_running")
	set guifont=IBM_Plex_Mono:h12
	colorscheme habamax
endif
```

---

# Neovim - Lazyvim

1. `:w` `lua/plugins/colorscheme.lua` in `:echo stdpath ('config')`
2. then `:e` same file (edit)
3. then `:w` / `:wq` to finish
4. Paste the following

```
return {
    -- add gruvbox
    { "ellisonleao/gruvbox.nvim" },
    -- Configure LazyVim to load gruvbox
    {
      "LazyVim/LazyVim",
      opts = {
        colorscheme = "gruvbox",
      },
    }
  }
```

---

# Linux/Bash

Find differences - `grep -Fxvf file1 file2 > file3`
</br>Remove all lines containing something - `awk '!/[something]/' file > file2`
</br>Copy timestamp = `touch -r [source] [target]`
</br>

`paste -d ' ' 1.txt 0.txt` -> merge 2 files line by line
</br>`awk NF [input txt]` -> remove empty lines
</br>`sort /tmp/uniques.txt | uniq` -> display 1 of each line

---

```
realesrgan-ncnn-vulkan.exe -i C:\Users\adria\Downloads\nihui_cat.png -o C:\Users\adria\Downloads\nihui_cat_lollypop.png -n 4x_NMKD-Siax_200k\esrgan-x4
```

---

**_SSH install_**
default port -> 8022 (termux only), normally 22

---

Extract .pkg - `xar -xvf [file] -C [output directory]`
</br>Extract .tar.gz - `tar –xvf [file] -C [output directory]`
</br>Extract .lz4 - `lz4 -d [input] [output]`

**_Protip: if you dont know the maximum mx value just put -mx100_**
</br>`7z a -mm=LZMA -mmt=on -mx9 ../grades.7z *`
</br>`7z a -mm=Lizard -mmt=on ../grades.7z *`

-   Fast compress (More than 3-4x faster??)
-   Lizard == optimized LZ4 (https://github.com/inikep/lizard)

`7z a -mm=Deflate -mmt=on ../archive.zip *`

-   Fastest compress (universal zip)

`7z a -mmt=on ../archive.tar *`

-   No compress (multiple files -> 1 file)

`7z h -scrcsha256 [input]` - Compute file hash

`find . -type d -empty` (`-delete` empty folders)

---

`sudo mount [e.g. /dev/sdb1] [target directory e.g. /mnt/media]`
</br>`sudo mount -o rw,exec,dev,remount [target directory]`
</br>`sudo mount -o ro,remount [target directory]`
</br>`sudo umount [target to unmount]`

---

`sudo dd if=[source] of=[target] bs=2048 status=progress`
`sudo dd if=[source] of=[target] bs=[depends] count=[also depends] status=progress`

1. `sudo ddrescue -b 2048 -n -v [source] [target e.g. iso] output.log`
2. `sudo ddrescue -b 2048 -d -r 3 -v [source] [target e.g. iso] output.log`
3. `sudo ddrescue -b 2048 -d -R -r 3 -v [source] [target e.g. iso] output.log`

**_Remember:_** 1M blocksize x 9 count = 9MB cloned

-   https://www.calculatorsoup.com/calculators/math/factors.php

---

`sfdisk -d [source] > part_table`
`sfdisk [target] < part_table`

-   Clone partitions without cloning data

`pv /dev/sda1 > /dev/sdb1`
`rm -rfv part_table`

---

`rsync –a --progress [src] [dst]`

-   `--remove-source-files`
-   `--max-size=100M`

---

`cat <someinput> | pv -p -s sizeof_someimput | xargs -0 -n 1 -P 5 <somecmd>`

---

```
sed -n 1,1000000p ./after.reg > after1
sed -n 1000000,2000000p ./after.reg > after2
sed -n 2000000,3000000p ./after.reg > after3
sed -n '3000000,$p' ./after.reg > after4
```

---

`find . -type d -print0 >dirs -> xargs -0 mkdir -p <dirs` (copy folder structure)
`find "$(cd .; pwd)" -type f`

---

```
split -b 4000M [input] [outputname].7z.
cat outputname.7z.a{a..g} > [output file].
```

---

`e2fsck /dev/sd*` - Chkdsk but for linux partitions
</br>`fsck_hfs -fy -x /dev/rdisk0s2` - Chkdsk but for mac hfs partitions

---

```
lspci | grep -i network
ifconfig
```

Force unmount - `fuser -km /dev/sd*`
`cp -rvT` [cp files without host folder]
`apt install fdisk parted gparted pv gcc-libs`

# Windows

`imagex.exe /apply d:\install.wim 1 c:\`
`mklink /J [new folder on a separate drive] [orig folder]`