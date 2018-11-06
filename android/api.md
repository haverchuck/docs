# API
<div class="nav-tab create" data-group='create'>
<ul class="tabs">
    <li class="tab-link java current" data-tab="java">Java</li>
    <li class="tab-link kotlin" data-tab="kotlin">Kotlin</li>
</ul>

## GraphQL: Realtime and Offline

AWS AppSync integrates with the [Apollo GraphQL client](https://github.com/apollographql/apollo-client) when building client applications. AWS provides plugins for offline support, authorization, and subscription handshaking to make this process easier. You can use the Apollo client directly, or with some client helpers provided in the AWS AppSync SDK when you get started.

For a step-by-step tutorial describing how to build an Android application, including code generation for Java types, by using AWS AppSync see `aws-appsync-building-a-client-app-android`.

The following information gives an overview of how the AWS SDKs work as well as code generation for your GraphQL application using the [AWS Amplify toolchain](https://aws-amplify.github.io/).

### Application Configuration

The AWS SDKs support configuration through a centralized file called `awsconfiguration.json` defining all the regions and service endpoints to communicate. When you configure categories in the Amplify CLI and run `amplify push`, this file is updated allowing you to focus on your Android application code. On Android Studio projects the `awsconfiguration.json` are placed in the `./src/main/res/raw` directory when using the CLI. If you are building an application and starting from the AWS AppSync console, choose the **Download Config** button on the **App Integration** page to download the `awsconfiguration.json` file, which is already populated for that specific API. You need to place it in the `./src/main/res/raw`.

### Code Generation

To execute GraphQL operations in Android you need to run a code generation process, which requires both the GraphQL schema and the statements (for example, queries, mutations, or subscriptions) that your client defines. The Amplify CLI toolchain makes this easy for you by automatically pulling down your schema and generating default GraphQL queries, mutations, and subscriptions before kicking off the code generation process using Gradle. If your client requirements change, you can alter these GraphQL statements and kick off a Gradle build again to regenerate the types. Install the CLI with the following command:

```bash
    npm install -g @aws-amplify/cli
```

Next, open a terminal, go to your Android Studio project root, and then run the following:

```bash
    amplify init
    amplify add codegen --apiId XXXXXX
```

The `XXXXXX` is the unique AppSync API identifier that you can find in the console in the root of your API's integration page. When you run this command you can accept the defaults, which create a `./src/main.graphql` folder structure with your statements. When you add the required Gradle dependencies later, the generated packages are automatically added to your project.

### Import SDK and Config

To use AppSync in your Android studio project, modify the project's `build.gradle` with the following dependency in the build script:

```bash
    classpath 'com.amazonaws:aws-android-sdk-appsync-gradle-plugin:2.6.+'
```

Next, in the app's build.gradle add in a plugin of `apply plugin: 'com.amazonaws.appsync'` and a dependency of `compile 'com.amazonaws:aws-android-sdk-appsync:2.6.+'`. For example:


```bash
    apply plugin: 'com.android.application'
    apply plugin: 'com.amazonaws.appsync'
    android {
        // Typical items
    }
    dependencies {
        // Typical dependencies
        compile 'com.amazonaws:aws-android-sdk-appsync:2.6.+'
        compile 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.2.0'
        compile 'org.eclipse.paho:org.eclipse.paho.android.service:1.1.1'
    }
```


Finally, update your AndroidManifest.xml with updates to `<uses-permissions>` for network calls and offline state. Also, add a `<service>` entry under `<application>` for `MqttService` to use subscriptions:

```xml
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>

            <!--other code-->

        <application
            android:allowBackup="true"
            android:icon="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:roundIcon="@mipmap/ic_launcher_round"
            android:supportsRtl="true"
            android:theme="@style/AppTheme">

            <service android:name="org.eclipse.paho.android.service.MqttService" />

            <!--other code-->
        </application>
```

**Build your project** ensuring there are no issues.

### Client Initialization

Inside your application code, such as the `onCreate()` lifecycle method of your activity class, you can initialize the AppSync client using an instance of `AWSConfiguration()` in the `AWSAppSyncClient` builder like the following:

```java

    private AWSAppSyncClient mAWSAppSyncClient;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mAWSAppSyncClient = AWSAppSyncClient.builder()
                .context(getApplicationContext())
                .awsConfiguration(new AWSConfiguration(getApplicationContext()))
                .build();
    }
```

This reads configuration information in the `awsconfiguration.json` file. By default, the information in the `Default` section of the json file is used.

### Run a Query

Now that the client is configured, you can run a GraphQL query. The syntax of the callback is `GraphQLCall.Callback<{NAME>Query.Data>` where `{NAME}` comes from the GraphQL statements that `amplify codegen` created after you ran a Gradle build. You invoke this from an instance of the AppSync client with a similar syntax of `.query(<NAME>Query.builder().build())`. For example, if you have a `ListTodos` query, your code will look like the following:

```java
    public void query(){
        mAWSAppSyncClient.query(ListTodosQuery.builder().build())
                .responseFetcher(AppSyncResponseFetchers.CACHE_AND_NETWORK)
                .enqueue(todosCallback);
    }

    private GraphQLCall.Callback<ListTodosQuery.Data> todosCallback = new GraphQLCall.Callback<ListTodosQuery.Data>() {
        @Override
        public void onResponse(@Nonnull Response<ListTodosQuery.Data> response) {
            Log.i("Results", response.data().listTodos().items().toString());
        }

        @Override
        public void onFailure(@Nonnull ApolloException e) {
            Log.e("ERROR", e.toString());
        }
    };
```

Optionally, you can change the cache policy on `AppSyncResponseFetchers`, but we recommend leaving `CACHE_AND_NETWORK` because it pulls results from the local cache first before retrieving data over the network. This gives a snappy UX and offline support.

### Run a Mutation

To add data you need to run a GraphQL mutation. The syntax of the callback is `GraphQLCall.Callback<{NAME}Mutation.Data>` where `{NAME}` comes from the GraphQL statements that `amplify codegen` created after a Gradle build. However, most GraphQL schemas organize mutations with an `input` type for maintainability, which is what the Amplify CLI does as well. Therefore you'll pass this as a parameter called `input` created with a second builder. You invoke this from an instance of the AppSync client with a similar syntax of `.mutate({NAME}Mutation.builder().input({Name}Input).build())` like the following:

```java
    public void mutation(){
        CreateTodoInput createTodoInput = CreateTodoInput.builder().
            name("Use AppSync").
            description("Realtime and Offline").
            build();

        mAWSAppSyncClient.mutate(CreateTodoMutation.builder().input(createTodoInput).build())
            .enqueue(mutationCallback);
    }

    private GraphQLCall.Callback<CreateTodoMutation.Data> mutationCallback = new GraphQLCall.Callback<CreateTodoMutation.Data>() {
        @Override
        public void onResponse(@Nonnull Response<CreateTodoMutation.Data> response) {
            Log.i("Results", "Added Todo");
        }

        @Override
        public void onFailure(@Nonnull ApolloException e) {
            Log.e("Error", e.toString());
        }
    };
```

### Subscribe to Data

Finally, it's time to set up a subscription to real-time data. The callback is just `AppSyncSubscriptionCall.Callback` and you invoke it with a client `.subscribe()` call and pass in a builder with syntax of `{NAME}Subscription.builder()` where `{NAME}` comes from the GraphQL statements that `amplify codegen` and Gradle build created. Note that the AppSync console and Amplify GraphQL transformer have a common nomenclature that puts the word `On` in front of a subscription as in the following example:

```java
    private AppSyncSubscriptionCall subscriptionWatcher;

    private void subscribe(){
        OnCreateTodoSubscription subscription = OnCreateTodoSubscription.builder().build();
        subscriptionWatcher = mAWSAppSyncClient.subscribe(subscription);
        subscriptionWatcher.execute(subCallback);
    }

    private AppSyncSubscriptionCall.Callback subCallback = new AppSyncSubscriptionCall.Callback() {
        @Override
        public void onResponse(@Nonnull Response response) {
            Log.i("Response", response.data().toString());
        }

        @Override
        public void onFailure(@Nonnull ApolloException e) {
            Log.e("Error", e.toString());
        }

        @Override
        public void onCompleted() {
            Log.i("Completed", "Subscription completed");
        }
    };
```

Subscriptions can also take input types like mutations, in which case they will be subscribing to particular events based on the input. To learn more about subscription arguments, see [Real-Time data](./aws-appsync-real-time-data).

## REST API

### Overview

Add RESTful APIs handled by your serverless Lambda functions. The CLI deploys your APIs and handlers using [Amazon API Gateway](http://docs.aws.amazon.com/apigateway/latest/developerguide/) and [AWS Lambda](http://docs.aws.amazon.com/lambda/latest/dg/).

### Set Up Your Backend

1. Complete the [Get Started](./get-started) steps before you proceed.

2. Use the CLI to add api to your cloud-enabled backend and app.

 In a terminal window, navigate to your project folder (the folder that typically contains your project level `build.gradle`), and add the SDK to your app. Note that the friendly name that specified for the `api` category will be the package name of the generated code.

	```bash
	$ cd ./YOUR_PROJECT_FOLDER
	$ amplify add api
	```

3. Choose `> REST` as your API service.

4. Choose `> Create a new Lambda function`.

5. Choose the `> Serverless express function` template.

6. Restrict API access? Choose `Yes`

7. Who should have access? Choose `Authenticated and Guest users`

8. When configuration of your API is complete, the CLI displays a message confirming that you have configured local CLI metadata for this category. You can confirm this by viewing status.

    ```bash
    $ amplify status
    | Category  | Resource name   | Operation | Provider plugin   |
    | --------- | --------------- | --------- | ----------------- |
    | Function  | lambda01234567  | Create    | awscloudformation |
    | Api       | api012345678    | Create    | awscloudformation |
    ```

9. To create your backend AWS resources run:

    ```bash
    $ amplify push
    ```

   Use the steps in the next section to connect your app to your backend.

### Connect to Your Backend

Use the following steps to add Cloud Logic to your app.

<div id="java" class="tab-content current">
1. Add the following to your `app/build.gradle`:

	```groovy
	dependencies {
		implementation 'com.amazonaws:aws-android-sdk-apigateway-core:2.6.+'
		implementation ('com.amazonaws:aws-android-sdk-mobile-client:2.6.+@aar') { transitive = true }
		implementation ('com.amazonaws:aws-android-sdk-auth-userpools:2.6.+@aar') { transitive = true }
	}
	```

2. Get your API client name.

    The CLI generates a client code file for each API you add. The API client name is the name of that file, without the extension.

    The path of the client code file is `./src/main/java/YOUR_API_RESOURCE_NAME/YOUR_APP_NAME_XXXXClient.java`.

    So, for an app named `useamplify` with an API resource named `xyz123`, the path of the code file will be `./src/main/java/xyz123/useamplifyabcdClient.java`. The API client name will be `useamplifyabcdClient`.

    - Find the resource name of your API by running `amplify status`.
    - Copy your API client name to use when invoking the API in the following step.

3. Invoke a Cloud Logic API.

    The following code shows how to invoke a Cloud Logic API using your API's client class,
    model, and resource paths.

    ```java
    import android.support.v7.app.AppCompatActivity;
    import android.os.Bundle;
    import android.util.Log;

    import com.amazonaws.http.HttpMethodName;
    import com.amazonaws.mobile.client.AWSMobileClient;
    import com.amazonaws.mobile.client.AWSStartupHandler;
    import com.amazonaws.mobile.client.AWSStartupResult;
    import com.amazonaws.mobileconnectors.apigateway.ApiClientFactory;
    import com.amazonaws.mobileconnectors.apigateway.ApiRequest;
    import com.amazonaws.mobileconnectors.apigateway.ApiResponse;
    import com.amazonaws.util.IOUtils;
    import com.amazonaws.util.StringUtils;

    import java.io.InputStream;
    import java.util.HashMap;
    import java.util.Map;

    // TODO Replace this with your api friendly name and client class name
    import YOUR_API_RESOURCE_NAME.YOUR_APP_NAME_XXXXClient;

    public class MainActivity extends AppCompatActivity {
        private static final String TAG = MainActivity.class.getSimpleName();

        // TODO Replace this with your client class name
        private YOUR_APP_NAME_XXXXClient apiClient;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            // Initialize the AWS Mobile Client
            AWSMobileClient.getInstance().initialize(this, new AWSStartupHandler() {
                @Override
                public void onComplete(AWSStartupResult awsStartupResult) {
                    Log.d(TAG, "AWSMobileClient is instantiated and you are connected to AWS!");
                }
            }).execute();


            // Create the client
            apiClient = new ApiClientFactory()
                    .credentialsProvider(AWSMobileClient.getInstance().getCredentialsProvider())
                    .build(YOUR_API_CLIENT_NAME.class);

            callCloudLogic();
        }

        public void callCloudLogic() {
            // Create components of api request
            final String method = "GET";
            final String path = "/items";

            final String body = "";
            final byte[] content = body.getBytes(StringUtils.UTF8);

            final Map parameters = new HashMap<>();
            parameters.put("lang", "en_US");

            final Map headers = new HashMap<>();

            // Use components to create the api request
            ApiRequest localRequest =
                    new ApiRequest(apiClient.getClass().getSimpleName())
                            .withPath(path)
                            .withHttpMethod(HttpMethodName.valueOf(method))
                            .withHeaders(headers)
                            .addHeader("Content-Type", "application/json")
                            .withParameters(parameters);

            // Only set body if it has content.
            if (body.length() > 0) {
                localRequest = localRequest
                        .addHeader("Content-Length", String.valueOf(content.length))
                        .withBody(content);
            }

            final ApiRequest request = localRequest;

            // Make network call on background thread
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Log.d(TAG,
                                "Invoking API w/ Request : " +
                                        request.getHttpMethod() + ":" +
                                        request.getPath());

                        final ApiResponse response = apiClient.execute(request);

                        final InputStream responseContentStream = response.getContent();

                        if (responseContentStream != null) {
                            final String responseData = IOUtils.toString(responseContentStream);
                            Log.d(TAG, "Response : " + responseData);
                        }

                        Log.d(TAG, response.getStatusCode() + " " + response.getStatusText());

                    } catch (final Exception exception) {
                        Log.e(TAG, exception.getMessage(), exception);
                        exception.printStackTrace();
                    }
                }
            }).start();
        }
      }
    ```
</div>
<div id="kotlin" class="tab-content">
1. Add the following to your `app/build.gradle`:

	```groovy
	dependencies {
		implementation 'com.amazonaws:aws-android-sdk-apigateway-core:2.6.+'
		implementation ('com.amazonaws:aws-android-sdk-mobile-client:2.6.+@aar') { transitive = true }
		implementation ('com.amazonaws:aws-android-sdk-auth-userpools:2.6.+@aar') { transitive = true }
	}
	```

2. Get your API client name.

    The CLI generates a client code file for each API you add. The API client name is the name of that file, without the extension.

    The path of the client code file is `./src/main/java/YOUR_API_RESOURCE_NAME/YOUR_APP_NAME_XXXXClient.java`.

    So, for an app named `useamplify` with an API resource named `xyz123`, the path of the code file will be `./src/main/java/xyz123/useamplifyabcdClient.java`. The API client name will be `useamplifyabcdClient`.

    - Find the resource name of your API by running `amplify status`.
    - Copy your API client name to use when invoking the API in the following step.

3. Invoke a Cloud Logic API.

    The following code shows how to invoke a Cloud Logic API using your API's client class,
    model, and resource paths.

    ```kotlin
    import android.os.Bundle
    import android.support.v7.app.AppCompatActivity
    import android.util.Log
    import com.amazonaws.http.HttpMethodName
    import com.amazonaws.mobile.client.AWSMobileClient
    import com.amazonaws.mobileconnectors.apigateway.ApiClientFactory
    import com.amazonaws.mobileconnectors.apigateway.ApiRequest
    import com.amazonaws.util.IOUtils
    import com.amazonaws.util.StringUtils

    // TODO Replace this with your api friendly name and client class name
    import YOUR_API_RESOURCE_NAME.YOUR_APP_NAME_XXXXClient
    import kotlin.concurrent.thread

    class MainActivity : AppCompatActivity() {
        companion object {
            private val TAG = MainActivity.javaClass.simpleName
        }

        // TODO Replace this with your client class name
        private var apiClient: YOUR_APP_NAME_XXXXClient? = null

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)
            setContentView(R.layout.activity_main)

            // Initialize the AWS Mobile Client
            AWSMobileClient.getInstance().initialize(this) { Log.d(TAG, "AWSMobileClient is instantiated and you are connected to AWS!") }.execute()

            // Create the client
            apiClient = ApiClientFactory().credentialsProvider(AWSMobileClient.getInstance().credentialsProvider)
                    // TODO Replace this with your client class name
                    .build(YOUR_APP_NAME_XXXXClient::class.java)

            callCloudLogic()
        }

        fun callCloudLogic() {
            val body = ""

            val parameters = mapOf("lang" to "en_US")
            val headers = mapOf("Content-Type" to "application/json")

            val request = ApiRequest(apiClient?.javaClass?.simpleName)
                    .withPath("/items")
                    .withHttpMethod(HttpMethodName.GET)
                    .withHeaders(headers)
                    .withParameters(parameters)

            if (body.isNotEmpty()) {
                val content = body.toByteArray(StringUtils.UTF8)
                request.addHeader("Content-Length", content.size.toString())
                        .withBody(content)
            }

            thread(start = true) {
                try {
                    Log.d(TAG, "Invoking API")
                    val response = apiClient?.execute(request)
                    val responseContentStream = response?.getContent()
                    if (responseContentStream != null) {
                        val responseData = IOUtils.toString(responseContentStream)
                        // Do something with the response data here
                        Log.d(TAG, "Response: $responseData")
                    }
                } catch (ex: Exception) {
                    Log.e(TAG, "Error invoking API")
                }
            }
        }
    }

    ```
</div>
