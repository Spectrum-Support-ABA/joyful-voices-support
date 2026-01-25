# StoreKit 2 Implementation Guide for Joyful Voices

## Required: Minimal StoreKit Implementation

If your iOS app currently has **NO** StoreKit implementation, you need to add this code to enable subscriptions.

---

## Step 1: Create Store Manager (StoreKit 2)

Create a new Swift file: `StoreManager.swift`

```swift
import StoreKit
import SwiftUI

@MainActor
class StoreManager: ObservableObject {

    // MARK: - Published Properties
    @Published var products: [Product] = []
    @Published var purchasedProductIDs: Set<String> = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    // MARK: - Product Identifiers
    // ⚠️ CRITICAL: These MUST match exactly what's in App Store Connect
    private let productIdentifiers: [String] = [
        "com.joyfulvoices.monthly",    // Replace with your actual Product ID
        "com.joyfulvoices.annual"      // Replace with your actual Product ID
    ]

    // MARK: - Initialization
    init() {
        Task {
            await loadProducts()
            await updatePurchasedProducts()

            // Listen for transaction updates
            await observeTransactionUpdates()
        }
    }

    // MARK: - Load Products
    func loadProducts() async {
        isLoading = true
        errorMessage = nil

        do {
            // Load products from App Store
            let storeProducts = try await Product.products(for: productIdentifiers)

            if storeProducts.isEmpty {
                errorMessage = "No products found. Check Product IDs in App Store Connect."
                print("❌ ERROR: Product IDs may not match App Store Connect")
                print("Expected IDs: \(productIdentifiers)")
            } else {
                products = storeProducts
                print("✅ Loaded \(storeProducts.count) products successfully")
            }

        } catch {
            errorMessage = "Failed to load products: \(error.localizedDescription)"
            print("❌ StoreKit Error: \(error)")

            // Retry after delay
            try? await Task.sleep(nanoseconds: 2_000_000_000) // 2 seconds
            await loadProducts()
        }

        isLoading = false
    }

    // MARK: - Purchase Product
    func purchase(_ product: Product) async throws {
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            // Verify the transaction
            let transaction = try checkVerified(verification)

            // Update purchased products
            await updatePurchasedProducts()

            // Finish the transaction
            await transaction.finish()

        case .userCancelled:
            print("User cancelled purchase")

        case .pending:
            print("Purchase pending (e.g., parental approval required)")

        @unknown default:
            print("Unknown purchase result")
        }
    }

    // MARK: - Update Purchased Products
    func updatePurchasedProducts() async {
        var purchasedIDs: Set<String> = []

        // Check current entitlements
        for await result in Transaction.currentEntitlements {
            if case .verified(let transaction) = result {
                if transaction.revocationDate == nil {
                    purchasedIDs.insert(transaction.productID)
                }
            }
        }

        self.purchasedProductIDs = purchasedIDs
    }

    // MARK: - Restore Purchases
    func restorePurchases() async {
        do {
            try await AppStore.sync()
            await updatePurchasedProducts()
        } catch {
            errorMessage = "Failed to restore purchases: \(error.localizedDescription)"
        }
    }

    // MARK: - Transaction Observer
    private func observeTransactionUpdates() async {
        for await result in Transaction.updates {
            if case .verified(let transaction) = result {
                await updatePurchasedProducts()
                await transaction.finish()
            }
        }
    }

    // MARK: - Verification Helper
    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified:
            throw StoreError.failedVerification
        case .verified(let safe):
            return safe
        }
    }

    // MARK: - Check Subscription Status
    func hasActiveSubscription() -> Bool {
        return !purchasedProductIDs.isEmpty
    }
}

// MARK: - Store Error
enum StoreError: Error {
    case failedVerification
}
```

---

## Step 2: Create Subscription View

Create `SubscriptionView.swift`:

