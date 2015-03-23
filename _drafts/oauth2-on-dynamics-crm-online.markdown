---
layout: post
title:  "OAuth2 on Dynamics CRM Online"
date:   2015-03-03 23:56:00
categories: microsoft dynamics crm, crm online, oauth2, authentication
---
Authentication on Dynamics CRM Online follows an OAuth2 flow and is fairly straightforward. While there are many libraries that handle OAuth2 authentication (such as Microsoft ADAL), it can be useful to understand what's happening under the hood.

# Initial configuration #

Microsoft Azure AD is necessary to support OAuth2 on Dynamics CRM Online. There is a free tier for this service, though.

1. Create a new Active Directory on Microsoft Azure
1. Navigate to the directory and select the tab 'Applications'
1. Add a new (click on the 'Add' button on the command bar)
1. Select the option 'Add an application my organization is developing'
1. Provide a name to the application and select 'Native Client Application'
1. Provide a redirect URL, which for the purposes of this tutorial can be the URL to your Dynamics CRM Online organization
1. Navigate to the newly created application and select the tab 'Configure'
1. Save the Client ID string
1. Find the 'Permissions to Other Applications' section and add a new application
1. Select 'Dynamics CRM Online' and save
1. Back to the screen with the 'Permissions to Other Applications', add the delegate permission 'Access CRM Online as organization users'

# Exchanging tokens #

First an auth endpoint discovery request is sent:

{% highlight http %}
GET https://ORG_NAME.crm.dynamics.com/XRMServices/2011/Organization.svc/web?SdkClientVersion=6.1.0.533 HTTP/1.1
{% endhighlight %}

Which returns the auth endpoint on the response header:

{% highlight http %}
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer authorization_uri=https://login.windows.net/TENTAND_ID/oauth2/authorize

HTTP Error 401 - Unauthorized: Access is denied
{% endhighlight %}

Next, a request to the auth endpoint is made with resource, client id, response type (= code), redirect uri and prompt (= login) parameters on the query string:

{% highlight http %}
GET https://login.windows.net/TENTAND_ID/oauth2/authorize?resource=RESOURCE_UI&client_id=CLIENT_ID&response_type=code&redirect_uri=REDIRECT_URI&prompt=login HTTP/1.1
{% endhighlight %}

Which returns the interactive auth form, in this case login.windows.net. Once the user is authenticated, a POST request is made on the client side to get the authorization code:

{% highlight http %}
POST https://login.windows.net/<TENANT_ID>/wsfederation HTTP/1.1
{% endhighlight %}

Which gets a redirect to the browser with the application's URI and the authorization code, which is passed on the query string.

{% highlight http %}
HTTP/1.1 302 Found
Location: REDIRECT_URI/?code=AUTHORIZATION_CODE
{% endhighlight %}

Back into the application, it can issue a request with the authorization code to get the access token. Parameters grant type (= authorization_code), the actual authorization code, client id, redirect uri and resource are passed into the request body:

{% highlight http %}
POST https://login.windows.net/c9c8d837-ca0b-4ebd-88a5-ff578008c93d/oauth2/token HTTP/1.1

resource=ORG_NAME.crm.dynamics.com&client_id=CLIENT_ID&grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=REDIRECT_URI
{% endhighlight %}

The response will be a JSON object containing the access token. Also, refresh token and token duration are passed so that the application can handle the token lifecycle:

{% highlight http %}
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"token_type":"Bearer","expires_in":"3600","expires_on":"1422482879","not_before":"1422478979","resource":"https://ORG_NAME.crm.dynamics.com","access_token":"ACCESS_TOKEN","refresh_token":"REFRESH_TOKEN","scope":"user_impersonation","id_token":"ID_TOKEN","pwd_exp":"6174798","pwd_url":"https://portal.microsoftonline.com/ChangePassword.aspx"}
{% endhighlight %}

Every subsequent request to the service needs to include the access token:

{% highlight http %}
Authorization: Bearer ACCESS_TOKEN
{% endhighlight %}

The token can also be refresh with the refresh token.