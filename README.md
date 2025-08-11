## Usage

```kotlin
import com.smallplanet.roverandroid.*

// Create a custom subclass of RoverApplication for you app
class ReferenceApplication(): RoverApplication() { }

// Alternatively, you can provide these two overrides manually
class ReferenceApplication(): Application() {
    override fun getPackageName(): String? {
        return Rover.getPackageName() ?: super.getPackageName()
    }
    override fun getPackageManager(): PackageManager {
        return Rover.getPackageManager(
            super.getPackageName(),
            super.getPackageManager()
        ) ?: super.getPackageManager()
    }
}
```

```kotlin
// To interact with a Rover collector you provide a RoverDelegate. A minimal
// delegate would know when the collection is finished (either successfully or
// as a result of an error) and be able to process any receipts collected.
//
// IMPORTANT NOTE: Some delegate methods provide a callback. Collection will not continue
// until the callback is made.
class ReferenceDelegate(): RoverDelegate() {
    override fun roverDidFinish(sessionUUID: String,
						    		  resultsGzip: ByteArray,
                                error: String?,
                                userError: String?,
                                verboseError: String?) {
        if (error != null) {
            Log.d("RoverDelegate", "[${sessionUUID}] finished with error: ${error}")
        } else {
            Log.d("RoverDelegate", "[${sessionUUID}] finished")
        }
    }
    override fun roverDidCollect(sessionUUID: String,
                                 receipts: MutableList<Receipt>) {
        // Receipts will be passed to you in batches during the collection process
        Log.d("RoverDelegate", "[${sessionUUID}] captured ${receipts.size} receipts")
    }
}

```

```kotlin
// Note: You will need to call configure to provide your license key
// and receive the list of merchants you will be able to collect from.
// Each merchant will be an int identifier and a user facing name.
// You may call configure multiple times if you wish.
// The application context is used to get the cacheDir, 
// filesDir and the packageName. The fragment manager is used to
// display merchant login UI when required.
Rover.configure(
    "MY_ROVER_LICENSE_KEY",
    Rover.Environment.staging,
    "unknown",
    applicationContext, 
    supportFragmentManager) { merchants, error ->
	
}

// When you are ready to connect and collect from a merchant, call 
// Rover.collect() with the desired merchant id, the date back to
// which Rover should collect from, and your delegate instance
// to collect the results with
val formatter = SimpleDateFormat("dd-MM-yyyy", Locale.ENGLISH)
val fromDate: Date = formatter.parse("01-01-2020") ?: Date()

// 1. Create a new connection to a merchant
// userId: [optional] identifier you provide and passed through by Rover
// account: [optional] account name for this merchant to connect to (nil for new connection)
// merchantId: the identifier for the merchant to connect to (passed back in configure merchants array)
// fromDate: how far back you'd like to collect receipts
// collectItemInfo: [optional] collect extra information about items when possible (like UPC)
// isEphemeral: [optional] encrypt and store this connection locally to reconnect at later date
Rover.collect(
	userId = null,
	account = null,
	merchantId = MerchantId.gmail.rawValue,
	fromDate = fromDate,
	collectItemInfo = true,
	isEphemeral = false,
	delegate = ReferenceDelegate())

// 2. List current merchant connections
// connections: array of existing merchant connections
Rover.connections { connections ->
	
}

// 3. Recollect from an existing connection
Rover.collect(
	userId = null,
	account = connection.account,
	merchantId = MerchantId.gmail.rawValue,
	fromDate = connection.fromDate,
	collectItemInfo = true,
	isEphemeral = false,
	delegate = ReferenceDelegate())

// 4. Remove a connection
Rover.shared.remove(connection: connection) { }

```

### Background Collection

To perform collections in the background (via [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager)) you need to perform the following steps.

- Create a Worker class in which you will call Rover.collect() on the connections you want to collect from.
- Schedule your Worker class for periodic work via WorkManager API