```swift
import SwiftUI
import StoreKit

struct SubscriptionView: View {
    @StateObject private var store = StoreManager()
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationView {
            VStack(spacing: 20) {

                // Header
                VStack(spacing: 8) {
                    Text("Unlock Joyful Voices Premium")
                        .font(.title2)
                        .bold()

                    Text("Access all features and communication boards")
                        .font(.subheadline)
                        .foregroundColor(.secondary)
                }
                .padding(.top)

                // Products List
                if store.isLoading {
                    ProgressView("Loading subscriptions...")
                        .padding()

                } else if let error = store.errorMessage {
                    VStack(spacing: 12) {
                        Image(systemName: "exclamationmark.triangle")
                            .font(.largeTitle)
                            .foregroundColor(.orange)
                        Text(error)
                            .multilineTextAlignment(.center)
                            .foregroundColor(.secondary)
                        Button("Retry") {
                            Task { await store.loadProducts() }
                        }
                        .buttonStyle(.bordered)
                    }
                    .padding()

                } else if store.products.isEmpty {
                    VStack(spacing: 12) {
                        Image(systemName: "exclamationmark.circle")
                            .font(.largeTitle)
                            .foregroundColor(.red)
                        Text("No subscriptions available")
                            .foregroundColor(.secondary)
                        Button("Retry") {
                            Task { await store.loadProducts() }
                        }
                        .buttonStyle(.bordered)
                    }
                    .padding()

                } else {
                    // Display products
                    ForEach(store.products, id: \.id) { product in
                        ProductRow(
                            product: product,
                            isPurchased: store.purchasedProductIDs.contains(product.id)
                        ) {
                            Task {
                                do {
                                    try await store.purchase(product)
                                } catch {
                                    print("Purchase failed: \(error)")
                                }
                            }
                        }
                    }
                    .padding(.horizontal)
                }

                Spacer()

                // Restore purchases button
                Button("Restore Purchases") {
                    Task {
                        await store.restorePurchases()
                    }
                }
                .buttonStyle(.borderless)
                .foregroundColor(.blue)
                .padding(.bottom)

                // Legal links
                HStack(spacing: 16) {
                    Link("Terms of Use", destination: URL(string: "https://your-domain.com/terms.html")!)
                    Text("•")
                    Link("Privacy Policy", destination: URL(string: "https://your-domain.com/privacy.html")!)
                }
                .font(.caption)
                .foregroundColor(.secondary)
                .padding(.bottom)
            }
            .navigationTitle("Subscribe")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Close") { dismiss() }
                }
            }
        }
    }
}

// MARK: - Product Row
struct ProductRow: View {
    let product: Product
    let isPurchased: Bool
    let onPurchase: () -> Void

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(product.displayName)
                    .font(.headline)

                Text(product.description)
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }

            Spacer()

            if isPurchased {
                Image(systemName: "checkmark.circle.fill")
                    .foregroundColor(.green)
                    .font(.title2)
            } else {
                Button(action: onPurchase) {
                    Text(product.displayPrice)
                        .bold()
                        .padding(.horizontal, 16)
                        .padding(.vertical, 8)
                        .background(Color.blue)
                        .foregroundColor(.white)
                        .cornerRadius(8)
                }
            }
        }
        .padding()
        .background(Color(.systemGray6))
        .cornerRadius(12)
    }
}
```

---

## Step 3: Configure App Store Connect

### A. Create In-App Purchase Products

1. Go to **App Store Connect** → Your App → **Features** → **In-App Purchases**
2. Click **+** to create new subscription group
3. Add two auto-renewable subscriptions:

**Monthly Subscription:**
- Product ID: `com.joyfulvoices.monthly` (or your bundle ID prefix)
- Reference Name: `Joyful Voices Monthly`
- Subscription Duration: 1 Month
- Price: Set your price tier

**Annual Subscription:**
- Product ID: `com.joyfulvoices.annual`
- Reference Name: `Joyful Voices Annual`
- Subscription Duration: 1 Year
- Price: Set your price tier (typically 20-30% discount vs monthly)

4. **Submit for review** (must be approved separately)

### B. Update Product IDs in Code

Replace the placeholder IDs in `StoreManager.swift`:

```swift
private let productIdentifiers: [String] = [
    "com.joyfulvoices.monthly",    // ← Use YOUR actual Product ID
    "com.joyfulvoices.annual"      // ← Use YOUR actual Product ID
]
```

