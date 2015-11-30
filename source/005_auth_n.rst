.. _authn:

*****************************************
5. Authenticating Accounts with Stormpath
*****************************************

Authentication is the process by which a system identifies that someone is who they say they are. Perhaps the most accessible example of this process is at the airport, where you must present your passport and your plane ticket. The passport is used to authenticate you, that you are who you present yourself to be, and the plane ticket represents your authorization to board a specific flight. 

In this chapter we will cover three of the ways that Stormpath allows you to authenticate users: :ref:`password authentication <password-authn>`, :ref:`token authentication <token-authn>`, and :ref:`social authentication <social-authn>`.

.. _password-authn:

a. How Password Authentication Works in Stormpath
=================================================

Probably the single most common way of authenticating a user is to ask them for their account credentials. When a user creates an account in Stormpath, it is required that they provide a username (or email) and a password. Those credentials can then be provided in order to authenticate an account.

Authenticating An Account
-------------------------

After an Account resource has been created, you can authenticate it given an input of a ``username`` or ``email`` and a ``password`` from the end-user. When authentication occurs, you are authenticating an Account within a specific Application against that Application’s Organizations, Directories and Groups (more on that :ref:`below <how-login-works>`). The key point is that the :ref:`Application resource <ref-application>` is the starting point for authentication attempts.

Once you have the Application resource you may attempt authentication by sending a POST request to the Application’s ``/loginAttempts`` endpoint and providing a base64 encoded ``username``/``email`` and ``password`` pair that is separated with a colon (for example ``testuser``:``testpassword``). Stormpath requires that the ``username``/``email`` and ``password`` are base64 encoded so that these values are not passed as clear text. For more information about the ``/loginAttempts`` endpoint please see the :ref:`Reference Chapter <ref-loginattempts>`.
     
So, if we had a user Account "Han Solo" in the "Captains" Directory, and we wanted to log him in, we would first need to take the combination of his ``username`` and ``password`` ("first2shoot:Change+me1") and then Base64 encode them: ``Zmlyc3Qyc2hvb3Q6Q2hhbmdlK21lMQ==``.

We would issue the following POST to our Application with ID ``1gk4Dxzi6o4PbdleXaMPLE``:

.. code-block:: http

  POST /v1/applications/1gk4Dxzi6o4PbdleXaMPLE/loginAttempts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
    "type": "basic",
    "value": "Zmlyc3Qyc2hvb3Q6Q2hhbmdlK21lMQ==",
    "accountStore": {
           "href": "https://api.stormpath.com/v1/groups/2SKhstu8Plaekcai8lghrp"
     }
  }

We are using the Base64 encoded ``value`` from above, and specifying that the Account can be found in the "Captains" Directory from :ref:`earlier <about-cloud-dir>`.

On success we would get back the ``href`` for the "Han Solo" Account:

.. code-block:: http  

  HTTP/1.1 200 OK
  Location: https://api.stormpath.com/v1/accounts/72EaYgOaq8lwTFHILydAid
  Content-Type: application/json;charset=UTF-8

  {
    "account": {
      "href": "https://api.stormpath.com/v1/accounts/72EaYgOaq8lwTFHILydAid"
    }
  }

The reason this succeeds is because there is an existing **Account Store Mapping** between the "Han Solo" Account's "Captains" Directory and our Application. This mapping is what allows this Account to log in to the Application.

.. note::

  Instead of just receiving an Account's ``href`` after successful authentication, it is possible to receive the full Account resource in the JSON response body. To do this, simply add the **expand=account** parameter to the end of your authentication query:

    ``https://api.stormpath.com/v1/applications/$YOUR_APPLICATION_ID/loginAttempts?expand=account``

  For more information about link expansion, please see :ref:`the Reference chapter <about-links>`.

.. _how-login-works:

How Login Attempts Work in Stormpath 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When the "Han Solo" Account tries to log in to the Application, the user submits a request to the Application’s ``/loginAttempts`` endpoint. Stormpath then consults the Application’s assigned **Account Stores** (Organizations, Directories, and Groups) in the order that they are assigned to the Application. When a matching Account is discovered in a mapped Account Store, it is used to verify the authentication attempt and all subsequent Account Stores are ignored. In other words, Accounts are matched for Application login based on a "first match wins" policy.

Let's look at an example to illustrate this behavior. Assume that two Account Stores, a "Customers" Directory and an "Employees" Directory, have been assigned (mapped) to a "Foo" application. "Customers" was assigned first, and "Employees" was assigned next, and this will dictate the order in which they are checked. 

The following flow chart shows what happens when an Account attempts to log in to the Foo application:

.. figure:: images/auth_n/LoginAttemptFlow.png
    :align: center
    :scale: 100%
    :alt: Login Attempt Flow 

    *The Login Attempt Flow* 

As you can see, Stormpath tries to find the Account in the "Customers" Directory first because it has a higher priority than the "Employees" directory. If not found, the "Employees" Directory is tried next as it has a lower priority.

You can map multiple Account Stores to an Application, but only one is required to enable login for an Application. Mapping multiple Account Stores to an Application, as well as configuring their priority, allows you precise control over the Account populations that may log in to your Application.

.. _non-cloud-login:

How Login Works with Mirrored and Social Directories 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For both :ref:`Mirrored <about-mirror-dir>` and :ref:`Social<about-social-dir>` Directories, Stormpath has a default behavior that links the Mirror/Social Directories with corresponding "master" Directories. 

