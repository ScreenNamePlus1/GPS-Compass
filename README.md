# GPS-Compass
GPS compass
### Overview
Building an Android app like this is straightforward using Android's built-in APIs for sensors (compass and level), GPS/location services, and sharing intents. I'll use Kotlin for the examples (it's modern and concise), but you can adapt to Java if preferred. We'll create a simple GUI with a compass rose (drawable or custom view), a bubble level indicator, text displays for direction and location, and a share button.

Key Android components:
- **Sensors**: Use `SensorManager` for magnetometer (compass) and accelerometer (level and orientation helper).
- **GPS/Location**: Use `FusedLocationProviderClient` from Google Play Services for efficient location without needing internet (GPS satellites work offline).
- **GUI**: Use ConstraintLayout for the main screen, with ImageView for compass, a custom view for level, TextViews for info, and a Button for sharing.
- **Permissions**: Need `ACCESS_FINE_LOCATION` for GPS. Sensors don't require runtime permissions.
- **Sharing**: Use `Intent.ACTION_SEND` to share location via apps like SMS/email when connected.

Tools needed: Android Studio (free), an emulator or physical device for testing.

### Step 1: Set Up the Project
1. Open Android Studio, create a new project with "Empty Activity".
2. Set min SDK to API 21 (Android 5.0) for broad compatibility.
3. In `build.gradle.kts` (app module), add dependencies:
   ```kotlin
   dependencies {
       implementation("com.google.android.gms:play-services-location:21.3.0")  // For FusedLocationProvider
   }
   ```
4. Sync Gradle.

### Step 2: Handle Permissions
In `AndroidManifest.xml`, add:
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />  <!-- Fallback -->
```

In your Activity, request runtime permission (for API 23+):
```kotlin
private fun requestLocationPermission() {
    if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
        ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.ACCESS_FINE_LOCATION), 1)
    } else {
        // Permission granted, start location updates
    }
}

// Override onRequestPermissionsResult to handle response
```

### Step 3: Design the GUI (layout.xml)
In `res/layout/activity_main.xml`, use this ConstraintLayout:
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!-- Compass Image -->
    <ImageView
        android:id="@+id/compass_image"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:src="@drawable/compass_rose"  <!-- Add a compass PNG drawable -->
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Direction Text -->
    <TextView
        android:id="@+id/direction_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Direction: North"
        app:layout_constraintTop_toBottomOf="@id/compass_image"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Location Text -->
    <TextView
        android:id="@+id/location_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Location: Loading..."
        app:layout_constraintTop_toBottomOf="@id/direction_text"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Level Indicator (Custom View or simple Text for simplicity) -->
    <TextView
        android:id="@+id/level_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Level: Flat"
        app:layout_constraintTop_toBottomOf="@id/location_text"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Share Button -->
    <Button
        android:id="@+id/share_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Share Location"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
- Add a compass rose image (e.g., download a PNG and put in `res/drawable/compass_rose.png`).
- For a better level, create a custom View (see Step 6).

### Step 4: Implement Compass
In `MainActivity.kt`:
```kotlin
class MainActivity : AppCompatActivity(), SensorEventListener {

    private lateinit var sensorManager: SensorManager
    private var accelerometer: Sensor? = null
    private var magnetometer: Sensor? = null
    private val rotationMatrix = FloatArray(9)
    private val orientationAngles = FloatArray(3)
    private var accelerometerReading = FloatArray(3)
    private var magnetometerReading = FloatArray(3)

    private lateinit var compassImage: ImageView
    private lateinit var directionText: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        compassImage = findViewById(R.id.compass_image)
        directionText = findViewById(R.id.direction_text)

        sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        magnetometer = sensorManager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD)

        // Other init code...
    }

    override fun onResume() {
        super.onResume()
        sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL)
        sensorManager.registerListener(this, magnetometer, SensorManager.SENSOR_DELAY_NORMAL)
    }

    override fun onPause() {
        super.onPause()
        sensorManager.unregisterListener(this)
    }

    override fun onSensorChanged(event: SensorEvent?) {
        if (event == null) return

        if (event.sensor.type == Sensor.TYPE_ACCELEROMETER) {
            System.arraycopy(event.values, 0, accelerometerReading, 0, accelerometerReading.size)
        } else if (event.sensor.type == Sensor.TYPE_MAGNETIC_FIELD) {
            System.arraycopy(event.values, 0, magnetometerReading, 0, magnetometerReading.size)
        }

        SensorManager.getRotationMatrix(rotationMatrix, null, accelerometerReading, magnetometerReading)
        SensorManager.getOrientation(rotationMatrix, orientationAngles)

        val azimuth = Math.toDegrees(orientationAngles[0].toDouble()).toFloat()  // 0 to 360 degrees
        compassImage.rotation = -azimuth  // Rotate image to point North

        val direction = getDirection(azimuth)
        directionText.text = "Direction: $direction"
    }

    private fun getDirection(azimuth: Float): String {
        return when ((azimuth + 360) % 360) {
            in 337.5..360.0, in 0.0..22.5 -> "North"
            in 22.5..67.5 -> "North-East"
            in 67.5..112.5 -> "East"
            in 112.5..157.5 -> "South-East"
            in 157.5..202.5 -> "South"
            in 202.5..247.5 -> "South-West"
            in 247.5..292.5 -> "West"
            in 292.5..337.5 -> "North-West"
            else -> "Unknown"
        }
    }

    override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}  // Required but empty
}
```
This uses sensors to calculate orientation and rotate the compass image.

### Step 5: Implement GPS Location
Add to `MainActivity.kt`:
```kotlin
private lateinit var fusedLocationClient: FusedLocationProviderClient
private lateinit var locationText: TextView
private var currentLocation: Location? = null

// In onCreate:
fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
locationText = findViewById(R.id.location_text)
requestLocationPermission()  // Call this

private fun startLocationUpdates() {
    val locationRequest = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 10000)  // Every 10s
        .build()

    val locationCallback = object : LocationCallback() {
        override fun onLocationResult(locationResult: LocationResult) {
            currentLocation = locationResult.lastLocation
            locationText.text = "Location: Lat ${currentLocation?.latitude}, Lon ${currentLocation?.longitude}"
        }
    }

    fusedLocationClient.requestLocationUpdates(locationRequest, locationCallback, Looper.getMainLooper())
}

// Call startLocationUpdates() after permission granted
```
GPS works offline via satellites, but initial fix might take time outdoors.

### Step 6: Implement Leveling Mechanism
Use accelerometer for tilt. Add to `MainActivity.kt`:
```kotlin
private lateinit var levelText: TextView

// In onCreate:
levelText = findViewById(R.id.level_text)

// In onSensorChanged, after getOrientation:
val pitch = Math.toDegrees(orientationAngles[1].toDouble()).toFloat()  // Front-back tilt
val roll = Math.toDegrees(orientationAngles[2].toDouble()).toFloat()   // Left-right tilt

levelText.text = "Level: Pitch $pitch°, Roll $roll°"  // Or "Flat" if abs < 5°

// For a bubble level, create a custom View:
class BubbleLevelView(context: Context) : View(context) {
    // Draw a circle that moves based on pitch/roll
    // Use Canvas in onDraw, invalidate() on sensor change
}
```
// Replace the TextView with this custom View for a visual bubble.

### Step 7: Implement Sharing
Add to `MainActivity.kt`:
```kotlin
private lateinit var shareButton: Button

// In onCreate:
shareButton = findViewById(R.id.share_button)
shareButton.setOnClickListener {
    if (currentLocation != null) {
        val shareIntent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, "My location: Lat ${currentLocation?.latitude}, Lon ${currentLocation?.longitude}")
        }
        startActivity(Intent.createChooser(shareIntent, "Share Location"))
    } else {
        Toast.makeText(this, "Location not available", Toast.LENGTH_SHORT).show()
    }
}
```
This shares via any app (SMS if no data, but SMS needs carrier; email/WhatsApp needs data).

### Step 8: Testing and Polish
- Test on a physical device (emulators have limited sensor simulation).
- Handle edge cases: No GPS signal, sensor calibration (prompt user to figure-8 wave phone).
- Add error handling, like Toast messages for permissions denied.
- For better UX: Animate compass smoothly, add a map link in share (e.g., "https://maps.google.com/?q=lat,lon").
- If you want voice output or more features, extend with TextToSpeech.

This should give you a functional app. Start with the basics and iterate! If you run into issues, share specific errors for more help.
