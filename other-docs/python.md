## PDF Rendering

### Convert PDF to images

Source: `convert_pdf_to_img.py.txt`

``` py
import fitz
import os

# ---- configuration ----
pdf_path = "input.pdf"     # change this
out_dir = "output_images"  # change this if you want
zoom = 2                   # 2 is a decent default; bigger = higher resolution
image_prefix = "page_"     # output filenames: page_1.png, page_2.png, ...

# ---- open the pdf ----
doc = fitz.open(pdf_path)

# make sure the output folder exists
os.makedirs(out_dir, exist_ok=True)

# ---- render each page to an image ----
paths = []

for pno in range(doc.page_count):
    # get the page
    page = doc[pno]

    # scale up rendering (like your original Matrix(zoom, zoom) approach)
    mat = fitz.Matrix(zoom, zoom)

    # render page to a raster image (pixmap)
    pix = page.get_pixmap(matrix=mat, alpha=False)

    # save it
    out_path = os.path.join(out_dir, f"{image_prefix}{pno + 1}.png")
    pix.save(out_path)

    # keep track of output files
    paths.append(out_path)

# ---- done ----
doc.close()

# print the generated files
for p in paths:
    print(p)
```

---

## Set Operations

### Compare text files with sets

``` py
# Compare two text files using Python set operations
# and write the results to separate text files.

with open("file1.txt", "r", encoding="utf-8") as f1:
    set1 = set(f1.readlines())

with open("file2.txt", "r", encoding="utf-8") as f2:
    set2 = set(f2.readlines())

# Set operations
only_file1 = set1 - set2
only_file2 = set2 - set1
common = set1 & set2
differences = set1 ^ set2

# Write lines found only in file1
with open("only_file1.txt", "w", encoding="utf-8") as f:
    f.writelines(sorted(only_file1))

# Write lines found only in file2
with open("only_file2.txt", "w", encoding="utf-8") as f:
    f.writelines(sorted(only_file2))

# Write lines common to both files
with open("common.txt", "w", encoding="utf-8") as f:
    f.writelines(sorted(common))

# Write all differing lines
with open("differences.txt", "w", encoding="utf-8") as f:
    f.writelines(sorted(differences))
```

## Section comparer

### Compare text files with sets

``` py
from pathlib import Path
from sys import argv

# Read a text file and detect its encoding from its byte-order mark (BOM).
# Files without a recognized BOM are assumed to be UTF-8.
def read(path):
    data = Path(path).read_bytes()

    # UTF-16 may use either little-endian or big-endian byte order.
    # UTF-8 files may optionally begin with a BOM.
    encoding = 'utf-16' if data.startswith((b'\xff\xfe', b'\xfe\xff')) else 'utf-8-sig' if data.startswith(b'\xef\xbb\xbf') else 'utf-8'

    return Path(path).read_text(encoding=encoding), encoding

# argv[1] = original file
# argv[2] = file containing sections to subtract
# argv[3] = output file
first, encoding = read(argv[1])
second, _ = read(argv[2])

# Split the second file into sections separated by blank lines.
# A set allows fast lookups and automatically removes duplicates.
remove = set(second.strip('\n').split('\n\n'))

# Keep sections from the first file only if they have no exact match
# in the second file. Original section order is preserved.
result = '\n\n'.join(
    section
    for section in first.strip('\n').split('\n\n')
    if section not in remove
)

# Write the result
Path(argv[3]).write_text(result, encoding=encoding, newline='\r\n')
```