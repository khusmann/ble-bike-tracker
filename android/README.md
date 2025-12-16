# Bike Tracker Android App

Android companion app for the ESP32 bike tracker system. Connects via BLE to
display real-time cadence and automatically sync workout sessions to Google
Health Connect.

## Features

### Real-time Display (Device Tab)
- Live cadence display (RPM)
- Total revolution count tracking
- BLE connection status indicator
- Manual connect/disconnect controls

### Background Sync (Sync Settings Tab)
- Automatic periodic syncing via WorkManager (every 15 minutes in dev)
- Manual "Sync Now" button
- Sync diagnostics: timestamps, success/failure counts, error messages
- Google Health Connect integration (writes ExerciseSessionRecord)
- Multi-bike support via BLE device address
- Battery optimization warning and detection
- Timestamp divergence detection for debugging

### Battery Efficiency
- Low-power BLE scanning with service UUID filters
- Short connection windows (under 30 seconds per sync)
- No persistent foreground service
- Minimal battery usage during background sync

## Requirements

- Android 8.0 (API 26) or higher
- Device with BLE support
- Java 17 or higher for building

## Building

### Setup

The project uses the Gradle wrapper, so you don't need to install Gradle
manually. However, you need the Android SDK:

Create a `local.properties` file in the `android` directory:

```properties
sdk.dir=/path/to/your/Android/Sdk
```

Typical SDK locations:

- Linux: `~/Android/Sdk` or `$HOME/Android/Sdk`
- macOS: `~/Library/Android/sdk`
- Windows: `C:\Users\YourUsername\AppData\Local\Android\Sdk`

Note: `local.properties` is git-ignored and won't be committed to the repo.

### Command Line Build

```bash
# Build debug APK
./gradlew assembleDebug

# Install to connected device
./gradlew installDebug

# Build and install
./gradlew installDebug
```

The APK will be generated at: `app/build/outputs/apk/debug/app-debug.apk`

### Android Studio

1. Open Android Studio
2. File -> Open -> select the `android` directory
3. Wait for Gradle sync to complete
4. Run -> Run 'app'

## Architecture

The app follows functional programming principles with immutable data
structures and reactive state management.

### Key Components

- **MainActivity.kt**: Two-tab Jetpack Compose UI (Device and Sync Settings)
- **BleManager.kt**: BLE connection and CSC measurement handling using Kotlin
  Flow
- **BikeViewModel.kt**: State management with StateFlow for reactive updates
- **BackgroundSyncWorker.kt**: WorkManager implementation for periodic syncing
- **SyncScheduler.kt**: Schedules and manages background sync jobs
- **HealthConnectHelper.kt**: Queries and writes to Google Health Connect
- **SyncPreferences.kt**: Local sync state persistence via SharedPreferences
- **Models.kt**: Immutable data classes for state management

### BLE Implementation

The app uses two BLE services from the ESP32 firmware:

**1. CSC Service** (Cycling Speed and Cadence - standard profile)
- Service UUID: `0x1816`
- CSC Measurement Characteristic UUID: `0x2A5B`
- Used for real-time cadence display
- Scans with `SCAN_MODE_LOW_POWER` and service UUID filter
- Subscribes to notifications for real-time updates
- Calculates instantaneous cadence from consecutive measurements

**2. Sync Service** (custom protocol for background data transfer)
- Service UUID: `0000FF00-0000-1000-8000-00805f9b34fb`
- Session Data Characteristic UUID: `0xFF01`
- Used by BackgroundSyncWorker for retrieving historical sessions
- Protocol:
  1. Write last synced timestamp (uint32, little-endian, Unix epoch)
  2. Read response: JSON session data + remaining session count
  3. Repeat until count reaches zero
- MTU negotiated to 512 bytes (minimum 185 required)

### State Management

Uses Kotlin Flow for reactive state updates:

- `BikeState`: Immutable state containing cadence, revolutions, connection
  status, and sync information
- `ConnectionState`: Sealed class representing BLE connection lifecycle
- `SyncState`: Sync status, timestamps, and error tracking
- Pure function `calculateCadence()` for cadence computation