The default Stormpath behavior is as a follows: a new user visits your site, and chooses something to log-in with their LDAP credentials, or social login like "Sign-in with Google". Once they log in using their social or LDAP credentials, if it doesn't already exist, a new user Account is created in your Mirror/Social Directory. After this Account is created, a search is performed inside the Application's master Directory for their email address, to see if they already exist in there. If the user Account is already in the master Directory, no action is taken. If the user Account is not found, a new one is created in the master Directory, and populated with the information pulled from the Google account. The customData resource for that Account is then used to store an ``href`` link to their Account in the Mirror/Social Directory. 

.. code-block:: json 

  {
    "customData": {
      "accountLink": "https://api.stormpath.com/v1/accounts/3fLduLKlQu"
    }
  }

If the user then chooses at some point to "Sign in with Facebook", then a similar process will occur, but this time with a link created to the user Account in the Facebook Directory. 

.. code-block:: json 

  {
    "customData": {
      "accountLinks": {
          "Link1": "https://api.stormpath.com/v1/accounts/3fLduLKlQu",
          "Link2": "https://api.stormpath.com/v1/accounts/X3rjfa4Ljd", 
          "Link3": "https://api.stormpath.com/v1/accounts/a05Ghpjd30"
      }
  }

This approach has two major benefits: It allows for a user to have one unified identity in your Application, regardless of how many social or LDAP identities they choose to log in with; this central identity can also be the central point that all authorization permissions (whether they be implicit or explicit) are then applied to.

.. _managing-login:

Manage Who Can Log Into Your Application 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As is hopefully evident by now, controlling which Accounts can log in to your Application is largely a matter of manipulating the Application's Account Store Mappings. For more detailed information about this resource, please see the :ref:`ref-account-store-mapping` section of the Reference chapter.

The reason why our user "Han Solo" was able to log in to our application is because the Application resource that represents our Application: ``https://api.stormpath.com/v1/applications/1gk4Dxzi6o4PbdleXaMPLE``, and our "Captains" Directory: ``https://api.stormpath.com/v1/directories/2SKhstu8Plaekcai8lghrp`` are mapped to one another by an **Account Store Mapping**. 

We can find this Mapping by sending a ``GET`` to our Application's ``/accountStoreMappings`` endpoint, which would yield the following response:

.. code-block:: http 

  HTTP/1.1 200 OK
  Content-Type: application/json;charset=UTF-8

  {
    "href":"https://api.stormpath.com/v1/applications/1gk4Dxzi6o4PbdleXaMPLE/accountStoreMappings",
    "offset":0,
    "limit":25,
    "size":1,
    "items":[
      {
        "href":"https://api.stormpath.com/v1/accountStoreMappings/5WKhSDXNR8Wiksjv808XHp",
        "listIndex":1,
        "isDefaultAccountStore":true,
        "isDefaultGroupStore":true,
        "application":{
          "href":"https://api.stormpath.com/v1/applications/1gk4Dxzi6o4PbdleXaMPLE"
        },
        "accountStore":{
          "href":"https://api.stormpath.com/v1/directories/2SKhstu8Plaekcai8lghrp"
        }
      }
    ]
  }

.. note::

  Any new Accounts and Groups added to this Application via it's `/accounts` and `/groups` endpoints will be added to this Directory by default, since ``isDefaultAccountStore`` and ``isDefaultGroupStore`` are both set to ``true``. 

.. _create-asm:

Mapping a new Account Store
"""""""""""""""""""""""""""

We would now like to map a new Account Store that will have the following characteristics:

#. It will have the highest login priority. This means that it will be consulted first during :ref:`the login process <how-login-works>`, before any other Account Stores.
#. It will be the default Account Store for any new Accounts.
#. It will be the default Group Store for any new Groups. 

To accomplish this, we will send a ``POST``:

.. code-block:: http 

  POST v1/accountStoreMappings HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
    "listIndex": 0,
    "isDefaultAccountStore": true,
    "isDefaultGroupStore": true,
    "application": {
      "href": "https://api.stormpath.com/v1/applications/1gk4Dxzi6o4PbdleXaMPLE"
    },
    "accountStore": {
      "href": "https://api.stormpath.com/v1/directories/2jw4Kslj97zYjYRXEh2KYf"
    }
  }

We are mapping the Application (id: ``1gk4Dxzi6o4PbdleXaMPLE``) to a new Directory (id: ``2jw4Kslj97zYjYRXEh2KYf``). Additionally, we are setting 

#. the login priority to the highest priority, by sending a ``listIndex`` of ``0``.
#. ``isDefaultAccountStore`` to ``true`` and 
#. ``isDefaultGroupStore`` to ``true`` as well.

So by sending a ``POST`` with these contents, we are able to create a new Account Store Mapping that supersedes the old one.

Updating an Existing Account Store
""""""""""""""""""""""""""""""""""

Updating an existing Account Store simply involves sending a ``POST`` to the ``v1/accountStoreMappings/$ACCOUNT_STORE_MAPPING_ID`` endpoint with the attributes that you would like to update. 

**Changing Login Priority**

For example, if you want to update an existing Account Store to now have highest login priority, simple send a ``POST`` with "listIndex": 0 in the body, and the accountStoreMapping resource will be updated. Additionally, all of the other Account Stores will have their ``listIndex`` incremented up by 1. 

**Changing the Default Account or Group Store**

Sending ``"isDefaultAccountStore": true`` and/or ``"isDefaultAccountStore": true`` in the JSON body to a ``v1/accountStoreMappings/$ACCOUNT_STORE_MAPPING_ID`` endpoint would result in those values being updated on the target resource, and whichever resource had those values as ``true`` would have them changed to ``false``.

.. note::

  Setting an AccountStoreMapping’s ``isDefaultGroupStore`` or ``isDefaultAccountStore`` to ``false`` will **not** automatically set another AccountStoreMapping’s ``isDefaultGroupStore`` or ``isDefaultAccountStore`` to ``true``. You are responsible for setting this yourself if you would like your Application to create new Accounts/Groups.

.. _token-authn:

b. How Token-Based Authentication Works
=======================================

In this section, we will discuss how to use Stormpath to use Stormpath to generate and manage OAuth 2.0 Access Token.

Introduction to Token-Based Authentication
------------------------------------------

Since HTTP is considered a stateless protocol, if your application authenticates a user for one HTTP request, a problem arises when the next request is sent and your application doesn't know who the user is. This is why many applications today pass some information to tie the request to a user. Traditionally, this requires **Server-based authentication**, where state is stored on the server and only a session identifier is stored on the client.

**Token-based authentication** is a alternate, stateless strategy. With token-based authentication, you secure an application based on a security token that is generated for the user on authentication and then stored on the client-side. Token-based Authentication is all about removing the need to store information on the server while giving extra security to keep the token secure on the client. This helps you as a developer build stateless and scalable applications.

Stormpath's approach to token-based authentication has two elements: JSON Web Tokens (JWTs) for authentication, and OAuth 2.0 for authorization. 

Why OAuth 2.0?
^^^^^^^^^^^^^^

OAuth 2.0 is an authorization framework and provides a protocol to interact with a service that can delegate authentication or provide authorization. Its primary advantage as a standard is its wide adoption rate across many mobile and web applications today. If you have ever logged-in to a website using Facebook or Google, you have used one of OAuth 2.0's many authorization flows. You can read more about the different OAuth 2.0 authorization flows or grant types in depth on `Stormpath’s blog <https://stormpath.com/blog/what-the-heck-is-oauth/>`_.

Even though OAuth 2.0 has many authorization modes or "grant types", Stormpath currently supports three of them:

- **Password Grant Type**: Provides the ability to get an Access Token based on a login and password.

- **Refresh Grant Type**: Provides the ability to generate another Access Token based on a special Refresh Token.

- **Client Credentials Grant Type**: Provides the ability to exchange an API Key for the Access Token. This is supported through the API Key Management feature.

To understand how to use Token-based Authentication, we need to talk about the different types of tokens that are available.

What Tokens Are Available for Token-Based Authentication?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For Token Based Authentication, there are a two different types of tokens that need to be managed. These are:

- Access Token
- Refresh Token

The **Access Token** is what grants access to a protected resource. The Access Token that Stormpath generates for Accounts on authentication is a **JSON Web Token**, or JWT. The JWT has security built-in to make sure that the Access Token is not tampered with on the client, and is only valid for a specified duration. 

The **Refresh Token** is a special token that is used to generate additional Access Tokens. This allows you to have an short-lived Access Token without having to collect credentials every single time you need a new Access Token.

When using OAuth 2.0, the Access Token and Refresh Token are returned in the same response during the token exchange, this is called an **Access Token Response**.

.. _token-authn-config:

Using Stormpath for Token-Based Authentication
---------------------------------------------------

Stormpath can be used to generate, manage, check, and revoke both Access and Refresh Tokens. Before diving in, let's talk about configuration.

Configuring Token-Based Authentication
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Stormpath is configurable so you can set the time to live (TTL) for both the Access and Refresh tokens. This is important for many applications because it gives the ability to define how the tokens expire. For example, you could decide that your application requires a user to log in daily, but the access should only live for 10 minutes. Or, you could decide that for your application, users should be able to stay logged-in for two months and the access token expires in an hour.

Each Application resource in Stormpath has an associated :ref:`OAuth Policy resource <ref-oauth-policy>` where the TTLs for a particular Application's tokens are stored inside properties called ``accessTokenTtl`` and ``refreshTokenTtl``:

.. code-block:: json 

  {
      "href": "https://api.stormpath.com/v1/oAuthPolicies/1gk4Dxzi6o4PbdleXaMPLE",
      "accessTokenTtl": "PT1H",
      "refreshTokenTtl": "P60D",
      "comment":" // This JSON has been truncated for readability"
  }

The values for both properties are stored as `ISO 8601 Durations <https://en.wikipedia.org/wiki/ISO_8601#Durations>`_. By **default**, the TTL ``duration`` for the Access Token is 1 hour and the Refresh Token's is 60 days, while the **maximum** ``duration`` is 180 days.

If we wanted to change the TTL for the Access Token to 30 minutes and the Refresh Token to 7 days, we could simply make a POST request to the ``/oAuthPolicies/$APPLICATION_ID`` endpoint with the following payload:

.. code-block:: http

  POST /v1/oAuthPolicies/1gk4Dxzi6o4PbdleXaMPLE HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
    "accessTokenTtl": "PT30M",
    "refreshTokenTtl": "P7D"
  }

