# Fix Guide: Capacitor IAP Subscriptions Not Loading

## Apple Review Issue
**Guideline 2.1 - Performance - App Completeness**
- Device: iPad Air 11-inch (M3)
- OS: iPadOS 26.2
- Issue: App failed to load subscriptions

---

## Current Setup Analysis

**Your Stack:**
- ✅ Capacitor hybrid app
- ✅ CdvPurchase plugin (Cordova Purchase)
- ✅ Product IDs defined in code
- ✅ Comprehensive error logging
- ❌ Missing StoreKit configuration in Xcode (likely)
- ⚠️ Development server URL in production config

**Product IDs:**
```typescript
joyfulvoicesios001  // basic_monthly
joyfulvoicesios002  // premium_monthly
joyfulvoicesios003  // basic_yearly
joyfulvoicesios004  // premium_yearly
```

**Bundle ID:** `com.joyfjohnson.app`

---

## Fix Steps (In Order)

### Step 1: Verify Product IDs in App Store Connect ⚠️ CRITICAL

1. **Go to App Store Connect**
   - Navigate to your app
   - Go to **Features** → **In-App Purchases**

2. **Check EXACT Product IDs**
   - For each subscription, click to view details
   - **Product ID must match EXACTLY** (case-sensitive):
     - ✅ `joyfulvoicesios001`
     - ✅ `joyfulvoicesios002`
     - ✅ `joyfulvoicesios003`
     - ✅ `joyfulvoicesios004`

3. **Verify Product Status**
   - Each product should be **"Ready to Submit"** or **"Waiting for Review"**
   - NOT "Missing Metadata" or "Developer Action Needed"

4. **Common Mistakes:**
   - ❌ Extra spaces: `joyfulvoicesios001 ` (space at end)
   - ❌ Wrong prefix: `com.joyfjohnson.app.joyfulvoicesios001`
   - ❌ Capitalization: `JoyfulVoicesIOS001`

**If Product IDs don't match, you have two options:**
- **Option A:** Update App Store Connect to match code (easier)
- **Option B:** Update code to match App Store Connect (requires new build)

---

### Step 2: Add StoreKit Configuration File to Xcode ⚠️ REQUIRED

**Why:** Capacitor + CdvPurchase needs this file for StoreKit to load products properly.

#### A. Create StoreKit Configuration File

1. **Open your iOS project in Xcode:**
   ```bash
   cd ios/App
   open App.xcworkspace
   ```

2. **Create StoreKit Configuration:**
   - In Xcode: **File** → **New** → **File...**
   - Search for "StoreKit"
   - Select **StoreKit Configuration File**
   - Name it: `Products.storekit`
   - Save location: `ios/App/App/` (inside the App folder)

3. **Add Products to Configuration:**
   - Click the **+** button at the bottom
   - Select **Add Auto-Renewable Subscription**
   - Fill in details for each product:

   **Product 1: Basic Monthly**
   - Product ID: `joyfulvoicesios001`
   - Reference Name: `Joyful Voices Basic Monthly`
   - Price: Your monthly price (e.g., $9.99)
   - Subscription Duration: 1 Month

   **Product 2: Premium Monthly**
   - Product ID: `joyfulvoicesios002`
   - Reference Name: `Joyful Voices Premium Monthly`
   - Price: Your premium monthly price (e.g., $19.99)
   - Subscription Duration: 1 Month

   **Product 3: Basic Yearly**
   - Product ID: `joyfulvoicesios003`
   - Reference Name: `Joyful Voices Basic Yearly`
   - Price: Your yearly price (e.g., $99.99)
   - Subscription Duration: 1 Year

   **Product 4: Premium Yearly**
   - Product ID: `joyfulvoicesios004`
   - Reference Name: `Joyful Voices Premium Yearly`
   - Price: Your premium yearly price (e.g., $199.99)
   - Subscription Duration: 1 Year

#### B. Enable StoreKit Testing in Xcode Scheme

1. **Edit Scheme:**
   - In Xcode: **Product** → **Scheme** → **Edit Scheme...**
   - Or press: `Cmd + <`

2. **Configure StoreKit:**
   - Select **Run** in the left sidebar
   - Go to **Options** tab
   - Under **StoreKit Configuration**:
     - Select: `Products.storekit`

3. **Save and close**

---

### Step 3: Fix Capacitor Config for Production Build ⚠️ REQUIRED

Your current `capacitor.config.ts` has a development server URL that **breaks production builds**.

**Current (WRONG for production):**
```typescript
const config: CapacitorConfig = {
  appId: 'com.joyfjohnson.app',
  appName: 'joyfulvoices',
  webDir: 'dist',
  server: {
    url: 'https://8ac46971-2e8b-4942-8245-2b72ad9d4fe7.lovableproject.com?forceHideBadge=true',
    cleartext: true
  }
};
```

**Fix Option A - Comment Out for Production (Recommended):**
```typescript
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.joyfjohnson.app',
  appName: 'joyfulvoices',
  webDir: 'dist',
  // ⚠️ IMPORTANT: Comment out server config for production builds!
  // Only use this for local development with live reload
  // server: {
  //   url: 'https://8ac46971-2e8b-4942-8245-2b72ad9d4fe7.lovableproject.com?forceHideBadge=true',
  //   cleartext: true
  // }
};

export default config;
```

