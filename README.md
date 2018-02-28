What is ActiveDocs?
===================

ActiveDocs is the 3scale feature that is based on [OAI 2.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md) & [SwaggerUI](https://github.com/swagger-api/swagger-ui). You can host any OAI compliant spec on 3scale & publish it in the Developer Portal for your community's reference & testing. One of the great advantages of the 3scale ActiveDocs is that it has its own proxy which supports CORS (only when using the SaaS platform). Perfect, no need to configure your own API to support CORS for this purpose.

Additionally there are some [custom 3scale fields](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/api_documentation/create-activedocs-spec#useful_tools) that you can use inside the OAI spec to expose the currently logged in user's credentials for easy use. No copy-pasting those multiple sets of credentials to files where you're never going to remember them. The ActiveDocs feature doesn't support OAuth2.0 out-of-the-box. Therefore this "How to" is intended to provide a means of enabling an OAuth2 flow on the documentation page exposing your API services.

Prerequisites
=============

*   A Red Hat Single Sign-on server configured according to our [Supported Configurations](https://access.redhat.com/articles/2798521#apicast-3x-support-4). Create the realm as [documented](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/developer_portal/authentication#enabling_and_disabling_authentication_via_red_hat_single_sign_on_7_0).
*   An OAI compliant spec with an `Authorization` header field for each operation that requires a token to call the API.

What will be covered?
=====================

*   How to configure Red Hat Single Sign-On server and the client.
*   How to configure 3scale.
*   Implementing the custom JavaScript client and Liquid to enable **Authorization Code flow**.

Configure Red Hat Single Sign-On & example client
-------------------------------------------------

Once the server and realm are configured according to the documentation mentioned above the client should then be set up following these steps.

### Step 1

Add a `redirect_uri` equivalent to the developer portal domain plus the path where the documentation is hosted. This value can also be a wildcard if the ActiveDocs specs are to be hosted on multiple pages in the portal.

### Step 2

Add the developer portal domain as the `Web Origin` value. For example: `https://{account-name}.example.com`. This is necessary for CORS requests to succeed for each client.

### Step 3

Enable the `Standard Flow enabled` switch for the **Authorization Code flow**.

Configure 3scale & the client
-----------------------------

If you're using the [OpenID Connect integration](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/api_authentication/rhsso) then the 3scale platform manages the synchronisation of clients into your Red Hat Single Sign-On server for you. In which case you can skip step 1. If you are also using the [Red Hat Single Sign-On devloper portal integration](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/developer_portal/authentication#enabling_and_disabling_authentication_via_red_hat_single_sign_on_7_0) then skip step 2 as well. Otherwise, follow all the steps below.

### Step 1

Create a client in 3scale via API if the client already has been created in your Red Hat Single Sign-On server. Use the credentials (`client_id` & `client_secret`) in the example request as shown here.

    curl -v  -X POST "https://{account-name}-admin.3scale.net/admin/api/accounts/{account_id}/applications.xml" -d 'access_token={access_token}&plan_id={plan_id}&name={application_name}&description={application_description}&application_id={client_id}&application_key={client_secret}'
    

This is probably a bit quicker and easier for testing purposes. However, in production it makes much more sense that the synchronisation is done from 3scale to Red Hat Single Sign-On as these are the client and token masters respectively. The clients can be created via [API in Red Hat Single Sign-On](https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.0/html/securing_applications_and_services_guide/client_registration#example_using_curl_2) also.

### Step 2

Add the Red Hat Single Sign-On URL to your developer portal SSO integrations if you haven't already done so. Follow the [Configuring 3scale](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/developer_portal/authentication#configuring_3scale) section to do this. This will then be reused in the Liquid templates in the developer portal.

### Step 3

Import the OAI spec using the ActiveDocs API. The easiest way to manage all your different API specifications is to host them directly on the 3scale platform. An example API call is shown here to import a spec.

    curl -v  -X POST "https://{account-name}-admin.3scale.net/admin/api/active_docs.json" -d 'access_token={access_token}&name={spec_friendly_name}&system_name={spec_system_name}&body={spec.json}&published=true'
    

Ensure that the spec has the following field definition in the parameters array for each operation that requires a token to call the API endpoint.

    "parameters": [
              {
                "type": "string",
                "description": "Authorization header\n(format: Bearer [access_token])",
                "name": "Authorization",
                "in": "header",
                "required": true
              },
    

Add the JavaScript client & custom Liquid
-----------------------------------------

First let's add the [cookie.js](https://github.com/kevprice83/activedocs-keycloak-client/blob/master/cookie.js) module to the 3scale CMS. In the _Developer Portal_ tab of the admin portal you can choose _"New Page"_ or _"New File"_ from the dropdown button. Configure the relevant attributes whether you add it as a file or page. Choose a **Title** that is appropriate, **Section** should be _javascripts_, **Path** must be the format `{filename}.js`, **Layout** must be empty and finally the **Content Type** _JavaScript_.

The partials for the [oauth widget](https://github.com/kevprice83/activedocs-keycloak-client/blob/master/widget.js) & the [keycloak client](https://github.com/kevprice83/activedocs-keycloak-client/blob/master/auth.js) should now be uploaded to the CMS. The name you choose here will be reused in the main template in the `{% include %}` Liquid tag. From the same dropdown button choose _"New Partial"_. Now upload the changes required to your documentation template. You can see the necessary Liquid tags in the [example docs template](https://github.com/kevprice83/activedocs-keycloak-client/blob/master/docs.html.liquid). This will work with both **SwaggerUI 2.1.3** & **2.2.10**. In the older version the Liquid tag to include the ActiveDocs spec would have looked something like: `{% active_docs version: '2.0', services: 'spec_system_name' %}`. If you want to upgrade to the latest version supported in the 3scale platform then follow the [upgrade tutorial](https://access.redhat.com/documentation/en-us/red_hat_3scale/2.saas/html/api_documentation/activedocs-upgrade-22). The OAuth widget partial should be referenced in the first `{% include %}` and the Keycloak client last.

Everything in the JavaScript and Liquid is fully dynamic, therefore all the account specific attributes like; developer portal domain, documentation page URL, application `client_id`, `client_secret` etc do not need to be hardcoded anywhere.

### How the client works

The OAuth widget checks if the current page contains a `state` parameter in the URL and renders the appropriate button to **authorize** or **get_token_**_. A dropdown of applications become available for the logged in user to retrieve a token. The application name and Service name are rendered but this is customisable to meet your needs. The **authorize** will execute the \*cookie.js\* module and a `state` value will be stored in the cookies with a default expiration of 60 seconds, this can also be configured as you wish. The redirect to the login page will take place and upon successful authorization a success message will be displayed and when the user is redirected to the developer portal a **get**_**token** button will be rendered. The same application should be selected for the next leg of the flow which will result in a token returned to the browser if successful. The state value returned by the Red Hat Single Sign-On server during the _callback_ is validated by the client against the original value stored in the cookie.