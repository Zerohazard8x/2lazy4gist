## Pro Tips

-   **Using `%` with `xargs`**  
    When you need to use `xargs -I%` inside single quotes, you can escape the `%` character as follows:
    ```bash
    '"%"'
    ```

This allows you to embed `xargs -I%` commands without breaking out of single quotes.

---

## Regular Expressions

-   **Match every line**

    ```regex
    ^.*$
    ```

    or, with multiline mode enabled:

    ```regex
    /^.*$/gm
    ```

-   **Match lines that do _not_ contain `twitch.tv`, `youtube.com`, or `youtube-nocookie.com`**

    ```regex
    /^((?!twitch\.tv|youtube\.com|youtube-nocookie\.com).)*$/
    ```

---

## Example of `tr`, `nl`, and I/O Redirection

-   Replace spaces with underscores, number each line, and read from standard input:

    ```bash
    cat file | tr ' ' _ | nl
    ```

    This can be rewritten (more idiomatically) using input redirection:

    ```bash
    <file tr ' ' _ | nl
    ```

---

## ExifTool

Copy metadata **from one image** to another:

```bash
exiftool -TagsFromFile src.jpg -all:all dst.jpg
```

Make a **copy of an image without any metadata**:

```bash
exiftool -all= dst.jpg -o output.jpg
```

---

## Vim

1. Open (or create) your Vim configuration file:

    ```vim
    :w $HOME/_vimrc       " Windows
    :w $HOME/.vimrc        " Unix-like systems
    ```

2. Re-open the same file to edit it:

    ```vim
    :e $HOME/.vimrc
    ```

3. Save or save and quit:

    ```vim
    :w
    :wq
    ```

4. To view or set the GUI font:

    ```vim
    :set guifont=*
    :set guifont?
    ```

5. Press `I` (capital i) to enter **Insert Mode**.
6. Paste the following snippet into your `.vimrc` (this sets a font and colorscheme when running in a GUI):

    ```vim
    if has("gui_running")
      set guifont=IBM_Plex_Mono:h12
      colorscheme habamax
    endif
    ```

---

## Neovim – LazyVim

1. In Neovim, write to the `colorscheme.lua` file inside your config directory. To find that path:

    ```vim
    :echo stdpath('config')
    ```

    Then:

    ```vim
    :w <path-to-config>/lua/plugins/colorscheme.lua
    ```

2. Re-open the same file to edit:

    ```vim
    :e <path-to-config>/lua/plugins/colorscheme.lua
    ```

3. Save or save and quit:

    ```vim
    :w
    :wq
    ```

4. Paste the following Lua snippet to add and configure Gruvbox for LazyVim:

    ```lua
    return {
      -- Add gruvbox plugin
      { "ellisonleao/gruvbox.nvim" },

      -- Configure LazyVim to load gruvbox
      {
        "LazyVim/LazyVim",
        opts = {
          colorscheme = "gruvbox",
        },
      },
    }
    ```

---

## Linux / Bash

-   **Find lines in `file1` that are _not_ in `file2`**:

    ```bash
    grep -Fxvf file1 file2 > file3
    ```

-   **Remove all lines containing a pattern**:

    ```bash
    awk '!/[something]/' file > file2
    ```

-   **Copy a file’s timestamp (modify time)**:

    ```bash
    touch -r [source] [target]
    ```

-   **Merge two files line by line, separating with a space**:

    ```bash
    paste -d ' ' 1.txt 0.txt
    ```

-   **Remove empty lines**:

    ```bash
    awk 'NF' input.txt
    ```

-   **Sort lines and remove duplicates**:

    ```bash
    sort /tmp/uniques.txt | uniq
    ```

---

## Compression & Archives

-   **Extract a `.pkg` file**:

    ```bash
    xar -xvf [file.pkg] -C [output-directory]
    ```

-   **Extract a `.tar.gz` file**:

    ```bash
    tar -xvf [file.tar.gz] -C [output-directory]
    ```

-   **Decompress an `.lz4` file**:

    ```bash
    lz4 -d [input.lz4] [output]
    ```

