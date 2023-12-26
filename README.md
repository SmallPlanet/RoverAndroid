## Usage

```kotlin
import com.smallplanet.roverandroid.*

// In a subclass of Application, provide this override for the getPackageName() method
class ReferenceApplication(): Application() {
    override fun getPackageName(): String? {
        return Rover.getPackageName() ?: super.getPackageName()
    }
}
```

```kotlin
// To interact with a Rover collector you provide a RoverDelegate. A minimal
// delegate would know when the collection is finished (either successfully or
// as a result of an error) and be able to process any receipts collected.
//
// Note: Some delegate methods provide a callback. Collection will not continue
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

Rover.collect(
	merchantId = MerchantId.gmail.rawValue,
	fromDate = fromDate,
	isEphemeral = false,
	delegate = ReferenceDelegate())
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

To embed a specific version of Rover in your Android Studio project, download the Android Archive file from the releases directory ( https://github.com/SmallPlanet/RoverAndroid/tree/main/Releases ). Place the .aar file somewhere in your project directory and then
add the following:

```kotlin
// Add the following dependency to your build.gradle. Make sure to update the path correctly to your downloaded .aar file.
dependencies {
	implementation files('./path/to/downloaded/RoverAndroid.aar')
}
```

### Disable SIGUSR1 breakpoints

By default, debugging in Android Studio will break on (any?) signal. [This is a well-known issue documented here](https://issuetracker.google.com/issues/240007217?pli=1). This is particularly annoying because usleep() is implemented using SIGUSR1 in the Android NDK. To work around this issue, you can make the following configuration change:

- Go to Run -> **Edit Configurations**
- Select the **Debugger** tab
- Select the **LLDB Post Attach Commands** tab
- Click the **+** button
- add ```process handle SIGUSR1 --pass true --stop false --notify true```


Latest version: v0.3.2