And we would get the following response:

.. code-block:: HTTP 

  HTTP/1.1 200 OK
  Location: https://api.stormpath.com/v1/oAuthPolicies/1gk4Dxzi6o4PbdleXaMPLE
  Content-Type: application/json;charset=UTF-8

  {
    "href": "https://api.stormpath.com/v1/oAuthPolicies/1gk4Dxzi6o4PbdleXaMPLE",
    "accessTokenTtl": "PT30M",
    "refreshTokenTtl": "P7D",
    "comment":" // This JSON has been truncated for readability"
  }

.. note::

    Refresh Tokens are optional. If you would like to disable the Refresh Token from being generated, set a ``duration`` value of 0 (e.g. PT0M).

Generating an OAuth 2.0 Access Token
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Stormpath can generate Access Tokens using the above-mentioned OAuth 2.0 **Password Grant** flow. Stormpath exposes an endpoint for each Application resource to support the OAuth 2.0 protocol::

    https://api.stormpath.com/v1/applications/$YOUR_APPLICATION_ID/oauth/token

This endpoint is used to generate an OAuth token for any valid Account associated with the specified Application. It uses the same validation as the ``/loginAttempt`` endpoint, as described in :ref:`how-login-works`.

Your application will act as a proxy to the Stormpath API. For example:

