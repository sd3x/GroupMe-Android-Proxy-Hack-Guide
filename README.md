# How to Setup GroupMe apk for Use with Proxy
This guide will document how to use the proxy software [burp](https://portswigger.net/burp) to intercept network requests for the GroupMe Android apk. The GroupMe Android app uses https requests, so the burp CA Certificate must be injected into the apk. To accomplish this, [apktool](https://ibotpeaches.github.io/Apktool/) will be used to decompile the GroupeMe app so that the certificate can be added and the app configured to use it. The app will then be recompiled and signed using [uber apk signer](https://github.com/patrickfav/uber-apk-signer/releases/tag/v1.2.1) for installation. This guide will walk through the process of doing this, using the app with the burp proxy on the android emulator built into [android studio](https://developer.android.com/studio).
## Step 1: Burp Install and Obtaining the CA Certificate
Install Burp Suite Community Edition from the website linked in the beginning of the guide. Start burp using the default configuration, and navigate to the Proxy tab. Turn off Intercept, the Intercept feature captures and holds individual requests for manual forwarding/dropping. Then navigate to the Proxy Options. 

Add a new Proxy Listener. You can bind it to any port, this example will use port `42069` and set it to bind on all interfaces. Set the TLS protocols of the new Proxy Listener to use custom protocols, and leave only `TLSv1.1` selected. Add the new Proxy Listener.

On the computer with burp setup, navigate to `http://localhost:42069` to verify burp is running correctly and download the CA Certificate. It should save as `cacert.der`, however for use with Android, rename the file to `cacert.cer`. Leave burp running, though the certificate stays valid between burp uses.
## Step 2: Decompiling the GroupMe apk
This example will assume you have a GroupMe apk valid for the architecture of the android device or emulator you are using named `groupme.apk`. With apktool installed, run the command `apktool d groupme.apk` to decompile the app. This will create a new folder with the same name as the original apk, in this case `./groupme/`.
## Step 3: Injecting the CA Certificate

Navigate to the base app folder, `./groupme/`, and open `AndroidManifest.xml` in a text editor. Locate the `application` tag. It should look something like: `<application android:allowBackup="false" ... />` add the following attribute to the `application` tag: `android:networkSecurityConfig="@xml/network_security_config"`. Save and close the file.

Navigate to `./groupme/res/xml/` and create a new file named `network_security_config.xml` with the following contents:
```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
		<base-config cleartextTrafficPermitted="true">
		    <trust-anchors>
		        <certificates src="@raw/cacert"/>
		    </trust-anchors>
		</base-config>
</network-security-config>
```

Navigate to `./groupme/res/` and create a folder named `raw`. Copy `cacert.cer` from Step 1 into the new `raw` folder.
## Step 4: Recompiling & Signing
Recompile the apk using apktool:
`apktool b groupme -o groupme_unsigned.apk`

Copy the uber apk signer jar into the folder where the unsigned apk is and run:
`java -jar uber-apk-signer-1.2.1.jar -a groupme_unsigned.apk`

This will create a new signed apk named `groupme_unsigned-aligned-debugSigned.apk` which is the final apk ready for install.
## Step 5: Installation onto Android Studio Emulator
Launch Android Studio and use it to run an emulated device. If there is not one, you can create it using the AVD manager. More info on that [here](https://developer.android.com/studio/run/emulator).
Simply drag the apk file onto the phone screen window to install the apk. Before launching the app, go to the emulator's Extended Controls > Settings > Proxy. Set to use a manual proxy configuration with host `127.0.0.1` and port number `42069` and apply the settings. Launch the newly installed GroupMe app. You should see all sorts of requests in burp's Proxy HTTP History. From here, you can use the app as normal while viewing all the network requests going on.


