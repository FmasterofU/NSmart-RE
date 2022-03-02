# NSmart-RE
Reverse engineering NSmart API

[NSmart Google Play page](https://play.google.com/store/apps/details?id=buslogic.nsmartapp&hl=en&gl=US)

[APK online downloader - NSmart (latest)](https://apkcombo.com/apk-downloader/#package=buslogic.nsmartapp)

[NSmart 4.6.apk](Nsmart_4.6.apk)


## Strings
[All strings contained in APK](nsmart-strings.txt)
```bash
strings Nsmart_4.6.apk > nsmart-strings.txt
```

Since the APK is (obviously) a Java archive (essentially a ZIP file), we open it and primarly take a look at three files:
 - `classes.dex`,
 - `classes2.dex` and
 - `resources.arsc`.

Running `strings` on those three files enables us to easily extract some REST API endpoints and classpaths. They are available [here](nsmart-strings-extracted-path.txt).

## Decompiling

Tools:
  - 
