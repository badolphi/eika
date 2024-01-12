# Prerequisites

To be able to follow along with the demos and work with the challenge, there is some software that you should install beforehand. How these tools can be installed is described below.

If you experience any issues during the setup, please send an email to benjamin@promon.no and we will try to assist you.

## Android Studio
Android Studio and many of the other tools require Java to be installed. So as a first step you should make sure that you have a recent JDK installed.

You can download the latest version of Android Studio for your platform here: [https://developer.android.com/studio](https://developer.android.com/studio). After installing it, launch it and go through the setup screen. After that, create a new project using the default settings.

We are now setting up an emulator to run the challenge in. Open "Tools" -> "Device Manager" go to the "Virtual" tab and click on "Create Device". On the "Select Hardware" screen, click "Next". On the "System Image" screen go to the “ARM Images” tab if you are on an ARM machine or to the “x86 Images” tab if you are on an Intel machine. Then click the download button besides the following image:

* Release Name: S
* API Level: 31
* ABI (ARM machine): arm64-v8a
* ABI (Intel machine): x86_64
* Target: Android 12.0

It is important to select the target “Android 12.0” and not “Android 12.0 (Google Play)” as only the images without Google Play services allow for root access. Also, if you get the option to install  "Google APIs", do not select that option.

After the image has been installed, click "Next" and then "Finish".

If you are not able to finish the setup of the emulator because of "The skin directory does not point to a valid skin", you should do the following: 
- Select advanced setting.
- Find "Custom skin definition" and select an option from the drop down
- The Finish button should now be clickable

You can test that the emulator works by clicking the play button besides the emulator entry in the Device Manager.

Lastly, you want to add the path to the Android platform tools to you `PATH` variable so you can access the tools conveniently from a terminal. To determine where the Android SDK is installed, go to "Android Studio" -> "Preferences" -> "Languages & Frameworks" -> "Android SDK" and check "Android SDK Location" if you are on macOS and to "File" -> "Settings" -> "Languages & Frameworks" -> "Android SDK" of you are on Linux or Windows. You then add `<sdk-location>/platform-tools` to your `PATH` variable.

You can test that everything is setup correctly by running this command: `adb shell su root id`. This should not give an error and you should get an output like this:

```
uid=0(root) gid=0(root) groups=0(root),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid) context=u:r:su:s0
```

## Jadx
The latest version of Jadx can be downloaded here: https://github.com/skylot/jadx/releases/download/v1.4.7/jadx-1.4.7.zip. You can test that it works by launching `bin/jadx-gui` on Linux or macOS or `bin/jadx-gui.bat` on Windows. There is not further setup required other than that.

## Ghidra
The latest version of Ghidra can be downloaded here: https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_10.3.3_build/ghidra_10.3.3_PUBLIC_20230829.zip. You can test that it works by launching `ghidraRun` on Linux or macOS and `ghidraRun.bat` on Windows. There is not further setup required other than that.

## Apktool
The latest version of Apktool can be downloaded here: https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.8.1.jar. You can test that it works by running `java -jar apktool_<version>.jar`. There is not further setup required other than that.

## Frida
Frida requires Python 3 to be installed. So as a first step, make sure that you have Python 3 installed on your system.

To download and install Frida on your computer, run the following command: `pip3 install frida-tools`.

We also want to install the Frida server on the emulator. You can download it here: https://github.com/frida/frida/releases. If you are running on an ARM machine, you want to download the file `frida-server-<version>-android-arm64.xz`. If you are running on an Intel machine, you want to download the file `frida-server-<version>-android-x86_64.xz`.

After download, you want to decompress the file (e.g. using the `xz` command: `xz -d frida-server-<version>-android-<arch>.xz`).

Then you can push the file to the emulator (make sure you have started the emulator): `adb push frida-server-<version>-android-<arch> /data/local/tmp/frida-server`.

The file needs to be executable, so run this command: `adb shell su root chmod +x /data/local/tmp/frida-server`.

To launch the server, run the following command: `adb shell su root /data/local/tmp/frida-server`. This will hang your terminal as long as the server runs.

You can now test if Frida works as expected by running the following command in a new terminal: `frida-trace -U -f com.android.deskclock`. This should not give any errors and launch the "Clock" app in the emulator.