**Fix Option B - Environment-Based Config (Better):**
```typescript
import type { CapacitorConfig } from '@capacitor/cli';

// Only use dev server in development mode
const isDevelopment = process.env.NODE_ENV === 'development';

const config: CapacitorConfig = {
  appId: 'com.joyfjohnson.app',
  appName: 'joyfulvoices',
  webDir: 'dist',
  ...(isDevelopment && {
    server: {
      url: 'https://8ac46971-2e8b-4942-8245-2b72ad9d4fe7.lovableproject.com?forceHideBadge=true',
      cleartext: true
    }
  })
};

export default config;
```

**After changing:**
1. Save the file
2. Run: `npm run build` (or your build command)
3. Run: `npx cap sync ios`
4. Rebuild in Xcode

---

### Step 4: Verify CdvPurchase Plugin Installation

**Check if plugin is installed:**
```bash
npm list @awesome-cordova-plugins/purchase
```

**If not installed or outdated:**
```bash
npm install cordova-plugin-purchase --save
npm install @awesome-cordova-plugins/purchase --save
npx cap sync ios
```

**Verify in `package.json`:**
```json
{
  "dependencies": {
    "cordova-plugin-purchase": "^13.x.x",
    "@awesome-cordova-plugins/purchase": "^6.x.x"
  }
}
```

---

### Step 5: Enable In-App Purchase Capability in Xcode

1. **Open Xcode project:**
   ```bash
   cd ios/App
   open App.xcworkspace
   ```

2. **Select App target:**
   - Click on your app in the left sidebar (blue icon)
   - Select the **App** target

3. **Enable In-App Purchase:**
   - Go to **Signing & Capabilities** tab
   - Click **+ Capability**
   - Add **In-App Purchase**

4. **Verify Bundle ID:**
   - Under **General** tab
   - Bundle Identifier should be: `com.joyfjohnson.app`
   - Must match App Store Connect exactly

---

### Step 6: Rebuild and Test

#### A. Clean Build
```bash
# In your project root
rm -rf node_modules ios/App/Pods ios/App/build
npm install
npx cap sync ios
```

#### B. Build in Xcode
1. Open Xcode: `cd ios/App && open App.xcworkspace`
2. Select a real device (not simulator)
3. Clean build folder: **Product** → **Clean Build Folder** (Shift + Cmd + K)
4. Build: **Product** → **Build** (Cmd + B)
5. Fix any errors

