The goal is to add OpenID Connect authentication flow to Pachyderm. This proposal aims to specify how we will implement that.

# Setting up OIDC
Unfortunately, the user set up required to enable OIDC will be a bit complex. That's because the OIDC spec requires a `redirect_uri` to send the Authorization Code to. We can't host a dedicated URL for this because that doesn't give our users an appropriate level of security. 

So instead, we'll create a server that runs in `pachd` which will receive the redirect to take the Authorization Code. For this to work, the user will need to create an address (local etc/hosts is fine) to point to it, and then will need to create a Pachyderm app for their IdP with this URL. Then they'll need to add this app to their IdP admin account. 

Once this has all been done, they will be able to login with OIDC using the flow described next.

# Flow

We'll use the Authorization Code flow option: https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth The steps of this flow are copied here as the numbered lines.

When a user wants to login (i.e. uses `pachctl auth login`), and their account is configured for OIDC, we will print their Identity Providers URL. That's because the IdP will need them to login in and sign a consent form in the browser.

    1. Client prepares an Authentication Request containing the desired request parameters.
    2. Client sends the request to the Authorization Server.
The user will then go to the URL to the IdP, and enter their login information there.
   
    3. Authorization Server Authenticates the End-User.
    4. Authorization Server obtains End-User Consent/Authorization.
The IdP will authenticate the user, and request their consent to use OIDC for Pachyderm.

    5. Authorization Server sends the End-User back to the Client with an Authorization Code.
This is where things get a little tricky. According to the OIDC spec: 
> When using the Authorization Code Flow, the Authorization Response MUST return the parameters defined in Section 4.1.2 of OAuth 2.0 [RFC6749] by adding them as query parameters to the redirect_uri specified in the Authorization Request using the application/x-www-form-urlencoded format, unless a different Response Mode was specified. 

This is where the URL we required the user to set up comes in: it's what we send in for the IdP to use as the `redirect_uri` here. So the IdP sends the Authorization Code to the user's instance of `pachd`.

    6. Client requests a response using the Authorization Code at the Token Endpoint.
    7. Client receives a response that contains an ID Token and Access Token in the response body.
 The user's Pachyderm then turns around and makes a request to the IdP's token endpoint using this Authorization Code. Assuming the Authorization Code is authenticated, the response will include the Access Token. 

    8. Client validates the ID token and retrieves the End-User's Subject Identifier.

Pachyderm will then ask the IdP for the username associated with the Access Token, in order to authenticate it. Assuming it is authentic, we will then create a Pachyderm Token, which we will write to the user's config file at `~/.pachyderm/config.json`.


# Testing
For integration tests, I'm planning to copy the testing currently done for SAML. I'll also test the entire flow with Okta, Keycloak, and Google OIDC. 
