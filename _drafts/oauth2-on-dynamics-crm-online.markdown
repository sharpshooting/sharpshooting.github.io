---
layout: post
title:  "OAuth 2.0 with Dynamics CRM Online"
date:   2015-03-23 14:03:00
categories:
- crm
- mobile
- authentication
---
Authentication on Dynamics CRM Online follows an OAuth 2.0 authorization code grant flow and is fairly straightforward. There are many libraries that handle OAuth 2.0 such as <a href="https://msdn.microsoft.com/en-us/library/azure/dn151135.aspx" target="_blank">Microsoft ADAL</a>, but it can be useful to understand what's happening under the hood.

# Initial configuration #

If you're developing a native client application that will use Dynamics CRM data, <a href="http://azure.microsoft.com/en-us/pricing/details/active-directory/" target="_blank">Microsoft Azure AD</a> can be used to perform authentication against Dynamics CRM Online. Microsoft Azure AD includes a free tier. The following steps are all it takes to set it up:

1. Create a new Active Directory on Microsoft Azure
1. Navigate to the directory and select the tab 'Applications'
1. Add a new (click on the 'Add' button on the command bar)
1. Select the option 'Add an application my organization is developing'
1. Provide a name to the application and select 'Native Client Application'
1. Provide a redirect URL, which for the purposes of this tutorial can be the URL to your Dynamics CRM Online organization
1. Navigate to the newly created application and select the tab 'Configure'
1. Save the Client ID string. The application must identify itself by using this code
1. Find the 'Permissions to Other Applications' section and add a new application
1. Select 'Dynamics CRM Online' and save
1. Back to the screen with the 'Permissions to Other Applications', add the delegate permission 'Access CRM Online as organization users'

# Exchanging tokens #

First, an auth endpoint discovery request is sent. Note that specifying the SDK version is required:

{% highlight http %}
GET https://ORG_NAME.crm.dynamics.com/XRMServices/2011/Organization.svc/web?SdkClientVersion=6.1.0.533 HTTP/1.1
{% endhighlight %}

This returns the auth endpoint on the response header:

{% highlight http %}
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer authorization_uri=https://login.windows.net/TENTAND_ID/oauth2/authorize

HTTP Error 401 - Unauthorized: Access is denied
{% endhighlight %}

Next, a request to the auth endpoint is made with resource, client id, response type (= "code"), redirect uri and prompt (= "login") parameters on the query string:

{% highlight http %}
GET https://login.windows.net/TENTAND_ID/oauth2/authorize?resource=RESOURCE_UI&client_id=CLIENT_ID&response_type=code&redirect_uri=REDIRECT_URI&prompt=login HTTP/1.1
{% endhighlight %}

Which returns the interactive auth form on login.windows.net. Once the user is authenticated, a POST request is made from the browser to get the authorization code:

{% highlight http %}
POST https://login.windows.net/<TENANT_ID>/wsfederation HTTP/1.1
{% endhighlight %}

But the code is passed directly to the application as a query string parameter since a redirect instruction is returned in the response. The OS should handle application switching:

{% highlight http %}
HTTP/1.1 302 Found
Location: REDIRECT_URI/?code=AUTHORIZATION_CODE
{% endhighlight %}

The application can then issue a request with the authorization code to get the access token. Grant type (= "authorization_code"), the actual authorization code, client id, redirect uri and resource are passed into the request body:

{% highlight http %}
POST https://login.windows.net/c9c8d837-ca0b-4ebd-88a5-ff578008c93d/oauth2/token HTTP/1.1

resource=ORG_NAME.crm.dynamics.com&client_id=CLIENT_ID&grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=REDIRECT_URI
{% endhighlight %}

The response will be a JSON object containing the access token. Also, refresh token and token duration are passed so that the application can handle the token life cycle:

{% highlight http %}
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
{"token_type":"Bearer","expires_in":"3600","expires_on":"1234567890","not_before":"1234567890","resource":"https://ORG_NAME.crm.dynamics.com","access_token":"ACCESS_TOKEN","refresh_token":"REFRESH_TOKEN","scope":"user_impersonation","id_token":"ID_TOKEN","pwd_exp":"1234567","pwd_url":"https://portal.microsoftonline.com/ChangePassword.aspx"}
{% endhighlight %}

Every subsequent request to the service needs to include the access token as such:

{% highlight http %}
GET https://ORG_NAME.crm.dynamics.com/xrmservices/2011/organizationdata.svc/ContactSet HTTP/1.1
Authorization: Bearer ACCESS_TOKEN
{% endhighlight %}