```
// Kotlin example for CoroutineWorker which will be called when it is time to process in the background
class BackgroundCollectionWorker(val context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return Rover.scheduleBackgroundCollections(
            {
                // Call Rover.collect() for the connections you want to refresh here. You can use
                // your own business logic to determine which connections are higher priority than
                // others and schedule them first.
                // NOTE: it is important then Rover.configure() is called before collections
                // are scheduled.
                Rover.configure(
                    "MY_ROVER_LICENSE_KEY",
                    Rover.Environment.staging,
                    "unknown",
                    applicationContext, 
                    null) { merchants, error ->
	
                    Rover.connections { connections ->

                        // Perform collections for at more one connection; that connection should:
                        // - not be userInteractionRequired
                        // - have the oldest collection date
                        val distantPast = Date(Long.MIN_VALUE)
                        val filteredConnections = connections
                            .filter { it.userInteractionRequired == false }
                            .sortedWith { lhs, rhs ->
                                (lhs.attemptedDate ?: distantPast).compareTo(rhs.attemptedDate ?: distantPast)
                            }
                        for (connection in filteredConnections) {
                            Log.d("Rover", "Schedule collection on ${connection.merchantId}:${connection.account}")
                            Rover.collect(account = connection.account,
                                merchantId = connection.merchantId,
                                fromDate = connection.fromDate ?: Date(),
                                isEphemeral = false,
                                delegate = ReferenceDelegate())
                        }
                    }
                }
            },
            { connections ->
                // When all collections are completed this code will execute, allowing for
                // any clean up or reporting.
                Log.d("Rover", "Background collection finished for ${connections.size} connections")
            }
        )
    }
}

// Kotlin example for scheduling your BackgroundCollectionWorker class
// Schedule background collections via WorkManager API. A good place to call
// this is in the callback of Rover.configure()
val manager = WorkManager.getInstance(this.applicationContext)
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)
    .build()
val request = PeriodicWorkRequestBuilder<BackgroundCollectionWorker>(24, TimeUnit.HOURS)
    .setInitialDelay(24, TimeUnit.HOURS)
    .setConstraints(constraints)
    .build()
manager.enqueueUniquePeriodicWork("BackgroundCollectionWorker", ExistingPeriodicWorkPolicy.UPDATE, request)
    

```


## SDK Integration

### Required build.gradle settings

At the time of this writing, the following changes to your build.gradle are required:


```kotlin
// Add the following repository to your build.gradle:
android {

	// Rover requires a minSdk of 24
	defaultConfig {
        minSdk 24
    }

	// Rover uses view bindings when displaying merchant log in screens
	buildFeatures {
		viewBinding true
	}
	
	// Rover includes several native libraries. Android Studio will attempt to automatically
	// strip these included .so files, mangling them to the point they will no longer work.
	// These packaging options disable the stripping of these libraries.
	packagingOptions {
		pickFirst '**/libjsc.so'
		pickFirst '**/libc++_shared.so'
		doNotStrip "*/arm64-v8a/*.so"
		doNotStrip "*/armeabi-v7a/*.so"
	}
}
```

### Manual AAR

To embed a specific version of Rover in your Android Studio project, download the Android Archive file from the releases directory ( https://github.com/SmallPlanet/RoverAndroid/releases ). Place the .aar file somewhere in your project directory and then
add the following:

```kotlin
// Add the following dependency to your build.gradle. Make sure to update the path correctly to your downloaded .aar file.
dependencies {
	implementation files('./path/to/downloaded/RoverAndroid.aar')
}
```

### Proguard

If you are using Proguard in your Android app, please add the following rule:

```
-keep class com.smallplanet.** { *; }
```

### Disable SIGUSR1 breakpoints

By default, debugging in Android Studio will break on (any?) signal. [This is a well-known issue documented here](https://issuetracker.google.com/issues/240007217?pli=1). This is particularly annoying because usleep() is implemented using SIGUSR1 in the Android NDK. To work around this issue, you can make the following configuration change:

- Go to Run -> **Edit Configurations**
- Select the **Debugger** tab
- Select the **LLDB Post Attach Commands** tab
- Click the **+** button
- add ```process handle SIGUSR1 --pass true --stop false --notify true```


Latest version: v0.4.26
