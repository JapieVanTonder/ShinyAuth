# ShinyAuth

`ShinyAuth` provides a way to protect a [Shiny](https://shiny.rstudio.com/) application behind oAuth and OpenID Connect service providers.

This package is not relying on the Pro version of Shiny. It works with the Open Source edition, and it will give you a user object in the server session.

## Supported Service Providers

### KeyCloak

[Keycloak](https://www.keycloak.org/) is an open source Identity and Access Management solution aimed at modern applications and services.
It is a solution you can host yorself.
Keycloak supports identity brokering and social login through common platforms such as, but not limited to *Google*, *Facebook*, *Twitter* and *Github*.
You can also make use of built in support for user federation to other identity providers such as *LDAP*, *Active Directory* or a *relational database*.

### Auth0

[Auth0](https://auth0.com/) is a hosted solution to add authentication to your applications.
Auth0 also supports identity brokering and social login through common platforms such as, but not limited to *Google*, *Facebook*, *Twitter* and *Github*.
You can also make use of built in support for user federation to other identity providers such as *LDAP*, *Active Directory* or a *relational database*.

* [List of supported social identity providers](https://auth0.com/docs/connections/identity-providers-social)
* [List of enterprise identity providers](https://auth0.com/docs/connections/identity-providers-enterprise)

## Install

```r
# From github
remotes::install_github('capiaas/ShinyAuth')
```

## Getting started

You either need an [account at Auth0](https://auth0.com/) or have a [working setup of Keycloak](https://www.keycloak.org/getting-started).

You shouldn't keep secrets in code, and handling of secrets / configuration is outside of scope for this package.
To properly handle secrets, read up on [Managing secrets by Hadley Wickham](https://cran.r-project.org/web/packages/httr/vignettes/secrets.html).

For this minimalistic example - environment variables are used.


```r
# app.R
library(shiny)
library(ShinyAuth)

options(shiny.port = 3000)

auth <- ShinyAuth_Auth0$new(
  client_id = Sys.getenv('AUTH0_CLIENT_ID'),
  client_secret = Sys.getenv('AUTH0_CLIENT_SECRET'),
  auth_domain = Sys.getenv('AUTH0_AUTH_DOMAIN'),
  app_url = 'http://127.0.0.1:3000/'
)

ui <- fluidPage(
  auth$scripts(),
  tableOutput('userinfo')
)

server <- function(input, output, session) {
  output$userinfo <- renderTable({session$user})
}

auth$app(ui, server)
```

Your shiny server function, which receives a *session* attribute, now contains a *user* object `session$user`.
This Minimalistic app, which only dumps the user properties onto a table - results in something  looking like this:

sub | given_name | family_name | nickname | name | picture | locale | updated_at | email | email_verified
--- | ---------- | ----------- | -------- | ---- | ------- | ------ | ---------- | ----- | --------------
unique_id | Stian | Berger | stiberger | Stian Berger | https://example.com/photo.jpg | en | 2020-06-24T10:45:57.028Z | stian@example.com | TRUE

If you have split the code in more files, it goes like this.

```r
#global.R
library(shiny)
library(ShinyAuth)

auth <- ShinyAuth_Auth0$new(
  client_id = Sys.getenv('AUTH0_CLIENT_ID'),
  client_secret = Sys.getenv('AUTH0_CLIENT_SECRET'),
  auth_domain = Sys.getenv('AUTH0_AUTH_DOMAIN'),
  app_url = 'http://127.0.0.1:3000/'
)
```

```r
# ui.R
ui <- fluidPage(
  auth$scripts(),
  tableOutput('userinfo')
)
# UI as function also works, and required for ?enableBookmarking in Shiny.
# ui <- function(req) {} 
auth$ui(ui)
```

```r
#server.R
server <- function(input, output, session) {
  output$userinfo <- renderTable({session$user})
}
auth$server(server)
```

## Mode of Operation

* User enters application
* No authentication code is detected in URL
* A minimalistic shiny UI is loaded, with only needed JS to redirect and store oAuth state parameter in session storage.
* A session state is set
* User is redirected to auth/login entrypoint of authenticating service provider
* User logs in or is already logged in.
* User is redirected back to our application url
* An authentication code is detected in URL
* The provided UI is served
* State and code is verified
* Access token and userinfo is fetched from service provider.
* Userinfo is injected into Shiny server session
* Provided server function is launched
* Application is launched and ready

## Things that go wrong

### Shiny random ports

When testing locally - Shiny launches on random ports. Service providers are strict about redirect urls back to application. It makes life easier to force Shiny to launch on a specific port.

```r
options(shiny.port = 3838)
# or
shiny::runApp('/app-path/', port = 3838)
```

### User roles

User roles can be a bit finicky to get passed along in the userinfo entrypoint of the different service providers.

* [A guide for Auth0](https://auth0.com/docs/authorization/concepts/sample-use-cases-rules#add-user-roles-to-tokens)
* [Docs for Keycloak](https://www.keycloak.org/docs/latest/server_admin/#_role_scope_mappings)

## Limitations

The entire App is authenticated. A user will be directed to identity provider immideatly after page entry.
No partial App access is possible at the moment.

Basic Role Based Authorization is possible through forwarding roles from service provider.
Other Authorization methods throug oAuth2 authorization channels are not built in.

Since we can't do full check of valid user on UI side of shiny, we can leak the empty application UI. The server will not launch, so no data should leak.
If full isolation is needed, you would need to render the UI from server.

## Disclaimer

This package is not affiliated with any of the service providers it supports.
