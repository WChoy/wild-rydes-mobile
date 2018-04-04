# Wild Rydes Mobile

Welcome to the Wild Rydes Mobile project.  In this project, we will take you through the steps necessary to build a complete website with analytics, authentication, email list sign-up and booking of a ride.

## Lesson 1: Web Hosting

The design team has dropped the final version of the web site, but has not set up web hosting.  In this lesson, we will quickly set up a scalable web site with global reach and deploy the code to the web site.

Download and run the project locally:

```
git clone https://github.com/aws-samples/wild-rydes-mobile
cd wild-rydes-mobile
yarn install
yarn start
```

To build the website in the cloud:

```
awsmobile init
```

For the questions:

* Where is your project's source directory: (src) **src**
* Where is your project's distribution directory that stores build artifacts (build) **build**
* What is your project's build command: (npm run-script build) **yarn run build**
* What is your project's start command for local test run: (npm run-script start) **yarn start**
* What awsmobile project name would you like to use: (wild-rydes-mobile-2018-03-09-14-57-35) **wild-rydes-mobile**

The AWS Mobile Hub project will be created in the cloud.  Use the following to build and publish the project to the cloud:

```
awsmobile publish
```

A browser will open pointing to the newly created site once publication completes.

## Lesson 2: Email Campaign

One of the features on the home page is an email sign-up page.  In this lesson, you will link the email sign-up to Amazon Pinpoint, then use the Amazon Pinpoint user segmentation and campaigns features to send an email to all the registered users.

Add the AWS Amplify and the AWS SDK for JavaScript to the project:

```
yarn add aws-sdk aws-amplify uuid
```

Edit the `src/index.js` file.  Add the following lines to the top of the file (below all the other imports):

```
import Amplify from 'aws-amplify';
import awsConfig from './aws-exports';

Amplify.configure(awsConfig);
```

Edit the `src/components/EmailSignUp.js` file.  Add the following lines to the top of the file:

```
import uuid from 'uuid';
import { Auth } from 'aws-amplify';
import * as Pinpoint from 'aws-sdk/clients/pinpoint';
import awsConfig from '../aws-exports';
```

Then look for `/* TODO: HANDLE FORM INPUT */` - this is where we need to add code to submit the email address into the Amazon Pinpoint analytics platform.  Replace the onEmailSubmitted() method that is there now with the following:

```
  async onEmailSubmitted(event) {
    event.preventDefault();

    /* TODO: HANDLE FORM INPUT */
    // Create a Pinpoint connection
    const credentials = await Auth.currentCredentials();
    const pinpointClient = new Pinpoint({
      region: awsConfig.aws_mobile_analytics_app_region,
      credentials: Auth.essentialCredentials(credentials)
    });
    if (!this.endpointId)
      this.endpointId = uuid.v4();

    // Create an endpoint definition
    const params = {
      ApplicationId: awsConfig.aws_mobile_analytics_app_id,
      EndpointId: this.endpointId,
      EndpointRequest: {
        Address: this.state.email,
        ChannelType: 'EMAIL',
        EffectiveDate: new Date().toISOString(),
        OptOut: 'NONE',
        RequestId: uuid.v4(),
        User: {
          UserAttributes: {
            email: [ this.state.email ],
            emailsignup: [ 'true' ]
          },
          UserId: this.state.email
        }
      }
    };

    // Update the endpoint definition
    pinpointClient.updateEndpoint(params, (err, data) => {
      if (err) {
        alert('Email Signup Failed');
        console.error('updateEndpoint: ', err);
      } else {
        this.setState({ email: '', emailsubmitted: true });
      }
    });
    /* END OF PINPOINT CHANGES */
  }
```

Publish the code with `awsmobile publish`.  Once the app has been published, enter your email address within the email sign-up form and click **Submit**.

### Configure the Email Channel

* Open the AWS Mobile Hub console, and select your project.
* Choose **Analytics** (in the top-right corner).
* Choose **Settings** (in the left-hand menu).
* Choose **Channels** (along the top)
* Choose **Email** > **Enable Email Channel**
* Choose **Email Address**, enter your email address.
* If required, click Verify and complete the verification process.
* Choose **Save**.

