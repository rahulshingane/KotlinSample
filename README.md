## KotlinSample With Retrofit
# Base Respose
  - Create base response class which is used to pasre api response in model class
```
public class ApiData {

    private String response;
    private String settings; // This is for cit

    public String getResponse() {
        return response;
    }

    public void setResponse(String response) {
        this.response = response;
    }

    public void setSettings(String settings) {
        this.settings = settings;
    }

    public Settings getSettings() {
        return new Gson().fromJson(settings, Settings.class);
    }

    public <T> List<T> getData(Class<T> c) {
        Type type = new ListParameterizedType<T>(c);
        return new Gson().fromJson(response, type);
    }

    public static <T> List<T> getData(Class<T> c, String response) {
        Type type = new ListParameterizedType<T>(c);
        return new Gson().fromJson(response, type);
    }

    public <T> T getDataAsObject(Class<T> c) {
        List<T> list = getData(c);
        if (list != null && list.size() > 0) {
            return list.get(0);
        }
        return null;
    }

}
```
# ParameterizedType class
  - Create ParameterizedType class used to convert generic type to model class
```
public class ListParameterizedType<T> implements ParameterizedType {

    private Class<?> wrapped;

    public ListParameterizedType(Class<T> wrapped) {
        this.wrapped = wrapped;
    }

    public Type[] getActualTypeArguments() {
        return new Type[] {wrapped};
    }

    public Type getRawType() {
        return List.class;
    }

    public Type getOwnerType() {
        return null;
    }

}
```

# Retrofit Helper class in Kotlin
  - This class is used to perform all api related operations
```
class RetrofitHelper {

    companion object {
        val KEY_SETTINGS = "settings"
        val KEY_DATA = "data"

        var progressDialog: ProgressDialog? = null;

        fun create(): Retrofit {
            val builder = OkHttpClient().newBuilder()
            builder.readTimeout(10, TimeUnit.SECONDS)
            builder.connectTimeout(5, TimeUnit.SECONDS)

            if (BuildConfig.DEBUG) {
                val interceptor = HttpLoggingInterceptor()
                interceptor.level = HttpLoggingInterceptor.Level.BODY
                builder.addInterceptor(interceptor)
            }

            builder.addInterceptor { chain ->
                val request = chain.request().newBuilder().addHeader("key", "value").build()
                chain.proceed(request)
            }

            val client = builder.build()
            val retrofit = Retrofit.Builder()
                    .addCallAdapterFactory(
                            RxJava2CallAdapterFactory.create())
                    .addConverterFactory(
                            GsonConverterFactory.create())
                    .baseUrl(WebServicesK.BASE_URL)
                    .client(client)
                    .build()

            return retrofit
        }

        fun getWebServices(): WebServices {
            return create().create(WebServicesK::class.java)
        }

        fun call(context: Context?, call: Single<ResponseBody>) {
            call(context, call, null, -1, false, null)
        }

        fun call(context: Context?, call: Single<ResponseBody>, callback: ApiCallback?) {
            call(context, call, null, -1, false, callback)
        }

        fun call(context: Context?, call: Single<ResponseBody>, requestCode: Int, callback: ApiCallback?) {
            call(context, call, null, requestCode, false, callback)
        }

        fun call(context: Context?, call: Single<ResponseBody>, disposableContainer: CompositeDisposable?, requestCode: Int, callback: ApiCallback?) {
            call(context, call, disposableContainer, requestCode, false, callback)
        }

        fun call(context: Context?, call: Single<ResponseBody>, disposableContainer: CompositeDisposable?, requestCode: Int, showProgress: Boolean, callback: ApiCallback?) {

            if(!checkInternetConnection(context)){
                callback?.onNoInternetConnection("Network Not available", requestCode)
                return
            }

            if (showProgress) {
                if (progressDialog == null) {
                    progressDialog = ProgressDialog(context)
                    progressDialog?.setMessage("Please waite...")
                    progressDialog?.setCancelable(false)
                }
                progressDialog?.show()
            }
            call
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(object : SingleObserver<ResponseBody> {
                        override fun onSuccess(t: ResponseBody?) {
                            try {
                                parseResponse(t, callback, requestCode)
                                callback?.onSuccessWithRawData(t, requestCode)
                            } catch (ex: Exception) {
                                ex.printStackTrace();
                            }
                            if ((progressDialog != null) && progressDialog!!.isShowing()) {
                                progressDialog?.dismiss()
                            }
                        }

                        override fun onSubscribe(d: Disposable) {
                            disposableContainer?.add(d)
                        }

                        override fun onError(e: Throwable) {
                            callback?.onFailure(e.message, requestCode)
                            if ((progressDialog != null) && progressDialog!!.isShowing()) {
                                progressDialog?.dismiss()
                            }
                        }
                    })
        }

        fun parseResponse(response: ResponseBody?, callback: ApiCallback?, requestCode: Int) {

            val mApiData = ApiData()
            var responseStr = response?.string();
            Log.d("Api Response", responseStr);
            if (!responseStr.isNullOrBlank()) {

                val jsonElement: JsonElement? = JsonParser().parse(responseStr)

                if (jsonElement!!.isJsonObject) {

                    val jsonObject = jsonElement.asJsonObject;

                    if (jsonObject.has(KEY_SETTINGS) && jsonObject.has(KEY_DATA)) {
                        // For CIT response
                        mApiData?.setSettings(jsonObject.getAsJsonObject(KEY_SETTINGS).toString())
                        var jArray = JsonArray()
                        if (jsonObject.get(KEY_DATA) is JsonObject) {
                            jArray?.add(jsonObject?.getAsJsonObject(KEY_DATA))
                            mApiData.setResponse(jArray?.toString());
                        } else {
                            mApiData.setResponse(jsonObject.get(KEY_DATA)?.toString());
                        }
                    } else {
                        var jArray = JsonArray()
                        jArray?.add(jsonObject)
                        mApiData.setResponse(jArray?.toString());
                    }

                } else {
                    mApiData.setResponse(responseStr);
                }

                callback?.onSuccess(mApiData, requestCode)


            } else {
                callback?.onFailure("Not able to process your request", requestCode)
            }
        }

        fun checkInternetConnection(context: Context?): Boolean {
            val _connManager = context?.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
            var _isConnected = false
            val _activeNetwork = _connManager.activeNetworkInfo
            if (_activeNetwork != null) {
                _isConnected = _activeNetwork.isConnectedOrConnecting
            }
            return _isConnected
        }
    }
}
```
# API Interface
 - All calling api's methods will be written here. Return type must be ResponseBody
