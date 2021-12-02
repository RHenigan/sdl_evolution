# Feature name

* Proposal: [SDL-0000](0000-android-12-issues.md)
* Author: [Robert Henigan](https://github.com/RHenigan)
* Status: **Awaiting review**
* Impacted Platforms: [Java Suite]

## Introduction

This proposal is to address the issues that have been introduced as a result of Android 12.

## Motivation

Starting in Android 12 Apps will no longer be able to start foreground services from the background except for in a few select cases. This will directly impact how apps are able to start their own `SdlService` implementation.

Android 12 also introduces new runtime bluetooth permissions that will be required by the library ([`BLUETOOTH_CONNECT` and `BLUETOOTH_SCAN`](https://developer.android.com/about/versions/12/features#bluetooth-permissions)).
 
Apps will also need to explicitly set the exported flag for any services, receivers, and activities in the manifest that have an `intent-filter`.

PendingIntents also now require the mutability flag to be set in all cases which impacts some notifications sent by the router service.

Android 12 may sometimes choose to hide notifications from services for up to 10 seconds to allow the service time to finish. This should be fine in most cases but may be something that needs to be considered by the developer and may have some cases in the library where a notification should be displayed immediately.

## Proposed solution

### Foreground Services
With the new restrictions around starting foreground services from the background we will need to meet one of the exceptions that Android 12 would allow tos start a foreground service from the background or we would need to start the foreground service from a foreground context.

The proposed solution for this case would be to create a PendingIntent in the SdlRouterService that can be given to the SdlBroadcastReceiver. At this point the SdlReceiver will be able to send the PendingIntent with and updated Intent where the developer specifies their unique SdlService class.

This will require that the exported flag be set to true in the manifest for the apps SdlService and an external apps RouterService would be the one starting the service.

~~~ xml
<service
    android:name="com.sdl.hellosdlandroid.SdlService"
    android:exported="true"
    android:foregroundServiceType="connectedDevice">
</service>
~~~

~~~ java
//Creating the PendingIntent

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && intent.getBooleanExtra(TransportConstants.PENDING_BOOLEAN_EXTRA, false)) {
    Intent pending = new Intent();
    PendingIntent pendingIntent = PendingIntent.getForegroundService(context, (int) System.currentTimeMillis(), pending, PendingIntent.FLAG_MUTABLE | Intent.FILL_IN_COMPONENT);
    intent.putExtra(TransportConstants.PENDING_INTENT_EXTRA, pendingIntent);
}

context.sendBroadcast(intent);

~~~

~~~ java
//Retrieve, Update, and Send the PendingIntent
@Override
public void onSdlEnabled(Context context, Intent intent) {
    DebugTool.logInfo(TAG, "SDL Enabled");
    intent.setClass(context, SdlService.class);

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        if (intent.getParcelableExtra(TransportConstants.PENDING_INTENT_EXTRA) != null) {
            PendingIntent pendingIntent = (PendingIntent) intent.getParcelableExtra(TransportConstants.PENDING_INTENT_EXTRA);
            try {
                //Here we are allowing the RouterSerivce that is in the Foreground to start the SdlService on our behalf
                pendingIntent.send(context, 0, intent);
            } catch (PendingIntent.CanceledException e) {
                e.printStackTrace();
            }
        }
    } else {
        // SdlService needs to be foregrounded in Android O and above
        // This will prevent apps in the background from crashing when they try to start SdlService
        // Because Android O doesn't allow background apps to start background services
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            context.startForegroundService(intent);
        } else {
            context.startService(intent);
        }
    }
}
~~~

### Bluetooth Runtime Permissions
With the new required Bluetooth Runtime permissions we will need developers to include the new permissions in the AndroidManifest.xml file. If the permissions are only necessary for the library and not a unique usecase in the application then the developer can specify that the `BLUETOOTH_SCAN` permission is never used for location.

~~~ xml
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"
    tools:targetApi="31"/>
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation"
    tools:targetApi="31" />
~~~

The developers will also need to request these permissions from the user as they are runtime permissions. If the user does not grant these permissions for a given application then that application will not receive any Intent with the `ACL_CONNECTED` action in the BroadcastReceiver.

~~~ java
//Check if Permissions have been granted
private boolean checkPermission() {
    int btConnectPermission = ContextCompat.checkSelfPermission(getApplicationContext(), BLUETOOTH_CONNECT);
    int btScanPermission = ContextCompat.checkSelfPermission(getApplicationContext(), BLUETOOTH_SCAN);

    return btConnectPermission == PackageManager.PERMISSION_GRANTED && btScanPermission == PackageManager.PERMISSION_GRANTED;
}

//Prompt the user to grant permissions
private void requestPermission() {
    ActivityCompat.requestPermissions(this, new String[]{BLUETOOTH_CONNECT, BLUETOOTH_SCAN}, REQUEST_CODE);
}

