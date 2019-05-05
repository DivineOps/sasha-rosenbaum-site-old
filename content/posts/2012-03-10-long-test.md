---
title: "Prompt for Sign-In on 401 – Unauthorized Errors in MVC App with Azure Active Directory Using WS Federation"
date: 2012-03-10T00:00:00-05:00
draft: true
---

Following my post on Partial Authentication with AAD I am going to elaborate on how to prompt the user to sign into the application instead of throwing a 401 – unauthorized error.

First step is to add HTTP error 401 redirect to IIS configuration in Web.config

{{<highlight xml>}}
<system.webServer>
    <httpErrors existingResponse="Replace" 
			defaultResponseMode="Redirect" errorMode="Custom">
      <remove statusCode="401"/>
      <error statusCode="401" responseMode="ExecuteURL" path="/Account/SignInRedirect"/>
    </httpErrors>
	...
<system.webServer>
{{</highlight>}}

Note that the responseMode is ExecuteURL (and not Redirect). This allows the requested URL to be passed through to the SignInRedirect method.

Now all we need is to add proper redirect conditions to our /Account/SignInRedirect method

<!-- readmore -->


{{<highlight cs>}}
public ActionResult SignInRedirect()
{
	if (Request.IsAuthenticated)
	{
		// The user is signed in 
		// but does not have permissions to access a specific URL
		return RedirectToAction("InsufficientPermissions", "Account");
	}

	// Redirect to home page after signing in.	
	WsFederationConfiguration config = 
		FederatedAuthentication.FederationConfiguration.WsFederationConfiguration;
		
	string callbackUrl = Url.Action("Index", "Home", 
			routeValues: null, protocol: Request.Url.Scheme);
	
	//The URL the user originally requested is passed in Request.RawUrl
	if (Request.RawUrl != null)
	{
		callbackUrl = Request.RawUrl;
	}

	SignInRequestMessage signInRequest = 
		FederatedAuthentication.WSFederationAuthenticationModule
		.CreateSignInRequest(
			uniqueId: String.Empty,
			returnUrl: callbackUrl,
			rememberMeSet: false);

	signInRequest.SetParameter("wtrealm", IdentityConfig.Realm ?? config.Realm);
	return new RedirectResult(signInRequest.RequestUrl.ToString());
}

{{</highlight>}}

There are 2 main use cases this method aims to cover:

### A non-authenticated user gets tries to access a URL that requires authorization ###
When a non-authenticated user tries to access a URL that is behind an [Authorize] attribute she will get a 401 – Unauthorized error. This error will be redirected by IIS to the path specified in httpErrors configuration, in our case /Account/SignInRedirect. The original URL will be passed on as well in Request.RawUrl. The sign in method will then attempt to sign the user in, and if the sign in is successful redirect her to the originally requested URL.

### An authenticated user tries to access a URL he does not have access to ###
In this case the first if clause of the sign in method will be activated, and the user will be redirected to a custom error page. You can inform the user that she does not have permissions to view the page, or craft any other custom error message. You could also choose to redirect the user back to home page. The one thing you should _not_ do is redirecting the user back to the Request.RawUrl, as this will result in an infinite redirect loop.
Another possible behavior in this case would be to prompt the user to sign in with an account that does have access to the requested URL. This behavior is, however, not typical for AD scenarios, where it is unlikely that a single user has access to multiple AD accounts.

Why am I separating the /Account/SignInRedirect from the regular /Account/SignIn method?
The reason is the case when the user tries to sign in via /Account/SignIn, and her credentials are already stored by the browser. In this case the user should be redirected to home, rather than hitting the insufficient permissions error page. Please view the original post for the /Account/SignIn method implementation.