#### C. Test on Device
1. Run on real iPad/iPhone (StoreKit doesn't work well in simulator)
2. Check Xcode console logs for:
   ```
   [AppleIAP] ✅ Products successfully loaded: 4
   ```
3. Navigate to subscription screen in app
4. Verify products display with prices

#### D. Test Purchase Flow (Sandbox)
1. **Create Sandbox Test Account:**
   - App Store Connect → Users and Access → Sandbox Testers
   - Create test account with unique email

2. **On Device:**
   - Settings → App Store → Sign Out (of real account)
   - Don't sign in yet (will prompt during purchase)

3. **In Your App:**
   - Tap a subscription
   - Sign in with sandbox account when prompted
   - Complete test purchase
   - Verify subscription activates

---

### Step 7: Verify App Store Connect Configuration

#### A. Check Paid Apps Agreement

1. **App Store Connect** → **Agreements, Tax, and Banking**
2. **Paid Apps Agreement** must be:
   - ✅ Active (green checkmark)
   - ✅ Latest version accepted
3. If not active, click to review and accept

#### B. Check Product Status

For each of the 4 products:
1. Status should be: **"Ready to Submit"** or **"Waiting for Review"**
2. All required metadata filled:
   - Display Name ✅
   - Description ✅
   - Review Screenshot ✅ (if required)

#### C. Check Subscription Group

1. All 4 products should be in the **same subscription group**
2. Subscription group should have:
   - Group Name (e.g., "Joyful Voices Subscriptions")
   - At least one localization (English)

---

## Testing Checklist Before Resubmitting

- [ ] Product IDs match exactly in code and App Store Connect
- [ ] All 4 products are "Ready to Submit" status
- [ ] StoreKit Configuration file created in Xcode
- [ ] StoreKit Configuration enabled in Xcode scheme
- [ ] In-App Purchase capability added in Xcode
- [ ] Development server URL removed from `capacitor.config.ts`
- [ ] Bundle ID matches: `com.joyfjohnson.app`
- [ ] Paid Apps Agreement is active
- [ ] CdvPurchase plugin installed and synced
- [ ] Clean build completed without errors
- [ ] Tested on real device (iPad if possible)
- [ ] Products load and display prices
- [ ] Test purchase works in Sandbox
- [ ] Subscription activates after purchase

---

## Expected Console Output (Success)

When everything works, you should see:
```
[AppleIAP] INITIALIZATION STARTED
[AppleIAP] Platform check - isNativeIOS: true
[AppleIAP] ✅ CdvPurchase plugin loaded successfully
[AppleIAP] Registering products with store...
[AppleIAP] ✅ Products registered
[AppleIAP] ✅ Store initialized
[AppleIAP] ✅ Store refresh completed
[AppleIAP] ✅ Store is ready!
[AppleIAP] FETCHING PRODUCTS FROM STOREKIT:
[AppleIAP] ✅ Product FOUND: "joyfulvoicesios001"
[AppleIAP]    Title: Joyful Voices Basic Monthly
[AppleIAP]    Price: $9.99
[AppleIAP] ✅ Product FOUND: "joyfulvoicesios002"
[AppleIAP]    Title: Joyful Voices Premium Monthly
[AppleIAP]    Price: $19.99
[AppleIAP] ✅ Product FOUND: "joyfulvoicesios003"
[AppleIAP]    Title: Joyful Voices Basic Yearly
[AppleIAP]    Price: $99.99
[AppleIAP] ✅ Product FOUND: "joyfulvoicesios004"
[AppleIAP]    Title: Joyful Voices Premium Yearly
[AppleIAP]    Price: $199.99
[AppleIAP] LOADING SUMMARY:
[AppleIAP] Products successfully loaded: 4
[AppleIAP] ✅ SUCCESS - Products available for purchase
```

---

## Troubleshooting Common Issues

### Issue: "Products successfully loaded: 0"

**Causes:**
1. Product IDs don't match App Store Connect
2. Products not approved/ready in App Store Connect
3. Bundle ID mismatch
4. StoreKit Configuration file missing/not enabled
5. No internet connection

**Debug:**
```
[AppleIAP] ❌ Product NOT FOUND: "joyfulvoicesios001"
[AppleIAP]    This could mean:
[AppleIAP]    - Product ID doesn't match App Store Connect
[AppleIAP]    - Product not approved in App Store Connect
[AppleIAP]    - Bundle ID mismatch
```

**Fix:** Verify Steps 1, 2, and 5 above

---

### Issue: "CdvPurchase plugin NOT available"

**Causes:**
1. Plugin not installed
2. Plugin not synced to iOS
3. Using web browser instead of native app

**Fix:**
```bash
npm install cordova-plugin-purchase --save
npx cap sync ios
# Rebuild in Xcode
```

---

### Issue: Products load but purchase fails

**Causes:**
1. Not using Sandbox test account
2. Sandbox account still signed into App Store
3. Network issue
4. Receipt validation failing

**Fix:**
1. Sign out of App Store in Settings
2. Use fresh sandbox test account
3. Check Supabase edge function logs for validation errors

---

### Issue: "Cannot connect to iTunes Store"

**Causes:**
1. Using simulator (StoreKit works but can be unreliable)
2. Not signed into any account
3. Network blocked

**Fix:**
1. Test on real device
2. Sign in with sandbox account
3. Check network/VPN

---

## Reply to Apple After Fixing

Once you've completed all steps and verified products load on device:

**In App Store Connect → App Review → Resolution Center:**

> We have identified and resolved the subscription loading issue:
>
> **Root Cause:** Missing StoreKit configuration file in Xcode project
>
> **Fixes Applied:**
> 1. Added StoreKit Configuration file with all 4 subscription products
> 2. Verified Product IDs match App Store Connect exactly
> 3. Removed development server configuration from production build
> 4. Enabled In-App Purchase capability in Xcode
> 5. Tested successfully on iPad Air M3 with iPadOS 18.2
>
> **Verification:**
> - All 4 products now load correctly
> - Prices display properly
> - Purchase flow completes successfully
> - Subscription activates as expected
>
> **Product IDs Configured:**
> - joyfulvoicesios001 (Basic Monthly)
> - joyfulvoicesios002 (Premium Monthly)
> - joyfulvoicesios003 (Basic Yearly)
> - joyfulvoicesios004 (Premium Yearly)
>
> The app is now ready for review. All subscriptions load and function correctly on iPadOS 26.2.

---

## Resources

- [Capacitor Documentation](https://capacitorjs.com/docs)
- [CdvPurchase Plugin Docs](https://github.com/j3k0/cordova-plugin-purchase)
- [StoreKit Configuration](https://developer.apple.com/documentation/xcode/setting-up-storekit-testing-in-xcode)
- [App Store Connect IAP Guide](https://help.apple.com/app-store-connect/#/devae49fb316)

---

## Quick Reference: Product IDs

```typescript
// In your code (useAppleIAP.ts)
export const APPLE_PRODUCTS = {
  basic_monthly: "joyfulvoicesios001",     // Must match App Store Connect
  premium_monthly: "joyfulvoicesios002",   // Must match App Store Connect
  basic_yearly: "joyfulvoicesios003",      // Must match App Store Connect
  premium_yearly: "joyfulvoicesios004",    // Must match App Store Connect
};
```

**Bundle ID:** `com.joyfjohnson.app` (must match everywhere)

---

**Last Updated:** 2026-01-25
**For:** Joyful Voices iOS App
**Apple Review Issue:** Guideline 2.1 - Subscriptions Not Loading
