## Injecting Frida gadget into APKs
This is what has worked for me. Obviously this won't apply to all use cases but I have found that this is generally the process that I take.

Decompile the app using `apktool`.
```
apktool d appname.apk
```

Add the Frida gadget to the decompiled apk. You can find a gadget for your architecture [here](https://github.com/frida/frida/releases).

Put the gadget in `lib/[arch]/libfrida-gadget.so`

Open the `AndroidManifest.xml` and find the main activity path. It should look something like this:
```
<activity android:label="@string/app_name" android:name="com.packagename.path.to.MainActivity">
```

In `MainActivity.smali`, we need to inject `libfrida-gadget.so`. Ideally, we need to do it before anything else loads. We can load it using the following smali:

```
const-string v0, "frida-gadget"

invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
```

Which can be read as `System.loadLibrary("frida-gadget")`. It's important that this is done early in the app's lifecycle, so we can do it in the `MainActivity` static constructor. In the app that I am using, it looks like this:

```
.method static constructor <clinit>()V
    .locals 1 # this is the number of non-param registers
    ...
```

Insert the smali above in the beginning of the static constructor (after the `.locals` line if present).

Now we need to rebuild the app.

```
apktool b -o appname_patched.apk decompiledfolder
```

Sign the app

```
jarsigner -sigalg SHA1withRSA -digestalg SHA1 -keystore my.keystore appname_patched.apk keyname
jarsigner -verify appname_patched.apk
```

And zipalign.

```
zipalign 4 appname_patched.apk appname_patched_aligned.apk
```

Now we can install this on our target device and use your frida library of choice to poke around. :)
