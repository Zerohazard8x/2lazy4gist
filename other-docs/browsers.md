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

# Disable history - Firefox user.js

```js
// Permanent Private Browsing
user_pref("browser.privatebrowsing.autostart", true);

// Browsing history
user_pref("places.history.enabled", false);

// Form history
user_pref("browser.formfill.enable", false);

// Password saving
user_pref("signon.rememberSignons", false);

// URL bar
user_pref("browser.urlbar.suggest.history", false);
user_pref("browser.urlbar.suggest.bookmark", true);
user_pref("browser.urlbar.suggest.openpage", false);

// New Tab page
user_pref("browser.newtabpage.activity-stream.feeds.topsites", false);
user_pref("browser.newtabpage.activity-stream.feeds.section.highlights", false);

// Windows "Recent Documents"
user_pref("browser.download.manager.addToRecentDocs", false);
```

# "Stateless" browser - Firefox user.js

```js
/*** History ***/
user_pref("browser.privatebrowsing.autostart", true);
user_pref("places.history.enabled", false);
user_pref("browser.formfill.enable", false);
user_pref("signon.rememberSignons", false);

/*** URL bar ***/
user_pref("browser.urlbar.suggest.history", false);
user_pref("browser.urlbar.suggest.bookmark", true);
user_pref("browser.urlbar.suggest.openpage", false);

/*** New Tab page ***/
user_pref("browser.newtabpage.activity-stream.feeds.topsites", false);
user_pref("browser.newtabpage.activity-stream.feeds.section.highlights", false);

/*** Download history ***/
user_pref("browser.download.manager.addToRecentDocs", false);
user_pref("browser.download.lastDir.savePerSite", false);

/*** Cache ***/
user_pref("browser.cache.disk.enable", false);
user_pref("browser.cache.disk_cache_ssl", false);
user_pref("browser.cache.offline.enable", false);
user_pref("browser.cache.memory.enable", false);

/*** Session restore ***/
user_pref("browser.sessionstore.resume_from_crash", false);
user_pref("browser.sessionstore.max_resumed_crashes", 0);
user_pref("browser.sessionstore.max_tabs_undo", 0);
user_pref("browser.sessionstore.max_windows_undo", 0);
user_pref("browser.sessionstore.privacy_level", 2);
user_pref("browser.startup.page", 1);

/*** Per-site state ***/
user_pref("browser.zoom.siteSpecific", false);

/*** Background network activity ***/
user_pref("network.prefetch-next", false);

/*** Service Workers ***/
user_pref("dom.serviceWorkers.enabled", false);
```