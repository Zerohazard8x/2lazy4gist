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
#!/usr/bin/env bash
set -euo pipefail

dest="../destFolder"
mkdir -p "$dest"

# Loop over Firefox profile directories (handles spaces)
shopt -s nullglob
for profile in *"Firefox profile"*/ ; do
  [[ -d "$profile" ]] || continue
  echo "Processing profile: $profile"

  # 1) Move the entire 'extensions/' directory
  if [[ -d "$profile/extensions" ]]; then
    mv -fv "$profile/extensions" "$dest/"
  fi

  # 2) Move extension metadata JSONs + prefs.js
  for f in extensions.json extension-preferences.json extension-settings.json cookies.sqlite places.sqlite addonStartup.json.lz4 user.js; do
    if [[ -f "$profile/$f" ]]; then
      mv -fv "$profile/$f" "$dest/"
    fi
  done

  # 3) Move 'browser-extension-data/'
  if [[ -d "$profile/browser-extension-data" ]]; then
    mv -fv "$profile/browser-extension-data" "$dest/"
  fi

  # 4) Move storage/default
  if [[ -d "$profile/storage/default" ]]; then
    mv -fv "$profile/storage/default" "$dest/"
  fi
done

echo "Done"

```