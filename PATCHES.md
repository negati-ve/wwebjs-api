# Patches for whatsapp-web.js

This document describes the patches and fork used for `whatsapp-web.js` to fix critical bugs that are not yet resolved in the official repository.

> **⚠️ IMPORTANT**: These are **temporary fixes** and should be removed once the official whatsapp-web.js repository releases versions that include these fixes.

## Current Setup

### Using Fork: `timothydillan/whatsapp-web.js#fix/duplicate-events-and-bindings`

Instead of using the official package with patches, we're using a fork that already includes the duplicate events fix.

**Why:** The duplicate events patch caused issues with session restoration, preventing the READY event from firing and blocking message reception.

### Patch File: `patches/whatsapp-web.js+1.34.3.patch`

This patch file is applied on top of timothydillan's fork to add the markedUnread fix.

---

## Fix 1: MarkedUnread Property Error

### Problem
When sending messages, the following error occurs:
```
Cannot read properties of undefined (reading 'markedUnread')
```

This happens because WhatsApp Web's internal structure changed, and the `chat.markedUnread` property may not exist in certain chat types (channels, status broadcasts, or newer WhatsApp versions).

### Solution
**File**: `node_modules/whatsapp-web.js/src/util/Injected/Utils.js`

**Changes**:
- Added check for `markedUnread` property existence before using it
- Detects special chat types (channels and status broadcasts)
- Implements multiple fallback layers:
  1. Try `sendSeen(chat)` if `markedUnread` exists and not a channel/status
  2. Fall back to `markSeen(chat)` if available
  3. Catch any errors and retry with `markSeen`
- Ensures stream is always marked unavailable using `finally` block

### Benefits
- Prevents crashes when sending messages
- Handles all chat types gracefully
- Provides robust error handling

### Upstream Reference
- Related issue: https://github.com/pedroslopez/whatsapp-web.js/issues
- Similar fix proposed by community developers

---

## Fix 2: Duplicate Events and Bindings (Via Fork)

### Problem
Multiple webhook events are being fired for the same action, particularly:
- Duplicate `authenticated` events
- Duplicate `ready` events
- Multiple event bindings causing repeated callbacks

This occurs because WhatsApp Web's `hasSynced` state can toggle multiple times (true → false → true), triggering the initialization flow repeatedly.

### Solution
**Using Fork**: `github:timothydillan/whatsapp-web.js#fix/duplicate-events-and-bindings`

Instead of patching the official version, we use @timothydillan's fork which includes:
1. Guard flags to prevent duplicate event listeners
2. Proper handling of restored sessions (important!)
3. Better CDP binding management after page navigation
4. Updated `sendSeen` format with `{chat, threadId}` structure

**Why Fork Instead of Patch?**
- The duplicate events patch caused the READY event to never fire on restored sessions
- This blocked all message reception
- Timothy's implementation handles session restoration correctly

### Benefits
- Prevents duplicate webhook calls
- Ensures clean single initialization
- Works correctly with session restoration
- Reduces unnecessary API calls to webhook endpoints

### Upstream Reference
- Fix by @timothydillan: https://github.com/timothydillan/whatsapp-web.js/tree/fix/duplicate-events-and-bindings
- Addresses duplicate event issues in WhatsApp Web 2.3000.x+

---

## Additional Fixes (Not in Patch File)

### Fix 3: CDN Request Error Log Filtering

**File**: `src/sessions.js` (local project file, not in patch)

**Changes**: Filters non-critical WhatsApp CDN message history download failures from error logs to debug level.

**Problem**: `ERR_ABORTED` errors from `cdn.whatsapp.net/mms/md-msg-hist` clutter logs during sync.

**Solution**: These expected failures are now logged at `debug` level instead of `error` level.

---

## How Patches Are Applied

Patches are automatically applied after every `npm install` via the `postinstall` script in `package.json`:

```json
"scripts": {
  "postinstall": "patch-package"
}
```

The `patch-package` tool reads patch files from the `patches/` directory and applies them to the installed packages in `node_modules/`.

---

## How to Remove Patches (When Official Fixes Are Released)

### Step 1: Check if Official Version Has Fixes