### Create a User Segment

* Choose **Segments** (in the left-hand menu).
* Choose **New segment**.
* Give your segment a name like "Email List Users"
* Fill in the form:
	* **Build segment**.
	* **Email**.
	* App usage criteria: Have **used** the app **at least once**.
        * Filter by user attributes: **emailsignup** > **true**.
* Validate that the segment estimate is 1 user.
* Choose **Create segment**.

### Create an Email Campaign

* Choose **Campaigns**
* Choose **New campaign**
* Name the campaign "New Email List Campaign", then choose **Next step**
* Choose **Use a previously defined segment**
* Select your segment, then choose **Next step**
* Enter a subject and body, then choose **Next step**
* Select **Immediate**, then choose **Next step**
* Choose **Launch campaign**

Check your email for your test message!

## Lesson 3: Authentication

Sign up, sign in, and forgot password screens have been provided, but need to be wired up to the main application.  First, create an Amazon
Cognito user pool:

```
awsmobile user-signin enable
awsmobile push
```

This will update the backend without publishing updated code.  Now, update the `src/auth/SignUp.js` file to do the sign-up process.  First, add the appropriate import at the top of the file:

```
import { Auth } from 'aws-amplify';
```

Next, adjust the `onSubmitForm(e)` method:

```
  async onSubmitForm(e) {
    e.preventDefault();
    try {
      const params = {
        username: this.state.email.replace(/[@.]/g, '|'),
        password: this.state.password,
        attributes: {
          email: this.state.email,
          phone_number: this.state.phone
        },
        validationData: []
      };
      const data = await Auth.signUp(params);
      console.log(data);
      this.setState({ stage: 1 });
    } catch (err) {
      alert(err.message);
      console.error("Exception from Auth.signUp: ", err);
      this.setState({ stage: 0, email: '', password: '', confirm: '' });
    }
  }
```

Adjust the `onSubmitVerification(e)` method:

```
  async onSubmitVerification(e) {
    e.preventDefault();
    try {
      const data = await Auth.confirmSignUp(
        this.state.email.replace(/[@.]/g, '|'),
        this.state.code
      );
      console.log(data);
      // Go to the sign in page
      this.props.history.replace('/signin');
    } catch (err) {
      alert(err.message);
      console.error("Exception from Auth.confirmSignUp: ", err);
      this.setStatate({ stage: 0, email: '', password: '', confirm: '', code: '' });
    }
  }
```

Update the `src/auth/SignIn.js` file to do the sign-in process.  First add the required import:

```
import { Auth } from 'aws-amplify';
```

Adjust the `onSubmitForm(e)` method:

```
  async onSubmitForm(e) {
    e.preventDefault();
    try {
        const userObject = await Auth.signIn(
            this.state.email.replace(/[@.]/g, '|'),
            this.state.password
        );
        console.log('userObject = ', userObject);
        this.setState({ userObject, stage: 1 });
    } catch (err) {
        alert(err.message);
        console.error('Auth.signIn(): ', err);
    }
  }
```

Also, adjust the `onSubmitVerification(e)` method:

```
  async onSubmitVerification(e) {
    e.preventDefault();
    try {
        const data = await Auth.confirmSignIn(
            this.state.userObject,
            this.state.code
        );
        console.log('data = ', data);
        this.setState({ stage: 0, email: '', password: '', code: '' });
        this.props.history.replace('/app');
    } catch (err) {
        alert(err.message);
        console.error('Auth.confirmSignIn(): ', err);
    }
  }
```

Finally, take a look at the route configuration in `src/index.js`.  We
need to update so that the current authentication is read from local
storage, then adjust the routing based on authentication.  We need to 
read the current state of the authentication:

```
const isAuthenticated = () => Amplify.Auth.user != null;
```

Run the app locally with `awsmobile run`.  You should be able to sign up (using the Apply link in the menu), or click **Giddy Up!** on the home page.  If you are already signed in, it will take you straight to the app.  Otherwise, it will ask you to sign-in first.

Run the following to publish the new site:

```
awsmobile publish -c -n -f
```

