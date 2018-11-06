# API

The API category provides a solution for making HTTP requests to REST and GraphQL endpoints. It includes a [AWS Signature Version 4](http://docs.aws.amazon.com/general/latest/gr/signature-version-4.html) signer class which automatically signs all AWS API requests for you as well as methods to use API Keys, Amazon Cognito User Pools, or 3rd party OIDC providers.

## GraphQL: Realtime and Offline

AWS AppSync helps you build data-driven apps with real-time and offline capabilities. The [AppSync iOS SDK](https://github.com/awslabs/aws-mobile-appsync-sdk-ios/) enables you to integrate your app with the AWS AppSync service and is based off of the Apollo project found [here](https://github.com/apollographql/apollo-ios). The SDK supports multiple authorization models, handles subscription handshake protocols for real-time updates to data, and has built-in capabilities for offline support that makes it easy to integrate into your app.

You can integrate with AWS AppSync using the following steps:

1. Setup the API endpoint and authentication information in the client side configuration.
2. Generate Swift code from the API schema.
3. Write app code to run queries, mutations and subscriptions.

The Amplify CLI provides support for AppSync that make this process easy. Using the CLI, you can configure an AWS AppSync API, download required client side configuration files, and generate client side code within minutes by running a few simple commands on the command line.

### Configuration

The AWS SDKs support configuration through a centralized file called `awsconfiguration.json` that defines your AWS regions and service endpoints. You obtain this file in one of two ways, depending on whether you are creating your AppSync API in the AppSync console or using the Amplify CLI.

* If you are creating your API in the console, navigate to the `Getting Started` page, and follow the steps in the `Integrate with your app` section. The `awsconfiguration.json` file you download is already populated for your specific API. Place the file in the root directory of your iOS project, and add it to your Xcode project.

* If you are creating your API with the Amplify CLI (using `amplify add api`), the `awsconfiguration.json` file is automatically downloaded and updated each time you run `amplify push` to update your cloud resources. The file is placed in the root directory of your iOS project, and you need to add it to your Xcode project.


### Code Generation

To execute GraphQL operations in iOS you need to run a code generation process, which requires both the GraphQL schema and the statements (for example, queries, mutations, or subscriptions) that your client defines. The Amplify CLI toolchain helps you do this by automatically pulling down your schema and generating default GraphQL queries, mutations, and subscriptions before kicking off the code generation process. If your client requirements change, you can alter these GraphQL statements and regenerate your types.

#### AppSync APIs Created in the Console

After installing the Amplify CLI open a terminal, go to your Xcode project root, and then run the following:

```bash
amplify init
amplify add codegen --apiId XXXXXX
```

The `XXXXXX` is the unique AppSync API identifier that you can find in the console in the root of your API's integration page. When you run this command you can accept the defaults, which create an `API.swift` file, and a `graphql` folder with your statements, in your root directory.

#### AppSync APIs Created Using the CLI

Navigate in your terminal to an XCode project directory and run the following:

```terminal
$amplify init     ## Select iOS as your platform
$amplify add api  ## Select GraphQL, API key, "Single object with fields Todo application"
```
Select *GraphQL* when prompted for service type:

```terminal
? Please select from one of the below mentioned services (Use arrow keys)
❯ GraphQL
  REST
```

The `add api` flow above will ask you some questions, such as if you already have an annotated GraphQL schema. If this is your first time using the CLI select **No** and let it guide you through the default project **"Single object with fields (e.g., “Todo” with ID, name, description)"** as it will be used in the code examples below. Later on, you can always change it.

Name your GraphQL endpoint and select authorization type:

```terminal
? Please select from one of the below mentioned services GraphQL
? Provide API name: myTodosApi
? Choose an authorization type for the API (Use arrow keys)
❯ API key
  Amazon Cognito User Pool
```

AWS AppSync API keys expire seven days after creation, and using API KEY authentication is only suggested for development. To change AWS AppSync authorization type after the initial configuration, use the `$ amplify update api` command and select `GraphQL`.
{: .callout .callout--info}

When you update your backend with *push* command, you can go to [AWS AppSync Console](http://console.aws.amazon.com/appsync/home) and see that a new API is added under *APIs* menu item:

```bash
$ amplify push
```

The `amplify push` process will prompt you to enter the codegen process and walk through configuration options. Accept the defaults and it will create a file named `API.swift` in your root directory (unless you choose to name it differently) as well as a directory called `graphql` with your documents. You also will have an `awsconfiguration.json` file that the AppSync client will use for initialization. 


### Import SDK and Config
To use AppSync in your Xcode project, modify your Podfile with a dependency of the AWS AppSync SDK as follows:

```ruby
target 'PostsApp' do
    use_frameworks!
    pod 'AWSAppSync', ' ~> 2.6.20'
end
```

Run `pod install` from your terminal and open up the `.xcworkspace` Xcode project. Add the `API.swift` and `awsconfiguration.json` files to your project **(File->Add Files to ..->Add)** and then build your project, ensuring there are no issues.

### Client Initialization

Initialize the AppSync client your application delegate by creating `AWSAppSyncClientConfiguration` and `AWSAppSyncClient` like the following:

```swift
import AWSAppSync

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

var appSyncClient: AWSAppSyncClient?

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    //You can choose your database location
    let databaseURL = URL(fileURLWithPath:NSTemporaryDirectory()).appendingPathComponent("database_name")

    do {
        //AppSync configuration & client initialization
        let appSyncConfig = try AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo(),databaseURL: databaseURL)
        appSyncClient = try AWSAppSyncClient(appSyncConfig: appSyncConfig)
        // Set id as the cache key for objects. See architecture section for details
        appSyncClient?.apolloClient?.cacheKeyForObject = { $0["id"] }
        } catch {
            print("Error initializing appsync client. \(error)")
        }
        //other methods
        return true
}
```

`AWSAppSyncClientInfo` represents the configuration information present in awsconfiguration.json file. Next, in your application code, you reference this in an appropriate lifecycle method such as `viewDidLoad()`:

```swift
import AWSAppSync

class Todos: UIViewController{
    //Reference AppSync client
    var appSyncClient: AWSAppSyncClient?

    override func viewDidLoad() {
        super.viewDidLoad()
        //Reference AppSync client from App Delegate
        let appDelegate = UIApplication.shared.delegate as! AppDelegate
        appSyncClient = appDelegate.appSyncClient
    }
}
```

### Run a Query

Now that the client is set up, you can run a GraphQL query. The syntax is `appSyncClient?.fetch(query: <NAME>Query() {(result, error)})` where `<NAME>` comes from the GraphQL statements that `amplify codegen` created. For example, if you have a `ListTodos` query your code will look like the following:

```swift
//Run a query
    appSyncClient?.fetch(query: ListTodosQuery())  { (result, error) in
    if error != nil {
        print(error?.localizedDescription ?? "")
        return
    }
    result?.data?.listTodos?.items!.forEach { print(($0?.name)! + " " + ($0?.description)!) }
}
```

Optionally, you can set a cache policy on the query as follows:

```swift
appSyncClient?.fetch(query: ListTodosQuery(), cachePolicy: .returnCacheDataAndFetch)  { (result, error) in
```
`returnCacheDataAndFetch` pulls results from the local cache first before retrieving data over the network. This gives a snappy UX and offline support.

### Run a Mutation

To add data you need to run a GraphQL mutation. The syntax is `appSyncClient?.perform(mutation: <NAME>Mutation() {(result, error)})` where `<NAME>` comes from the GraphQL statements that `amplify codegen` created. However, most GraphQL schemas organize mutations with an `input` type for maintainability, which is what the AppSync console and Amplify CLI do as well. Therefore, you need to pass this as a parameter called `input`, as in the following example:

```swift
let mutationInput = CreateTodoInput(name: "Use AppSync", description:"Realtime and Offline")

appSyncClient?.perform(mutation: CreateTodoMutation(input: mutationInput)) { (result, error) in
    if let error = error as? AWSAppSyncClientError {
        print("Error occurred: \(error.localizedDescription )")
    }
    if let resultError = result?.errors {
        print("Error saving the item on server: \(resultError)")
        return
    }
}
```

### Subscribe to Data

Finally, it's time to set up a subscription to real-time data. The syntax `appSyncClient?.subscribe(subscription: <NAME>Subscription() {(result, transaction, error)})` where `<NAME>` comes from the GraphQL statements that `amplify codegen` created. Note that the AppSync console and Amplify GraphQL transformer have a common nomenclature that puts the word `On` in front of a subscription as in the following example:

```swift
//Set a variable to discard at the class level
var discard: Cancellable?

//In your app code
do {
    discard = try appSyncClient?.subscribe(subscription: OnCreateTodoSubscription(), resultHandler: { (result, transaction, error) in
        if let result = result {
        print(result.data!.onCreateTodo!.name + " " + result.data!.onCreateTodo!.description!)
        } else if let error = error {
        print(error.localizedDescription)
        }
    })
    } catch {
    print("Error starting subscription.")
}
```

Subscriptions can also take input types like mutations, in which case they will be subscribing to particular events based on the input. To learn more about subscription arguments, see :ref:`Real-Time data <aws-appsync-real-time-data>`.

### Client Architecture

The AppSync client supports offline scenarios with a programing model that provides a "write through cache". This allows you to both render data in the UI when offline as well as add/update through an "optimistic response". The below diagram shows how the AppSync client interfaces with the network GraphQL calls, it's offline mutation queue, the Apollo cache, and your application code.

**//TODO - Insert diagram**

Your application code will interact with the AppSync client to perform GraphQL queries, mutations, or subscriptions. The AppSync client automatically performs the correct authorization methods when interfacing with the HTTP layer adding API Keys, tokens, or signing requests depending on how you have configured your setup. When you do a mutation, such as adding a new item (like a blog post) in your app the AppSync client adds this to a local queue (persisted to disk with SQLite) when the app is offline. When network connectivity is restored the mutations are sent to AppSync in serial allowing you to process the responses one by one. 

Any data returned by a query is automatically written to the Apollo Cache (e.g. “Store”) that is persisted to disk via SQLite. The cache is structured as a key value store using a reference structure. There is a base “Root Query” where each subsequent query resides and then references their individual item results. You specify the reference key (normally “id”) in your application code. An example of the cache that has stored results from a “listPosts” query and “getPost(id:1)” query is below.

| Key | Value |
| ROOT_QUERY | [ROOT_QUERY.listPosts, ROOT_QUERY.getPost(id:1)]
| ROOT_QUERY.listPosts | {0, 1, …,N} |
| Post:0 |{author:"Nadia", content:"ABC"} |
| Post:1 | {author:"Shaggy", content:"DEF"} |
| ... | ... |
| Post:N | {author:"Pancho", content:"XYZ"} |
| ROOT_QUERY.getPost(id:1) |ref: $Post:1 |

Notice that the cache keys are normalized where the `getPost(id:1)` query references the same element that is part of the `listPosts` query. This only happens when you define a common cache key to uniquely identify the objects. This is done when you configure the AppSync client in your AppDelegate with:

```swift
//Use something other than "id" if your GraphQL type is different
appSyncClient?.apolloClient?.cacheKeyForObject = { $0["id"] }
```

If you are performing a mutation, you can write an “optimistic response” anytime to this cache even if you are offline. You use the AppSync client to connect by passing in the query to update, reading the items off the cache. This normally returns a single item or list of items, depending on the GraphQL response type of the query to update. At this point you would add to the list, remove, or update it as appropriate and write back the response to the store persisting it to disk. When you reconnect to the network any responses from the service will overwrite the changes as the authoritative response.

#### Offline Mutations

As outlined in the architecture section, all query results are automatically persisted to disc with the AppSync client. For updating data through mutations when offline you will need to use an "optimistic response" with a transaction. This is done by passing an `optimisticUpdate` in the `appSyncClient?.perform()` mutation method using a `transaction`, where you pass in a query that will be updated in the cache. Inside of this transaction, you can write to the store via `appSyncClient?.store?.withinReadWriteTransaction`.

For example, the below code shows how you would update the `CreateTodoMutation` mutation from earlier by adding a `optimisticUpdate: { (transaction) in do {...} catch {...}` argument with a closure. This adds an item to the cache with `transaction?.update()` using a locally generated unique identifier. This might be enough for your app, however if the AppSync response returns a different value for `ID` (which many times is the case as best practice is generation of IDs at the service layer) then you will need to replace the value locally when a response is received. this can be done in the `resultHandler` by using `appSyncClient?.store?.withinReadWriteTransaction()` and `transaction?.update()` again.

```swift
func optimisticCreateTodo(input: CreateTodoInput, query:ListTodosQuery){
        let createTodoInput = CreateTodoInput(name: input.name, description: input.description)
        let createTodoMutation = CreateTodoMutation(input: createTodoInput)
        let UUID = NSUUID().uuidString
        
        self.appSyncClient?.perform(mutation: createTodoMutation, optimisticUpdate: { (transaction) in
            do {
                try transaction?.update(query: query) { (data: inout ListTodosQuery.Data) in
                    data.listTodos?.items?.append(ListTodosQuery.Data.ListTodo.Item.init(id: UUID, name: input.name, description: input.description!))
                }
            } catch {
                print("Error updating cache with optimistic response for \(createTodoInput)")
            }
        }, resultHandler: { (result, error) in
            if let result = result {
                print("Added Todo Response from service: \(String(describing: result.data?.createTodo?.name))")
                //Now remove the outdated entry in cache from optimistic write
                let _ = self.appSyncClient?.store?.withinReadWriteTransaction { transaction in
                    try transaction.update(query: ListTodosQuery())
                    { (data: inout ListTodosQuery.Data) in
                        var pos = -1, counter = 0
                        for item in (data.listTodos?.items!)! {
                            if item?.id == UUID {
                                pos = counter
                                continue
                            }; counter += 1
                        }
                        if pos != -1 { data.listTodos?.items?.remove(at: pos) }
                    }
                }
            } else if let error = error {
                print("Error adding Todo: \(error.localizedDescription)")
            }
        })
    }
```

You might add similar code in your app for updating or deleting items using an optimistic response, it would look largely similar except that you might overwrite or remove an element from the `data.listTodos?.items` array.

### Authentication Modes

For client authorization AppSync supports API Keys, Amazon IAM credentials (we recommend using Amazon Cognito Identity Pools for this option), Amazon Cognito User Pools, and 3rd party OIDC providers. This is inferred from the `awsconfiguration.json` when you call `AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo()`.

#### API Key

API Key is the easiest way to setup and prototype your application with AppSync. It's also a good option if your application is completely public. If your application needs to interact with other AWS services besides AppSync, such as S3, you will need to use IAM credentials provided by Cognito Identity Pools, which also supports "Guest" access. See [the authentication section for more details](./authentication). For manual configuration, add the following snippet to your `awsconfiguration.json` file:

```json
{
  "AppSync": {
        "Default": {
            "ApiUrl": "YOUR-GRAPHQL-ENDPOINT",
            "Region": "us-east-1",
            "ApiKey": "YOUR-API-KEY",
            "AuthMode": "API_KEY"
        }
   }
}
```

Add the following code to your app:

```swift
// You can choose your database location, accessible by the SDK
let databaseURL = URL(fileURLWithPath:NSTemporaryDirectory()).appendingPathComponent(database_name)
    
do {
    // Initialize the AWS AppSync configuration
    let appSyncConfig = try AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo(), 
                                                                databaseURL: databaseURL)
    
    // Initialize the AWS AppSync client
    appSyncClient = try AWSAppSyncClient(appSyncConfig: appSyncConfig)
} catch {
    print("Error initializing appsync client. \(error)")
}
```

#### Cognito User Pools

Amazon Cognito User Pools is the most common service to use with AppSync when adding user Sign-Up and Sign-In to your application. If your application needs to interact with other AWS services besides AppSync, such as S3, you will need to use IAM credentials with Cognito Identity Pools. The Amplify CLI can automatically configure this for you when running `amplify add auth` and can also automatically federate User Pools with Identity Pools. This allows you to have both User Pool credentials for AppSync and AWS credentials for S3. You can then use the `AWSMobileClient` for automatic credentials refresh [as outlined in the authentication section](./authentication). For manual configuration, add the following snippet to your `awsconfiguration.json` file:

```json
{
  "CognitoUserPool": {
        "Default": {
            "PoolId": "POOL-ID",
            "AppClientId": "APP-CLIENT-ID",
            "AppClientSecret": "APP-CLIENT-SECRET",
            "Region": "us-east-1"
        }
    },
  "AppSync": {
        "Default": {
            "ApiUrl": "YOUR-GRAPHQL-ENDPOINT",
            "Region": "us-east-1",
            "AuthMode": "AMAZON_COGNITO_USER_POOLS"
        }
   }
}
```

Add the following code to your app:

```swift                                
    func initializeAppSync() {
        // You can choose your database location, accessible by the SDK
        let databaseURL = URL(fileURLWithPath:NSTemporaryDirectory()).appendingPathComponent("todos_db")
        
        do {
            // Initialize the AWS AppSync configuration
            let appSyncConfig = try AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo(),
                                                                  userPoolsAuthProvider: {
                                                                    class MyCognitoUserPoolsAuthProvider : AWSCognitoUserPoolsAuthProviderAsync {
                                                                        func getLatestAuthToken(_ callback: @escaping (String?, Error?) -> Void) {
                                                                            AWSMobileClient.sharedInstance().getTokens { (tokens, error) in
                                                                                if error != nil {
                                                                                    callback(nil, error)
                                                                                } else {
                                                                                    callback(tokens?.idToken?.tokenString, nil)
                                                                                }
                                                                            }
                                                                        }
                                                                    }
                                                                    return MyCognitoUserPoolsAuthProvider()}(),
                                                                  databaseURL:databaseURL)
            
            // Initialize the AWS AppSync client
            appSyncClient = try AWSAppSyncClient(appSyncConfig: appSyncConfig)
        } catch {
            print("Error initializing appsync client. \(error)")
        }
    }
```

#### IAM

When using AWS IAM in a mobile application you should leverage Amazon Cognito Identity Pools. The Amplify CLI can automatically configure this for you when running `amplify add auth`. You can then use the `AWSMobileClient` for automatic credentials refresh [as outlined in the authentication section](./authentication) For manual configuration, add the following snippet to your `awsconfiguration.json` file:

```json
{
  "CredentialsProvider": {
      "CognitoIdentity": {
          "Default": {
              "PoolId": "YOUR-COGNITO-IDENTITY-POOLID",
              "Region": "us-east-1"
          }
      }
  },
  "AppSync": {
    "Default": {
          "ApiUrl": "YOUR-GRAPHQL-ENDPOINT",
          "Region": "us-east-1",
          "AuthMode": "AWS_IAM"
     }
   }
}
```

Add the following code to your app:

```swift
// Set up the Amazon Cognito CredentialsProvider
let credentialsProvider = AWSCognitoCredentialsProvider(regionType: CognitoIdentityRegion,
                                                        identityPoolId: CognitoIdentityPoolId)
                                                                
// You can choose your database location, accessible by the SDK
let databaseURL = URL(fileURLWithPath:NSTemporaryDirectory()).appendingPathComponent(database_name)
    
do {
  // Initialize the AWS AppSync configuration
            let appSyncConfig = try AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo(),
                                                                  credentialsProvider: credentialsProvider,
                                                                  databaseURL: databaseURL)
    
    // Initialize the AWS AppSync client
    appSyncClient = try AWSAppSyncClient(appSyncConfig: appSyncConfig)
} catch {
    print("Error initializing appsync client. \(error)")
}
```

#### OIDC

If you are using a 3rd party OIDC provider you will need to configure it and manage the details of token refreshes yourself. Update the `awsconfiguration.json` file and code snippet as follows:

```json
{
  "AppSync": {
        "Default": {
            "ApiUrl": "YOUR-GRAPHQL-ENDPOINT",
            "Region": "us-east-1",
            "AuthMode": "OPENID_CONNECT"
        }
   }
}
```

Add the following code to your app:

```swift
class MyOidcProvider: AWSOIDCAuthProvider {
    func getLatestAuthToken() -> String {
        // Fetch the JWT token string from OIDC Identity provider
        // after the user is successfully signed-in
        return "token"
    }
}

let databaseURL = URL(fileURLWithPath:NSTemporaryDirectory()).appendingPathComponent(database_name)
    
do {
  // Initialize the AWS AppSync configuration
    let appSyncConfig = try AWSAppSyncClientConfiguration(appSyncClientInfo: AWSAppSyncClientInfo(),
                                                          oidcAuthProvider: MyOidcProvider(),
                                                          databaseURL:databaseURL)
    
    appSyncClient = try AWSAppSyncClient(appSyncConfig: appSyncConfig)
} catch {
    print("Error initializing appsync client. \(error)")
}
```

## REST API

### Overview

Add RESTful APIs handled by your serverless Lambda functions. The CLI deploys your APIs and handlers using [Amazon API Gateway](http://docs.aws.amazon.com/apigateway/latest/developerguide/) and [AWS Lambda](http://docs.aws.amazon.com/lambda/latest/dg/).

### Set Up Your Backend

1. Complete the [Get Started](./get-started) steps before you proceed.

2. Use the CLI to add api to your cloud-enabled backend and app.

 In a terminal window, navigate to your project folder (the folder that contains your app `.Xcodeproj` file), and add the SDK to your app.

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

1. `Podfile` that you configure to install the AWS Mobile SDK must contain:

	```ruby
	platform :ios, '9.0'

	target :'YOUR-APP-NAME' do
	  use_frameworks!

	     # For auth
	     pod 'AWSAuthCore', '~> 2.6.33'
	     pod 'AWSMobileClient', '~> 2.6.33'

	     # For API
	     pod 'AWSAPIGateway', '~> 2.6.33'

	     # other pods

	end
	```

	Run `pod install --repo-update` before you continue.

	If you encounter an error message that begins `[!] Failed to connect to GitHub to update the CocoaPods/Specs . . .`, and your internet connectivity is working, you may need to [update openssl and Ruby](https://stackoverflow.com/questions/38993527/cocoapods-failed-to-connect-to-github-to-update-the-cocoapods-specs-specs-repo/48962041#48962041).

2. Classes that call |ABP| APIs must use the following import statements:

	```
	import AWSAuthCore
	import AWSCore
	import AWSAPIGateway
	import AWSMobileClient
	```

3. Next, import files generated by CLI. The CLI generates a client code file and request-response structure file for each API you add.

4. Add those files by going to your Xcode Project Navigator project, right-click on project's name in top left corner, and select "Add Files to YOUR_APP_NAME".

5. Select all the files under `generated-src` folder of your application's root folder and add them to your project.

6. Next, set the bridging header for Swift in your project settings. Double-click your project name in the Xcode Project Navigator, choose the Build Settings tab and search for  `Objective-C Bridging Header`. Enter `generated-src/Bridging_Header.h`

	This is needed because the AWS generated code has some Objective-C code which requires bridging to be used for Swift.

	> If you already have a bridging header in your app, you can just append an extra line to it: `#import "AWSApiGatewayBridge.h"` instead of above step.

7. Use the files generated by CLI to determine the client name of your API. In the `generated-src` folder, files ending with name `*Client.swift` are the names of your client (without .swift extension).

	The path of the client code file is `./generated-src/YOUR_API_RESOURCE_NAME+YOUR_APP_NAME+Client.swift`.

	So, for an app named `useamplify` with an API resource named `xyz123`, the path of the code file might be `./generated-src/xyz123useamplifyabcdClient.swift`. The API client name would be `xyz123useamplifyabcdClient`.

	- Find the resource name of your API by running `amplify status`.
	- Copy your API client name to use when invoking the API in the following step.


8. Invoke a Cloud Logic API.

	To invoke a Cloud Logic API, create code in the following form and substitute your API's
	client class, model, and resource paths. Replace `YOUR_API_CLIENT_NAME` with the value you copied from the previous step.

	```swift
  import UIKit
  import AWSAuthCore
  import AWSCore
  import AWSAPIGateway
  import AWSMobileClient

  // ViewController or application context . . .

    func doInvokeAPI() {
         // change the method name, or path or the query string parameters here as desired
         let httpMethodName = "POST"
         // change to any valid path you configured in the API
         let URLString = "/items"
         let queryStringParameters = ["key1":"{value1}"]
         let headerParameters = [
             "Content-Type": "application/json",
             "Accept": "application/json"
         ]

         let httpBody = "{ \n  " +
                 "\"key1\":\"value1\", \n  " +
                 "\"key2\":\"value2\", \n  " +
                 "\"key3\":\"value3\"\n}"

         // Construct the request object
         let apiRequest = AWSAPIGatewayRequest(httpMethod: httpMethodName,
                 urlString: URLString,
                 queryParameters: queryStringParameters,
                 headerParameters: headerParameters,
                 httpBody: httpBody)

        // Create a service configuration
        let serviceConfiguration = AWSServiceConfiguration(region: AWSRegionType.USEast1,
              credentialsProvider: AWSMobileClient.sharedInstance().getCredentialsProvider())

        // Initialize the API client using the service configuration
        xyz123useamplifyabcdClient.registerClient(withConfiguration: serviceConfiguration!, forKey: "CloudLogicAPIKey")

        // Fetch the Cloud Logic client to be used for invocation
        let invocationClient = xyz123useamplifyabcdClient.client(forKey: "CloudLogicAPIKey")

        invocationClient.invoke(apiRequest).continueWith { (task: AWSTask) -> Any? in
                 if let error = task.error {
                     print("Error occurred: \(error)")
                     // Handle error here
                     return nil
                 }

                 // Handle successful result here
                 let result = task.result!
                 let responseString = String(data: result.responseData!, encoding: .utf8)

                 print(responseString)
                 print(result.statusCode)

                 return nil
             }
         }
	```
