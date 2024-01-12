 # Demos
This is a detailed summary of steps taken during the demos. If you haven't done so already, you should install the prerequisites described in `Prerequisites.md`. The app used to demo the attacks is `app.apk`.

## Reverse engineering

As a first step to understand what the app does, you can take a quick look at which files are in the `app.apk` file. It is a zip file so any tool that can read zip files can be used. You will see that there are a lot of files in there that are not too interesting like meta information and resources. But you also see that there is a file called `classes.dex` which contains all the Java code of the app as well as the files `lib/arm64-v8a/libdemo.so` and `lib/x86_64/libdemo.so` which contain the native code of the app depending on the architecture of the device the app runs on. Another interesting file is `AndroidManifest.xml`, which is a binary xml file that has important metadata about the app.

As a first step to understand what the app does, you can just install it on your emulator.

You can run the emulator from Android Studio by creating a new dummy project. Then you click on "Tools" -> "Device Manager". There you should see the emulator you have setup as a prerequisite. You can launch it by clicking the green play button.

Once the emulator has started, you can install the app with the following command: `adb install app.apk`.

Now you should see the demo app installed on the device and should be able to launch it.

The demo app takes a password and has a button to check it either in Java or native code. If you press one of the buttons, you get a message telling you if the password was correct or not.

Now that we understand what the app is doing, we can see if we can find out how the password is checked and potentially what the correct password is.

We first look into the Java code. This can be done with the Jadx tool that you should already have installed. You can launch it by running `bin/jadx-gui` on macOS and Linux or `bin/jadx-gui.bat` on Windows.

To open the app, you click "Open File" and then select the `app.apk` file. On the left side under "Source Code", you will see all the Java code of the app. Most of the code is not interesting as it is from either Google or the IDE that was used to build the app. The only thing that sticks out is the `no.promon.demoapp` package. Looking through the code of the classes in the package, you can see that the interesting functionality happens in the class `DemoActivity`. This class installs listeners on the buttons to get notified when they get clicked. They then get the content of the password text field and check if with either `checkPasswordJava` or `checkPasswordNative` depending on which button was clicked. And depending on the return value of these check methods, a dialog is displayed that either days "Correct!" or "Wrong!". So the interesting functionality is really in `checkPasswordJava` and `checkPasswordNative`. Looking at what `checkPasswordJava`, it should be quite obvious that the password for the Java check is `secret`. You can verify this by entering the password in the emulator. For `checkPasswordNative`, we do not have an implementation and the method is marked as `native`. This is because it is implemented in native code, which we will look at next.

We will check the native code using Ghidra, which you should have installed already. You can launch it by running `ghidraRun` on macOS and Linux and `ghidraRun.bat` on Windows.

The first thing you have to do is to create a project for the analysis. This can be done by going to "File" -> "New Project", clicking "Next", giving the project a name and then clicking "Finish".

You can now drag the `app.apk` file into the file view in the middle. Since it is a zip file, it asks you if you want to open the zip file itself or files inside the zip file. The easiest thing to do is to select "Batch" and then click "OK", which will import everything it can read from the zip file. You can now select a file you want to look at. To do that, you can double click on either `lib/arm64-v8a/libdemo.so` or `lib/x86_64/libdemo.so` and then on "Yes" and "Analyze".

We want to look at the code of the app so under "Symbol Tree" on the left, you select "Functions" and then you should see the function that implements the `checkPasswordNative` method (`Java_no_promon_demoapp_DemoActivity_checkPasswordNative`). When you click on it, you should see the decompiled code of the method. If not, you can click on "Window" -> "Decompile Java_no_promon_demoapp_DemoActivity_checkPasswordNative". The function needs to convert the Java string it receives as an argument (the password) to a C string but you do not need to understand that part. The important line is `iVar1 = strcmp(__s1,"supersecret");` where it compares the C version of the password against the expected password `supersecret`. You can again verify this by typing this password in the emulator.
## Repackaging

Now that we know how the app checks the password, we can think about how to manipulate it to do something it was not intending of doing like accepting any password. This will be different depending on what kind of code you want to patch.
### Java

If we want the app to accept any password for the Java check, we could for example modify the `checkPasswordJava` method to always return `true` instead of checking the password.

To make modifications to the app, we use apktool, which you should have installed already.

We first decompile the app using apktool:
```bash
java -jar apktool_2.8.1.jar d --no-res app.apk
```

This tells apktool to decompile the app but not to process the ressources (which it has problems with in the case of that app).

You now have a new folder called `app`. That folder contains a folder called `smali` which contains the smali version of the app. We want to patch the code in the `DemoActivity` class, so you want to open the file `app/smali/no/promon/demoapp/DemoActivity.smali` in a text editor.

