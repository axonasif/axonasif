---
title: vivaldi-managed-policies-macos
date: 2026-02-16
tags:
  - seed
  - guide
---
# Locking Down Vivaldi Settings on macOS with Managed Policies

You can use macOS managed preferences (the same mechanism used for enterprise/MDM deployments) on a personal Mac to lock Vivaldi browser settings — preventing changes to DNS-over-HTTPS configuration and force-installing extensions that can't be removed.

## How It Works

Chromium-based browsers (including Vivaldi) read policy files from `/Library/Managed Preferences/`. When a policy is set there, the corresponding setting in the browser UI becomes grayed out and shows "managed by your organization." Since the plist file is owned by root, it can't be modified without `sudo`.

## Creating the Policy File

Write the plist directly using `sudo tee`:

```bash
sudo mkdir -p /Library/Managed\ Preferences

sudo tee /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DnsOverHttpsMode</key>
    <string>secure</string>
    <key>DnsOverHttpsTemplates</key>
    <string>https://your-doh-provider.example/dns-query</string>
    <key>ExtensionInstallForcelist</key>
    <array>
        <string>EXTENSION_ID_1;https://clients2.google.com/service/update2/crx</string>
        <string>EXTENSION_ID_2;https://clients2.google.com/service/update2/crx</string>
    </array>
</dict>
</plist>
EOF
```

Replace `EXTENSION_ID_1`, `EXTENSION_ID_2`, etc. with the actual extension IDs from their Chrome Web Store URLs, and replace the DoH template URL with your provider.

Set proper ownership:

```bash
sudo chown root:wheel /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
sudo chmod 644 /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
```

Then **flush the preferences cache** — this is the key step that makes changes take effect:

```bash
sudo killall cfprefsd
```

Restart Vivaldi and verify at `vivaldi://policy`.

## Policy Reference

**DnsOverHttpsMode** options:
- `secure` — only use DoH, fail if unavailable
- `automatic` — try DoH first, fall back to plain DNS
- `off` — disable DoH

**ExtensionInstallForcelist** format: each entry is `EXTENSION_ID;UPDATE_URL`. The update URL for Chrome Web Store extensions is always `https://clients2.google.com/service/update2/crx`. You can find the extension ID in its Chrome Web Store URL.

## Modifying the Policy Later

To surgically update a single key (e.g. adding extensions) without rewriting the whole file, use `plutil -replace`:

```bash
sudo plutil -replace ExtensionInstallForcelist -json '[
  "ext1_id;https://clients2.google.com/service/update2/crx",
  "ext2_id;https://clients2.google.com/service/update2/crx",
  "ext3_id;https://clients2.google.com/service/update2/crx"
]' /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
```

Always run `sudo killall cfprefsd` afterwards, then restart Vivaldi.

## Verifying

- `vivaldi://policy` — all policies should show Source: **Platform**, Level: **Mandatory**, Status: **OK**
- `vivaldi://extensions` — force-installed extensions show as managed with no remove button
- DoH setting in Vivaldi's privacy settings should be grayed out

## Removing the Policy

```bash
sudo rm /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
sudo killall cfprefsd
```

Restart Vivaldi.

## Gotchas

- **Don't use `defaults write`** for this — it's unreliable with managed preferences paths and can silently write to the wrong location.
- **Always run `sudo killall cfprefsd`** after any changes. macOS aggressively caches preferences, and without this, Vivaldi may see stale values even after a restart.
- **Binary conversion is optional** — XML plists work fine. You can convert with `sudo plutil -convert binary1 <path>` if you prefer.
- **Read the current state** anytime with `sudo plutil -p /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist`.
