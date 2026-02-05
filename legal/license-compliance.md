---
title: Open Source License Compliance Guide
description: Technical guidance for distributing Honeymelon (GPL v3) while fulfilling open-source licensing obligations.
---

# Open Source License Compliance Guide for Honeymelon

This document provides technical guidance for ensuring Honeymelon remains compliant with GPL v3 and all third-party open-source licenses.

---

## Architecture Overview

Honeymelon's licensing compliance is built on a fundamental architectural principle:

```
┌─────────────────────────────────────┐
│   Honeymelon Application            │
│   (GPL-3.0-or-later)                │
│                                     │
│   - Vue.js UI (MIT)                 │
│   - Tauri (MIT/Apache-2.0)          │
│   - TypeScript/Rust code (GPL v3)   │
│   - Job orchestration               │
└──────────────┬──────────────────────┘
               │
               │ Process spawn
               │ (No linking)
               ↓
┌─────────────────────────────────────┐
│   FFmpeg Binary                     │
│   (LGPL v2.1+)                      │
│                                     │
│   - Runs as separate process        │
│   - No shared memory                │
│   - No library linking              │
└─────────────────────────────────────┘

```

---

## GPL v3 + LGPL Compliance: Technical Implementation

### How Honeymelon Executes FFmpeg

**File**: runner modules under `src-tauri/src/runner`

```rust
// FFmpeg is spawned as a separate process
let child = Command::new(ffmpeg_path)
    .args(&args)
    .stdout(Stdio::piped())
    .stderr(Stdio::piped())
    .spawn()?;

```

This is **process execution**, identical to running a command in Terminal:

```bash
ffmpeg -i input.mp4 output.mp4

```

### GPL v3 Compatibility with LGPL FFmpeg

