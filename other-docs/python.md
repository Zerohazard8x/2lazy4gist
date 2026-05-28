## PDF Rendering

### Convert PDF to images

Source: `convert_pdf_to_img.py.txt`

```python
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

Source: `python_set_operations.py.txt`

```python
# open the first text file and read its lines into a set
with open("file1.txt", "r") as f1:
    set1 = set(f1.readlines())

# open the second text file and read its lines into a set
with open("file2.txt", "r") as f2:
    set2 = set(f2.readlines())

# subtract the second set from the first set
set3 = set1 - set2

# intersection
# set3 = set1 & set2

# open a new text file and write the lines of the new set into it
with open("out.txt", "w") as out:
    for line in set3:
        out.write(line)
```

