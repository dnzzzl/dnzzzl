+++
date = '2026-03-06T08:18:41-04:00'
draft = true
title = 'Azure_identity_integration'
+++

My task today involves setting up Azure Identity integration for our application. This will allow us to leverage Azure's authentication and authorization infrastructure, ensuring that our app is secure and compliant with organizational policies that are already in place, ergo, google corporate emails.

Here are the moving pieces:

- Azure App Service Web application with authentication settings enabled and Google provider configured.
- .NET Core MVC application with Microsoft.Identity.Web, `ApplicationUser`s and Roles Based Access Control already set up independently.

Some requirements unique to our implementation:

- Users do not register in the platform, instead the users are preregistered by the admin and matched on login, once we have the logged in user's claims, we should correlate their email with our database of users. This ensures continuity with our current domain.
- Users log in with their corporate Google accounts. Our application should reject any log in with an account not belonging to our configured domain.
- Let's set up a mixed authentication within our Azure Web App to allow for a landing page with a 'log in with google button'.

So far this is my understanding of the auth mechanism in Azure:

It is known as Easy Auth and consists of a gateway set at the Azure infrastructure level before the request hits our Application, in turn our application only receives authenticated requests.

This is unless we configure mixed authentication where we have a unauthenticated landing page, which we will be taking advantage of to show a nice informational static page, along with login options.

The mechanism consists on intercepting requests at the gateway level and validating the user's authentication status before allowing access to the application. After access is allowed, the gateway attaches some headers along with the request which we should be able to access via the HttpContext in our application controller code. These headers contain claims about the user that will be used to correlate and integrate with our identity set up from database.

At this point, integration will involve mapping the claims from the headers to our application's user model, ensuring that we have a seamless experience for the user while maintaining security and compliance with our organization's policies by rejecting any unauthorized access including non-domain emails.

After planning we land on this set of requirements:

1. Deploy to Azure App Service with Authentication enabled, Google provider configured, allow unauthenticated requests.
2. Visit / → landing page shown without auth.
3. Click Google button → redirected to Google OAuth → returns to /.auth/login/google → Easy Auth validates → redirects to /Account/PostLogin.
4. Middleware runs, reads header, finds user → Identity cookie set → role routing redirect fires.
5. Visit any protected route directly without auth → redirected to / (landing).
6. Login with non-domain Google account → Access Denied page.
7. Login with valid domain but unregistered email → NotRegistered page.
8. Logout → both Identity cookie and Easy Auth session cleared → back to landing page.
9. Locally with AzureEasyAuth:Enabled: false → dev login path works normally.

Soon enough we had our middleware class code written, it accomplished this set of steps:
- Reads X-MS-CLIENT-PRINCIPAL-NAME header for email.
- Skips if: user is already signed in (has valid Identity cookie), or path is /, /Account/NotRegistered, /Account/AccessDenied, /Account/Logout.
- Validates @domain suffix against AllowedEmailDomain.
- Calls UserManager<ApplicationUser>.FindByEmailAsync(email).
- If found and Active == true: calls SignInManager.SignInAsync(user, isPersistent: false).
- If found but inactive (locked out): redirect to /Account/AccessDenied.
- If not found in DB: redirect to /Account/NotRegistered.
- If domain mismatch: redirect to /Account/AccessDenied.
