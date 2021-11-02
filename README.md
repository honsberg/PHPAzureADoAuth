# PHP: Implement Azure AD login to your site

Katy Nicholson

https://katystech.blog/

PHP Class and associated code for Azure AD/Office 365 oAuth login to a demo web site.



A while ago I wrote some code to enable one of my PHP projects to log in via authentication with ADFS. I’ve recently updated this to talk directly with Azure AD, and have split this off into a separate project which I’ll share here.

Basically this works using oAuth2, browser sessions, a database and a couple of scripts, and on the Azure AD side you need to create an App Registration. Within this sample project the following flow happens:

*    User lands on index.php. If they do not have a session key cookie, one is generated, and this is stored in the database along with the page the user was attempting to access. They are redirected to login.microsoftonline.com to authenticate.
*    If you allowed authentication from any tenant, and used the common endpoint (rather than your specific tenant ID), the user may be asked to allow your app to access their account. If they are on their home tenancy, you will have already approved this for all users.
*    The user is redirected to the oauth.php file, where a background request is made back to login.microsoftonline.com to obtain a token. Once this has been successful, the user is redirected back to their original destination.
*    If the user lands on index.php and their session key cookie already exists, and exists in the database, and has not expired, they will be allocated that token’s data.
*    If the user lands on index.php with a session key cookie, but it is going to expire in the next 10 minutes, we will perform a refresh request in the background.
*    If the user lands on index.php with a session key cookie, but it’s expired, they are redirected back to login.microsoftonline.com – which may automatically log them back in, or may prompt, depending on their settings.

The code is available on my GitHub repository. The various files within are:

*    database.sql – the table used by this project. Create a database first, and note down the details (host, username, password, database).
*    inc/config.inc – configuration file, put your database details in here, along with your tenant ID, client ID and client secret from the Azure AD App Registration. There’s also an entry for URL which is the URL of your site, included rather than automatically determined incase your site is accessible by multiple URLs as this needs to match what you’ve entered into the App Registration. I’ve defined constants for all these settings, there is probably a better way to do this but they work.
*    inc/mysql.php – class used for accessing the database, nothing needs editing in here.
*    inc/auth.php – main authentication class – again nothing to edit in here.
*    www/index.php – sample page which will request login and then display your username and a logout link, along with a print of all the data retrieved during logon.
*    www/oauth.php – callback script, Azure AD login page returns you back to this script.

When putting this on a web server, the web root should be www and the inc directory should not be accessible from the web browser.

Note: You will need to have the PHP CURL library installed on the server for this to work.

To create the App Registration, go to Azure AD > App registrations https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps and click New registration. Fill out your site display name, select the account types, then enter the Redirect URI which will be https://your.domain.name/oauth.php. Make sure that you put https://your.domain.name (with no trailing slash) into \_URL within config.inc. Click Register once done.
![grafik](https://user-images.githubusercontent.com/12712721/139848514-5ec8e568-79fb-4b5b-9a44-f495daab700d.png)


A quick note on supported account types – if you use “common” as your tenant ID within config.inc, any account on any Azure AD tenant, plus personal (outlook.com etc) accounts can log in. If you use your actual tenant ID, then only accounts which are on your tenant (either as normal accounts, or as guests from other tenancies) can log in, but personal Microsoft accounts will work if they are guests on your tenancy, assuming you’ve set the supported account type in the App Registration accordingly.

Once you’ve finished creating the app, you should see the Overview screen. Copy the client ID and tenant ID, pasting them into _OAUTH_SERVER and _OAUTH_CLIENTID in config.inc. The _OAUTH_SERVER entry should be the login.microsoftonline.com URL but with TENANT_ID replaced with your directory (tenant) ID.

![grafik](https://user-images.githubusercontent.com/12712721/139848722-53424bf5-d0c6-4050-b8ff-ee21425959c9.png)


Now on the Certificates & secrets page, add a new secret and select the appropriate time. Don’t forget you will need to update this before it expires, so make a note in your calendar. Once done, copy the secret value and paste this into _OAUTH_SECRET within config.inc.

Hopefully you should now be able to browse to your application and be prompted to log in. On your first go, you’ll be asked to allow permissions for everyone on your tenant (assuming you have the appropriate admin rights).

![grafik](https://user-images.githubusercontent.com/12712721/139848813-832be390-a115-4f0f-89ad-380922cc619c.png)


## Update (1st Sept 2021)

I’ve updated this to show some basic use of the Graph API. The default index.php page will now pull the logged on user’s profile photo and basic profile data. You’ll need to configure the _OAUTH_SCOPE constant in config.inc and make sure this includes “user.read” for this bit to work. It’s not required but if you want to read the logged on user’s directory entry, you can do.

There’s more detailed use of Graph API in my other PHP post, https://katystech.blog/2020/08/php-graph-mailer, which demonstrates reading a mailbox and sending mail through Office 365. You’ll need to alter the scope in config.inc to add the appropriate permissions in if doing this in your own application, and query using the logged on user’s access token (show in this project) rather than the application’s own token (show in the Graph Mailer project).


## Update (15th & 16th Oct 2021) – Restricting Access 

You can further restrict access to your application in Azure AD. Go to the Azure AD > Enterprise Applications https://portal.azure.com/#blade/Microsoft_AAD_IAM/StartboardApplicationsMenuBlade/AllApps/menuId/ page, find your app and then go to the Properties page. Toggle “Assignment required” to “Yes”. Now go to the Users and groups page, and add the users/groups you wish to have access.

You can further extend this by adding different roles to the application – in Azure AD > App Registrations, find your app and go to the App roles page. Create an app role for each role you want to assign, e.g. Admin and User:
Screenshot of Create app role
You can create many app roles to help control access within your application, in this sample project I’ve gone for Role.Admin and Role.User.

The display name is shown when you assign users in Azure AD, and the Value is what is shown in the PHP code on your application. When assigning users in the Enterprise Application screen you will be asked to select their role.

In the index.php sample, you will notice $Auth->userRoles referred to – this will be an array of user roles. You can use the checkUserRole function in auth.php to verify if a user holds the role required for a specific part of your application, i.e. if ($Auth->checkUserRole(‘Role.Admin’)) { //admin only code here }

## Further Reading:

Microsoft identity platform and OAuth 2.0 authorization code flow – Microsoft identity platform | Microsoft Docs https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow

rfc7636 (ietf.org) https://datatracker.ietf.org/doc/html/rfc7636

Graph Explorer – Microsoft Graph https://developer.microsoft.com/en-us/graph/graph-explorer