-   **7-Zip compression tips**

    > _Pro Tip: If you don’t know the maximum `-mx` value, just put `-mx=100`._

    -   **LZMA (Max compression)**:

        ```bash
        7z a -mm=LZMA -mmt=on -mx=9 ../grades.7z *
        ```

    -   **Lizard (optimized LZ4; very fast)**:

        ```bash
        7z a -mm=Lizard -mmt=on ../grades.7z *
        ```

        -   Lizard is an optimized version of LZ4.
            GitHub: [https://github.com/inikep/lizard](https://github.com/inikep/lizard)

    -   **Deflate (classic ZIP; fastest and most universal)**:

        ```bash
        7z a -mm=Deflate -mmt=on ../archive.zip *
        ```

    -   **Create a tar archive without compression**:

        ```bash
        7z a -mmt=on ../archive.tar *
        ```

        -   Packs multiple files/folders into a single `.tar` file but does **not** compress.

    -   **Compute a SHA-256 hash of a file**:

        ```bash
        7z h -scrcsha256 [input-file]
        ```

    -   **Find and delete empty directories**:

        ```bash
        find . -type d -empty -delete
        ```

---

## Mounting Filesystems

-   **Mount a device** (e.g., `/dev/sdb1`):

    ```bash
    sudo mount /dev/sdb1 /mnt/media
    ```

-   **Remount an already-mounted directory as read-write, with `exec` and `dev` options**:

    ```bash
    sudo mount -o rw,exec,dev,remount /mnt/media
    ```

-   **Remount as read-only**:

    ```bash
    sudo mount -o ro,remount /mnt/media
    ```

-   **Unmount a target**:

    ```bash
    sudo umount /mnt/media
    ```

---

## Disk Imaging & Cloning

### `dd`

-   **Basic `dd` copy (block size 2048 bytes, show progress)**:

    ```bash
    sudo dd if=[source] of=[target] bs=2048 status=progress
    ```

-   **Specify block size and count**:

    ```bash
    sudo dd if=[source] of=[target] bs=[blocksize] count=[count] status=progress
    ```

### `ddrescue`

1. **First pass (no retry, sparse copy, block size 2048)**:

    ```bash
    sudo ddrescue -b 2048 -n -v [source] [target.iso] output.log
    ```

2. **Second pass (retry 3 times, fill gaps, block size 2048)**:

    ```bash
    sudo ddrescue -b 2048 -d -r 3 -v [source] [target.iso] output.log
    ```

3. **Third pass (reverse direction retry 3 times, block size 2048)**:

    ```bash
    sudo ddrescue -b 2048 -d -R -r 3 -v [source] [target.iso] output.log
    ```

> **Important:** Using a block size of `1M` and a count of `9` clones the first 9 MB of data:
>
> ```bash
> sudo dd if=[source] of=[target] bs=1M count=9
> ```

-   **Useful reference for factors/calculators:**
    [https://www.calculatorsoup.com/calculators/math/factors.php](https://www.calculatorsoup.com/calculators/math/factors.php)

---

## Partition Table Cloning (`sfdisk`)

-   **Dump partition table of a source disk to a file**:

    ```bash
    sfdisk -d [source-disk] > part_table
    ```

-   **Apply that partition table to a target disk**:

    ```bash
    sfdisk [target-disk] < part_table
    ```

    -   This clones the partition layout without copying the actual data.

-   **Copy the contents of a partition literally** (using `pv` and a pipe):

    ```bash
    pv /dev/sda1 > /dev/sdb1
    ```

-   **Remove the `part_table` file when done**:

    ```bash
    rm -rfv part_table
    ```

---

## File Synchronization (`rsync`)

-   **Basic archive sync with progress**:

    ```bash
    rsync -a --progress [source-dir] [destination-dir]
    ```

-   **Additional `rsync` flags**:

    -   `--remove-source-files` – Delete source files after transfer
    -   `--max-size=100M` – Skip files larger than 100 MB

---

## Command Pipelines & Parallel Processing

-   **Use `pv` to show progress (with size), pipe into `xargs` for parallel execution**:

    ```bash
    cat <someinput> | pv -p -s sizeof_someinput | xargs -0 -n 1 -P 5 <somecmd>
    ```

    -   `-p` – Show percentage progress
    -   `-s` – Total size for progress calculation
    -   `-0` – Expect NUL-delimited input
    -   `-n 1` – One argument per command execution
    -   `-P 5` – Run up to 5 processes in parallel

---

## Text Splitting with `sed`

-   **Split a large file (`after.reg`) into four parts**:

    ```bash
    sed -n '1,1000000p' ./after.reg   > after1
    sed -n '1000000,2000000p' ./after.reg > after2
    sed -n '2000000,3000000p' ./after.reg > after3
    sed -n '3000000,$p' ./after.reg   > after4
    ```

---

## Directory & File Structure Operations

-   **Copy folder structure only (create empty directories)**:

    ```bash
    find . -type d -print0 > dirs
    xargs -0 mkdir -p < dirs
    ```

-   **List all files with full paths**:

    ```bash
    find "$(cd .; pwd)" -type f
    ```

---

## File Splitting & Reassembly

-   **Split a large file into 4000 MB chunks with a `.7z.` prefix**:

    ```bash
    split -b 4000M [input-file] [outputname].7z.
    ```

-   **Reassemble split chunks (for a `.7z` archive)**:

    ```bash
    cat outputname.7z.a{a..g} > [reconstructed-file].7z
    ```

---

## Filesystem Checks

-   **Check (and repair) an ext family filesystem**:

    ```bash
    e2fsck /dev/sd*
    ```

    -   Similar to Windows’ CHKDSK but for Linux ext partitions.

-   **Check (and repair) an HFS (macOS) filesystem**:

    ```bash
    fsck_hfs -fy -x /dev/rdisk0s2
    ```

---

## Networking

-   **List PCI devices, filter for “network”**:

    ```bash
    lspci | grep -i network
    ```

-   **Show network interfaces**:

    ```bash
    ifconfig
    ```

-   **Force unmount (kill processes using the mount point)**:

    ```bash
    fuser -km /dev/sd*
    ```

---

## SSH Installation (Termux)

-   By default, SSH listens on port **8022** for Termux installations (standard SSH uses port 22).

---

## Windows Commands

-   **Apply a Windows image (`.wim`) from `install.wim`**:

    ```powershell
    imagex.exe /apply d:\install.wim 1 c:\
    ```

-   **Create a directory junction (symbolic link) to a folder on another drive**:

    ```powershell
    mklink /J [new-folder] [original-folder]
    ```