---

## Step 4: Add to Settings

In your `SettingsView.swift` or equivalent:

```swift
struct SettingsView: View {
    @StateObject private var store = StoreManager()
    @State private var showingSubscription = false

    var body: some View {
        List {
            Section(header: Text("Subscription")) {
                if store.hasActiveSubscription() {
                    HStack {
                        Text("Status")
                        Spacer()
                        Text("Active")
                            .foregroundColor(.green)
                    }

                    Button("Manage Subscription") {
                        // Open iOS subscription management
                        if let url = URL(string: "https://apps.apple.com/account/subscriptions") {
                            UIApplication.shared.open(url)
                        }
                    }
                } else {
                    Button("Subscribe to Premium") {
                        showingSubscription = true
                    }
                }
            }

            Section(header: Text("Account")) {
                Button(role: .destructive) {
                    // Implement account deletion
                } label: {
                    Label("Delete Account", systemImage: "trash")
                        .foregroundColor(.red)
                }
            }
        }
        .sheet(isPresented: $showingSubscription) {
            SubscriptionView()
        }
    }
}
```

---

## Step 5: Configure StoreKit Configuration File (For Testing)

1. In Xcode, go to **File** → **New** → **File**
2. Select **StoreKit Configuration File**
3. Name it `Products.storekit`
4. Add your products manually for local testing:
   - Click **+** → Add subscription
   - Enter Product ID, name, price

5. In Xcode scheme:
   - **Product** → **Scheme** → **Edit Scheme**
   - **Run** → **Options** tab
   - **StoreKit Configuration**: Select `Products.storekit`

---

## Step 6: Test in Sandbox

### A. Create Sandbox Test Account

1. Go to **App Store Connect** → **Users and Access** → **Sandbox Testers**
2. Create test account with unique email
3. Sign out of real App Store account on test device
4. Sign in with sandbox account when prompted during purchase

### B. Test Flow

1. Run app on device
2. Navigate to subscription screen
3. Verify products load
4. Attempt purchase
5. Complete with sandbox account
6. Verify "Active" status appears

---

## Troubleshooting

### Products Not Loading

**Check these:**
1. Product IDs match **exactly** (case-sensitive)
2. Products are in "Ready to Submit" status in App Store Connect
3. Paid Apps Agreement is signed
4. Bundle ID matches
5. Internet connection works

**Add debug logging:**
```swift
print("🔍 Looking for products: \(productIdentifiers)")
print("📦 Found products: \(storeProducts.map { $0.id })")
```

### "Cannot connect to iTunes Store"

- Use real device, not simulator (for actual testing)
- Sign out of production App Store account
- Sign in with Sandbox tester account
- Check network connection

### Purchases Not Persisting

- Check `Transaction.currentEntitlements` logic
- Verify transactions are being finished
- Test restore purchases functionality

---

## Common Mistakes

1. ❌ Product IDs don't match App Store Connect
2. ❌ Products not submitted for review in App Store Connect
3. ❌ Testing on simulator (use real device)
4. ❌ Not signing Paid Apps Agreement
5. ❌ Bundle ID mismatch
6. ❌ Forgetting to call `transaction.finish()`

---

## Apple Review Tips

**What Apple Tests:**
- Products load on first launch
- Purchase flow works end-to-end
- Restore purchases works
- Subscription status displays correctly
- App handles network errors gracefully

**Make Sure:**
- Add loading states (spinners)
- Show clear error messages
- Implement retry logic
- Test on iPad Air if possible
- Products load within 10 seconds

---

## Resources

- [StoreKit 2 Documentation](https://developer.apple.com/documentation/storekit)
- [Testing In-App Purchases](https://developer.apple.com/documentation/storekit/in-app_purchase/testing_in-app_purchases_in_xcode)
- [App Store Connect Help](https://help.apple.com/app-store-connect/)

---

**Next Steps:**
1. Implement `StoreManager.swift` in your iOS project
2. Create subscription UI
3. Configure products in App Store Connect
4. Test in Sandbox thoroughly
5. Submit for review