- The user inputs their credentials (e.g. ``username`` and ``password``) into a form and submits them.
- Your application in turn takes the credentials and formulates the OAuth 2.0 Access Token request to Stormpath.
- When Stormpath returns with the Access Token Response, you can then return the Access Token and/or the Refresh Token to the client.

So you would send the following API call:

.. code-block:: http 

  POST /v1/applications/$YOUR_APPLICATION_ID/oauth/token HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/x-www-form-urlencoded

  grant_type=password&username=tom%40stormpath.com&password=Secret1


This would result in this response:

.. code-block:: http

  HTTP/1.1 200 OK
  Content-Type: application/json;charset=UTF-8

  {
    "access_token": "eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiIxdkhJMGpCWERybW12UHFBRmYyWHNWIiwiaWF0IjoxNDQxMTE4Nzk2LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxMTIwNTk2LCJydGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2In0.xlCXL7UUVnMoBKj0p0bXM_cnraWo5Io-TvUt2WBOl3k",
    "refresh_token": "eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2IiwiaWF0IjoxNDQxMTE4Nzk2LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxNzIzNTk2fQ.xUjcxTZhWx74aa6adnUXjuvUgqjC8TvvrB7cBEmNF_g",
    "token_type": "Bearer",
    "expires_in": 1800,
    "stormpath_access_token_href": "https://api.stormpath.com/v1/accessTokens/1vHI0jBXDrmmvPqAFf2XsV"
  }

This is an **OAuth 2.0 Access Token Response** and includes the following:

.. list-table:: 
    :widths: 15 10 60
    :header-rows: 1

    * - Property
      - Type
      - Description
    
    * - access_token
      - String (JSON Web Token)
      - The access token for the response.
    
    * - refresh_token
      - String (JSON Web Token)
      - The refresh token that can be used to get refreshed Access Tokens.
        
    * - token_type
      - String
      - The type of token returned.
    
    * - expires_in
      - Number
      - The time in seconds before the token expires.
    
    * - stormpath_access_token_href 
      - String
      - The href location of the token in Stormpath.

.. note::

    Just like with logging-in a user, it is possible to generate a token against a particular Application's Account Store resource. To do so, specify the Account Store's ``href`` as a parameter in the body::

        grant_type=password&username=tom@stormpath.com&password=Secret1&accountStore=https://api.stormpath.com/v1/directories/2SKhstu8Plaekcai8lghrp

Validating an Access Token
^^^^^^^^^^^^^^^^^^^^^^^^^^

Once an ``access_token`` has been generated, we have taken care of the Authentication part of our workflow. Now, the OAuth token can be used to authorize individual requests that the user makes. To do this, the client will need to pass it to your application.

For example, if you have a route ``https://yourapplication.com/secure-resource``, the client would request authorization to access the resource by passing the access token as follows:

.. code-block:: http 

    GET /secure-resource HTTP/1.1
    Host: https://yourapplication.com
    Authorization: Bearer eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiIxdkhJMGpCWERybW12UHFBRmYyWHNWIiwiaWF0IjoxNDQxMTE4Nzk2LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxMTIwNTk2LCJydGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2In0.xlCXL7UUVnMoBKj0p0bXM_cnraWo5Io-TvUt2WBOl3k

Once your application receives the request, the first thing to do is to validate the token, either using Stormpath, or using local application-side logic. The benefit of using Stormpath to validate the token through the REST API (or an SDK that is using the REST API) is that Stormpath can validate the token against the state of your Application and Account resources. To illustrate the difference:

.. list-table:: 
    :widths: 60 15 15
    :header-rows: 1

    * - Validation Criteria
      - Locally
      - Stormpath
    
    * - Token hasn't been tampered with
      - Yes
      - Yes
        
    * - Token hasn't expired
      - Yes
      - Yes
    
    * - Token hasn't been revoked
      - No
      - Yes
        
    * - Account hasn't been disabled or deleted
      - No
      - Yes
    
    * - Issuer is Stormpath
      - Yes
      - Yes
        
    * - Issuing Application is still enabled, and hasn't been deleted
      - No
      - Yes
    
    * - Account is still in an Account Store for the issuing Application
      - No
      - Yes

It is up to you to determine which kind of validation is important for your application. If you need to validate the state of the Account and/or Application resources, or if you need to use token revocation, then using Stormpath to validate the token is the obvious choice. If you only require that the token has not expired and has not been tampered with, you can validate the token locally and minimize the network requests to Stormpath.

.. _about-token-validation:

Using Stormpath to Validate Tokens
""""""""""""""""""""""""""""""""""
To see how to validate tokens with the Stormpath REST API, let's go back to the example where a user has already generated an access token. 

To recap, we have done the following: 

1. Sent a POST to ``https://api.stormpath.com/v1/applications/$YOUR_APPLICATION_ID/oauth/token`` with a body that included information about the OAuth Grant Type we wanted, as well as our user's username and password.
2. Received back an **Access Token Response**, which contained - among other things - an **Access Token** in JWT format.

The user now attempts to access a secured resource by passing the ``access_token`` JWT value from the Access Token Response in the ``Authorization`` header:

.. code-block:: http 

  GET /secure-resource HTTP/1.1
  Host: https://yourapplication.com
  Authorization: Bearer eyJraWQiOiIyWkZNVjRXV[...]

The ``Authorization`` header contains the Access Token. To validate this Token with Stormpath, you can issue an HTTP GET to your Stormpath Application’s ``/authTokens/`` endpoint with the JWT token::

    https://api.stormpath.com/v1/applications/$YOUR_APPLICATION_ID/authTokens/eyJraWQiOiIyWkZNVjRXV[...]

If the access token can be validated, Stormpath will return a 302 to the Access Token resource:

.. code-block:: http 

  HTTP/1.1 302 Location Found
  Location: https://api.stormpath.com/v1/accessTokens/6zVrviSEIf26ggXdJG097f

With the confirmation that the token is valid, you can now allow the user access to the secured resource that they requested.

Validating the Token Locally
""""""""""""""""""""""""""""

