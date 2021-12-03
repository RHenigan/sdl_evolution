# Feature name

* Proposal: [SDL-0000](0000-android-12-issues.md)
* Author: [Robert Henigan](https://github.com/RHenigan)
* Status: **Awaiting review**
* Impacted Platforms: [Java Suite]

## Introduction

With almost every Android version, Google updates the OS to better distribute the device resources among the running apps to enhance the performance and optimize battery life. However, sometimes the new enhancements come with new restrictions on how apps can use the resources on the device. Android 12 is one of the major Android updates that heavily modify how apps can run in the background and use device resources. This affects how the `SdlRouterService` starts and communicates with other SDL apps and also how the `SdlService` for SDL apps is started. This proposal is to address the issues that have been introduced as a result of Android 12.

## Motivation

Android 12 adds multiple restrictions on how apps can work in the background and access device resources. Some of the new restrictions that affect how SDL Android works are:

- Apps will no longer be able to start foreground services from the background except for in a few select cases. This will directly impact how apps are able to start their own `SdlService` implementation.

- Android 12 introduces new runtime Bluetooth permissions that will be required by the library to successfully establish a Bluetooth connection with the head unit ([`BLUETOOTH_CONNECT` and `BLUETOOTH_SCAN`](https://developer.android.com/about/versions/12/features#bluetooth-permissions)).
 
- Apps will need to explicitly set the exported flag for any services, receivers, and activities that have an `intent-filter` in the manifest.

- `PendingIntents` now require the mutability flag to be set in all cases which impacts some notifications sent by the router service.

- Android 12 may sometimes choose to hide notifications from services for up to 10 seconds to allow short-running services to finish without interrupting the user. This is in most cases a good change. However, in some cases in the library, the notifications should be updated to be displayed immediately.

## Proposed solution

### Foreground Services
With the new restrictions around starting foreground services from the background we will need to meet one of the exceptions that Android 12 would allow to start a foreground service from the background or we would need to start the foreground service from a foreground context.

PendingIntents allows us to send an intent on behalf of another app using the context and permissions of the application that created the PendingIntent. Since the `SdlRouterService` will run in the foreground, the `SdlRouterService` could create a PendingIntent to start an apps  `SdlService`. Since the `SdlRouterService` is creating the PendingIntent, the intent would be sent from the `SdlRouterServices` Context.

The PendingIntent in the `SdlRouterService` can be given to the `SdlBroadcastReceiver`. At this point the `SdlReceiver` will be able to send the PendingIntent with and updated Intent where the developer specifies their unique `SdlService` class. By calling `PendingIntent.send()` we are starting a new foreground service of the unique `SdlService` class from the context of the `SdlRouterService`. Since the `SdlRouterService` is running in the foreground we are not longer trying to start a foreground service from the background.

This will require that the exported flag be set to true in the manifest for the apps `SdlService` as an external apps `SdlRouterService` would be the one starting the service.

#### Library Changes
~~~ java
//Creating the PendingIntent from SdlRouterService.java

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && intent.getBooleanExtra(TransportConstants.PENDING_BOOLEAN_EXTRA, false)) {
    Intent pending = new Intent();
    PendingIntent pendingIntent = PendingIntent.getForegroundService(context, (int) System.currentTimeMillis(), pending, PendingIntent.FLAG_MUTABLE | Intent.FILL_IN_COMPONENT);
    intent.putExtra(TransportConstants.PENDING_INTENT_EXTRA, pendingIntent);
}

context.sendBroadcast(intent);

~~~

#### App Developer Changes
~~~ xml
<!--AndroidManifest.xml-->
<service
    android:name="com.sdl.hellosdlandroid.SdlService"
    android:exported="true" <!--New Addition-->
    android:foregroundServiceType="connectedDevice">
</service>
~~~

~~~ java
//SdlReceiver.java
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
With the new required Bluetooth Runtime permissions we will need developers to include the new permissions in the `AndroidManifest.xml` file. 

#### Developer Changes
~~~ xml
//AndroidManifest.xml
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"
    tools:targetApi="31"/>
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation"
    tools:targetApi="31" />
~~~

The developers will also need to request these permissions from the user as they are runtime permissions. If the user does not grant these permissions for a given application then that application will not receive any Intent with the `ACL_CONNECTED` action in the `SdlBroadcastReceiver`.

In the event that the user denies bluetooth permissions from the application this complicates the use cases surrounding the `SdlBroadcastReceiver` and `SdlRouterService`. If the permissions are denied for the application then the apps `SdlBroadcastReceiver` will not receive any Intent with the `ACL_CONNECTED` action and therefore will not know to start its own `SdlRouterService` when the device connects over bluetooth. In the event that all apps on the device support the `SdlDeviceListener` (SDL Library Version 4.12 and newer) we can update the `SdlBroadcastReceiver` to try to find the first app with these permissions granted and allow that app to start up their own `SdlRouterService`.

#### Library Changes
~~~ java
//SDLBroadcastReceiver.java
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

We also need to consider interactions with applications supporting older versions of the `SdlRouterService`. In the event there is an application on the phone with SDL Library version 4.11 or older, we might run into a situation where that application tries to start another applications `SdlRouterService` directly. If this happens and the application does not have Bluetooth Permissions granted then that `SdlRouterService` will not be able to start the `bluetoothTransport`. The suggested solution is to update the `initCheck` within the `SdlRouterService` to check the apps Bluetooth Permissions. If the permissions are not granted we can fail the `initCheck` and try to deploy the next `SdlRouterService`.

If a phone connects to the head unit over USB we can still start the `SdlRouterService` but if the designated app does not have Bluetooth Permissions the `bluetoothTransport` will still not be able to start. This could cause confusion for the user if they expect the `SdlRouterService` to connect over USB and Bluetooth but the `SdlRouterService` will only connect over USB. The suggested solution is to present a notification to the user reminding them to enable Bluetooth Permissions. By clicking on the notification the user will be directed to the apps settings page to grant those permissions. Meanwhile the `SdlRouterService` will wait to initialize the `bluetoothTransport` and will continuously check the permission status. Once the permissions are granted the `bluetoothTransport` will be started.

#### Library Changes
~~~ java
//SdlRouterService.java
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
~~~

### AndroidManifest Exported Flag

Starting in Android 12 any activities, services, or broadcast receivers that use intent filters will need to explicitly declare the `android:exported` attribute for the given app components. The `SdlRouterService` and `SdlReceiver` should already have the exported attribute defined and set to true. But The `USBAccessoryAttachmentActivity` will now also require this attribute to be set. Any activity that had an `intent-filter` would have a default exported value of true. Now we need to explicitly set it.

~~~ xml
<!--AndroidManifest.xml-->
<activity
    android:name="com.smartdevicelink.transport.USBAccessoryAttachmentActivity"
    android:exported="true" <!--New Addition-->
    android:launchMode="singleTop">
    <intent-filter>
        <action android:name="android.hardware.usb.action.USB_ACCESSORY_ATTACHED" />
    </intent-filter>
    .....
</activity>

~~~

### PendingIntent Mutable Flag

In Android 12, you must specify the mutability of each PendingIntent object that your app creates. This will impact the notifications that the `SdlRouterService` is trying to display, As we do not need to update these intents at any point we can flag them with `FLAG_IMMUTABLE`.

~~~ java
//SdlRouterService.java
int flag = android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ? PendingIntent.FLAG_IMMUTABLE : 0;
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, flag);

~~~

### Service Notification Delays

Starting in Android 12 when a service tries to present a notification, Android may delay showing the notification for up to 10 seconds. This is to try to allow the service to complete before the notification is presented. If we have connected to a system before we may want to present the `SdlRouterService` notifications immediately. App developers may also want to display the notifications related to the `SdlService` immediately. This can be achieved by setting the foregroundServiceBehavior flag to `Notification.FOREGROUND_SERVICE_IMMEDIATE`.

~~~ java
//SdlRouterSerivce.java
Notification.Builder builder =
new Notification.Builder(this, SDL_NOTIFICATION_CHANNEL_ID)
                    .setContentTitle("SmartDeviceLink")
                    .setContentText("Service Running")
                    .setForegroundServiceBehavior(Notification.FOREGROUND_SERVICE_IMMEDIATE);
notification = builder.build();
~~~

## Potential downsides

### Foreground Services

Using `PendingIntents` to start the apps `SdlService` from the `SdlRouterService` creates a potential security risk as the apps `SdlService` will now be required to have the `android:exported` attribute set to true. This could expose apps to have their `SdlService` be started by apps that are not SDL certified. We could also create a custom SDL `<permission>` to be used by apps as a requirement to be able to start the `SdlService` but nothing is stoping developers from listing that custom permission in their `AndroidManifest.xml`.   

### Bluetooth Runtime Permissions

With the new runtime permissions, users will have to grant bluetooth permissions to SDL enabled apps at runtime. This means that if the user denies permissions for a specific app, the app will not know when the device connects to the head unit over bluetooth nor will it be able to start its `bluetoothTransport` in the `SdlRouterService`. However, apps that have their bluetooth permissions denied will still be able to bind to another app's `SdlRouterService` and will still be able to start up a `SdlRouterService` for a USB connection only. If there are not any apps on the device with bluetooth permissions granted then the `SdlRouterService` would never be started when connecting over bluetooth.

### Service Notification Delays

This change would be only to make sure a notification is always displayed immediately otherwise android may choose to delay the notification up to 10 seconds. If the notification is delayed there is no risk to the user and could result in a better user experience as we can hide some of the initial `SdlRouterService` notifications while the `SdlRouterService` connects. If we choose to display the notification immediately the user experience will mirror its current implementation. For the `SdlService` it will be up to the individual app developers to implement the immediate flag.

## Impact on existing code

### Foreground Services

The `SdlRouterService` currently sends an intent out for each apps `SdlBroadcastReceiver` which notifies the apps to start the `SdlService`. If the app targets Android 12 and tries to start the `SdlService` from the background like this the app will crash. We will need to add a PendingIntent as an extra to this intent to be used by the `SdlReceiver`
These changes will require app developers to change how they implement the `SdlReceiver.onReceive` method to check the incoming intent for a pendingIntent and send that pendingIntent with the appropriate class name.
This implementation will also require the `SdlService` to have the `android:exported` attribute set to `true` in the `AndroidManifest.xml`.

### Bluetooth Runtime Permissions

The bluetooth logic in the library will now require these 2 new runtime permissions. With these permissions granted there is not any issue with the bluetooth logic but we now need to account for the use cases where a user has denied permissions. Specifically without these permissions the `SdlBroadcastReceiver` will never receive Intents with the `ACL_CONNECTED` action and the `SdlRouterService` will not be able to start the `bluetoothTransport`.

### AndroidManifest Exported Flag

These attributes were already defined where required. 

### PendingIntent Mutable Flag

The `SdlRouterService` will crash if the the app targets Android 12 and does not have this flag defined on the PendingIntent used for `SdlRouterService` notifications.

### Service Notification Delays

These notifications may be delayed by Android by up to 10 seconds

## Alternatives considered

### Foreground Services

Alternatives for the Foreground Service restrictions are limited. We either need to start the `SdlService` from a foreground context or the conditions need to meet one of the exceptions listed by Google. These conditions include:

* Starting the `SdlService` from an Activity.
* Starting the `SdlService` from user interaction with a notification
* Requesting the user ignores battery optimizations for each SDL Application

These options would either require an Activity to be launched for each SDL app, the user to interact with a notification for each SDL app, or for the user to choose battery optimization options for each app which then creates a situation where the user dictates if the given app will have its `SdlService` start when the `SdlRouterService` connects.

### Bluetooth Runtime Permissions

//TODO Discuss "optimal router service" solution and limitations

### PendingIntent Mutable Flag

The one case where PendingIntents were being used previously in the library we can set the flag to be `PendingIntent.FLAG_MUTABLE` but as we are handing the PendingIntent off to the notificationManager and the intent does not need to be changed at any point we should use `PendingIntent.FLAG_IMMUTABLE`.

### Service Notification Delays

We can leave the notification implementations unchanged and Android would simply delay displaying them if the service they are related to is still running.