Monitor the official repository for fixes:
- https://github.com/pedroslopez/whatsapp-web.js/releases
- https://github.com/pedroslopez/whatsapp-web.js/commits/main

Look for commits or releases that mention:
- "markedUnread fix"
- "duplicate events fix"
- "duplicate READY events"
- Similar issues

### Step 2: Switch to Official Version

Once you confirm the official version includes the fixes:

```bash
# Install the official version
npm install whatsapp-web.js@1.35.0  # Replace with actual fixed version
```

Update `package.json` to remove the fork reference:
```json
{
  "dependencies": {
    "whatsapp-web.js": "^1.35.0"  // Instead of github:timothydillan/...
  }
}
```

### Step 3: Test and Remove Patches

```bash
# Test if the markedUnread issue is fixed
npm run start
# Send test messages

# If markedUnread is also fixed, delete the patch file
rm patches/whatsapp-web.js+1.34.3.patch

# If markedUnread still needs fixing, the patch will be reapplied automatically
# Just update the patch file name if the version changed
```

### Step 4: Test Thoroughly

After removing patches, test these scenarios:

1. **Message Sending**:
   ```bash
   # Send a message via API
   curl -X POST http://localhost:3000/client/sendMessage/YOUR_SESSION \
     -H "Content-Type: application/json" \
     -d '{"chatId": "1234567890@c.us", "contentType": "string", "content": "Test message"}'
   ```

2. **Check for Duplicate Events**:
   - Monitor your webhook endpoint
   - Verify only ONE `ready` event is received
   - Verify only ONE `authenticated` event per session initialization

3. **Check Logs**:
   - Ensure no `markedUnread` errors appear
   - Verify no excessive CDN error logs

### Step 5: Clean Up (Optional)

If patches are no longer needed and you want to remove `patch-package`:

```bash
# Remove patch-package from devDependencies
npm uninstall patch-package

# Remove postinstall script from package.json
# Edit package.json and remove: "postinstall": "patch-package"

# Remove patches directory
rm -rf patches/
```

---

## Verifying Patches Are Applied

To verify the patches are correctly applied:

```bash
# Check if patch file exists
ls -la patches/

# Verify patch-package is installed
npm list patch-package

# Check postinstall script
grep postinstall package.json

# Test by reinstalling dependencies
rm -rf node_modules
npm install

# Should see: "patch-package 8.0.1" output during install
```

---

## Troubleshooting

### Patch Fails to Apply

If you see errors during `npm install`:

```
**ERROR** Failed to apply patch for package whatsapp-web.js
```

**Causes**:
1. whatsapp-web.js version changed (no longer 1.34.4)
2. Official package already includes the fixes
3. File structure changed in the package

**Solutions**:
1. Check current installed version: `npm list whatsapp-web.js`
2. If version is different, regenerate patch:
   ```bash
   # Make manual changes to node_modules/whatsapp-web.js
   npx patch-package whatsapp-web.js
   ```
3. Or remove the patch and test if fixes are needed

### Dependencies Issues

If `patch-package` is not found:

```bash
npm install --save-dev patch-package
```

---

## Contributing

If you find issues with these patches or want to improve them:

1. Make changes to the appropriate files in `node_modules/whatsapp-web.js/`
2. Regenerate the patch:
   ```bash
   npx patch-package whatsapp-web.js
   ```
3. Test thoroughly
4. Commit the updated patch file

---

## References

- **whatsapp-web.js Official Repository**: https://github.com/pedroslopez/whatsapp-web.js
- **patch-package Documentation**: https://github.com/ds300/patch-package
- **Duplicate Events Fix Source**: https://github.com/timothydillan/whatsapp-web.js/tree/fix/duplicate-events-and-bindings

---

## Version History

| Date | Patch Version | whatsapp-web.js Version | Description |
|------|--------------|------------------------|-------------|
| 2026-01-28 | 1.0 | 1.34.4 | Initial combined patch (markedUnread + duplicate events) |

---

**Last Updated**: 2026-01-28  
**Maintainer**: Project Team  
**Status**: ⚠️ Active - Remove when upstream fixes are available