Local validation would also begin at the point of the request to a secure resource:

.. code-block:: http 

  GET /secure-resource HTTP/1.1
  Host: https://yourapplication.com
  Authorization: Bearer eyJraWQiOiIyWkZNVjRXV[...]

The token specified in the Authorization header has been digitally signed with the Stormpath API Key Secret that was used to generate the token. This means that you can use a JWT library for your specific language to validate the token locally if necessary. For more information, please see one of our `Integration Guides <https://docs.stormpath.com/home/>`_.

Refreshing Access Tokens
^^^^^^^^^^^^^^^^^^^^^^^^

In the event that the Access Token expires, the user can generate a new one using the Refresh Token without re-entering their credentials. To use this Refresh Token, simply make an HTTP POST to your Applications ``/oauth/token`` endpoint with it and you will get a new token back.

.. code-block:: http 

  POST /v1/applications/$YOUR_APPLICATION_ID/oauth/token HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/x-www-form-urlencoded

  grant_type=refresh_token&refresh_token=eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2IiwiaWF0IjoxNDQxMTE4Nzk2LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxNzIzNTk2fQ.xUjcxTZhWx74aa6adnUXjuvUgqjC8TvvrB7cBEmNF_g

This would be the response:

.. code-block:: http 

  HTTP/1.1 200 OK
  Content-Type: application/x-www-form-urlencoded

  {
    "access_token": "eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiI2TnJXSXM1aWttSVBWSkNuMnA0bnJyIiwiaWF0IjoxNDQxMTMzNjQ1LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxMTM1NDQ1LCJydGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2In0.SbSmuPz0-v4J2BO9-lpyz_2_T62mSB1ql_0IMrftpgg",
    "refresh_token": "eyJraWQiOiIyWkZNVjRXVlZDVkczNVhBVElJOVQ5Nko3IiwiYWxnIjoiSFMyNTYifQ.eyJqdGkiOiIxdkhEZ2Z0THJ4Slp3dFExc2hFaTl2IiwiaWF0IjoxNDQxMTE4Nzk2LCJpc3MiOiJodHRwczovL2FwaS5zdG9ybXBhdGguY29tL3YxL2FwcGxpY2F0aW9ucy8xZ2s0RHh6aTZvNFBiZGxCVmE2dGZSIiwic3ViIjoiaHR0cHM6Ly9hcGkuc3Rvcm1wYXRoLmNvbS92MS9hY2NvdW50cy8zYXBlbll2TDBaOXY5c3BkenBGZmV5IiwiZXhwIjoxNDQxNzIzNTk2fQ.xUjcxTZhWx74aa6adnUXjuvUgqjC8TvvrB7cBEmNF_g",
    "token_type": "Bearer",
    "expires_in": 1800,
    "stormpath_access_token_href": "https://api.stormpath.com/v1/accessTokens/6NrWIs5ikmIPVJCn2p4nrr"
  }

Note that this response contains the same Refresh Token as was in the request. This is because when Stormpath generates a new Access Token for a Refresh Token it does not generate a new Refresh token, nor does it modify its expiration time. This means that once the Refresh Token expires, the user must authenticate again to get a new Access and Refresh Tokens.


Revoking Access and Refresh Tokens
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are cases where you might want to revoke the Access and Refresh Tokens that you have generated for a user. For example:

- The user has explicitly logged out, and your application needs to revoke their access, requiring re-authentication.
- The application, device, and/or client has been compromised and you need to revoke tokens for an Account.

To revoke the tokens, simply delete the Account's ``/accessTokens/:accessTokenId`` resource. 

To retrieve an Account's Access and Refresh tokens, make an HTTP GET calls for the Account information, then you will find the tokens inside the ``/accessTokens`` and ``/refreshTokens`` collections:

.. code-block:: json 

  {
    "href": "https://api.stormpath.com/v1/accounts/3apenYvL0Z9v9spdzpFfey",
    "username": "jlpicard",
    
    [...]
    
    "accessTokens": {
      "href": "https://api.stormpath.com/v1/accounts/3apenYvL0Z9v9spdzpFfey/accessTokens"
    },
    "refreshTokens": {
      "href": "https://api.stormpath.com/v1/accounts/3apenYvL0Z9v9spdzpFfey/refreshTokens"
    }
  }

If you then perform a GET on the ``accessTokens`` link, you will get back the individual tokens:

.. code-block:: json 

    {
      "href": "https://api.stormpath.com/v1/accounts/3apenYvL0Z9v9spdzpFfey/accessTokens",
      "offset": 0,
      "limit": 25,
      "size": 1,
      "items": [
        {
          "href": "https://api.stormpath.com/v1/accessTokens/6NrWIs5ikmIPVJCn2p4nrr",
          [...]
        }
      ]
    }

To revoke the token, simply issue an HTTP Delete::

    DELETE https://api.stormpath.com/v1/accessTokens/6NrWIs5ikmIPVJCn2p4nrr

You will get back a ``204 No Content`` response back from Stormpath when the call succeeds. 

.. _social-authn:

c. How Social Authentication Works
==================================

