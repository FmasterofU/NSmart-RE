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

Since the APK is (obviously) a Java archive (essentially a ZIP file), we can extract it and take a look primarly at these three files:
 - `classes.dex`,
 - `classes2.dex` and
 - `resources.arsc`.

Running `strings` on those three files enables us to easily extract some REST API endpoints and classpaths. They are available [here](nsmart-strings-extracted-path.txt).

## Decompiling and analysis

Tools:
  - [Dex2jar](https://github.com/pxb1988/dex2jar)
  - [Java decompiler](https://github.com/java-decompiler/jd-gui)

Converting APK to standard Java Archive:
```ps1
.\dex2jar-2.1\dex-tools-2.1\d2j-dex2jar.bat -f -o nsmart.jar Nsmart_4.6.apk
```

Open the `nsmart.jar` file in the Java decompiler. Sample REST API calls are:
 - in `buslogic.nsmartapp.api.CitiesExtendedApi.class`
   ```java
   ...
   public CitiesExtendedApi(String paramString1, String paramString2) {
     this.companyApiKey = paramString1;
     this.companyUrl = paramString2;
   }

   public void getCitiesExtended() {
     OkHttpClient okHttpClient = new OkHttpClient();
     HttpUrl.Builder builder = HttpUrl.get(this.companyUrl + "/publicapi/v1/networkextended.php").newBuilder();
     builder.addQueryParameter("action", "get_cities_extended");
     Request request = (new Request.Builder()).url(builder.build()).addHeader("X-Api-Authentication", this.companyApiKey).build();
     okHttpClient.newBuilder().connectTimeout(7L, TimeUnit.SECONDS);
     okHttpClient.newBuilder().readTimeout(20L, TimeUnit.SECONDS);
     okHttpClient.newCall(request).enqueue(new Callback() { ... });
   }
   ...
   ```
 - in `buslogic.nsmartapp.api.AnnouncementApi.class`
   ```java
   ...
   public AnnouncementApi(String paramString1, String paramString2, int paramInt) {
     this.companyApiKey = paramString1;
     this.stationId = paramInt;
     this.companyUrl = paramString2;
   }
   
   ...

   public void getAnnouncements() {
     OkHttpClient okHttpClient = new OkHttpClient();
     HttpUrl.Builder builder = HttpUrl.get(this.companyUrl + "/publicapi/v1/announcement/announcement.php").newBuilder();
     builder.addQueryParameter("ibfm", "TM00000");
     builder.addQueryParameter("station_uid", "" + this.stationId);
     Request request = (new Request.Builder()).url(builder.build()).addHeader("X-Api-Authentication", this.companyApiKey).build();
     okHttpClient.newBuilder().connectTimeout(4L, TimeUnit.SECONDS);
     okHttpClient.newBuilder().readTimeout(10L, TimeUnit.SECONDS);
     okHttpClient.newCall(request).enqueue(new Callback() { ... });
   }
   ...
   ```

All the API call require an Authentication key, so we search for the code where its value is set. The relevant code is in `buslogic.nsmartapp.ui.SplashActivity.class`:
```java
...
import android.app.Activity;
...

public class SplashActivity extends Activity {
  ...
  
  private String companyApiKey;
  
  private String companyUrl;
  
  ...
  
  protected void onCreate(Bundle paramBundle) {
    super.onCreate(paramBundle);
    setContentView(2131427366);
    this.sharedPrefVersion = getSharedPreferences("com.example.testjedan.version", 0);
    this.mDatabase = ((BasicApp)getApplication()).getDatabase();
    this.companyApiKey = getString(2131820660);
    this.companyUrl = getString(2131820664);
    this.screenSplashDuration = Integer.parseInt(getString(2131820967));
    hideSystemUI();
    VersionApi versionApi = new VersionApi(this.companyApiKey, this.companyUrl);
    versionApi.setCallBack(new SplashActivity$$ExternalSyntheticLambda1(this));
    versionApi.getVersion();
  }
...
}
```

The line `this.companyApiKey = getString(2131820660);` represents an superclass (`android.app.Activity`) call to the application Context, where the parameter `2131820660` (hex `7F110074`) is the resource ID of the API key located in the Application resources.
Moreover, we can see that the same number is declared as a constant in the `buslogic.nsmartapp.R.class`:
```java
public final class R {
...
  public static final class string {
    ...
    public static final int company_api_key = 2131820660;
    ...
    public static final int company_url = 2131820664;
    ...
  }
...
}
```
These constant names can be a clue to the actual resource names of the sought after values (and, as it turns out, they are the same). 

## Resource extraction

Tools:
 - [APKtool](https://github.com/iBotPeaches/Apktool)

```ps1
.\apktool_2.6.1.jar d '.\Nsmart_4.6.apk'
```

We position ourselves to the `res` (resources) foler of the generated directory, and recursively search for the hex value of the company api key resource ID:
```bash
grep -r 7f110074
```
And the result is:
```grep
values/public.xml:    <public type="string" name="company_api_key" id="0x7f110074" />
```
As we can see, the resource name is the same as the constant name in the previously show piece of code from the R.class 

Now we again search for the api key value by its name `company_api_key`:
```bash
grep -r company_api_key
```
And the result is:
```grep 
values/public.xml:    <public type="string" name="company_api_key" id="0x7f110074" />
values/strings.xml:    <string name="company_api_key">4670f468049bbee2260</string>
```

**The API key is 4670f468049bbee2260.**

## Test

Now we create a few test call in accordance to the call's in the decompiled java code shown previously from `buslogic.nsmartapp.api.CitiesExtendedApi.class` and `buslogic.nsmartapp.ui.SplashActivity.class`.

```bash
curl --location --request GET 'https://online.nsmart.rs/publicapi/v1/networkextended.php?action=get_cities_extended' --header 'X-Api-Authentication: 4670f468049bbee2260'
```
From the [response JSON](sample-response-cities-extended.json) we extract station information, for instance, the ID for the station "Narodnog Fronta - Šekspirova" is `6532`.
Now we can form the Announcment API call for this station:
```bash
curl --location --request GET 'https://online.nsmart.rs/publicapi/v1/announcement/announcement.php?ibfm=TM00000&station_uid=6532' --header 'X-Api-Authentication: 4670f468049bbee2260'
```
The [JSON response](sample-response-announcment-6532.json), among other things, gives the exact current position for all busses that stop at the given bus station.
