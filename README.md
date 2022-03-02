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

All the API call require an Authentication key, so we search for the code where it's value is set. The relevant code is in `buslogic.nsmartapp.ui.SplashActivity.class`:
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