This file contains smali code, which you do not really need to understand to much for now. Everything needed to know to solve the challenge is explained here. In that file, you find lines starting with `.method`, which mark the beginning of a method. We want to patch the `checkPasswordJava` method, so look for this line: `.method private checkPasswordJava(Ljava/lang/String;)Z`. Below that line, you have the code of the method:
```
.method private checkPasswordJava(Ljava/lang/String;)Z
    .locals 1

    const-string v0, "secret"

    .line 20
    invoke-virtual {p1, v0}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result p1

    return p1
.end method
```

This code represents the bytecode instructions of that method. The bytecode is similar to assembly code in that you have different instructions and the instructions work on temporary variables (registers). You do not need to understand that code too much but to give you a feeling what it does, here is a description of its instructions:

The line `const-string v0, "secret"` puts the constant string `"secret"` into a register called `v0`.

The line `invoke-virtual {p1, v0}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z` calls the non static method `equals` (which returns a boolean) from the `java.lang.String` class on the object in register `p0` (the first argument to the method) with the content of the `v0` register (that was set in the line above) as an argument.

The line `move-result p1` moves the result of calling the method into the `p1` register.

The line `return p1` returns the value in the `p1` register to the caller.

We would like to patch this method to always return true. This can be done by modifying it like so:
```
.method private checkPasswordJava(Ljava/lang/String;)Z
    .locals 1
    const/4 v0, 0x1
    return v0
.end method
```

Here, we move the integer constant `0x1` (true) into the `v0` register and then return the value of that register to the caller.

After we have modified the code, we need to compile it to an apk file again. We can use apktool for this:
```bash
java -jar apktool_2.8.1.jar b app -o app-patched.apk
```

This will build a new apk called `app-patched.apk` with the contents of the `app` folder. You can verify that our patch did what we want it to by loading this file into Jadx.

Before we can install the app on our emulator to test it, we need to zipalign and sign it. For that, we need two tools (`zipalign` and `apksigner`) that are part of the Android SDK. To be able to launch them, you need to find out where they are installed. This can be done by first finding out where your Android SDK is installed. You can do this by launching Android Studio and opening it's settings. You then go to "Appearance & Behavior" -> "System Settings" -> "Android SDK". You should see the path besides "Android SDK Location".

The tools we need are located in the `build-tools/<some version>/` folder in the folder where the Android SDK is installed.

To zipalign the patched app, you run this command (in my case, the `zipalign` command is found in the `/opt/android/android-sdk/build-tools/34.0.0` folder, but this is likely different in your case):
```bash
/opt/android/android-sdk/build-tools/34.0.0/zipalign -p -v 4 app-patched.apk app-aligned.apk
```

This command takes the patched version of the app `app-patched.apk` as input and outputs the aligned version as `app-aligned.apk`.

To sign the app, you need a keystore which contains a certificate to sign with. You can use the keystore in this repository for that (`debug.keystore`).

You can then sign the app with this command (in my case, the `apksigner` command is found in the `/opt/android/android-sdk/build-tools/34.0.0` folder, but this is likely different in your case):
```bash
/opt/android/android-sdk/build-tools/34.0.0/apksigner sign --ks ../debug.keystore --ks-key-alias androiddebugkey --ks-pass pass:android --out app-signed.apk app-aligned.apk
```

This command takes the aligned version of the app `app-aligned.apk` as input and outputs the signed version as `app-signed.apk`.

You can now install this app in the emulator: `adb install -r app-signed.apk`. Please note the `-r` option that tells the device to install the app even if it is already installed. You should see that you can enter any password and the Java check will succeed.

### Native

If we want the app to accept any password for the Java check, we could for example modify the  `Java_no_promon_demoapp_DemoActivity_checkPasswordNative` function to always return `true` instead of checking the password.

To make modifications to the app, we use apktool, which you should have installed already.

We first decompile the app using apktool:
```bash
java -jar apktool_2.8.1.jar d --no-res app.apk
```

This tells apktool to decompile the app but not to process the ressources (which it has problems with in the case of that app).

You now have a new folder called `app`. That folder contains a folder called `lib` which contains the native libraries of the app. It is important that you patch the correct library that matches the architecture of your emulator (and your computer). If you are using an ARM based machine, you want to patch `libdemo.so` in the `arm64-v8a` folder. If you are using an Intel based machine, you want to patch `libdemo.so` in the `x86_64` folder.

To patch one of the libraries, you launch Ghidra and create a new project like before. Then you drag the library you want to patch into it. You can now open it. Then look for the `Java_no_promon_demoapp_DemoActivity_checkPasswordNative` function under "Symbol Tree" -> "Functions". We want to make that function always return true, so we need to patch the assembly code. We do that by first switching to the assembly view by going to "Window" -> "Listing: libdemo.so".