| Technical Approach                              | License Compatibility                          |
| ----------------------------------------------- | ---------------------------------------------- |
| Static linking to libavcodec, libavformat, etc. | Compatible - combined work is GPL v3           |
| Dynamic linking to .dylib files                 | Compatible - LGPL libraries with GPL main work |
| **Process execution (Honeymelon's approach)**   | **Compatible - separate works**                |

### Why Process Separation

**Process separation maintains clear license boundaries**:

- Honeymelon remains pure GPL v3 (no LGPL code mixed in)
- FFmpeg remains pure LGPL (can be replaced by users)
- Users can update FFmpeg independently
- No need to provide FFmpeg source code separately (readily available)

---

## File Distribution Requirements

### Required Files in App Bundle

```
Honeymelon.app/
└── Contents/
    ├── MacOS/
    │   └── Honeymelon              # Your app (GPL v3)
    ├── Resources/
    │   ├── LICENSE.txt             # ✅ REQUIRED: Honeymelon GPL v3 License
    │   ├── FFMPEG-LICENSE.txt      # ✅ REQUIRED: FFmpeg LGPL License
    │   ├── THIRD-PARTY-NOTICES.txt # ✅ REQUIRED: All dependencies
    │   └── bin/                     # Optional: Bundled FFmpeg
    │       ├── ffmpeg              # LGPL binary
    │       └── ffprobe             # LGPL binary
    └── Info.plist

```

### Automated Bundle Script

Add this script to ensure licenses are always included:

**File**: `scripts/bundle-licenses.sh`

```bash
#!/bin/bash
# Copy license files into macOS app bundle

APP_BUNDLE="$1"
RESOURCES="$APP_BUNDLE/Contents/Resources"

# Create Resources directory if it doesn't exist
mkdir -p "$RESOURCES"

# Copy license files
cp LICENSE "$RESOURCES/LICENSE.txt"
cp LICENSES/FFMPEG-LGPL.txt "$RESOURCES/FFMPEG-LICENSE.txt"
cp THIRD_PARTY_NOTICES.md "$RESOURCES/THIRD-PARTY-NOTICES.txt"

echo "✅ License files bundled successfully"

```

**Usage**:

```bash
chmod +x scripts/bundle-licenses.sh
./scripts/bundle-licenses.sh "src-tauri/target/release/bundle/macos/Honeymelon.app"

```

---

## Tauri Configuration for License Files

Update `src-tauri/tauri.conf.json` to include licenses in the bundle:

```json
{
  "bundle": {
    "resources": ["../LICENSE", "../LICENSES/FFMPEG-LGPL.txt", "../THIRD_PARTY_NOTICES.md"]
  }
}
```

This ensures license files are automatically included when running `npm run tauri:build`.

---

## In-App License Display

### Implementation Checklist

- [ ] Add "Licenses" menu item to "About" window
- [ ] Display LICENSE (GPL v3) for Honeymelon
- [ ] Display FFMPEG-LICENSE.txt (LGPL v2.1+)
- [ ] Display THIRD_PARTY_NOTICES.md
- [ ] Make it accessible via keyboard shortcut

### Example Implementation

**File**: `src/components/LicensesDialog.vue`

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { readTextFile } from '@tauri-apps/plugin-fs';
import { resolveResource } from '@tauri-apps/api/path';

const licenses = ref({
  honeymelon: '',
  ffmpeg: '',
  thirdParty: '',
});

onMounted(async () => {
  const licPath = await resolveResource('LICENSE.txt');
  const ffmpegPath = await resolveResource('FFMPEG-LICENSE.txt');
  const thirdPartyPath = await resolveResource('THIRD-PARTY-NOTICES.txt');

  licenses.value.honeymelon = await readTextFile(licPath);
  licenses.value.ffmpeg = await readTextFile(ffmpegPath);
  licenses.value.thirdParty = await readTextFile(thirdPartyPath);
});
</script>

<template>
  <div class="licenses-dialog">
    <h2>Honeymelon License (GPL v3)</h2>
    <pre>{{ licenses.honeymelon }}</pre>

    <h2>FFmpeg License (LGPL v2.1+)</h2>
    <pre>{{ licenses.ffmpeg }}</pre>

    <h2>Third-Party Notices</h2>
    <pre>{{ licenses.thirdParty }}</pre>
  </div>
</template>
```

---

## DMG Distribution

### What to Include in the DMG

```
Honeymelon_2.0.0_aarch64.dmg
├── Honeymelon.app              # Main application (GPL v3)
├── LICENSE.txt                 # Honeymelon GPL v3 License
├── FFMPEG-LICENSE.txt          # FFmpeg LGPL License
└── THIRD_PARTY_NOTICES.txt     # All dependencies

```

### DMG Build Script

**File**: `scripts/create-dmg-with-licenses.sh`

```bash
#!/bin/bash
# Create DMG with license files

VERSION="2.0.0"
DMG_DIR="dmg-staging"

# Clean and create staging directory
rm -rf "$DMG_DIR"
mkdir -p "$DMG_DIR"

# Copy app bundle
cp -r "src-tauri/target/release/bundle/macos/Honeymelon.app" "$DMG_DIR/"

# Copy license files to DMG root
cp LICENSE "$DMG_DIR/LICENSE.txt"
cp LICENSES/FFMPEG-LGPL.txt "$DMG_DIR/FFMPEG-LICENSE.txt"
cp THIRD_PARTY_NOTICES.md "$DMG_DIR/THIRD_PARTY_NOTICES.txt"

# Create DMG
hdiutil create -volname "Honeymelon" \
  -srcfolder "$DMG_DIR" \
  -ov -format UDZO \
  "Honeymelon_${VERSION}_aarch64.dmg"

echo "✅ DMG created with all license files"

```

---

## FFmpeg Build Considerations

### Recommended Configuration

To minimize patent and licensing concerns, build FFmpeg with:

```bash
./configure \
  --enable-videotoolbox \      # Hardware H.264/HEVC (Apple licensed)
  --enable-libopus \            # Patent-free audio
  --enable-libvpx \             # VP9 (open source)
  --disable-gpl \               # No GPL code
  --enable-version3 \           # LGPL v2.1+
  --arch=arm64 \                # Apple Silicon
  --prefix=/usr/local

```

### Codecs to Avoid (GPL/Patent Issues)

**DO NOT ENABLE**:

- `--enable-gpl` - Would make FFmpeg GPL, not LGPL
- `--enable-libx264` - GPL licensed
- `--enable-libx265` - GPL licensed
- `--enable-nonfree` - Proprietary codecs
- `--enable-libfdk-aac` - Non-free license

  **SAFE TO USE**:

- VideoToolbox (hardware, Apple licensed)
- Native AAC encoder (built-in)
- libvpx (VP9, open source)
- libopus (Opus, patent-free)
- libaom (AV1, open source)

---

## Compliance Verification

### Pre-Release Checklist

Before each release, verify:

```bash
# 1. Check that FFmpeg is not linked (optional verification)
otool -L Honeymelon.app/Contents/MacOS/Honeymelon
# Should NOT show any libavcodec, libavformat, etc.

# 2. Verify license files are in bundle
ls -la Honeymelon.app/Contents/Resources/
# Should show LICENSE.txt, FFMPEG-LICENSE.txt, etc.

# 3. Check DMG contents
hdiutil mount Honeymelon_2.0.0_aarch64.dmg
ls -la /Volumes/Honeymelon/
# Should show license files in DMG root

# 4. Verify FFmpeg runs as separate process
# Open Activity Monitor while converting
# Should see separate "ffmpeg" process
```

---

## Legal Q&A for Developers

### Q: Can I statically link FFmpeg with a GPL v3 application?

**A**: **YES**. GPL v3 is compatible with LGPL, so you could link FFmpeg libraries if needed. However, process separation is simpler and maintains clearer boundaries between Honeymelon code and FFmpeg.

### Q: What if I use FFmpeg libraries via FFI (Foreign Function Interface)?

**A**: This is dynamic linking and is fully compatible with GPL v3 + LGPL. Process separation is still preferred for simplicity.

### Q: Can I modify FFmpeg source code?

**A**: Yes. If you do, you MUST:

1. Provide modified FFmpeg source code to users
2. Document your modifications
3. Follow LGPL terms for the modified version

It's easier to NOT modify FFmpeg and use it as-is.

### Q: Can others redistribute Honeymelon commercially?

**A**: **YES**. GPL v3 explicitly allows commercial distribution. Anyone can:

- Download the source code
- Build and distribute binaries
- Charge money for distribution
- Modify and redistribute

They MUST provide source code and maintain GPL v3 licensing.

### Q: What about linking to other GPL/LGPL libraries?

**A**: GPL v3 is compatible with:
- LGPL v2.1, v3 (one-way compatibility)
- Apache License 2.0
- MIT License
- BSD licenses

All dependencies remain under their original licenses.

---

## Summary

**Honeymelon's GPL v3 compliance**:

1. ✅ Full GPL v3 license text in LICENSE file
2. ✅ Copyright headers in all source files
3. ✅ FFmpeg runs as separate process (LGPL v2.1+)
4. ✅ License files included with distribution
5. ✅ No FFmpeg source modifications
6. ✅ All dependencies GPL-compatible (MIT, Apache 2.0, ISC, BSD)

**To stay compliant**:

1. Always include LICENSE, FFMPEG-LICENSE.txt, and THIRD_PARTY_NOTICES.md
2. Maintain GPL v3 copyright headers in source files
3. Use bundle scripts to automate license inclusion
4. Display licenses in the app UI
5. Provide source code access (GitHub repository)

**GPL v3 Rights**:

Users can:
- Use Honeymelon for any purpose (commercial or personal)
- Study and modify the source code
- Redistribute original or modified versions
- Charge money for distribution

Users must:
- Provide source code with distributions
- Maintain GPL v3 licensing on derivatives
- Include copyright and license notices

---

**Questions?** See the [GNU GPL v3 FAQ](https://www.gnu.org/licenses/gpl-faq.html) or consult with a software licensing attorney.

**Last Updated**: 2025-01-15
