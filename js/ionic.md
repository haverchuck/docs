---
---

# Angular & Ionic

AWS Amplify helps developers to create high-quality Angular and Ionic apps quickly by handling the heavy lifting of configuring and integrating cloud services behind the scenes. It also provides a powerful high-level API and ready-to-use security best practices.

The `aws-amplify-angular` package is compatible with Angular 5+ and Ionic 4.

## Installation

AWS Amplify provides Angular Components that you can use with Ionic in [aws-amplify-angular](https://www.npmjs.com/package/aws-amplify-angular) npm package.

Install `aws-amplify` and `aws-amplify-angular` npm packages into your Angular app.

```bash
$ npm install --save aws-amplify
$ npm install --save aws-amplify-angular
```

## Setup the AWS Backend

To configure your Ionic app for AWS Amplify, you need to create a backend configuration with Amplify CLI and import the auto-generated configuration file into your project. 

Following commands will enable Auth category and will create `aws-exports.js` configuration file under your projects `/src` folder. 

```bash
$ npm install -g @aws-amplify/cli
$ amplify init
$ amplify add auth
$ amplify add storage
$ amplify push # Updates your backend
```

Please visit [Authentication Guide]({%if jekyll.environment == 'production'%}{{site.amplify.docs_baseurl}}{%endif%}/js/authentication)  and [Storage Guide]({%if jekyll.environment == 'production'%}{{site.amplify.docs_baseurl}}{%endif%}/js/storage) to learn more about enabling these categories.
{: .callout .callout--info}


A configuration file is placed inside your configured source directory. To import the configuration file to your Ionic app, you will need to rename `aws-exports.js` to `aws-exports.ts`. You can setup your `package.json` npm scripts to rename the file for you, so that any configuration changes which result in a new generated `aws-exports.js` file get changed over to the `.ts` extension.

```javascript	
"scripts": {	
    "start": "[ -f src/aws-exports.js ] && mv src/aws-exports.js src/aws-exports.ts || ng serve; ng serve",	
    "build": "[ -f src/aws-exports.js ] && mv src/aws-exports.js src/aws-exports.ts || ng build --prod; ng build --prod"	
}	
```

## Import and Configure Amplify

Import the configuration file and configure Amplify in your `main.ts` file. 

```javascript
import Amplify from 'aws-amplify';
import amplify from './aws-exports';
Amplify.configure(amplify);
```

In your [home page component](https://angular.io/guide/bootstrapping) `src/app/app.module.ts`, you can import Amplify modules as following:

```javascript
import { AmplifyAngularModule, AmplifyService } from 'aws-amplify-angular';

@NgModule({
  ...
  imports: [
    ...
    AmplifyAngularModule
  ],
  ...
  providers: [
    ...
    AmplifyService
  ]
  ...
});
```

NOTE: the service provider is optional. You can import the core categories normally i.e. `import { Analytic } from 'aws-amplify'` or create your own provider. The service provider does some work for you and exposes the categories as methods. The provider also manages and dispatches authentication state changes using observables which you can subscribe to within your components (see below).

## Using the AWS Amplify API

AmplifyService is a provider in your Angular app, and it provides AWS Amplify core categories through dependency injection. To use *AmplifyService* with [dependency injection](https://angular.io/guide/dependency-injection-in-action), inject it into the constructor of any component or service, anywhere in your application.

```javascript
import { AmplifyService } from 'aws-amplify-angular';

...
constructor(
    public navCtrl:NavController,
    public amplifyService: AmplifyService,
    public modalCtrl: ModalController
) {
    this.amplifyService = amplifyService;
}
...
```

You can access and work directly with AWS Amplify Categories via the built-in service provider:

```javascript
import { Component } from '@angular/core';
import { AmplifyService }  from 'aws-amplify-angular';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})

export class AppComponent {
  
  constructor( public amplify:AmplifyService ) {
      
      this.amplifyService = amplify;
      
      /** now you can access category APIs:
       *
       * this.amplifyService.auth();          // AWS Amplify Auth
       * this.amplifyService.analytics();     // AWS Amplify Analytics
       * this.amplifyService.storage();       // AWS Amplify Storage
       * this.amplifyService.api();           // AWS Amplify API
       * this.amplifyService.cache();         // AWS Amplify Cache
       * this.amplifyService.pubsub();        // AWS Amplify PubSub
       * this.amplifyService.interactions();  // AWS Amplify Interactions
       *     
      **/
  }
  
}
```

You can access all [AWS Amplify Category APIs](https://aws-amplify.github.io/amplify-js/media/developer_guide) with *AmplifyService*. 

## Subscribe to Authentication State Changes

Import AmplifyService into your component and listen for auth state changes:

```javascript
import { AmplifyService }  from 'aws-amplify-angular';

  // ...
constructor( public amplifyService: AmplifyService ) {

    this.amplifyService = amplifyService;

    this.amplifyService.authStateChange$
        .subscribe(authState => {
            this.signedIn = authState.state === 'signedIn';
            if (!authState.user) {
                this.user = null;
            } else {
                this.user = authState.user;
                this.greeting = "Hello " + this.user.username;
            }
        });

}
```

## Use View Components

AWS Amplifies provides components that you can use in your Angular view templates. 

### Authenticator

The Authenticator component creates an out-of-the-box signing/sign-up experience for your Angular app. 

Before using this component, please be sure that you have activated [Authentication category](https://aws-amplify.github.io/amplify-js/media/authentication_guide):
```bash
$ amplify add auth
```


To use Authenticator, just add the `amplify-authenticator` directive in your .html view:
```html
  ...
  <amplify-authenticator framework="Ionic"></amplify-authenticator>
  ...
```

#### SignUp Configuration
The SignUp component provides your users with the ability to sign up.  It is included as part of the ```authenticator``` component, but can also be used in isolation:

Usage: 
```<amplify-auth-sign-up [signUpConfig]="signUpConfig"></amplify-auth-sign-up>```
or
```<amplify-authenticator [signUpConfig]="signUpConfig"></amplify-authenticator>```

The SignUp Component accepts a 'signUpConfig' object which allows you to customize it.

{% include sign-up-attributes.html %}

The signUpFields array in turn consist of an array of objects, each describing a field that will appear in sign up form that your users fill out:

{% include sign-up-fields.html %}


### Photo Picker

Photo Picker component will render a file upload control that will allow choosing a local image and uploading it to Amazon S3. Once an image is selected, a base64 encoded image preview will be displayed automatically.

Before using this component, please be sure that you have activated [*user-files* with Amplify CLI](https://docs.aws.amazon.com/aws-mobile/latest/developerguide/aws-mobile-cli-reference.html):

```bash
$ amplify add storage
```

To render photo picker in an Angular view, use *amplify-photo-picker* component:

```html
<amplify-photo-picker framework="Ionic"
    (picked)="onImagePicked($event)"
    (loaded)="onImageLoaded($event)">
</amplify-photo-picker>
```

The component will emit two events:

 - `(picked)` - Emitted when an image is selected. The event will contain the `File` object which can be used for upload.
 - `(loaded)` - Emitted when an image preview has been rendered and displayed.
 - `path` - An optional S3 image path (prefix).
 - `storageOptions` - An object passed within the ‘options’ property in the Storage.put request. This can be used to set the permissions ‘level’ property of the objects being uploaded i.e. ‘private’, ‘protected’, or ‘public’.  
 
 [Learn more about S3 permissions.]({%if jekyll.environment == 'production'%}{{site.amplify.docs_baseurl}}{%endif%}/js/storage#file-access-levels).


### S3 Album

S3 Album component display a list of images from the connected S3 bucket.

Before using this component, please be sure that you have activated [*user-files* with Amplify CLI](https://docs.aws.amazon.com/aws-mobile/latest/developerguide/aws-mobile-cli-reference.html):

```bash
$ amplify add storage
```

To render the album, use *amplify-s3-album* component in your Angular view:

```html
<amplify-s3-album framework="Ionic" 
    path="pics" (selected)="onAlbumImageSelected($event)">
</amplify-s3-album>
```

- `options` - object which is passed as the 'options' parameter to the .get request.  This can be used to set the 'level' of the objects being requested (i.e. 'protected', 'private', or 'public'
- `(selected)` event can be used to retrieve the URL of the clicked image on the list:

```javascript
onAlbumImageSelected( event ) {
      window.open( event, '_blank' );
}
```

### Interactions

The `amplify-interactions` component provides you with a Chatbot user interface. You can pass it three parameters:

1. `bot`:  The name of the Amazon Lex Chatbot

2. `clearComplete`:  A flag indicating whether or not the messages should be cleared at the
end of the conversation.

3. `complete`: A function that is executed when the conversation is completed.

```html
<amplify-interactions bot="yourBotName" clearComplete="true" (complete)="onBotComplete($event)"></amplify-interactions>
```

See the [Interactions documentation]({%if jekyll.environment == 'production'%}{{site.amplify.docs_baseurl}}{%endif%}/js/interactions) for information on creating an Amazon Lex Chatbot.

### XR

#### Sumerian Scene

The `amplify-sumerian-scene` component provides you with a prebuilt UI for loading and displaying Amazon Sumerian scenes inside of your website:

{% include_relative common/scene-size-note.md %}

```javascript
// sceneName: the configured friendly scene you would like to load
<amplify-sumerian-scene sceneName="scene1" framework="Ionic"></amplify-sumerian-scene>
```

See the [XR documentation]({%if jekyll.environment == 'production'%}{{site.amplify.docs_baseurl}}{%endif%}/js/xr) for information on creating and publishing a Sumerian scene.

### Styles

To use the aws-amplify-angular components you will need to install '@aws-amplify/ui'.

Add the following to your styles.css file to use the default styles:
```@import '~aws-amplify-angular/theme.css';```

You can use custom theming for AWS Amplify components. Just import your custom *styles.css* that overrides the default styles which can be found in `/node_modules/aws-amplify-angular/theme.css`.

### Custom Styles

*Note: This feature is currently in beta, and is available only for the authentication components.*

The components in `aws-amplify-angular` use CSS modules from the `@amplify/ui` package.  This helps ensure consistency across components.

You can append classes to the various elements of the `aws-amplify-angular` components by using the classOverrides feature.  This feature allows you to pass in an array of classes that pertain to a given CSS module from `@amplify/ui` package; these classes are then appended the class attribute in the markup.  Your custom classes should reside in your own stylesheet(s).  

Be sure to remember that css classes derive precedence from the order in which they are encoutered in the stylesheet.

For components which serve as parents for other components, such as `amplify-authenticator`, you can provide overrides for all of the children by passing the classOverrides prop to the parent component:

<amplify-authenticator  
  [classOverrides]="classOverrides"
</amplify-authenticator>

```js 
  classOverrides = {
    sectionHeader: ['class1', 'class2']
  };
```

The same mechanism can be used for child components, but the classOverrides are properties if config objects for each child:

<amplify-authenticator  
  [classOverrides]="classOverrides"
  [signUpConfig]="signUpConfig"
  [signInConfig]="signInConfig"
  [confirmSignUpConfig]="confirmSignUpConfig"
  [confirmSignInConfig]="confirmSignInConfig"
  [requireNewPasswordConfig]="requireNewPasswordConfig"
  [forgotPasswordConfig]="forgotPasswordConfig"
</amplify-authenticator>

```js 
  signUpConfig = {
    classOverrides: {
      sectionHeader: ['class2']
    }
  };
  // other configs as necessary
```

The list of classes that are available include: 

{% include ui-classes-angular.html %}

## Angular 6 Support

Currently, the newest version of Angular (6.x) does not provide the shim for the  `global` object, which was provided in previous versions. Specific AWS Amplify dependencies rely on this shim.  While we evaluate the best path forward to address this issue, you have a couple of options for re-creating the shim in your Angular 6 app to make it compatible with Amplify.

1.  Add the following to your polyfills.ts: `(window as any).global = window;`.

2.  Add the following script to your index.html `<head>` tag:
``` 
    <script>
        if (global === undefined) {
            var global = window;
        }
    </script>
  ```