```
interface WebServices {
    companion object {
        val BASE_URL = "http://google.com/WS/"
    }
    @POST("notification_list")
    fun getNotificationList(@QueryMap map: Map<String, String>): Single<ResponseBody>

    @GET("http://ip-api.com/json")
    fun getLocation(): Single<ResponseBody>

    @GET("http://google.com")
    fun getList(): Single<ResponseBody>

}
```
# API Delagate
 - Add callbacks methods fro api related operations
```
interface ApiCallback {
    //This callback is used for get api response in CIT format. CIT api response has fixed format.
    // It has settings and data objects
    fun onSuccess(apiData: ApiData?, requestCode: Int)

    //This callback will notify if there is any failure in api calling
    fun onFailure(message: String?, requestCode: Int)

    //This callback will notify no internet connection status
    fun onNoInternetConnection(message: String?, requestCode: Int)

    // This callback will provide raw response of api
    fun onSuccessWithRawData(response: ResponseBody?, requestCode: Int)
}
``````

## Example
 - Initialize Retrofit
 ```
 val retroApiServe by lazy {
        RetrofitHelper.getWebServices()
    }
    var disposable: CompositeDisposable? = CompositeDisposable()
```    
 - Call Api
 ```
  val call = retroApiServe.getList()
   RetrofitHelper.call(this,call, disposable, 101, true,this)
```        
 - Callback Methods
 ```
    override fun onSuccess(apiData: ApiData?, requestCode: Int) {
 if(requestCode==101){
            val activityResponse : LocationData? = apiData?.getDataAsObject(LocationData::class.java)  
        }else if(requestCode == 102){
            val activityResponse: List<ListModel>? = apiData?.getData(ListModel::class.java)
        }
    }

    override fun onFailure(message: String?, requestCode: Int) {
//        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }

    override fun onNoInternetConnection(message: String?, requestCode: Int) {
//        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }

    override fun onSuccessWithRawData(response: ResponseBody?, requestCode: Int) {
//        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }
```
  - Location Model
 
 
 ```
 class LocationData {
    @SerializedName("as")
    @Expose
    private var `as`: String? = null
    @SerializedName("city")
   

    fun getAs(): String? {
        return `as`
    }

    fun setAs(`as`: String) {
        this.`as` = `as`
    }

}
```
