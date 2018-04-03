---
published: true
title: Retrieve an OAuth client token from Azure AD using Runscope
tags:
  - azure
  - security
header:
  image: /images/runscopeteststep.png
excerpt: >-
  Runscope is a great online tool to validate and test API endpoints. For a
  recent project, we are using it to mimic traffic from an external system that
  is supposed to submit XML files to our application. The application, however,
  requires an authorization token with a valid JWT to authenticate and authorize
  the caller. As there is a lifetime on those tokens, we need to retrieve a new
  one each time we start a test. Luckily that is not too difficult in Runscope.
---
[Runscope](http://www.runscope.com) is a great online tool to validate and test API endpoints. For a recent project, we are using it to mimic traffic from an external system that is supposed to submit XML files to our application.

The application, however, requires an authorization token with a valid JWT to authenticate and authorize the caller. As there is a lifetime on those tokens, we need to retrieve a new one each time we start a test. Luckily that is not too difficult in Runscope.

For this, I created a new test called **Token** with a single test step. In this step, I do a POST to `https://login.microsoftonline.com/yourtenantname/oauth2/token`, which is the Azure Active Directory endpoint to fetch tokens. Put in your own tenant name or id of your Azure AD.

As we do a form post, we need to add a _content-type_ of `application/x-www-form-urlencoded`.
We are going to post four parameters, the _client_id_ and _client_secret_, which are used in the client OAuth flow to identify and authorize the client, and a _resource_id_ and _grant_type_. The grant type is `client_credentials` as that is the way we authenticate. 

The _resource_id_ is the id of the application you want to access. You can find those id's in the Azure Portal under the Application registrations.

The client id and secret are coming from the application you have created in the same Azure portal. The client id is visible in the main overview and the secret can be generated from the settings.

You end up with a test like below

![runscopeteststep.png](/images/runscopeteststep.png)

When running the test, the call will be made and, if all parameters are correct, there will be a response in JSON. One of the parameters is the actual access code which we need in subsequent tests. Using the variables option, you can retrieve this property.

![runscopevariable.png](/images/runscopevariable.png)

The JSON body will be inspected for a property called `access_token` which is then stored in the variable called `token`.

We can now use this token in the other calls to our application by including it in an `Authorization: Bearer {{token}}` header.

However, it is not very efficient to repeat the above steps in each test. Runscope supports subtests to handle this. So create a new test, add a **subtest** step and refer to the test case containing the logic to fetch the token.

![runscopesubtest.png](/images/runscopesubtest.png)

The execution of this subtest produces a response with variables, which then contains the token we extracted before. So as shown above, we use variables again to extract from the JSON body the token from the `variables.token`.

Now in your further test steps, just include the `Authorization: Bearer {{token}}` to reuse the token in the calls.
