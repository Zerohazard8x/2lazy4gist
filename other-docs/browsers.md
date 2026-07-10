# Clean files - Chromium

```sh
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

# Clean files - Firefox

```sh
#!/usr/bin/env bash
set -euo pipefail

# Start terminal in Firefox profile folder

dest="../destFolder"
mkdir -p "$dest"

# Loop over Firefox profile directories (handles spaces)
shopt -s nullglob
for profile in ./ ; do
    [[ -d "$profile" ]] || continue
    # 1) Move the entire 'extensions/' directory
    if [[ -d "$profile/extensions" ]]; then
        mv -fv "$profile/extensions" "$dest/"
    fi

    # 2) Move files
    for f in extensions.json extension-preferences.json extension-settings.json cookies.sqlite places.sqlite addonStartup.json.lz4 containers.json storage.sqlite user.js prefs.js; do
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
        mkdir -p "$dest/storage"
        mv -fv "$profile/storage/default" "$dest/storage/"
    fi
done

echo "Done"
```

# Enable clear on shutdown - Firefox user.js

```js
user_pref("privacy.sanitize.sanitizeOnShutdown", true);

user_pref("privacy.clearOnShutdown.history", true);
user_pref("privacy.clearOnShutdown.cache", true);
user_pref("privacy.clearOnShutdown.cookies", false);
user_pref("privacy.clearOnShutdown.formdata", false);
user_pref("privacy.clearOnShutdown.sessions", false);
user_pref("privacy.clearOnShutdown.siteSettings", false);
user_pref("privacy.clearOnShutdown.offlineApps", false);

user_pref("privacy.clearOnShutdown_v2.browsingHistoryAndDownloads", true);
user_pref("privacy.clearOnShutdown_v2.cache", true);
user_pref("privacy.clearOnShutdown_v2.cookiesAndStorage", false);
user_pref("privacy.clearOnShutdown_v2.formdata", false);
user_pref("privacy.clearOnShutdown_v2.siteSettings", false);
```