//Execute if the user grants permissions
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    switch (requestCode) {
        case REQUEST_CODE:
            if (grantResults.length > 0) {

                boolean connectAccepted = grantResults[0] == PackageManager.PERMISSION_GRANTED;
                boolean scanAccepted = grantResults[1] == PackageManager.PERMISSION_GRANTED;

                if (connectAccepted && scanAccepted) {
                    SdlReceiver.queryForConnectedService(this);
                }
            }
            break;
    }
} 
~~~

In the even that the user denies bluetooth permissions from the application this compliacts the use cases surrounding the `BroadcastReceiver` and `SdlRouterService`. If the permissions are denied for the application then the apps `BroadcastReceiver` will not receive any Intent with the `ACL_CONNECTED` action and therefore will not know to start its own RouterService when the device connects over bluetooth. In the even that all apps on the device support the `SdlDeviceListener` (SDL Library Version 4.12 and newer) we can update the `BroadcastReceiver` to try to find the first app with these permissions granted and allow that app to start up their own RouterService.

~~~ java
//Before the SDLDeviceListener tries to start the router service
//Get the first app that has the permissions and is aware the device connected over Bluetooth

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    if (!AndroidTools.areBtPermissionsGranted(context, routerService.getPackageName()) && sdlAppInfoList.size() > 1) {
        for (SdlAppInfo appInfo : sdlAppInfoList) {
            if (AndroidTools.areBtPermissionsGranted(context, appInfo.getRouterServiceComponentName().getPackageName())) {
                routerService = appInfo.getRouterServiceComponentName();
                break;
            }
        }
    }
}
~~~

We also need to consider the cases where there are applications on the device that do not support the SDLDeviceListener. In these cases another app can try to directly start the RouterService of an application that does not have the permissions and there for the router service will not be able to start up the bluetoothTransport. In this case we can update the `initCheck` within the RouterService so in the event one of these apps tries to start the router service and this app does not have the bluetooth permissions, this app can fail the initCheck and try to deploy the next router service in the list. We also need to consider devices being connected over USB. In this case we can allow the RouterService to start and wait to start the bluetoothTransport. We can present a notification to the user that will navigate them to the applications permissions page. Once the permissions are granted the RouterService can then start the bluetoothTransport.

~~~ java
private boolean initCheck() {

    .....

    // If Android 12 or newer make sure we have BT Runtime permissions
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && !AndroidTools.areBtPermissionsGranted(this, this.getPackageName())) {
        //If Connected over USB start RouterService
        if (isConnectedOverUSB) {
            //Delay starting bluetoothTransport
            waitingForBTRuntimePermissions = true;
            btPermissionsHandler = new Handler(Looper.myLooper());
            //Continuously Check for the Bluetooth Permissions
            btPermissionsRunnable = new Runnable() {
                @Override
                public void run() {
                    if (!AndroidTools.areBtPermissionsGranted(SdlRouterService.this, SdlRouterService.this.getPackageName())) {
                        btPermissionsHandler.postDelayed(btPermissionsRunnable, BT_PERMISSIONS_CHECK_FREQUENCY);
                    } else {
                        waitingForBTRuntimePermissions = false;
                        initBluetoothSerialService();
                    }
                }
            };
            btPermissionsHandler.postDelayed(btPermissionsRunnable, BT_PERMISSIONS_CHECK_FREQUENCY);
            //Present Notification to take user to permissions page for the app
            showBTPermissionsNotification();
        } else {
            //If connected over bluetooth and permissions are not granted, fail the initCheck
            return false;
        }
    }

    .....

    return true;
}

private synchronized void initBluetoothSerialService() {
    if (waitingForBTRuntimePermissions) {
        return;
    }

    .....

}

private void showBTPermissionsNotification() {

    .....

    // Create an intent that will be fired when the user clicks the notification.
    Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
    Uri uri = Uri.fromParts("package", getPackageName(), null);
    intent.setData(uri);
    int flag = android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ? PendingIntent.FLAG_IMMUTABLE : 0;
    PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, flag);
    builder.setContentIntent(pendingIntent);

    .....

}
~~~

### AndroidManifest Exported Flag
### PendingIntent Mutable Flag
### Service Notification Delays

## Potential downsides

Describe any potential downsides or known objections to the course of action presented in this proposal, then provide counter-arguments to these objections. You should anticipate possible objections that may come up in review and provide an initial response here. Explain why the positives of the proposal outweigh the downsides, or why the downside under discussion is not a large enough issue to prevent the proposal from being accepted.

## Impact on existing code

Describe the impact that this change will have on existing code. Will some SDL integrations stop compiling due to this change? Will applications still compile but produce different behavior than they used to? Is it possible to migrate existing SDL code to use a new feature or API automatically?

## Alternatives considered

Describe alternative approaches to addressing the same problem, and why you chose this approach instead.