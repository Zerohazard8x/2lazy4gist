# Flags - Edge

```
(The following for 2023 Edge revamp)
edge-rounded-containers
edge-visual-rejuv-rounded-tabs
edge-overlay-scrollbars-win-style
edge-minimal-toolbar
```

---
# Clean files - Chromium
```
for chromeFiles in $(find *chrome://version profile*/ -maxdepth 1)
do
mkdir -p ./destFolder/
echo $chromeFiles | grep -Ei network | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei login | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei vault | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei database | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei sync | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei account | xargs -I% mv -fv % ./destFolder/
echo $chromeFiles | grep -Ei extension | xargs -I% mv -fv % ./destFolder/
done
```

---
# Clean files - Firefox
```
for fireFiles in $(find *Firefox profile*/ -maxdepth 1)
do
mkdir -p ../destFolder/
echo $fireFiles | grep -Ei cookies | xargs -I% mv -fv % ../destFolder/
echo $fireFiles | grep -Ei addon | xargs -I% mv -fv % ../destFolder/
echo $fireFiles | grep -Ei extension | xargs -I% mv -fv % ../destFolder/
echo $fireFiles | grep -Ei storage | xargs -I% mv -fv % ../destFolder/
echo $fireFiles | grep -Ei settings | xargs -I% mv -fv % ../destFolder/
echo $fireFiles | grep -Ei prefs | xargs -I% mv -fv % ../destFolder/
done
```