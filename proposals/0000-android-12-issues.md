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

Starting in Android 12 any activities, services, or broadcast receivers that use intent filters will need to explicitly declare the `android:exported` attribute for the given app components. The `SdlRouterService` and `SdlReceiver` should already have the exported attribute defined and set to true. But The `USBAccessoryAttachmentActivity` will now also require this attribute to be set. Any activity that had an `intent-filter` would have a default exported value of true. Now we need to explicitly set it.

~~~ xml

<activity
    android:name="com.smartdevicelink.transport.USBAccessoryAttachmentActivity"
    android:exported="true"
    android:launchMode="singleTop">
    <intent-filter>
        <action android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED" />
    </intent-filter>

    <meta-data
        android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED"
        android:resource="@xml/accessory_filter" />
</activity>

~~~

### PendingIntent Mutable Flag

In Android 12, you must specify the mutability of each PendingIntent object that your app creates. This will impact the notifications that the RouterService is trying to display, As we do not need to update these intents at anypoint we can flag them with `FLAG_IMMUTABLE`.

~~~ java

int flag = android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ? PendingIntent.FLAG_IMMUTABLE : 0;
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, flag);

~~~

### Service Notification Delays

Starting in Android 12 when a service tries to present a notification, Android may delay showing the notification for up to 10 seconds. This is to try to allow the service to complete before the notification is presented. If we have connected to a system before we may want to present the RouterService notifications immediately. App developers may also want to display the notifications related to the `SdlService` immediately. This can be achieved by setting the foregroundServiceBehavior flag to `Notification.FOREGROUND_SERVICE_IMMEDIATE`.

~~~ java

private void safeStartForeground(int id, Notification notification) {
    try {
        if (notification == null) {
            if (hasConnectedBefore && android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.S) {
                Notification.Builder builder =
                        new Notification.Builder(this, SDL_NOTIFICATION_CHANNEL_ID)
                            .setContentTitle("SmartDeviceLink")
                            .setContentText("Service Running")
                            .setForegroundServiceBehavior(Notification.FOREGROUND_SERVICE_IMMEDIATE);
                notification = builder.build();
            } else {
                //Try the NotificationCompat this time in case there was a previous error
                NotificationCompat.Builder builder =
                        new NotificationCompat.Builder(this, SDL_NOTIFICATION_CHANNEL_ID)
                                .setContentTitle("SmartDeviceLink")
                                .setContentText("Service Running");

                notification = builder.build();
            }
        }
        startForeground(id, notification);
        DebugTool.logInfo(TAG, "Entered the foreground - " + System.currentTimeMillis());
    } catch (Exception e) {
        DebugTool.logError(TAG, "Unable to start service in foreground", e);
    }
}

~~~

## Potential downsides

### Foreground Services

With the proposed solution of using PendingIntents to start the apps `SdlService` from the RouterService this creates a potential security risk as the apps `SdlService` will now be required to have the `android:exported` attribute set to true. This could expose apps to have their `SdlService` be started by apps that are not SDL certified. We could also create a custom SDL `<permission>` to be used by apps as a requirement to be able to start the `SdlService` but nothing is stoping developers from listing that custom permission in their `AndroidManifest.xml`.   

### Bluetooth Runtime Permissions

With the new runtime permissions users will have to grant bluetooth permissions to SDL enabled apps. This means that if the user denies permissions for a specific app, the app will not know when the device connects to the head unit nor will it be able to start its bluetoothTransport in the routerService. With the solutions mentioned above apps that have their permissions denied will still be able to bind to another apps RouterService and will still be able to start up a routerService for a USB connection. If there are not any apps on the device with permissions then the router service would never be started when connecting over bluetooth.

### AndroidManifest Exported Flag

For the `SdlReceiver` and `SdlRouterService` the android guides already directed app developers to define the `android:exported` attribute and set it to true. In these cases as well since they have `intent-filters` defined this attributes default value was true. For the `SdlService` becasue there was not necessarily an `intent-filter` the default value may have been false. As mentioned prior by adding the exported attribute set to true to the `SdlService` this may pose a risk to app developers.

### PendingIntent Mutable Flag

The RouterService is the only place in the library currently that uses a PendingIntent and the PendingIntent is sent to the NotificationManager. With this being the case there is no need for the intent to be changed so we can flag this PendingIntent with the `PendingIntent.FLAG_IMMUTABLE` flag.

### Service Notification Delays

This change would be only to make sure a notification is always displayed immediately otherwise android may choose to delay the notification up to 10 seconds. If the notification is delayed there is no risk to the user and could result in a better user experience as we can hide some of the initial RouterService notifications while the routerService connects. If we choose to display the notificaiton immediately the user experience will mirror its current implementation. For the `SdlService` it will be up to the individual app developers to implement the immediate flag.

## Impact on existing code

## Alternatives considered

### Foreground Services

Alternatives for the Foreground Service restrictions are limited. We either need to start the `SdlService` from a foreground context or the conditions need to meet one of the exceptiions listed by google. These conditions include:

* Starting the Service from an Activity.
* Starting the Service from a user interaction with a notification
* Requesting the user ignores battery optimizations for each SDL Application

These options would either recquire and Activity be launched for each SDL app, the user to interact with a notification for each SDL app, or for the user to choose battery optimization options for each app which then creates a situation where the user dictates if the given app will have its `SdlService` start when the RouterService conencts.

### Bluetooth Runtime Permissions

These permissions are required to be able to receive intents withe the `ACL_CONNECTED` action in the BroadcastReceiver and to start the bluetoothTransport in the RouterService. Without these permissions we cannot know when the device connects to a device over Bluetooth nor will the RouterService be able to connect over Bluetooth.

### AndroidManifest Exported Flag

For the cases where this attribute is now required by Android 12. We already required developers to include this attribute.

### PendingIntent Mutable Flag

The one case where PendingIntents were being used previously in the library we can set the flag to be `PendingIntent.FLAG_MUTABLE` but as we are handing the PendingIntent off to the notificationManager and the intent does not need to be changed at any point we should use `PendingIntent.FLAG_IMMUTABLE`.

### Service Notification Delays

We can leave the notification implementations unchanged and Android would simply delay displaying them if the service they are related to is still running.