Social authentication essentially means using the "Log in with x" button in your application, where "x" is a Social Login Provider of some kind. The Social Login Providers currently supported by Stormpath are: 

- Google
- Facebook
- Github,
- LinkedIn 

In general, the social login process works as follows: 

1. The user who wishes to authenticate will click a "Log in with x" link.

2. The user will be asked by the Provider to accept the permissions required by your app.

3. The Provider will return the user to your application with an access token.

4. Stormpath will take this access token and use it to query the provider for:  
   
   - an email address
   - a first name
   - a last name.
   
.. note::

    If Stormpath is unable to retrieve the user's first and last name, it will populate those attributes with a default value: ``NOT_PROVIDED``.

5. Stormpath will first search for a Directory that matches the provider of the access token. If one is not found, an error will return.

6. Once the Directory is located, Stormpath will look for an Account in your application's Directories that matches this information.

   a. If a matching Account is found, Stormpath will return the existing Account's ``href``.

   b. If a matching Account is not found, Stormpath will create one and return the new Account's ``href``.

7. At this point, a language/framework-specific integration would use this ``href`` to create a Session for the user.

As a developer, integrating Social Login into your application with Stormpath only requires three steps:

1. Create a Social Directory for your Provider.

2. Map the Directory as an Account Store to an Application resource. When an Account Store (in this case a Directory) is mapped to an Application, the Accounts in the AccountStore are considered the Application’s users and they can log in to it.

3. Include the provider-specific logic that will access the social account (e.g. embed the appropriate link in your site that will send an authentication request to the social provider) 

i. Google
---------

Before you integrate Google Login with Stormpath, you must complete the following steps:

- Create an application in the `Google Developer Console <https://console.developers.google.com/start>`_

- Enable Google Login for your Google application

- Retrieve the OAuth Credentials (Client ID and Secret) for your Google application

- Add your application's redirect URL, which is the URL the user will be returned to after successful authentication.
  
.. note::

    Be sure to only enter the Redirect URL you are currently using. So, if you are running your app in development mode, set it to your local URL, and if you're running your app in production mode, set it to your production URL.
  
For more information, please see the `Google OAuth 2.0 documentation <https://developers.google.com/identity/protocols/OAuth2>`_.

Step 1: Create a Social Directory for Google
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating this Directory for Google requires that you provide information from Google as a Provider resource. This can be accomplished by sending an HTTP POST:

.. code-block:: http 

  POST /v1/directories HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8
    
  {
      "name" : "my-google-directory",
      "description" : "A Google directory",
      "provider": {
          "providerId": "google",
          "clientId":"YOUR_GOOGLE_CLIENT_ID",
          "clientSecret":"YOUR_GOOGLE_CLIENT_SECRET",
          "redirectUri":"YOUR_GOOGLE_REDIRECT_URI"
      } 
  }

.. note::

    If you are using `Google+ Sign-In for server-side apps <https://developers.google.com/identity/sign-in/web/server-side-flow>`_, Google recommends that you leave the "Authorized Redirect URI" field blank in the Google Developer Console. In Stormpath, when creating the Google Directory, you must set the redirect URI to ``postmessage``.

Step 2: Map the Google Directory as an Account Store for Your Application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating an Account Store Mapping between your new Google Directory and your Stormpath Application can be done through the REST API, as described in :ref:`create-asm`.

Step 3: Access an Account with Google Tokens
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To access or create an Account in your new Google Directory, you must gather a Google **Authorization Code** on behalf of the user. This requires leveraging `Google’s OAuth 2.0 protocol <https://developers.google.com/identity/protocols/OAuth2>`_ and the user’s consent for your application’s permissions.

Generally, this will include embedding a link in your site that will send an authentication request to Google. Once the user has authenticated, Google will redirect the response to your application, including the **Authorization Code** or **Access Token**. This is documented in detail here: `Using OAuth 2.0 for Web Server Applications <https://developers.google.com/identity/protocols/OAuth2WebServer>`_.

.. note::

    It is required that your Google application requests the ``email`` scope from Google. If the authorization code or access token does not grant ``email`` scope, you will not be able to get an Account. For more information about scopes please see `Google's OAuth Login Scopes documentation <https://developers.google.com/+/web/api/rest/oauth#login-scopes>`_.

Once the Authorization Code is gathered, you send an HTTP POST:

.. code-block:: http 

  POST /v1/applications/YOUR_APP_ID/accounts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
      "providerData": {
        "providerId": "google",
        "code": "YOUR_GOOGLE_AUTH_CODE"
      }
  }

If you have already exchanged an Authorization Code for an Access Token, this can be passed to Stormpath in a similar fashion:

.. code-block:: http 

  POST /v1/applications/YOUR_APP_ID/accounts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8
    
  {
      "providerData": {
        "providerId": "google",
        "accessToken": "%ACCESS_TOKEN_FROM_GOOGLE%"
      }
  }

Either way, Stormpath will use the ``code`` or ``accessToken`` provided to retrieve information about your Google Account, then return a Stormpath Account. The HTTP Status code will tell you if the Account was created (HTTP 201) or if it already existed in Stormpath (HTTP 200). 

ii. Facebook
------------

Before you integrate Facebook Login with Stormpath, you must complete the following steps:

- Create an application on the `Facebook Developer Site <https://developers.facebook.com/>`_

- Retrieve your OAuth credentials (App ID and App Secret)

- Add your application's private and public root URLs
  
For more information, please see the `Facebook documentation <https://developers.facebook.com/docs/apps/register>`_.

Step 1: Create a Social Directory for Facebook
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating this Directory requires that you provide information from Facebook as a Provider resource. This can be accomplished by sending an HTTP POST:

