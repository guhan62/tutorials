---
copyright:
  years: 2017, 2018
lastupdated: "2017-11-27"

---

{:java: #java .ph data-hd-programlang='java'}
{:swift: #swift .ph data-hd-programlang='swift'}
{:ios: data-hd-operatingsystem="ios"}
{:android: data-hd-operatingsystem="android"}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}

# Mobile application with a serverless backend
In this tutorial, you will learn how to use Cloud Functions along with Cognitive and Data services to build a serverless backend for a mobile application. The application shown in this tutorial is a feedback app that smartly analyses the tone of the feedback text and appropriately acknowledges the customer through a Push Notification.
{:shortdesc}

This tutorial is configurable based on your target platform. You are currently viewing the documentation for the **iOS / Swift** version of this tutorial. Use the drop down switcher at the top of this documentation to select the **Android / Java** version of this tutorial.
{: swift}

This tutorial is configurable based on your target platform. You are currently viewing the documentation for the **Android / Java** version of this tutorial. Use the drop down switcher at the top of this documentation to select the **iOS / Swift** version of this tutorial.
{: java}

## Objectives

* Provision services to handle user authentication, feedback persistence and analysis.
* Configure push notifications.
* Deploy a serverless backend.
* Configure and run a native mobile application to collect user feedback.

## Products

This tutorial uses the following products:
   * [App ID](https://console.bluemix.net/catalog/services/AppID)
   * [Cloud Functions](https://console.bluemix.net/openwhisk)
   * [Cloudant NoSQL DB](https://console.bluemix.net/catalog/services/cloudantNoSQLDB)
   * [Tone Analyzer](https://console.bluemix.net/catalog/services/tone_analyzer)
   * [Push Notifications](https://console.bluemix.net/catalog/services/imfpush)

<p style="text-align: center;">
![](images/solution11/Architecture.png)
</p>

1. The user authenticates against [App ID](https://console.bluemix.net/catalog/services/AppID). App ID provides access and identification tokens.
2. Further calls to the backend API include the access token.
3. The backend is implemented with [Cloud Functions](https://console.bluemix.net/openwhisk). The serverless actions, exposed as Web Actions, expect the token to be sent in the request headers and verify its validity (signature and expiration date) before allowing access to the actual API.
4. When the user submits a feedback, the feedback is stored in [Cloudant](https://console.bluemix.net/catalog/services/cloudantNoSQLDB)
5. The feedback text is processed with [Tone Analyzer](https://console.bluemix.net/catalog/services/tone_analyzer).
6. Based on the analysis result, a notification is sent back to the user with [Push Notifications](https://console.bluemix.net/catalog/services/imfpush).
7. The user receives the notification on the device.

## Before you begin
{: #prereqs}

This tutorial uses the IBM Cloud command line tool to provision resources and deploy code. Make sure to install the `bx` command line tool.

* [IBM Cloud Developer Tools](https://github.com/IBM-Cloud/ibm-cloud-developer-tools) - Script to install bx CLI and required plug-ins (Cloud Foundry and Cloud Functions)

Additionally you will need the following software and accounts:

   1. Java 8
   2. Android Studio 2.3.3
   3. Google Developer account to configure Firebase Cloud Messaging
   4. Bash shell, cURL
   {: java}


   1. Xcode
   2. Apple Developer account to configure Apple Push Notification Service
   3. Bash shell, cURL
   {: swift}

In this tutorial, you will configure push notifications for the application. The tutorial assumes you have completed the basic Push Notifications tutorial for either [Android](./android-mobile-push-analytics.html) or [iOS](./ios-mobile-push-analytics.html) and you are familiar with the configuration of Firebase Cloud Messaging or Apple Push Notification Service.
{:tip}

For Windows 10 users to work with the command line instructions, we recommend installing the Windows Subsystem for Linux and Ubuntu as described in [this article](https://msdn.microsoft.com/en-us/commandline/wsl/install-win10).
{: tip}
{: java}

## Get the application code

The repository contains both the mobile application and the Cloud Functions actions code.

1. Checkout the code from the GitHub repository

   ```sh
   git clone https://github.com/IBM-Cloud/serverless-followupapp-android
   ```
   {: pre}
   {: java}

   ```sh
   git clone https://github.com/IBM-Cloud/serverless-followupapp-ios
   ```
   {: pre}
   {: swift}

2. Review the code structure

| File                                     | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| [**actions**](https://github.com/IBM-Cloud/serverless-followupapp-android/tree/master/actions) | Code for the Cloud Functions actions of the serverless mobile backend |
| [**android**](https://github.com/IBM-Cloud/serverless-followupapp-android/tree/master/android) | Code for the mobile application          |
| [**deploy.sh**](https://github.com/IBM-Cloud/serverless-followupapp-android/tree/master/deploy.sh) | Helper script to install, uninstall, update the Cloud Functions trigger, actions, rules |
{: java}

| File                                     | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| [**actions**](https://github.com/IBM-Cloud/serverless-followupapp-ios/tree/master/actions) | Code for the Cloud Functions actions of the serverless mobile backend |
| [**followupapp**](https://github.com/IBM-Cloud/serverless-followupapp-ios/tree/master/followupapp) | Code for the mobile application          |
| [**deploy.sh**](https://github.com/IBM-Cloud/serverless-followupapp-ios/blob/master/deploy.sh) | Helper script to install, uninstall, update the Cloud Functions trigger, actions, rules |
{: swift}

## Provision services to handle user authentication, feedback persistence and analysis
{: #provision_services}

In this section, you will provision the services used by the application. You can choose to provision the services from the IBM Cloud catalog or using the `bx` command line.

It is recommended that you create a new space to provision the services and deploy the serverless backend. This helps to keep all the resources together.

### Provision services from the IBM Cloud catalog

1. Go to the [IBM Cloud catalog](https://console.bluemix.net/catalog/)
2. Create a [Cloudant NoSQL DB](https://console.bluemix.net/catalog/services/cloudantNoSQLDB) service with the **Lite** plan. Set the name to **serverlessfollowup-db**.
3. Create a [Watson Tone Analyzer](https://console.bluemix.net/catalog/services/tone_analyzer) service with the **Standard** plan. Set the name to **serverlessfollowup-tone**.
4. Create an [App ID](https://console.bluemix.net/catalog/services/AppID) service with the **Graduated tier** plan. Set the name to **serverlessfollowup-appid**.
5. Create a [Push Notifications](https://console.bluemix.net/catalog/services/imfpush) service with the **Lite** plan. Set the name to **serverlessfollowup-mobilepush**.

### Provision services from the command line

With the command line, run the following commands to provision the services and retrieve their credentials:

   ```sh
   bx cf create-service cloudantNoSQLDB Lite serverlessfollowup-db
   ```
   {: pre}

   ```sh
   bx cf create-service-key serverlessfollowup-db for-cli
   ```
   {: pre}

   ```sh
   bx cf service-key serverlessfollowup-db for-cli
   ```
   {: pre}

   ```sh
   bx cf create-service tone_analyzer standard serverlessfollowup-tone
   ```
   {: pre}

   ```sh
   bx cf create-service-key serverlessfollowup-tone for-cli
   ```
   {: pre}

   ```sh
   bx cf service-key serverlessfollowup-tone for-cli
   ```
   {: pre}

   ```sh
   bx cf create-service AppID "Graduated tier" serverlessfollowup-appid
   ```
   {: pre}

   ```sh
   bx cf create-service-key serverlessfollowup-appid for-cli
   ```
   {: pre}

   ```sh
   bx cf service-key serverlessfollowup-appid for-cli
   ```
   {: pre}

   ```sh
   bx cf create-service imfpush lite serverlessfollowup-mobilepush
   ```
   {: pre}

   ```sh
   bx cf create-service-key serverlessfollowup-mobilepush for-cli
   ```
   {: pre}

   ```sh
   bx cf service-key serverlessfollowup-mobilepush for-cli
   ```
   {: pre}

## Configure push notifications
{: #push_notifications}

When a user submits a new feedback, the application will analyze this feedback and send back a notification to the user. The user may have moved to another task, or may not have the mobile app started so using push notifications is a good way to communicate with the user. The Push Notifications service makes it possible to send notifications to iOS or Android users via one unified API. In this section, you will configure the Push Notifications service for your target platform.

### Configure Firebase Cloud Messaging (FCM)
{: java}

   1. In the [Firebase console](https://console.firebase.google.com), create a new project. Set the name to **serverlessfollowup**
   2. Navigate to the Project **Settings**
   3. Under the **General** tab, add two applications:
      1. one with the package name set to: **com.ibm.mobilefirstplatform.clientsdk.android.push**
      2. and one with the package name set to: **serverlessfollowup.app**
   4. Download the `google-services.json` containing the two defined applications from Firebase console and place this file in the `android/app` folder of the checkout directory.
   5. Find the Sender ID and Server Key (also called API Key later on) under the **Cloud Messaging** tab.
   6. In the Push Notifications service dashboard, set the value of the Sender ID and API Key.
   {: java}

### Configure Apple Push Notifications Service (APNs)
{: swift}

1. Go to the [Apple Developer![External link icon](https://console.bluemix.net/docs/api/content/icons/launch-glyph.svg?lang=en?lang=en)](https://developer.apple.com/) portal and Register an App ID.
2. Create a development and distribution APNs SSL certificate.
3. Create a development provisioning profile.
4. Configure the Push Notifications service instance on IBM Cloud. Refer to [Obtain APNs credentials and configure Push Notifications service](https://console.bluemix.net/docs/tutorials/ios-mobile-push-analytics.html#obtain-apns-credentials-and-configure-push-notifications-service-instance-) for detailed steps.
{: swift}

## Deploy a serverless backend
{: #serverless_backend}

With all the services configured, you can now deploy the serverless backend. The following Cloud Functions artifacts will be created in this section:

| Artifact                      | Type                           | Description                              |
| ----------------------------- | ------------------------------ | ---------------------------------------- |
| `serverlessfollowup`          | Package                        | A package to group the actions and to keep all service credentials |
| `serverlessfollowup-cloudant` | Package Binding                | Bound to the built-in Cloudant package   |
| `serverlessfollowup-push`     | Package Binding                | Bound to the Push Notifications package  |
| `auth-validate`               | Action                         | Validates the access and identification tokens |
| `users-add`                   | Action                         | Persists the user information (id, name, email, picture, device id) |
| `users-prepare-notify`        | Action                         | Formats a message to use with Push Notifications |
| `feedback-put`                | Action                         | Stores a user feedback in the database   |
| `feedback-analyze`            | Action                         | Analyzes a feedback with Tone Analyzer   |
| `users-add-sequence`          | Sequence exposed as Web Action | `auth-validate` and `users-add`          |
| `feedback-put-sequence`       | Sequence exposed as Web Action | `auth-validate` and `feedback-put`       |
| `feedback-analyze-sequence`   | Sequence                       | `read-document` from Cloudant, `feedback-analyze`, `users-prepare-notify` and `sendMessage` with Push Notifications |
| `feedback-analyze-trigger`    | Trigger                        | Called by Cloud Functions when a feedback is stored in the database |
| `feedback-analyze-rule`       | Rule                           | Links the trigger `feedback-analyze-trigger` with the sequence `feedback-analyze-sequence` |

### Compile the code
{: java}
1. From the root of the checkout directory, compile the actions code
{: java}

   ```sh
   ./android/gradlew -p actions clean jar
   ```
   {: pre}
   {: java}

### Configure and deploy the actions
{: java}

2. Copy template.local.env to local.env

   ```sh
   cp template.local.env local.env
   ```
3. Get the credentials for Cloudant, Tone Analyzer, Push Notifications and App ID services from the IBM Cloud dashboard (or the output of the bx commands we ran before) and replace placeholders in `local.env` with corresponding values. These properties will be injected into a package so that all actions can get access to the database.
4. Deploy the actions to Cloud Functions. `deploy.sh` loads the credentials from `local.env` to create the Cloudant databases (users, feedback and moods) and deploy the Cloud Functions artifacts for the application.
   ```sh
   ./deploy.sh --install
   ```
   {: pre}

   You can use `./deploy.sh --uninstall` to remove the Cloud Functions artifacts once you have completed the tutorial.
   {: tip}

## Configure and run a native mobile application to collect user feedback
{: #mobile_app}

Our Cloud Functions actions are ready for our mobile app. Before running the mobile app, you need to configure its settings to target the services you created.

1. With Android Studio, open the project located in the `android` folder of your checkout directory
2. Edit `android/app/src/main/res/values/credentials.xml` and fill in the blanks with values from credentials. You will need the App ID `tenantId`, the Push Notification `appGuid` and `clientSecret` and the organization and space names where the Cloud Functions have been deployed.
3. Build the project
4. Start the application on a real device or with an emulator.
   For the emulator to receive push notifications, make sure to pick an image with the Google APIs and to log in with a Google account within the emulator.
   {: tip}
5. Watch the Cloud Functions in the background
   ```sh
   bx wsk activation poll
   ```
   {: pre}
6. In the application, select **Log in** to authenticate with a Facebook or Google account. Once logged in, type a feedback message and press the **Send Feedback** button. Few seconds after the feedback has been sent, you should receive a push notifications on the device. The notification text can be customized by modifying the template documents in the `moods` database in the Cloudant service instance. Use the **View token** button to inspect the access and identification tokens generated by App ID upon login.
{: java}


1. Push client SDK and other SDKs are available on CocoaPods and Carthage. For this solution, let's use CocoaPods
2. Open Terminal and `cd ` into `followupapp` folder. Run the below command to install the required dependencies.
   ```sh
   pod install
   ```
   {: pre}
3. Open the file with  `.xcworkspace` extension located under the `followupapp` folder of your checkout directory to launch your code in Xcode.
4. Edit `BMSCredentials.plist file` and fill in the blanks with values from credentials. You will need the App ID `tenantId`, the Push Notification `appGuid` and `clientSecret` and the organization and space names where the Cloud Functions have been deployed.
5. Build the project.
6. Start the application on a real device or with a simulator.
7. Watch the Cloud Functions in the background by running the below command on a Terminal.
   ```sh
   bx wsk activation poll
   ```
   {: pre}
8. In the application, select **Log in** to authenticate with a Facebook or Google account. Once logged in, type a feedback message and press the **Send Feedback** button. Few seconds after the feedback has been sent, you should receive a push notifications on the device. The notification text can be customized by modifying the template documents in the `moods` database in the Cloudant service instance. Use the **View token** button to inspect the access and identification tokens generated by App ID upon login.
{: swift}

## Clean up resources

1. Use `deploy.sh` to remove the Cloud Functions artifacts:

   ```sh
   ./deploy.sh --uninstall
   ```
   {: pre}

2. Delete the Cloudant, App ID, Push Notifications and Tone Analyzer services from the IBM Cloud console.

## Related information

* App ID provides a default configuration to help with the initial set up of your identity providers. Prior to publishing your app, [update the configuration to your own credentials](../services/appid/identity-providers.html). You will also be able to [customize the login widget](../services/appid/login-widget.html#login-widget).


* When you create an OpenWhisk Swift action with a Swift source file(.swift files under `actions` folder), it has to be compiled into a binary before the action is run. Once done, subsequent calls to the action are much faster until the container that holds your action is purged. This delay is known as the cold-start delay.
  To avoid the cold-start delay, you can compile your Swift file into a binary and then upload to OpenWhisk in a zip file. As you need the OpenWhisk scaffolding, the easiest way to create the binary is to build it within the same environment it runs in. refer [Package an Action as a Swift executable](https://console.bluemix.net/docs/openwhisk/openwhisk_actions.html#creating-swift-actions) for further steps.
{: swift}
