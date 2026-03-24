---
title: Vivaldi (Chromium) managed policies on MacOS
date: 2026-02-16
tags:
  - seed
  - guide
---
# Locking Down Vivaldi Settings on macOS with Managed Policies

On macOS, the reliable way to enforce Vivaldi managed policies on a personal machine is **not** to drop a plist into `/Library/Managed Preferences/` by hand. That can appear to work briefly, but macOS may remove those preferences after a reboot if they are not backed by a real configuration profile.

The durable approach is to install a `.mobileconfig` profile whose payload type matches Vivaldi's bundle identifier: `com.vivaldi.Vivaldi`.

## The Important Discovery

If you manually place `com.vivaldi.Vivaldi.plist` in `/Library/Managed Preferences/` on a non-MDM personal Mac, Vivaldi can read it at first, but macOS may later clean it up as an orphaned managed preference.

That means:
- the browser may show the setting as managed temporarily
- the policy can disappear after restart
- the setup is not trustworthy for long-term enforcement

So if your goal is persistent browser lockdown, use a configuration profile.

## How It Works

Chromium-based browsers such as Vivaldi read managed policy values via macOS managed preferences. When those values are delivered by a configuration profile:
- the settings persist across reboots
- the corresponding UI in Vivaldi becomes grayed out
- `vivaldi://policy` shows them as platform-managed policies

The key detail is that the profile payload uses Vivaldi's bundle ID as its `PayloadType`:

```xml
<key>PayloadType</key>
<string>com.vivaldi.Vivaldi</string>
```

## Create the Configuration Profile

Save a file such as `vivaldi-policy.mobileconfig` with contents like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.vivaldi.Vivaldi</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.vivaldi.Vivaldi.policy</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-E5F6-7890-ABCD-EF1234567890</string>

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
    </array>
    <key>PayloadDisplayName</key>
    <string>Vivaldi Browser Policy</string>
    <key>PayloadIdentifier</key>
    <string>com.vivaldi.policy.profile</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>B2C3D4E5-F6A7-8901-BCDE-F12345678901</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

Replace these values before installing:
- `https://your-doh-provider.example/dns-query` with your DoH endpoint
- `EXTENSION_ID_1`, `EXTENSION_ID_2`, etc. with real Chrome Web Store extension IDs
- the example UUIDs with your own if you want a cleaner long-term profile identity

## Install the Profile

Open the file:

```bash
open vivaldi-policy.mobileconfig
```

Then approve it in:

**System Settings -> Privacy & Security -> Profiles**

Once installed, macOS treats the policy as legitimate managed configuration instead of an orphaned plist.

If Vivaldi does not immediately reflect the profile after installation or after a profile change, flush the macOS preferences cache and then restart Vivaldi:

```bash
sudo killall cfprefsd
```

## Policy Reference

**DnsOverHttpsMode** options:
- `secure` - only use DoH, fail if unavailable
- `automatic` - try DoH first, fall back to plain DNS
- `off` - disable DoH

**ExtensionInstallForcelist** format:
- each entry is `EXTENSION_ID;UPDATE_URL`
- for Chrome Web Store extensions, the update URL is `https://clients2.google.com/service/update2/crx`

## Verifying

Check that the profile is installed:

```bash
sudo profiles list
```

Then verify inside Vivaldi:
- `vivaldi://policy` - policies should show Source: **Platform**, Level: **Mandatory**, Status: **OK**
- `vivaldi://extensions` - force-installed extensions should appear managed with no remove button
- Vivaldi's DNS-over-HTTPS setting should be grayed out

## Updating the Policy Later

Edit the `.mobileconfig` file, then remove and reinstall the profile so macOS picks up the new payload cleanly.

If you are iterating on values, treat the profile file as the source of truth rather than trying to patch `/Library/Managed Preferences/` directly.

After reinstalling or changing the profile, it is still worth flushing the preferences cache before reopening Vivaldi:

```bash
sudo killall cfprefsd
```

## Removing the Policy

Go to:

**System Settings -> Privacy & Security -> Profiles**

Select **Vivaldi Browser Policy** and remove it.

If Vivaldi still shows stale policy state immediately afterwards, run:

```bash
sudo killall cfprefsd
```

## Gotchas

- **Do not rely on a hand-written plist in `/Library/Managed Preferences/` for persistence.** It may work temporarily, then disappear after reboot.
- **The payload must target `com.vivaldi.Vivaldi`.** That is what Vivaldi reads as its managed preferences domain.
- **`vivaldi://policy` is your source of truth.** If the browser does not show the policy there, macOS delivery is not set up correctly.
- **A configuration profile beats manual filesystem tricks.** The problem is not just file ownership or plist syntax; it is that macOS wants managed preferences to come from a real profile authority.
- **`cfprefsd` caching still matters.** Even with the correct profile in place, macOS can serve stale preference data until you run `sudo killall cfprefsd`.

## Old Manual Plist Method

The older direct-plist approach is still useful as a reference and can help for short-lived testing, but it should not be treated as persistent on a personal non-MDM Mac.

### Creating the Policy File Manually

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

Replace `EXTENSION_ID_1`, `EXTENSION_ID_2`, and the DoH template URL with your real values.

Set ownership and permissions:

```bash
sudo chown root:wheel /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
sudo chmod 644 /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
```

Then flush the preferences cache:

```bash
sudo killall cfprefsd
```

Restart Vivaldi and verify at `vivaldi://policy`.

### Modifying the Manual Plist Later

To surgically update a single key without rewriting the whole file, use `plutil -replace`:

```bash
sudo plutil -replace ExtensionInstallForcelist -json '[
  "ext1_id;https://clients2.google.com/service/update2/crx",
  "ext2_id;https://clients2.google.com/service/update2/crx",
  "ext3_id;https://clients2.google.com/service/update2/crx"
]' /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
```

Always run this afterwards:

```bash
sudo killall cfprefsd
```

### Reading or Removing the Manual Plist

Read the current state:

```bash
sudo plutil -p /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
```

Remove it:

```bash
sudo rm /Library/Managed\ Preferences/com.vivaldi.Vivaldi.plist
sudo killall cfprefsd
```

Again, the key limitation of this method is persistence: macOS may clean up the file after restart if no backing configuration profile exists.