.. code-block:: http 

  POST /v1/directories HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
      "name" : "my-facebook-directory",
      "description" : "A Facebook directory",
      "provider": {
        "providerId": "facebook",
        "clientId":"YOUR_FACEBOOK_APP_ID",
        "clientSecret":"YOUR_FACEBOOK_APP_SECRET"
      }
  }

Step 2: Map the Facebook Directory as an Account Store for Your Application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating an Account Store Mapping between your new Facebook Directory and your Stormpath Application can be done through the REST API, as described in :ref:`create-asm`.

Step 3: Access an Account with Facebook Tokens
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To access or create an Account in your new Facebook Directory, you need to gather a **User Access Token** from Facebook before submitting it to Stormpath. This is possible either by using a `Facebook SDK Library <https://developers.facebook.com/docs/facebook-login/access-tokens/#usertokens>`_, or `Facebook’s Graph Explorer <https://developers.facebook.com/tools/explorer>`_ for testing.

.. note::

    It is required that your Facebook application requests the ``email`` scope from Facebook. If the access token does not grant ``email`` scope, you will not be able to get an Account with an access token. For more information about scopes please see `Permissions with Facebook Login <https://developers.facebook.com/docs/facebook-login/permissions/>`_.

Once the User Access Token is gathered, you send an HTTP POST:

.. code-block:: http 

  POST /v1/applications/YOUR_APP_ID/accounts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
      "providerData": {
        "providerId": "facebook",
        "accessToken": "USER_ACCESS_TOKEN_FROM_FACEBOOK"
      }
  }

Stormpath will use the ``accessToken`` provided to retrieve information about your Facebook Account, then return a Stormpath Account. The HTTP Status code will tell you if the Account was created (HTTP 201) or if it already existed in Stormpath (HTTP 200). 


iii. Github
-----------

Before you integrate GitHub Login with Stormpath, you must complete the following steps:

- Create an application in the `GitHub Developer Site <https://developer.github.com/>`_

- Retrieve OAuth Credentials (Client ID and Secret) for your GitHub application

- Add your application's redirect URL, which is the URL the user will be returned to after successful authentication.
  
For more information, please see the `GitHub documentation on registering your app <https://developer.github.com/guides/basics-of-authentication/#registering-your-app>`_.

Step 1: Create a Social Directory for GitHub
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating this Directory requires that you provide information from GitHub as a Provider resource. This can be accomplished by sending an HTTP POST:

.. code-block:: http 

  POST /v1/directories HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8
    
  {
      "name" : "my-github-directory",
      "description" : "A GitHub directory",
      "provider": {
        "providerId": "github",
        "clientId":"YOUR_GITHUB_CLIENT_ID",
        "clientSecret":"YOUR_GITHUB_CLIENT_SECRET"
      }
  }

Step 2: Map the GitHub Directory as an Account Store for Your Application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating an Account Store Mapping between your new GitHub Directory and your Stormpath Application can be done through the REST API, as described in :ref:`create-asm`.

Step 3: Access an Account with GitHub Tokens
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To access or create an Account in your new Github Directory, you must gather a Github **Authorization Code** on behalf of the user. This requires leveraging `Github's OAuth 2.0 protocol <https://developer.github.com/v3/oauth>`_ and the user’s consent for your application’s permissions.

Generally, this will include embedding a link in your site that will send an authentication request to Github. Once the user has authenticated, Github will redirect the response to your application, including the **Authorization Code**. This is documented in detail `here <https://developer.github.com/v3/oauth/#web-application-flow>`_.

.. note::

    It is required that your GitHub application requests the ``user:email`` scope from GitHub. If the access token does not grant ``user:email`` scope, you will not be able to get an Account with an access token. For more information about see `Github's documentation on OAuth scopes <https://developer.github.com/v3/oauth/#scopes>`_. 

Once the Authorization Code is gathered, you can send an HTTP POST:

.. code-block:: http 

  POST /v1/applications/YOUR_APP_ID/accounts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
    "providerData": {
      "providerId": "github",
      "code": "AUTH_CODE_FROM_GITHUB"
    }
  }

Stormpath will use the ``code`` provided to retrieve information about your GitHub Account, then return a Stormpath Account. The HTTP Status code will tell you if the Account was created (HTTP 201) or if it already existed in Stormpath (HTTP 200). 

iv. LinkedIn
------------

Before you integrate LinkedIn Login with Stormpath, you must complete the following steps:

- Create an application in the `LinkedIn Developer Site <https://www.linkedin.com/secure/developer?newapp=>`_

- Add your application's redirect URLs, which are the URL the user will be returned to after successful authentication.

- Retrieve OAuth Credentials (Client ID and Secret) for your LinkedIn application
  
For more information, please see `LinkedIn's OAuth documentation <https://developer.linkedin.com/docs/oauth2>`_.

Step 1: Create a Social Directory for LinkedIn
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating this Directory requires that you provide information from LinkedIn as a Provider resource. This can be accomplished by sending an HTTP POST:

.. code-block:: http 

  POST /v1/directories HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
      "name" : "my-linkedin-directory",
      "description" : "A LinkedIn Directory",
      "provider": {
        "providerId": "linkedin",
        "clientId":"YOUR_LINKEDIN_APP_ID",
        "clientSecret":"YOUR_LINKEDIN_APP_SECRET"
      }
  }

Step 2: Map the LinkedIn Directory as an Account Store for Your Application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating an Account Store Mapping between your new LinkedIn Directory and your Stormpath Application can be done through the REST API, as described in :ref:`create-asm`.