### Background Sync Architecture

**WorkManager** schedules periodic sync jobs:
- Interval: 15 minutes (configurable to hourly for production)
- Constraints: Requires battery not low
- Work: Scan for bike, connect, retrieve sessions, write to Health Connect

**SyncPreferences** stores local state:
- Last sync attempt/success timestamps
- Last synced session timestamp (for resuming sync)
- Success/failure counts
- Last error message

**Why SharedPreferences?** Health Connect restricts background reads for
privacy, so we cache the last synced timestamp locally to avoid querying Health
Connect from the background worker.

### Health Connect Integration

- Writes `ExerciseSessionRecord` for each synced session
- Uses BLE device address in `clientRecordId` for multi-bike support:
  `bike-{address}-{timestamp}`
- No local database: Health Connect is the primary data store
- Queries most recent session per bike to determine sync resume point

## Permissions

Required permissions (requested at runtime):

**Bluetooth (for real-time display and background sync):**
- `BLUETOOTH_SCAN`: For discovering BLE devices
- `BLUETOOTH_CONNECT`: For connecting to the bike tracker

**Health Connect (for writing workout sessions):**
- `health.permission.WRITE_EXERCISE`: To create ExerciseSessionRecord entries

**Background Work:**
- `RECEIVE_BOOT_COMPLETED`: To reschedule WorkManager jobs after device reboot

**Note:** Users must disable battery optimization for the app to ensure reliable
background syncing. The app displays a warning if optimization is enabled.

## Usage

### Initial Setup

1. Install the app on your Android device
2. Grant Bluetooth and Health Connect permissions when prompted
3. Install and configure Google Health Connect if not already installed
4. Disable battery optimization for the app (recommended for reliable background
   sync)

### Real-time Display

1. Launch the app and navigate to the **Device** tab
2. Tap "Connect" to scan for and connect to your bike tracker
3. Start pedaling to see real-time cadence updates
4. Tap "Disconnect" to end the session

### Background Sync

1. Navigate to the **Sync Settings** tab
2. Enable "Background Sync Enabled" toggle
3. Optionally tap "Sync Now" to manually trigger a sync
4. The app will automatically sync sessions every 15 minutes (configurable)
5. View sync status, timestamps, and diagnostic information in the sync tab

### Multi-Bike Support

The app supports multiple bikes by tracking the BLE device address. Each bike's
sessions are stored separately in Health Connect using unique `clientRecordId`
values.

## Testing

The app requires the ESP32 firmware from the `firmware` directory to be running
and advertising both the CSC and Sync services.

**Testing Real-time Display:**
1. Ensure firmware is running and advertising CSC service (`0x1816`)
2. Launch app and connect from Device tab
3. Trigger reed switch to see cadence updates

**Testing Background Sync:**
1. Record a session on the bike (>5 minutes of pedaling)
2. Wait 5 minutes for idle timeout to save the session
3. Tap "Sync Now" in the Sync Settings tab
4. Verify session appears in Health Connect and sync counters update

## Troubleshooting

**Background sync not working:**
- Check that battery optimization is disabled for the app
- Verify Health Connect permissions are granted
- Check sync error message in Sync Settings tab
- Ensure the bike is within BLE range

**Sessions not appearing in Health Connect:**
- Verify the session met minimum duration requirement (5 minutes)
- Check for timestamp divergence warning in Sync Settings tab
- Try manual "Sync Now" to see error details

**Connection issues:**
- Ensure firmware is running and advertising
- Check Bluetooth is enabled on phone
- Verify you're within BLE range of the bike

## Next Steps (Planned)

**Stage 4: Cadence Time Series Data**
- Record periodic cadence samples during sessions
- Write `CyclingPedalingCadenceRecord` to Health Connect
- Enable cadence graphs in compatible fitness apps

**Future Consideration: Companion Device Manager API**
- Replace WorkManager with proximity-based sync
- Automatic battery optimization exemption
- More reliable than periodic polling