If you have opened the `arm64-v8a` version of the library, you should see something like this:

```
                    ********************************************
                    *                 FUNCTION                 *
                    ********************************************
                    undefined Java_no_promon_demoapp_DemoAc
         undefined    w0:1      <RETURN>
         undefined8   Stack[-0x local_10                    XREF[2]: 00100644(W), 
                                                                     00100698(*)  
         undefined8   Stack[-0x local_20                    XREF[2]: 00100640(W), 
                                                                     0010069c(R)  
         undefined8   Stack[-0x local_30                    XREF[1]: 0010063c(W)  
                    Java_no_promon_demoapp_DemoActiv  XREF[3]: Entry Point(*), 
                                                               001006c4, 001006f0(*)  
   0010063c f6 57      stp     x22,x21,[sp, #local_30]!
            bd a9
   00100640 f4 4f      stp     x20,x19,[sp, #local_20]
            01 a9
   00100644 fd 7b      stp     x29,x30,[sp, #local_10]
            02 a9
...
```

You do not really need to understand that code (what I am demonstrating here is enough to solve the challenge). The only thing we want to do is to patch the first instructions to always return true.

We do this by first right clicking on the first instruction `stp     x22,x21,[sp, #local_30]!` and selecting "Patch instruction". We then change the instruction `stp` to `mov` and the operands `x22,x21,[sp, #-0x30]!` to `x0, #1`. This will move the value 1 (true) into the register that is used to return values on arm64. We then just need to return so we do not run the rest of the code of the function. So we right click on the next line `stp     x20,x19,[sp, #local_20]` and again select "Patch instruction". We change the instruction `stp` to `ret` and the operands from `x20,x19,[sp, #local_20]` to an empty string. Your patched version should now look like this:

```
                      ************************************************
                      *                   FUNCTION                   *
                      ************************************************
                      undefined Java_no_promon_demoapp_DemoActivit
          undefined     w0:1       <RETURN>
          undefined8    Stack[-0x1 local_10                       XREF[2]:  00100644(W), 
                                                                             00100698(*)  
          undefined8    Stack[-0x2 local_20                       XREF[1]:  0010069c(R)  
          undefined8    Stack[-0x3 local_30
                      Java_no_promon_demoapp_DemoActivity_  XREF[3]:  Entry Point(*), 001006c4, 
                                                                      001006f0(*)  
     0010063c 20 00       mov      x0,#0x1
             80 d2
     00100640 c0 03       ret
             5f d6
     00100644 fd 7b       stp      x29,x30,[sp, #local_10]
             02 a9
```

If you have opened the `x86_64` version of the library, you should see something like this:
```
                      ************************************************
                      *                   FUNCTION                   *
                      ************************************************
                      undefined Java_no_promon_demoapp_DemoActivit
          undefined     AL:1       <RETURN>
                      Java_no_promon_demoapp_DemoActivity_  XREF[3]:  Entry Point(*), 
                                                                      00100708(*), 001007a0  
     00100610 55          PUSH     RBP
     00100611 41 57       PUSH     R15
     00100613 41 56       PUSH     R14

...
```

You do not really need to understand that code (what I am demonstrating here is enough to solve the challenge). The only thing we want to do is to patch the first instructions to always return true.

We do this by first right clicking on the first instruction `PUSH     RBP` and selecting "Patch instruction". We then change the instruction `PUSH` to `MOV` and the operand `RBP` to `EAX,1`. This will move the value 1 (true) into the register that is used to return values on Intel. We then just need to return so we do not run the rest of the code of the function. So we right click on the next line `PUSH     R15` and again select "Patch instruction". We change the instruction `PUSH` to `RET` and the operands from `R15` to an empty string. Your patched version should now look like this:

```
                      ************************************************
                      *                   FUNCTION                   *
                      ************************************************
                      undefined Java_no_promon_demoapp_DemoActivit
          undefined     AL:1       <RETURN>
                      Java_no_promon_demoapp_DemoActivity_  XREF[3]:  Entry Point(*), 
                                                                      00100708(*), 001007a0  
     00100610 b8 01       MOV      EAX,0x1
             00 00 00
     00100615 c3          RET
     00100616 50          PUSH     RAX
```

In both cases, if you look at the decompiled code of your changes (by going to "Window" -> "Decompile Java_no_promon_demoapp_DemoActivity_checkPasswordNative") you should see that the function now always returns 1 (true).