Step 3: Access an Account with LinkedIn Tokens
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To access or create an Account in your new LinkedIn Directory, you must gather a LinkedIn **Access Token** on behalf of the user. This requires leveraging `LinkedIn's OAuth 2.0 protocol <https://developer.linkedin.com/docs/oauth2>`_ and the user’s consent for your application’s permissions.

Generally, this will include embedding a link in your site that will send an authentication request to LinkedIn. Once the user has authenticated, LinkedIn will redirect the response to your application, including the Authorization Code that you will exchange for the Access Token. This is documented in detail in LinkedIn's `Authenticating with OAuth 2.0 page <https://developer.linkedin.com/docs/oauth2#hero-par_longformtext_3_longform-text-content-par_resourceparagraph_3>`_.

.. note::

    It is required that your LinkedIn application requests the ``r_basicprofile`` and ``r_emailaddress`` scopes from LinkedIn. If the access token does not grant these scopes, you will not be able to get an Account with an access token. For more information about LinkedIn scopes, see `LinkedIn's "Profile Fields" documentation <https://developer.linkedin.com/docs/fields>`_.

Once the Access Token is gathered, you can send an HTTP POST:

.. code-block:: http 

  POST /v1/applications/YOUR_APP_ID/accounts HTTP/1.1
  Host: api.stormpath.com
  Content-Type: application/json;charset=UTF-8

  {
    "providerData": {
      "providerId": "linkedin",
      "accessToken": "TOKEN_FROM_LINKEDIN"
    }
  }

Stormpath will use the ``accessToken`` provided to retrieve information about your LinkedIn Account, then return a Stormpath Account. The HTTP Status code will tell you if the Account was created (HTTP 201) or if it already existed in Stormpath (HTTP 200). 

.. _mirror-dir-authn:

d. Authenticating Against a Mirrored LDAP Directory
===================================================

This section assumes that you are already familiar both with :ref:`how-login-works` and the concept of Stormpath :ref:`about-mirror-dir` as well as how they are :ref:`modeled <modeling-mirror-dirs>`. 

Mirror Directories and LDAP 
---------------------------

To recap: With LDAP integration, Stormpath is simply mirroring the canonical LDAP user directory, so it is recommended that you maintain a "master" Directory alongside your Mirror Directory. Furthermore, a successful user login is the recommended time to provision, link, and synchronize an Account in the Mirror Directory to your master Directory.

Step 1: Create a Mirror Directory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

HTTP POST a new Directory resource to the ``/directories`` endpoint. This Directory will contain a :ref:`ref-provider` resource with ``providerId`` set to ``"ldap"``. This Provider resource will in turn contain an :ref:`ref-ldap-agent` object:

.. code-block:: http

    POST /v1/directories HTTP/1.1
    Host: api.stormpath.com
    Content-Type: application/json;charset=UTF-8
    
    {
      "name":"My LDAP Directory",
      "description":"An LDAP Directory created with the Stormpath API",
      "provider":{
        "providerId":"ldap",
        "agent":{
          "config":{
            "directoryHost":"ldap.local",
            "directoryPort":"636",
            "sslRequired":true,
            "agentUserDn":"tom@stormpath.com",
            "agentUserDnPassword":"StormpathRulez",
            "baseDn":"dc=example,dc=com",
            "pollInterval":60,
            "referralMode":"ignore",
            "ignoreReferralIssues":false,
            "accountConfig":{
              "dnSuffix":"ou=employees",
              "objectClass":"person",
              "objectFilter":"(cn=finance)",
              "emailRdn":"email",
              "givenNameRdn":"givenName",
              "middleNameRdn":"middleName",
              "surnameRdn":"sn",
              "usernameRdn":"uid",
              "passwordRdn":"userPassword"
            },
            "groupConfig":{
              "dnSuffix":"ou=groups",
              "objectClass":"groupOfUniqueNames",
              "objectFilter":"(ou=*-group)",
              "nameRdn":"cn",
              "descriptionRdn":"description",
              "membersRdn":"uniqueMember"
            }
          }
        }
      }
    }

For more information about all of these values, please see the Reference chapter :ref:`ref-directory` section.

Step 2: Install your LDAP Agent 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Installing your Agent is done in three steps.

**1. Download**

Download your Agent by following the Download link on the Agent page in the Admin Console.
   
**2. Configure**
   
*a.* Make sure Java 1.8 is installed

*b.* Unzip to a location in your file system, for example ``C:\stormpath\agent`` in Windows or ``/opt/stormpath/agent`` in Unix.

In the same location, open the file ``dapper.properties`` from the config folder and replace this line::

  agent.id = PutAgentSpecificIdHere

With this line::

  agent.id  = 72MlbWz6C4dLo1oBhgjjTt

Follow the instructions in the ``dapper.properties`` file to reference your account's API authentication.
   
**3. Start**

In Windows:

(cd to your agent directory, for example C:\stormpath\agent)

.. code-block:: powershell

  C:\stormpath\agent>cd bin
  C:\stormpath\agent\bin>startup.bat

In Unix:

cd to your agent directory, for example /opt/stormpath/agent

.. code-block:: bash 

  $ cd bin
  $ startup.sh

The Agent will start synchronizing immediately, pushing the configured data to Stormpath. You will see the synchronized user Accounts and Groups appear in the Stormpath Directory, and the Accounts will be able to log in to any Stormpath-enabled application that you assign. When the Agent detects local changes, additions or deletions to the mirrored Accounts or Groups, it will automatically propagate those changes to Stormpath.

Step 3: Map the Mirror Directory as an Account Store for Your Application
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Creating an Account Store Mapping between your new Mirror Directory and your Stormpath Application can be done through the REST API, as described in :ref:`create-asm`.

From this point on, any time a user logs in to your Application, their Account will be provisioned into Stormpath, as detailed above in :ref:`non-cloud-login`.