You can now export your changes by going to "File" -> "Export Program". Then you select the Format "Original File" and as the output file, you select the path to the file you have loaded into Ghirda (be careful to select the correct architecture folder).

You can now create a new version of the apk by using apktool like before:

```bash
java -jar apktool_2.8.1.jar b app -o app-patched.apk
```

And then you need to zipalign and sign the file like before (make sure to adjust the path of the tools):
```bash
/opt/android/android-sdk/build-tools/34.0.0/zipalign -p -v 4 app-patched.apk app-aligned.apk
/opt/android/android-sdk/build-tools/34.0.0/apksigner sign --ks ../debug.keystore --ks-key-alias androiddebugkey --ks-pass pass:android --out app-signed.apk app-aligned.apk
```

You can now install this app in the emulator: `adb install -r app-signed.apk`. Please note the `-r` option that tells the device to install the app even if it is already installed. And you should see that you can enter any password and the Native check will succeed.
# Hooking

Instead of patching the app on disk like in the repackaging case, we can also patch the app at runtime using the hooking framework Frida. Also here, our goal will be to patch the app to accept any password. This will be different depending on what kind of code you want to patch.

In both cases though, we want to first run the frida-server that you should already have installed on the device as instructed in the prerequisites. You can run it like so:
```bash
adb shell su root /data/local/tmp/frida-server
```

This command will hang your terminal as long as the server runs.

Also, if you have previously installed a patched version of the demo app, you want to uninstall it and re-install the original app:
```bash
adb uninstall no.promon.demoapp
adb install app.apk
```
### Java

For modifying the Java part, we instruct Frida to place a hook in the `checkPasswordJava` method at runtime to always return true. Frida scripts are written in JavaScript. Create a file with the following content:

```js
Java.perform(function()
{
    var activity = Java.use("no.promon.demoapp.DemoActivity");
    activity.checkPasswordJava.implementation = function (password)
    {
        console.log("Hook!");
        return true;
    };
});
```

This code does the following: It first calls `Java.perform` to make sure that it can work in a Java VM. This is required for all manipulations to Java code. Inside the function that runs in the Java VM, we first get the class we want to hook by calling `Java.use` with the name of the class which we can get from Jadx. Once we have the class, we can replace the implementation of one of its methods with our own. In our case, this is the `checkPasswordJava` method. What we do here is to assign a function that takes one `password` argument to the implementation of the `checkPasswordJava` method. Inside the method, we log a message so we know that our code was called and then we return true.

You can now test that the script works. For that, first launch the demo app on the emulator and make sure that it does not accept arbitrary passwords for the Java check. After that, leave the app running in the foreground. You then instruct Frida to inject and run the script in the app:
```bash
frida -U -F -l hook_java.js
```

This tells Frida to connect to the device via USB (the emulator is seen as a USB device). For this to work, make sure you do not have any phones connected to your computer via USB. In addition, we tell it to attach to the application that is currently in the foreground and we tell it to inject and run the script we wrote (which I call `hook_java.js` but you can call it whatever you like).

After that, you should see that Frida is launched in your terminal and you should not see any error messages. You can then try to enter an arbitrary password in the app and you should see the Java check succeed.
### Native

For modifying the Native part, we instruct Frida to place a hook in the `Java_no_promon_demoapp_DemoActivity_checkPasswordNative` function at runtime to always return true. Create a file with the following content:

```js
var module = Process.findModuleByName("libdemo.so");
var address = module.findExportByName("Java_no_promon_demoapp_DemoActivity_checkPasswordNative");

Interceptor.attach(address,
{
    onEnter: function(args)
    {
    },
    onLeave: function(ret)
    {
        console.log("Hook!");
        ret.replace(1);
    }
});
```

This code does the following: It first needs to find the library we want to modify in memory. This can be done with the `Process.findModuleByName` API. Once it has found the library, it can find the address of function we want to hook. This is done with the `findExportByName` API. Once we have found the address, we can use the `Interceptor.attach` API to put a hook at the address of the function. That method gets the address where we want to hook as well as an object as arguments. That object can provide a callback called `onEnter` for when the the function is entered (allowing you to inspect and modify the arguments) and a callback called `onLeave` for when the function is exited (allowing you to inspect and modify the return value). We use the `onLeave` callback to log that we have intercepted the function as well to replace the return value with 1 (true).

You can now test that the script works. For that, first launch the demo app on the emulator and make sure that it does not accept arbitrary passwords for the Native check. After that, leave the app running in the foreground. You then instruct Frida to inject and run the script in the app like in the Java case:

```bash
frida -U -F -l hook_native.js
```

After that, you should see that Frida is launched in your terminal and you should not see any error messages. You can then try to enter an arbitrary password in the app and you should see the Native check succeed.
