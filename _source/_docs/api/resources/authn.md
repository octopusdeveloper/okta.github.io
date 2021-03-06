---
layout: docs_page
title: Authentication API
redirect_from: "/docs/api/rest/authn.html"
---

# Authentication API

The Okta Authentication API provides operations to authenticate users, perform multi-factor enrollment and verification, recover forgotten passwords, and unlock accounts. It can be used as a standalone API to provide the identity layer on top of your existing application, or it can be integrated with the Okta [Sessions API](sessions.html) to obtain an Okta [session cookie](/docs/examples/session_cookie.html) and access apps within Okta.

The API is targeted for developers who want to build their own end-to-end login experience to replace the built-in Okta login experience and addresses the following key scenarios:

- **Primary authentication** allows you to verify username and password credentials for a user.

- **Multifactor authentication** (MFA) strengthens the security of password-based authentication by requiring additional verification of another factor such as a temporary one-time password or an SMS passcode. The Authentication API supports user enrollment with MFA factors enabled by the administrator, as well as MFA challenges based on your **Okta Sign-On Policy**.

- **Recovery** allows users to securely reset their password if they've forgotten it, or unlock their account if it has been locked out due to excessive failed login attempts. This functionality is subject to the security policy set by the administrator.

## Application Types

The behavior of the Okta Authentication API varies depending on the type of your application and your org's security policies such as the **Okta Sign-On Policy**, **MFA Enrollment Policy**, or **Password Policy**.

> Policy evaluation is conditional on the [client request context](../getting_started/design_principles.html#client-request-context) such as IP address.

### Public Application

A public application is an application that anonymously starts an authentication or recovery transaction without an API token, such as the [Okta Sign-In Widget](/docs/guides/okta_sign-in_widget.html).  Public applications are aggressively rate-limited to prevent abuse and require primary authentication to be successfully completed before releasing any metadata about a user.

### Trusted Application

Trusted applications are backend applications that act as authentication broker or login portal for your Okta organization and may start an authentication or recovery transaction with an administrator API token.  Trusted apps may implement their own recovery flows and primary authentication process and may receive additional metadata about the user before primary authentication has successfully completed.

> Trusted web applications may need to override the [client request context](../getting_started/design_principles.html#client-request-context) to forward the originating client context for the user.


## Getting Started with Authentication

1. Make sure you need the API. Check out the [Okta Sign-In Widget](/docs/guides/okta_sign-in_widget.html) which is built on the Authentication API. The Sign-In Widget is easier to use and supports basic use cases.
2. For more advanced use cases, learn [the Okta API basics](/docs/api/getting_started/api_test_client.html).
3. Explore the Authentication API:

    [![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/07df454531c56cb5fe71)

## Authentication Operations

### Primary Authentication
{:.api .api-operation}

{% api_operation post /api/v1/authn %}

Every authentication transaction starts with primary authentication which validates a user's primary password credential. **Password Policy**, **MFA Policy**,  and **Sign-On Policy** are evaluated during primary authentication to determine if the user's password is expired, a factor should be enrolled, or additional verification is required. The [transaction state](#transaction-state) of the response depends on the user's status, group memberships and assigned policies.

The requests and responses vary depending on the application type, and whether a password expiration warning is sent:

- [Primary Authentication with Public Application](#primary-authentication-with-public-application)&mdash;[Request Example](#request-example-public-application)
- [Primary Authentication with Trusted Application](#primary-authentication-with-trusted-application)&mdash;[Request Example](#request-example-trusted-application)
- [Primary Authentication with activationToken](#primary-authentication-with-activation-token)&mdash;[Request Example](#request-example-activation-token)
- [Primary Authentication with Password Expiration Warning](#primary-authentication-with-password-expiration-warning)&mdash;[Request Example](#request-example-password-expiration-warning)

> You must first enable MFA factors and assign a valid **Sign-On Policy** to a user to enroll and/or verify a MFA factor during authentication.

#### Request Parameters
{:.api .api-request .api-request-params}

As part of the authentication call either the username and password or the token parameter must be provided.

Parameter   | Description                                                                                                      | Param Type | DataType                          | Required | Default | MaxLength
----------- | ---------------------------------------------------------------------------------------------------------------- | ---------- | --------------------------------- | -------- | ------- | ---------
username    | User's non-qualified short-name (e.g. dade.murphy) or unique fully-qualified login (e.g dade.murphy@example.com) | Body       | String                            | FALSE    |         |
password    | User's password credential                                                                                       | Body       | String                            | FALSE    |         |
relayState  | Optional state value that is persisted for the lifetime of the authentication transaction                        | Body       | String                            | FALSE    |         | 2048
options     | Opt-in features for the authentication transaction                                                               | Body       | [Options Object](#options-object) | FALSE    |         |
context     | Provides additional context for the authentication transaction                                                   | Body       | [Context Object](#context-object) | FALSE    |         |
token       | Token received as part of activation user request                                                                | Body       | String                            | FALSE    |         |

##### Options Object

The authentication transaction [state machine](#transaction-state) can be modified via the following opt-in features:

|----------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------+----------+----------+--------+----------|
| Property                   | Description                                                                                                                                                | DataType | Nullable | Unique | Readonly |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------- | ------ | -------- |
| multiOptionalFactorEnroll  | Transitions transaction back to `MFA_ENROLL` state after successful factor enrollment when additional optional factors are available for enrollment        | Boolean  | TRUE     | FALSE  | FALSE    |
| warnBeforePasswordExpired  | Transitions transaction to `PASSWORD_WARN` state before `SUCCESS` if the user's password is about to expire and within their password policy warn period   | Boolean  | TRUE     | FALSE  | FALSE    |
|----------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------+----------+----------+--------+----------|

##### Context Object

The context object allows [trusted web applications](#trusted-application) such as an external portal to pass additional context for the authentication or recovery transaction.

> Overriding context such as `deviceToken` is a highly privileged operation limited to trusted web applications and requires making authentication or recovery requests with a valid *administrator API token*.

|-------------+-------------------------------------------------------------------------+----------+----------+--------+----------+-----------+-----------|
| Property    | Description                                                             | DataType | Nullable | Unique | Readonly | MinLength | MaxLength |
| ----------- | ----------------------------------------------------------------------- | -------- | -------- | ------ | -------- | --------- | --------- |
| deviceToken | A globally unique ID identifying the user's client device or user agent | String   | TRUE     | FALSE  | FALSE    |           | 32        |
|-------------+-------------------------------------------------------------------------+----------+----------+--------+----------+-----------+-----------|

> You must always pass the same `deviceToken` for a user's device with every authentication request for **per-device** or **per-session** Sign-On Policy factor challenges.  If the `deviceToken` is absent or does not match the previous `deviceToken`, the user will be challenged every-time instead of **per-device** or **per-session**.

It is recommended that you generate a UUID or GUID for each client and persist the `deviceToken` as a persistent cookie or HTML5 localStorage item scoped to your web application's origin.

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

`401 Unauthorized` status code is returned for requests with invalid credentials or locked out accounts.

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "errorCode": "E0000004",
  "errorSummary": "Authentication failed",
  "errorLink": "E0000004",
  "errorId": "oaeuHRrvMnuRga5UzpKIOhKpQ",
  "errorCauses": []
}
~~~

`403 Forbidden` status code is returned for requests with valid credentials but the user is denied access via **Sign-On Policy** or **MFA Policy**.

> Ensure the user is assigned to a valid **MFA Policy** if additional verification is required in the user's **Sign-On Policy**.

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000085",
    "errorSummary": "You do not have permission to access your account at this time.",
    "errorLink": "E0000085",
    "errorId": "oaeLngBsntPSj2IVcrEWcAZSA",
    "errorCauses": []
}
~~~

`429 Too Many Requests` status code may be returned when the rate-limit is exceeded.

> Authentication requests are aggressively rate-limited (e.g. 1 request per username/per second) for security purposes.

~~~http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-Rate-Limit-Limit: 1
X-Rate-Limit-Remaining: 0
X-Rate-Limit-Reset: 1447534590

{
    "errorCode": "E0000047",
    "errorSummary": "API call exceeded rate limit due to too many requests.",
    "errorLink": "E0000047",
    "errorId": "oaeWaNHfOyQSES34-a2Dw6Phw",
    "errorCauses": []
}
~~~

#### Primary Authentication with Public Application

Authenticates a user with username/password credentials via a [public application](#public-application)

##### Request Example: Public Application
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "password": "correcthorsebatterystaple",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "options": {
    "multiOptionalFactorEnroll": false,
    "warnBeforePasswordExpired": false
  }
}' "https://${org}.okta.com/api/v1/authn"
~~~

##### Response Example (Success)
{:.api .api-response .api-response-example}

Users with a valid password not assigned to a **Sign-On Policy** with additional verification requirements will successfully complete the authentication transaction.

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

##### Response Example (Invalid Credentials)
{:.api .api-response .api-response-example}

Primary authentication requests with an invalid `username` or `password` fail with a `401 Unauthorized` error.

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "errorCode": "E0000004",
  "errorSummary": "Authentication failed",
  "errorLink": "E0000004",
  "errorId": "oaeuHRrvMnuRga5UzpKIOhKpQ",
  "errorCauses": []
}
~~~

##### Response Example (Access Denied)
{:.api .api-response .api-response-example}

Primary authentication request has valid credentials but the user is denied access via  **Sign-On Policy** or **MFA Policy** with `403 Forbidden` error.

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000085",
    "errorSummary": "You do not have permission to access your account at this time.",
    "errorLink": "E0000085",
    "errorId": "oaeLngBsntPSj2IVcrEWcAZSA",
    "errorCauses": []
}
~~~

##### Response Example (Locked Out)
{:.api .api-response .api-response-example}

Primary authentication requests for a user with `LOCKED_OUT` status is conditional on the user's password policy.  Password policies define whether to hide or show  lockout failures which disclose a valid user identifier to the caller.

###### Hide Lockout Failures (Default)

If the user's password policy is configure to **hide lockout failures**, a `401 Unauthorized` error is returned preventing information disclosure of a valid user identifier.

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "errorCode": "E0000004",
  "errorSummary": "Authentication failed",
  "errorLink": "E0000004",
  "errorId": "oaeuHRrvMnuRga5UzpKIOhKpQ",
  "errorCauses": []
}
~~~

###### Show Lockout Failures

If the user's password policy is configure to **show lockout failures**, the authentication transaction completes with `LOCKED_OUT` status.

~~~json
{
  "status": "LOCKED_OUT",
  "_links": {
    "next": {
      "name": "unlock",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/unlock",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Expired Password)
{:.api .api-response .api-response-example}

User must [change their expired password](#change-password) to complete the authentication transaction.

> Users will be challenged for MFA (`MFA_REQUIRED`) before `PASSWORD_EXPIRED` if they have an active factor enrollment

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "PASSWORD_EXPIRED",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "policy": {
      "complexity": {
        "minLength": 8,
        "minLowerCase": 1,
        "minUpperCase": 1,
        "minNumber": 1,
        "minSymbol": 0
      }
    }
  },
  "_links": {
    "next": {
      "name": "changePassword",
      "href": "https://your-domain.okta.com/api/v1/authn/credentials/change_password",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Factor Challenge)
{:.api .api-response .api-response-example}

User is assigned to a **Sign-On Policy** that requires additional verification and must [select and verify](#verify-factor) a previously enrolled [factor](#factor-object) by `id` to complete the authentication transaction.

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_REQUIRED",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": [
      {
        "id": "rsalhpMQVYKHZKXZJQEW",
        "factorType": "token",
        "provider": "RSA",
        "profile": {
          "credentialId": "dade.murphy@example.com"
        },
        "_links": {
          "verify": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors/rsalhpMQVYKHZKXZJQEW/verify",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "id": "ostfm3hPNYSOIOIVTQWY",
        "factorType": "token:software:totp",
        "provider": "OKTA",
        "profile": {
          "credentialId": "dade.murphy@example.com"
        },
        "_links": {
          "verify": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors/ostfm3hPNYSOIOIVTQWY/verify",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "id": "sms193zUBEROPBNZKPPE",
        "factorType": "sms",
        "provider": "OKTA",
        "profile": {
          "phoneNumber": "+1 XXX-XXX-1337"
        },
        "_links": {
          "verify": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
         "id": "clf193zUBEROPBNZKPPE",
         "factorType": "call",
         "provider": "OKTA",
         "profile": {
           "phoneNumber": "+1 XXX-XXX-1337"
         },
         "_links": {
           "verify": {
             "href": "https://your-domain.okta.com/api/v1/authn/factors/clf193zUBEROPBNZKPPE/verify",
             "hints": {
               "allow": [
                 "POST"
                ]
              }
            }
         }
      },
      {
        "id": "opf3hkfocI4JTLAju0g4",
        "factorType": "push",
        "provider": "OKTA",
        "profile": {
          "credentialId": "dade.murphy@example.com",
          "deviceType": "SmartPhone_IPhone",
          "name": "Gibson",
          "platform": "IOS",
          "version": "9.0"
        },
        "_links": {
          "verify": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors/opf3hkfocI4JTLAju0g4/verify",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      }
    ]
  },
  "_links": {
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Factor Enroll)
{:.api .api-response .api-response-example}

User is assigned to a **MFA Policy** that requires enrollment during sign-on and must [select a factor to enroll](#enroll-factor) to complete the authentication transaction.

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": [
      {
        "factorType": "token",
        "provider": "RSA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "factorType": "token:software:totp",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "factorType": "sms",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "factorType": "call",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
               ]
            }
          }
         }
      },
      {
        "factorType": "push",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      }
    ]
  },
  "_links": {
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Primary Authentication with Trusted Application

Authenticates a user via a [trusted application](#trusted-application) or proxy that overrides [client request context](../getting_started/design_principles.html#client-request-context).

> Specifying your own `deviceToken` is a highly privileged operation limited to trusted web applications and requires making authentication requests with a valid *API token*.

> The **public IP address** of your [trusted application](#trusted-application) must be [whitelisted as a gateway IP address](../getting_started/design_principles.html#ip-address) to forward the user agent's original IP address with the `X-Forwarded-For` HTTP header

##### Request Example: Trusted Application
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36" \
-H "X-Forwarded-For: 23.235.46.133" \
-d '{
  "username": "dade.murphy@example.com",
  "password": "correcthorsebatterystaple",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "options": {
    "multiOptionalFactorEnroll": false,
    "warnBeforePasswordExpired": false
  },
  "context": {
    "deviceToken": "26q43Ak9Eh04p7H6Nnx0m69JqYOrfVBY"
  }
}' "https://${org}.okta.com/api/v1/authn"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Primary Authentication with Activation Token

Authenticates a user via a [trusted application](#trusted-application) or proxy that overrides [client request context](../getting_started/design_principles.html#client-request-context).

> Specifying your own `deviceToken` is a highly privileged operation limited to trusted web applications and requires making authentication requests with a valid *API token*.

> The **public IP address** of your [trusted application](#trusted-application) must be [whitelisted as a gateway IP address](../getting_started/design_principles.html#ip-address) to forward the user agent's original IP address with the `X-Forwarded-For` HTTP header

##### Request Example: Activation Token
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36" \
-H "X-Forwarded-For: 23.235.46.133" \
-d '{
  "token": "o7AFoTGE9xjQiHQK6dAa"
}' "https://${org}.okta.com/api/v1/authn"
~~~

##### Response Example (Success - User with Password, no MFA)
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2017-03-29T21:42:30.000Z",
  "status": "SUCCESS",
  "sessionToken": "20111DuMTdPoBlMOqX5R_OAV3ku2bTWxP6wUIRT_jqkU6XTvOsJLmDq",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2017-03-29T21:37:25.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

##### Response Example (Success - User with Password and Configured MFA)
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00bMktAiPaI0Jo97bpiKxEw7drTgtukJKs33abrSpb",
  "expiresAt": "2017-03-29T21:49:09.000Z",
  "status": "MFA_ENROLL",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2017-03-29T21:37:25.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": [
      {
        "factorType": "question",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "_links": {
          "questions": {
            "href": "https://your-domain.okta.com/api/v1/users/00u1nehnZ6qp4Qy8G0g4/factors/questions",
            "hints": {
              "allow": [
                "GET"
              ]
            }
          },
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      },
      {
        "factorType": "token:software:totp",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      },
      {
        "factorType": "token:software:totp",
        "provider": "GOOGLE",
        "vendorName": "GOOGLE",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      },
      {
        "factorType": "sms",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      },
      {
        "factorType": "push",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      },
      {
        "factorType": "token:hardware",
        "provider": "YUBICO",
        "vendorName": "YUBICO",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        },
        "status": "NOT_SETUP"
      }
    ]
  },
  "_links": {
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Success - User Without Password)
{:.api .api-response .api-response-example}

In the case where the user was created without credentials the response will trigger the workflow to [set the user's password](#change-password). After the password is configured, depending on the MFA setting, the workflow continues with MFA enrollment or a successful authentication completes.

~~~json
{
  "stateToken": "005Oj4_rx1yAYP2MFNobMXlM2wJ3QEyzgifBd_T6Go",
  "expiresAt": "2017-03-29T21:35:47.000Z",
  "status": "PASSWORD_RESET",
  "recoveryType": "ACCOUNT_ACTIVATION",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "policy": {
      "expiration": {
        "passwordExpireDays": 5
      },
      "complexity": {
        "minLength": 8,
        "minLowerCase": 1,
        "minUpperCase": 1,
        "minNumber": 1,
        "minSymbol": 0
      }
    }
  },
  "_links": {
    "next": {
      "name": "resetPassword",
      "href": "https://your-domain.okta.com/api/v1/authn/credentials/reset_password",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Failure - Invalid or Expired Token)
{:.api .api-response .api-response-example}

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "errorCode": "E0000004",
  "errorSummary": "Authentication failed",
  "errorLink": "E0000004",
  "errorId": "oae2fYYJmkuTg-NyozqBkb3sw",
  "errorCauses": []
}
~~~

#### Primary Authentication with Password Expiration Warning

Authenticates a user with a password that is about to expire.  The user should [change their password](#change-password) to complete the authentication transaction but may skip.

>Please note:
* The `warnBeforePasswordExpired` option must be explicitly specified as `true` to allow the authentication transaction to transition to `PASSWORD_WARN` status.
* Non-expired passwords successfully complete the authentication transaction if this option is omitted or is specified as `false`.

##### Request Example: Password Expiration Warning
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "password": "correcthorsebatterystaple",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "options": {
    "multiOptionalFactorEnroll": false,
    "warnBeforePasswordExpired": true
  }
}' "https://${org}.okta.com/api/v1/authn"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "PASSWORD_WARN",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "policy": {
      "expiration": {
        "passwordExpireDays": 5
      },
      "complexity": {
        "minLength": 8,
        "minLowerCase": 1,
        "minUpperCase": 1,
        "minNumber": 1,
        "minSymbol": 0
      }
    }
  },
  "_links": {
    "next": {
      "name": "changePassword",
      "href": "https://your-domain.okta.com/api/v1/authn/credentials/change_password",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "skip": {
      "name": "skip",
      "href": "https://your-domain.okta.com/api/v1/authn/skip",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Change Password
{:.api .api-operation}

{% api_operation post api/v1/authn/credentials/change_password %}

This operation changes a user's password by providing the existing password and the new password password for authentication transactions with either the `PASSWORD_EXPIRED` or `PASSWORD_WARN` state.

- A user *must* change their expired password for an authentication transaction with `PASSWORD_EXPIRED` status to successfully complete the transaction.
- A user *may* opt-out of changing their password (skip) when the transaction has a `PASSWORD_WARN` status.

#### Request Parameters
{:.api .api-request .api-request-params}

Parameter   | Description                                                | Param Type | DataType  | Required |
----------- | ---------------------------------------------------------- | ---------- | --------- | -------- |
stateToken  | [state token](#state-token) for current transaction        | Body       | String    | TRUE     |
oldPassword | User's current password that is expired or about to expire | Body       | String    | TRUE     |
newPassword | New password for user                                      | Body       | String    | TRUE     |

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the `oldPassword` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000014",
  "errorSummary": "Update of credentials failed",
  "errorLink": "E0000014",
  "errorId": "oaeYx8fd_-VQdONMI5OYcqoqw",
  "errorCauses": [
    {
      "errorSummary": "oldPassword: The credentials provided were incorrect."
    }
  ]
}
~~~

If the `newPassword` does not meet password policy requirements, you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000014",
  "errorSummary": "The password does meet the complexity requirements of the current password policy.",
  "errorLink": "E0000014",
  "errorId": "oaeuNNAquYEQkWFnUVG86Abbw",
  "errorCauses": [
    {
      "errorSummary": "Passwords must have at least 8 characters, a lowercase letter, an uppercase letter, a number, no parts of your username"
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "oldPassword": "correcthorsebatterystaple",
  "newPassword": "Ch-ch-ch-ch-Changes!"
}' "https://${org}.okta.com/api/v1/authn/credentials/change_password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

## Multifactor Authentication Operations

You can enroll, activate, and verify factors using the `/api/v1/authn/factors` endpoint.

> You can also enroll, manage, and verify factors outside the authentication context with [`/api/v1/users/:uid/factors/`](factors.html#factor-verification-operations).

### Enroll Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors %}

Enrolls a user with a [factor](factors.html#supported-factors-for-providers) assigned by their **MFA Policy**.

- [Enroll Okta Security Question Factor](#enroll-okta-security-question-factor)
- [Enroll Okta SMS Factor](#enroll-okta-sms-factor)
- [Enroll Okta Call Factor](#enroll-okta-call-factor)
- [Enroll Okta Verify TOTP Factor](#enroll-okta-verify-totp-factor)
- [Enroll Okta Verify Push Factor](#enroll-okta-verify-push-factor)
- [Enroll Google Authenticator Factor](#enroll-google-authenticator-factor)
- [Enroll RSA SecurID Factor](#enroll-rsa-securid-factor)
- [Enroll Symantec VIP Factor](#enroll-symantec-vip-factor)
- [Enroll YubiKey Factor](#enroll-yubikey-factor)
- [Enroll Duo Factor](#enroll-duo-factor)
- [Enroll U2F Factor](#enroll-u2f-factor)

> This operation is only available for users that have not previously enrolled a factor and have transitioned to the `MFA_ENROLL` [state](#transaction-state).

#### Request Parameters
{:.api .api-request .api-request-params}

Parameter   | Description                                                                   | Param Type  | DataType                                                     | Required |
----------- | ----------------------------------------------------------------------------- | ----------- | -------------------------------------------------------------| -------- |
stateToken  | [state token](#state-token) for current transaction                           | Body        | String                                                       | TRUE     |
factorType  | type of factor                                                                | Body        | [Factor Type](factors.html#factor-type)                      | TRUE     |
provider    | factor provider                                                               | Body        | [Provider Type](factors.html#provider-type)                  | TRUE     |
profile     | profile of a [supported factor](factors.html#supported-factors-for-providers) | Body        | [Factor Profile Object](factors.html#factor-profile-object)  | TRUE     |

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

> Some [factor types](factors.html#factor-type) require [activation](#activate-factor) to complete the enrollment process.  The [authentication transaction](#transaction-state) will transition to `MFA_ENROLL_ACTIVATE` if a factor requires activation.

#### Enroll Okta Security Question Factor
{:.api .api-operation}

Enrolls a user with the Okta `question` factor and [question profile](factors.html#question-profile).

> Security Question factor does not require activation and is `ACTIVE` after enrollment

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "question",
  "provider": "OKTA",
  "profile": {
    "question": "disliked_food",
    "answer": "mayonnaise"
  }
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~


##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00OhZsSfoCtbJTrU2XkwntfEl-jCj6ck6qcU_kA049",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~


#### Enroll Okta SMS Factor
{:.api .api-operation}

Enrolls a user with the Okta `sms` factor and an [SMS profile](factors.html#sms-profile).  A text message with an OTP is sent to the device during enrollment and must be [activated](#activate-sms-factor) by following the `next` link relation to complete the enrollment process.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "sms",
  "provider": "OKTA",
  "profile": {
    "phoneNumber": "+1-555-415-1337"
  }
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

> Use the `resend` link to send another OTP if user doesn't receive the original activation SMS OTP.

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "mbl198rKSEWOSKRIVIFT",
      "factorType": "sms",
      "provider": "OKTA",
      "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/mbl198rKSEWOSKRIVIFT/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "sms",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/mbl198rKSEWOSKRIVIFT/lifecycle/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~


#### Enroll Okta Call Factor
{:.api .api-operation}

Enrolls a user with the Okta `call` factor and a [Call profile](factors.html#call-profile).  A voice call with an OTP is sent to the device during enrollment and must be [activated](#activate-call-factor) by following the `next` link relation to complete the enrollment process.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "call",
  "provider": "OKTA",
  "profile": {
    "phoneNumber": "+1-555-415-1337"
  }
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

> Use the `resend` link to send another OTP if user doesn't receive the original activation Voice Call OTP.

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "clf198rKSEWOSKRIVIFT",
      "factorType": "call",
      "provider": "OKTA",
      "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/clf198rKSEWOSKRIVIFT/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "call",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/clf198rKSEWOSKRIVIFT/lifecycle/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~

#### Enroll Okta Verify TOTP Factor
{:.api .api-operation}

Enrolls a user with the Okta `token:software:totp` factor.  The factor must be [activated](#activate-totp-factor) after enrollment by following the `next` link relation to complete the enrollment process.

> This API implements [the TOTP standard](https://tools.ietf.org/html/rfc6238), which is used by apps like Okta Verify and Google Authenticator.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "token:software:totp",
  "provider": "OKTA"
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "ostf2xjtDKWFPZIKYDZV",
      "factorType": "token:software:totp",
      "provider": "OKTA",
      "profile": {
        "credentialId": "dade.murphy@example.com"
      },
      "_embedded": {
        "activation": {
          "timeStep": 30,
          "sharedSecret": "KBMTM32UJZSXQ2DW",
          "encoding": "base32",
          "keyLength": 6,
          "_links": {
            "qrcode": {
              "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/ostf2xjtDKWFPZIKYDZV/qr/00Mb0zqhJQohwCDkB2wOifajAsAosEAXvDwuCmsAZs",
              "type": "image/png"
            }
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/ostf2xjtDKWFPZIKYDZV/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Enroll Okta Verify Push Factor
{:.api .api-operation}

Enrolls a user with the Okta verify `push` factor. The factor must be [activated on the device](#activate-push-factor) by scanning the QR code or visiting the activation link sent via email or sms.

Use the published activation links to embed the QR code or distribute an activation `email` or `sms`.

> This API implements [the TOTP standard](https://tools.ietf.org/html/rfc6238), which is used by apps like Okta Verify and Google Authenticator.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "push",
  "provider": "OKTA"
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "WAITING",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "_embedded": {
        "activation": {
          "expiresAt": "2015-11-03T10:15:57.000Z",
          "_links": {
            "qrcode": {
              "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/qr/00fukNElRS_Tz6k-CFhg3pH4KO2dj2guhmaapXWbc4",
              "type": "image/png"
            },
            "send": [
              {
                "name": "email",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              },
              {
                "name": "sms",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              }
            ]
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "poll",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/poll",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Enroll Google Authenticator Factor
{:.api .api-operation}

Enrolls a user with the Google `token:software:totp` factor.  The factor must be [activated](#activate-totp-factor) after enrollment by following the `next` link relation to complete the enrollment process.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "token:software:totp",
  "provider": "GOOGLE"
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "ostf2xjtDKWFPZIKYDZV",
      "factorType": "token:software:totp",
      "provider": "GOOGLE",
      "profile": {
        "credentialId": "dade.murphy@example.com"
      },
      "_embedded": {
        "activation": {
          "timeStep": 30,
          "sharedSecret": "KYCRM33UJZSXQ2DW",
          "encoding": "base32",
          "keyLength": 6,
          "_links": {
            "qrcode": {
              "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/uftm3iHSGFQXHCUSDAND/qr/00Mb0zqhJQohwCDkB2wOifajAsAosEAXvDwuCmsAZs",
              "type": "image/png"
            }
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/uftm3iHSGFQXHCUSDAND/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Enroll RSA SecurID Factor
{:.api .api-operation}

Enrolls a user with a RSA SecurID factor and a [token profile](factors.html#token-profile).  RSA tokens must be verified with the [current pin+passcode](factors.html#factor-verification-object) as part of the enrollment request.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "token",
  "provider": "RSA",
  "profile": {
    "credentialId": "dade.murphy@example.com"
  },
  "passCode": "5275875498"
}' "https://${org}.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00OhZsSfoCtbJTrU2XkwntfEl-jCj6ck6qcU_kA049",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Enroll Symantec VIP Factor
{:.api .api-operation}

Enrolls a user with a Symantec VIP factor and a [token profile](factors.html#token-profile).  Symantec tokens must be verified with the [current and next passcodes](factors.html#factor-verification-object) as part of the enrollment request.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "factorType": "token",
  "provider": "SYMANTEC",
  "profile": {
    "credentialId": "VSMT14393584"
  },
  "passCode": "875498",
  "nextPassCode": "678195"
}' "https://${org}.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00OhZsSfoCtbJTrU2XkwntfEl-jCj6ck6qcU_kA049",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Enroll YubiKey Factor
{:.api .api-operation}

Enrolls a user with a Yubico factor (YubiKey).  YubiKeys must be verified with the [current passcode](factors.html#factor-verification-object) as part of the enrollment request.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "factorType": "token:hardware",
  "provider": "YUBICO",
  "passCode": "cccccceukngdfgkukfctkcvfidnetljjiknckkcjulji"
}' "https://${org}.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00OhZsSfoCtbJTrU2XkwntfEl-jCj6ck6qcU_kA049",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~


#### Enroll Duo Factor
{:.api .api-operation}

 The enrollment process starts with enrollment [request to Okta](#request-example-10), then continues with the Duo widget that is embedded in the page. The page needs to create an iframe with name "duo_iframe" as described [here](https://duo.com/docs/duoweb#3.-show-the-iframe) to host the widget. The script address is recieved in the response object in \_embedded.factor.\_embedded.\_links.script object. The information to initialize the Duo object is taken from \_embedded.factor.\_embedded.activation object as it is shown in the [full example](#full-page-example-for-duo-enrollment). In order to mainian the link between Duo and Okta the stateToken must be passed back when Duo calls the callback. This is done by populating the hidden element in the "duo_form" as it is decribed [here](https://duo.com/docs/duoweb#passing-additional-post-arguments-with-the-signed-response). After Duo enrollment and verification is done, the Duo script makes a call back to Okta. To complete the authentication process, make a call using [the poll link](#activation-poll-request-example) to get session token and verify sucessful state.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "factorType": "web",
  "provider": "DUO",
  "stateToken": "$(stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken":"00BlN4kOtm7wNxuM8nuXsOK1PFXBkvvTH-buJUrgWX",
  "expiresAt":"2016-07-13T13:14:52.000Z",
  "status":"MFA_ENROLL_ACTIVATE",
  "factorResult":"WAITING",
  "_embedded":{
      "user":{
          "id":"00uku2SnbUX49SAGb0g3",
          "passwordChanged":"2016-07-13T13:07:51.000Z",
          "profile":{
              "login":"first.last@gexample.com",
              "firstName":"First",
              "lastName":"Last",
              "locale":"en_US",
              "timeZone":"America/Los_Angeles"
          }
      },
      "factor":{
          "id":"dsflnpo99zpfMyaij0g3",
          "factorType":"web",
          "provider":"DUO",
          "vendorName":"DUO",
          "profile":{
              "credentialId":"first.last@gexample.com"
          },
          "_embedded":{
              "activation":{
                  "host":"api-your-host.duosecurity.com",
                  "signature":"TX|...your-signature",
                  "factorResult":"WAITING",
                  "_links":{
                      "complete":{
                          "href":"https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/lifecycle/duoCallback",
                          "hints":{
                              "allow":[
                                  "POST"
                             ]
                          }
                      },
                      "script":{
                          "href":"https://your-domain.okta.com/js/sections/duo/Duo-Web-v2.js",
                          "type":"text/javascript; charset=utf-8"
                      }
                  }
              }
          }
      }
  },
  "_links":{
      "next":{
          "name":"poll",
          "href":"https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/lifecycle/activate/poll",
          "hints":{
              "allow":[
                   "POST"
              ]
          }
      },
      "cancel":{
          "href":"https://your-domain.okta.com/api/v1/authn/cancel",
          "hints":{
              "allow":[
                  "POST"
              ]
          }
      },
      "prev":{
          "href":"https://your-domain.okta.com/api/v1/authn/previous",
          "hints":{
              "allow":[
                  "POST"
              ]
          }
      }
  }
}
~~~

#### Full Page Example for Duo enrollment
In this example we will put all the elements together in html page.

~~~html
<html>
    <body>
        <!--
            The Duo SDK will automatically bind to this iFrame and populate it for us.
            See https://www.duosecurity.com/docs/duoweb for more info.
         -->
        <iframe id="duo_iframe" width="620" height="330" frameborder="0"></iframe>
        <!--
            The Duo SDK will automatically bind to this form and submit it for us.
            See https://www.duosecurity.com/docs/duoweb for more info.
         -->
        <form method="POST" id="duo_form">
            <!-- The state token is required here (in order to bind anonymous request back into Auth API) -->
            <input type="hidden" name="stateToken" value='00BlN4kOtm7wNxuM8nuXsOK1PFXBkvvTH-buJUrgWX' />
        </form>

        <script src="https://your-domain.okta.com/js/sections/duo/Duo-Web-v2.js"></script>

        <!-- The host, sig_request, and post_action values will be given via the Auth API -->
        <script>
            Duo.init({
                'host': 'api-your-host.duosecurity.com',
                'sig_request': 'TX|...your-signature',
                'post_action': 'https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/lifecycle/duoCallback'
            });
        </script>
    </body>
</html>
~~~

##### Activation Poll Request Example
{:.api .api-response .api-request-example}
The poll is to verify successful authentication and to obtain session token.

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "$(stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors/${factorId}/lifecycle/activate/poll"
~~~

##### Activation Poll Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt":"2016-07-13T13:37:42.000Z",
  "status":"SUCCESS",
  "sessionToken":"20111zMXPaEe_lEw7pg2Ub810HDkpBwzSVBEPBRpA87LH5sW3Jj35_x",
  "_embedded":{
    "user":{
        "id":"00ukv3jVTgRjDctlX0g3",
        "passwordChanged":"2016-07-13T13:29:58.000Z",
        "profile":{
            "login":"first.last@example.com",
            "firstName":"First",
            "lastName":"Last",
            "locale":"en_US",
            "timeZone":"America/Los_Angeles"
        }
    }
  }
}
~~~

#### Enroll U2F Factor
{:.api .api-operation}

> Enrolling a U2F factor is an {% api_lifecycle ea %} feature.

Enrolls a user with a U2F factor.  The enrollment process starts with getting an `appId` and `nonce` from Okta and using those to get registration information from the U2F key using the U2F javascript API.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "factorType": "u2f",
  "provider": "FIDO",
  "stateToken": "$(stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00s7Yewe3Z4aujPLpR4qW4y1hMKzAbyXK5LSKJRW2G",
  "expiresAt": "2016-12-05T19:40:53.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "_embedded": {
    "user": {
      "id": "00ukv3jVTgRjDctlX0g3",
      "passwordChanged": "2015-10-28T23:27:57.000Z",
      "profile": {
        "login": "first.last@gmail.com",
        "firstName": "First",
        "lastName": "Last",
        "locale": "en",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "fuf8y1y14jaygfX5K0h7",
      "factorType": "u2f",
      "provider": "FIDO",
      "vendorName": "FIDO",
      "_embedded": {
        "activation": {
          "version": "U2F_V2",
          "appId": "https://your-domain.okta.com",
          "nonce": "s-NaltFnye-xNsJeAhnN",
          "timeoutSeconds": 20
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/fuf8y1y14jaygfX5K0h7/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Activate Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/lifecycle/activate %}

The `sms`,`call` and `token:software:totp` [factor types](factors.html#factor-type) require activation to complete the enrollment process.

- [Activate TOTP Factor](#activate-totp-factor)
- [Activate SMS Factor](#activate-sms-factor)
- [Activate Call Factor](#activate-call-factor)
- [Activate Push Factor](#activate-push-factor)
- [Activate U2F Factor] (#activate-u2f-factor) (EA feature)

#### Activate TOTP Factor
{:.api .api-operation}

Activates a `token:software:totp` factor by verifying the OTP.

> This API implements [the TOTP standard](https://tools.ietf.org/html/rfc6238), which is used by apps like Okta Verify and Google Authenticator.

#### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                          | Param Type | DataType | Required |
------------ | ---------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment              | URL        | String   | TRUE     |
stateToken   | [state token](#state-token)  for current transaction | Body       | String   | TRUE     |
passCode     | OTP generated by device                              | Body       | String   | TRUE     |

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the passcode is invalid you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your passcode doesn't match our records. Please try again."
    }
  ]
}
~~~

#### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "123456"
}' "https://${org}.okta.com/api/v1/authn/factors/ostf1fmaMGJLMNGNLIVG/lifecycle/activate"
~~~

#### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Activate SMS Factor
{:.api .api-operation}

Activates a `sms` factor by verifying the OTP.  The request and response is identical to [activating a TOTP factor](#activate-totp-factor)

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
passCode     | OTP sent to mobile device                           | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the passcode is invalid you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your passcode doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "123456"
}' "https://${org}.okta.com/api/v1/authn/factors/sms1o51EADOTFXHHBXBP/lifecycle/activate"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}

~~~

#### Activate Call Factor
{:.api .api-operation}

Activates a `call` factor by verifying the OTP.  The request and response is identical to [activating a TOTP factor](#activate-totp-factor)

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
passCode     | Passcode received via the voice call                | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the passcode is invalid you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your passcode doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "12345"
}' "https://${org}.okta.com/api/v1/authn/factors/clf1o51EADOTFXHHBXBP/lifecycle/activate"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}

~~~

#### Activate Push Factor
{:.api .api-operation}

Activation of `push` factors are asynchronous and must be polled for completion when the `factorResult` returns a `WAITING` status.

Activations have a short lifetime (minutes) and will `TIMEOUT` if they are not completed before the `expireAt` timestamp.  Use the published `activate` link to restart the activation process if the activation is expired.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
}' "https://${org}.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate"
~~~

##### Response Example (Waiting)
{:.api .api-response .api-response-example}

> Follow the the published `next` link to keep polling for activation completion

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "WAITING",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "_embedded": {
        "activation": {
          "expiresAt": "2015-11-03T10:15:57.000Z",
          "_links": {
            "qrcode": {
              "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/qr/00fukNElRS_Tz6k-CFhg3pH4KO2dj2guhmaapXWbc4",
              "type": "image/png"
            },
            "send": [
              {
                "name": "email",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              },
              {
                "name": "sms",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              }
            ]
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "poll",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/poll",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Response Example (Success)
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

##### Response Example (Timeout)
{:.api .api-response .api-response-example}

> Follow the the published `activate` link to restart the activation process

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "TIMEOUT",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "_embedded": {
        "activation": {
          "factorResult": "TIMEOUT",
          "_links": {
            "activate": {
              "href": "https://your-domain.okta.com/api/v1/users/00u4vi0VX6U816Kl90g4/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate",
              "hints": {
                "allow": [
                  "POST"
                ]
              }
            },
            "send": [
              {
                "name": "email",
                "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              },
              {
                "name": "sms",
                "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              }
            ]
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Poll for Push Factor Activation

After the push notification is sent to user's device we need to know when the user completes the activation. This is done by polling the "poll" link.

###### Poll Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb"
}' "https://${org}.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/poll"
~~~

###### Poll Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken":"007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt":"2016-12-22T21:36:47.000Z",
  "status":"MFA_ENROLL_ACTIVATE",
  "factorResult":"WAITING",
  "_embedded":{
    "user":{
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2016-12-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor":{
      "id":"opfh52xcuft3J4uZc0g3",
      "factorType":"push",
      "provider":"OKTA",
      "vendorName":"OKTA",
      "_embedded":{
        "activation":{
          "expiresAt":"2016-12-22T21:41:47.000Z",
          "factorResult":"WAITING",
          "_links":{
            "send":[
              {
                "name":"email",
                "href":"http://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email",
                "hints":{
                  "allow":["POST"]
                }
              },
              {
                "name":"sms",
                "href":"http://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms",
                "hints":{
                  "allow":["POST"]
                }
              }
            ],
            "qrcode":{
              "href":"http://your-domain.okta.com/api/v1/users/opfh52xcuft3J4uZc0g3/factors/opfn169oIx3k63Klh0g3/qr/20111huUFWDFTAeq_lFQKfKFS_rLABkE_pKgGl5PBUeLvJVmaIrWq5u",
              "type":"image/png"
            }
          }
        }
      }
    }
  },
  "_links":{
    "next":{
      "name":"poll",
      "href":"http://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/poll",
      "hints":{
        "allow":["POST"]
      }
    },
    "cancel":{
      "href":"http://your-domain.okta.com/api/v1/authn/cancel",
      "hints":{
        "allow":["POST"]
      }
    },
    "prev":{
      "href":"http://your-domain.okta.com/api/v1/authn/previous",
      "hints":{
        "allow":["POST"]
      }
    }
  }
}
~~~

##### Send Activation Links

Sends an activation email or SMS when when the user is unable to scan the QR code provided as part of an Okta Verify transaction.
If for any reason the user can't scan the QR code, they can use the link provided in email or SMS to complete the transaction.

> The user must click the link from the same device as the one where the Okta Verify app is installed.

###### Request Example (Email)
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb"
}' "https://${org}.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email"
~~~

###### Request Example (SMS)
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "profile": {
    "phoneNumber": "+1-555-415-1337"
  }
}' "https://${org}.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms"
~~~


###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL_ACTIVATE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "WAITING",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "_embedded": {
        "activation": {
          "expiresAt": "2015-11-03T10:15:57.000Z",
          "_links": {
            "qrcode": {
              "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/qr/00fukNElRS_Tz6k-CFhg3pH4KO2dj2guhmaapXWbc4",
              "type": "image/png"
            },
            "send": [
              {
                "name": "email",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/email",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              },
              {
                "name": "sms",
                "href": "https://your-domain.okta.com/api/v1/users/00ub0oNGTSWTBKOLGLNR/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/sms",
                "hints": {
                  "allow": [
                    "POST"
                  ]
                }
              }
            ]
          }
        }
      }
    }
  },
  "_links": {
    "next": {
      "name": "poll",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/lifecycle/activate/poll",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Activate U2F Factor
{:.api .api-operation}

> Activating a U2F factor is an {% api_lifecycle ea %} feature.

Activation gets the registration information from the U2F token using the platform APIs and passes it to Okta.

##### Get registration information from U2F token by calling the U2F Javascript call
{:.api .api-response .api-response-example}

~~~html
//Get the u2f-api.js from https://github.com/google/u2f-ref-code/tree/master/u2f-gae-demo/war/js
<script src="/u2f-api.js"></script>
<script>
//use the appId from the activation object
var appId = activation.appId;
var registerRequests = [{
    version: activation.version, //use the version from the activation object
    challenge: activation.nonce //use the nonce from the activation object
    }];
u2f.register(appId, registerRequests, [], function (data) {
  if (data.errorCode && data.errorCode !== 0) {
    //Error from U2F platform
  } else {
	  //Get the registrationData from the callback result
	  var registrationData = data.registrationData;

	  //Get the clientData from the callback result
	  var clientData = data.clientData;
  }
});
</script>
~~~

Activate a `u2f` factor by verifying the registration data and client data.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter        | Description                                               | Param Type | DataType | Required |
-----------------| ----------------------------------------------------------| ---------- | -------- | -------- |
fid              | `id` of factor returned from enrollment                   | URL        | String   | TRUE     |
stateToken       | [state token](#state-token) for current transaction       | Body       | String   | TRUE     |
registrationData | base64 encoded registration data from U2F javascript call | Body       | String   | TRUE     |
clientData       | base64 encoded client data from U2F javascript call       | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the registration nonce is invalid or if registration data is invalid, you will receive a `403 Forbidden` status code with the following error:

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your passcode doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "registrationData": "BQTl3Iu9V4caCvcI44pmYwIehICWyboL_J2Wl5FA6ZGNx9qT11Df-rHJIy9iP6MSJ_qAaKqdq8O0XVqBG46p6qbpQLIb471thYthrQiW9955tNdORCEhvZX9iYNI1peNlETOr7Qx_PgIZ6Ein6aB3wH9JCTGgsdd4JX3cYixbj1v9W8wggJEMIIBLqADAgECAgRVYr6gMAsGCSqGSIb3DQEBCzAuMSwwKgYDVQQDEyNZdWJpY28gVTJGIFJvb3QgQ0EgU2VyaWFsIDQ1NzIwMDYzMTAgFw0xNDA4MDEwMDAwMDBaGA8yMDUwMDkwNDAwMDAwMFowKjEoMCYGA1UEAwwfWXViaWNvIFUyRiBFRSBTZXJpYWwgMTQzMjUzNDY4ODBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABEszH3c9gUS5mVy-RYVRfhdYOqR2I2lcvoWsSCyAGfLJuUZ64EWw5m8TGy6jJDyR_aYC4xjz_F2NKnq65yvRQwmjOzA5MCIGCSsGAQQBgsQKAgQVMS4zLjYuMS40LjEuNDE0ODIuMS41MBMGCysGAQQBguUcAgEBBAQDAgUgMAsGCSqGSIb3DQEBCwOCAQEArBbZs262s6m3bXWUs09Z9Pc-28n96yk162tFHKv0HSXT5xYU10cmBMpypXjjI-23YARoXwXn0bm-BdtulED6xc_JMqbK-uhSmXcu2wJ4ICA81BQdPutvaizpnjlXgDJjq6uNbsSAp98IStLLp7fW13yUw-vAsWb5YFfK9f46Yx6iakM3YqNvvs9M9EUJYl_VrxBJqnyLx2iaZlnpr13o8NcsKIJRdMUOBqt_ageQg3ttsyq_3LyoNcu7CQ7x8NmeCGm_6eVnZMQjDmwFdymwEN4OxfnM5MkcKCYhjqgIGruWkVHsFnJa8qjZXneVvKoiepuUQyDEJ2GcqvhU2YKY1zBGAiEAxWDh5F7vr0AoEsi3N-uR6KR3ADXlZnQgzROUTVhff8ICIQCiUUG1FkQ9e8PW1dhRk6tjHjL22KZ9JqBrTfpytC5jaQ==",
  "clientData": "eyAiY2hhbGxlbmdlIjogImFYLS1wMTlibldWcUlnY25HU0hLIiwgIm9yaWdpbiI6ICJodHRwczpcL1wvc25hZ2FuZGxhLm9rdGFwcmV2aWV3LmNvbSIsICJ0eXAiOiAibmF2aWdhdG9yLmlkLmZpbmlzaEVucm9sbG1lbnQiIH0=",
  "stateToken": "00MBkDX0vBddsuU1VnDsa7-qqIOi7g51YLNQEen1hi"
}' "https://${org}.okta.com/api/v1/authn/factors/fuf1o51EADOTFXHHBXBP/lifecycle/activate"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9pe",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

### Verify Factor

Verifies an enrolled factor for an authentication transaction with the `MFA_REQUIRED` or `MFA_CHALLENGE` [state](#transaction-state)

- [Verify Security Question Factor](#verify-security-question-factor)
- [Verify SMS Factor](#verify-sms-factor)
- [Verify TOTP Factor](#verify-totp-factor)
- [Verify Push Factor](#verify-push-factor)
- [Verify Duo Factor](#verify-duo-factor)
- [Verify U2F Factor](#verify-u2f-factor)
- [Verify Call Factor](#verify-call-factor)

#### Verify Security Question Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

Verifies an answer to a `question` factor.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
answer       | answer to security question                         | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the `answer` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your answer doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "answer": "mayonnaise"
}' "https://${org}.okta.com/api/v1/authn/factors/ufs1pe3ISGKGPYKXRBKK/verify"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00ZD3Z7ixppspFljXV2t_Z6GfrYzqG7cDJ8reWo2hy",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Verify SMS Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor                                      | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
passCode     | OTP sent to device                                  | Body       | String   | FALSE    |

> If you omit `passCode` in the request a new OTP will be sent to the device, otherwise the request will attempt to verify the `passCode`

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the `passCode` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your answer doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Send SMS Challenge (OTP)

Omit `passCode` in the request to sent an OTP to the device.

###### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb"
}' "https://${org}.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify"
~~~

###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "sms193zUBEROPBNZKPPE",
      "factorType": "sms",
      "provider": "OKTA",
      "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
      }
    }
  },
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "sms",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~

##### Verify SMS Challenge (OTP)

Specify `passCode` in the request to verify the factor.

###### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "657866"
}' "https://${org}.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify"
~~~

###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00t6IUQiVbWpMLgtmwSjMFzqykb5QcaBNtveiWlGeM",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Verify TOTP Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

Verifies an OTP for a `token:software:totp` factor.

> This API implements [the TOTP standard](https://tools.ietf.org/html/rfc6238), which is used by apps like Okta Verify and Google Authenticator.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor                                      | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
passCode     | OTP sent to device                                  | Body       | String   | FALSE    |

##### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the passcode is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your passcode doesn't match our records. Please try again."
    }
  ]
}
~~~

###### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "657866"
}' "https://${org}.okta.com/api/v1/authn/factors/ostfm3hPNYSOIOIVTQWY/verify"
~~~

###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00t6IUQiVbWpMLgtmwSjMFzqykb5QcaBNtveiWlGeM",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

#### Verify Push Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

Sends an asynchronous push notification (challenge) to the device for the user to approve or reject.  The `factorResult` for the transaction will have a result of `WAITING`, `SUCCESS`, `REJECTED`, or `TIMEOUT`.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |


##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb"
}' "https://${org}.okta.com/api/v1/authn/factors/ufs1pe3ISGKGPYKXRBKK/verify"
~~~

##### Response Example (Waiting)
{:.api .api-response .api-response-example}

> Keep polling authentication transactions with `WAITING` result until the challenge completes or expires.

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "WAITING",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "profile": {
        "deviceType": "SmartPhone_IPhone",
        "name": "Gibson",
        "platform": "IOS",
        "version": "9.0"
      }
    }
  },
  "_links": {
    "next": {
      "name": "poll",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "push",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~

##### Response Example (Approved)
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00Fpzf4en68pCXTsMjcX8JPMctzN2Wiw4LDOBL_9xx",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

##### Response Example (Rejected)
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "REJECTED",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "profile": {
        "deviceType": "SmartPhone_IPhone",
        "name": "Gibson",
        "platform": "IOS",
        "version": "9.0"
      }
    }
  },
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "push",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~

##### Response Example (Timeout)
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorResult": "WAITING",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": {
      "id": "opfh52xcuft3J4uZc0g3",
      "factorType": "push",
      "provider": "OKTA",
      "profile": {
        "deviceType": "SmartPhone_IPhone",
        "name": "Gibson",
        "platform": "IOS",
        "version": "9.0"
      }
    }
  },
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": [
      {
        "name": "push",
        "href": "https://your-domain.okta.com/api/v1/authn/factors/opfh52xcuft3J4uZc0g3/verify/resend",
        "hints": {
          "allow": [
            "POST"
          ]
        }
      }
    ]
  }
}
~~~

#### Verify Duo Factor
{:.api .api-operation}

Verification of the Duo factor is implemented as an integration with Duo widget. The process is very similar to the  [enrollment](#full-page-example-for-duo-enrollment) where the widget is embedded in an iframe - "duo_iframe". Verification starts with request to the Okta API, then continues with a Duo widget that handles the actual verification. We need to pass the state token as hidden object in "duo_form". The authentication completes with call to [poll link](#verification-poll-request-example) to verify the state and obtain session token.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "${stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors/${factorId]/verify"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
    "stateToken":"00CzoxFVe4R2nv0hTxm32r1kayfrrOkuxcE2rfINwZ",
    "expiresAt":"2016-07-13T14:13:58.000Z",
    "status":"MFA_CHALLENGE",
    "factorResult":"WAITING",
    "_embedded":{
        "user":{
            "id":"00ukv3jVTgRjDctlX0g3",
            "passwordChanged":"2016-07-13T13:29:58.000Z",
            "profile":{
                "login":"first.last@gexample.com",
                "firstName":"First",
                "lastName":"Last",
                "locale":"en_US",
                "timeZone":"America/Los_Angeles"
            }
        },
        "factor":{
            "id":"dsflnpo99zpfMyaij0g3",
            "factorType":"web",
            "provider":"DUO",
            "vendorName":"DUO",
            "profile":{
                "credentialId":"first.last@example.com"
            },
            "_embedded":{
                "verification":{
                    "host":"api-your-host.duosecurity.com",
                    "signature":"TX|...your-signature",
                    "factorResult":"WAITING",
                    "_links":{
                        "complete":{
                            "href":"https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/lifecycle/duoCallback",
                            "hints":{
                                "allow":[
                                    "POST"
                                ]
                            }
                        },
                        "script":{
                            "href":"https://your-domain.okta.com/js/sections/duo/Duo-Web-v2.js",
                            "type":"text/javascript; charset=utf-8"
                        }
                    }
                }
            }
        },
        "policy":{
            "allowRememberDevice":true,
            "rememberDeviceLifetimeInMinutes":15,
            "rememberDeviceByDefault":false
        }
    },
    "_links":{
        "next":{
            "name":"poll",
            "href":"https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/verify",
            "hints":{
                "allow":[
                    "POST"
                ]
            }
        },
        "cancel":{
            "href":"https://your-domain.okta.com/api/v1/authn/cancel",
            "hints":{
                "allow":[
                    "POST"
                ]
            }
        },
        "prev":{
            "href":"https://your-domain.okta.com/api/v1/authn/previous",
            "hints":{
                "allow":[
                    "POST"
                ]
            }
        }
    }
}
~~~

##### Sample for Duo iFrame
{:.api .api-response .api-response-example}

~~~html
...
<!--
    The Duo SDK will automatically bind to this iFrame and populate it for us.
    See https://www.duosecurity.com/docs/duoweb for more info.
 -->
<iframe id="duo_iframe" width="620" height="330" frameborder="0"></iframe>
<!--
    The Duo SDK will automatically bind to this form and submit it for us.
    See https://www.duosecurity.com/docs/duoweb for more info.
 -->
<form method="POST" id="duo_form">
    <!-- The state token is required here (in order to bind anonymous request back into Auth API) -->
    <input type="hidden" name="stateToken" value='00CzoxFVe4R2nv0hTxm32r1kayfrrOkuxcE2rfINwZ' />
</form>

<script src="https://your-domain.okta.com/js/sections/duo/Duo-Web-v2.js"></script>

<!-- The host, sig_request, and post_action values will be given via the Auth API -->
<script>
    Duo.init({
        'host': 'api-your-host.duosecurity.com',
        'sig_request': 'TX|...your-signature',
        'post_action': 'https://your-domain.okta.com/api/v1/authn/factors/dsflnpo99zpfMyaij0g3/lifecycle/duoCallback'
    });
</script>
...

~~~

##### Verification Poll Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "${stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors/${factorId]/verify"
~~~

##### Verification Poll Response Example
{:.api .api-response .api-response-example}

~~~json
{
    "expiresAt":"2016-07-13T14:14:44.000Z",
    "status":"SUCCESS",
    "sessionToken":"201111XUk7La2gw5r5PV1IhU4WSd0fV6mvNYdlJoeqjuyej7S83x3Hr",
    "_embedded":{
        "user":{
            "id":"00ukv3jVTgRjDctlX0g3",
            "passwordChanged":"2016-07-13T13:29:58.000Z",
            "profile":{
                "login":"first.last@example.com",
                "firstName":"First",
                "lastName":"Last",
                "locale":"en_US",
                "timeZone":"America/Los_Angeles"
            }
        }
    }
}
~~~

#### Verify U2F Factor
{:.api .api-operation}

> Verifying a U2F factor is [an {% api_lifecycle ea %} feature](/docs/api/getting_started/releases-at-okta.html).

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor returned from enrollment             | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
clientData   | base64 encoded client data from the U2F token       | Body       | String   | TRUE     |
signatureData| base64 encoded signature data from the U2F token    | Body       | String   | TRUE     |

##### Start verification to get challenge nonce

Verification of the U2F factor starts with getting the challenge nonce and U2F token details and then using the client side
javascript API to get the signed assertion from the U2F token.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "${stateToken}"
}' "https://${org}.okta.com/api/v1/authn/factors/${factorId]/verify"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
   "stateToken":"00wCfuPA3qX3azDawSdPGFIhHuzbZX72Gv4bu_ew9d",
   "expiresAt":"2016-12-06T01:32:39.000Z",
   "status":"MFA_CHALLENGE",
   "_embedded":{
      "user":{
         "id":"00u21eb3qyRDNNIKTGCW",
         "passwordChanged":"2015-10-28T23:27:57.000Z",
         "profile":{
            "login":"first.last@gmail.com",
            "firstName":"First",
            "lastName":"Last",
            "locale":"en",
            "timeZone":"America/Los_Angeles"
         }
      },
      "factor":{
         "id":"fuf8y2l4n5mfH0UWe0h7",
         "factorType":"u2f",
         "provider":"FIDO",
         "vendorName":"FIDO",
         "profile":{
            "credentialId":"shvjvW2Fi2GtCJb33nm0105EISG9lf2Jg0jWl42URM6vtDH8-AhnoSKfpoHfAf0kJMaCx13glfdxiLFuPW_1bw",
            "appId":"https://your-domain.okta.com",
            "version":"U2F_V2"
         },
         "_embedded":{
            "challenge":{
               "nonce":"tT1MI7XGzMu48Ivnz3vB",
               "timeoutSeconds":20
            }
         }
      },
      "policy":{
         "allowRememberDevice":true,
         "rememberDeviceLifetimeInMinutes":0,
         "rememberDeviceByDefault":false
      }
   },
   "_links":{
      "next":{
         "name":"verify",
         "href":"https://your-domain.okta.com/api/v1/authn/factors/fuf8y2l4n5mfH0UWe0h7/verify",
         "hints":{
            "allow":[
               "POST"
            ]
         }
      },
      "cancel":{
         "href":"https://your-domain.okta.com/api/v1/authn/cancel",
         "hints":{
            "allow":[
               "POST"
            ]
         }
      },
      "prev":{
         "href":"https://your-domain.okta.com/api/v1/authn/previous",
         "hints":{
            "allow":[
               "POST"
            ]
         }
      }
   }
}
~~~

##### Get the signed assertion from the U2F token
{:.api .api-response .api-response-example}

~~~html
//Get the u2f-api.js from https://github.com/google/u2f-ref-code/tree/master/u2f-gae-demo/war/js
<script src="/u2f-api.js"></script>
<script>
var challengeNonce = factor._embedded.challenge.nonce; //use the nonce from the challenge object
var appId = factor.profile.appId; //use the appId from factor profile object

//Use the version and credentialId from factor profile object
var registeredKeys = [{version: factor.profile.version, keyHandle: factor.profile.credentialId }];

//Call the U2F javascript API to get signed assertion from the U2F token
u2f.sign(appId, factorData.challenge.nonce, registeredKeys, function (data) {
  if (data.errorCode && data.errorCode !== 0) {
    //Error from U2F platform
  } else {
	  //Get the client data from callback result
	  var clientData = data.clientData;

    //Get the signature data from callback result
	  var signatureData = data.signatureData;
  }
}
</script>
~~~

##### Post the signed assertion to Okta to complete verification
{:.api .api-request .api-request-example}

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "${stateToken}",
  "clientData":"eyAiY2hhbGxlbmdlIjogIlJ6ZDhQbEJEWUEyQ0VsbXVGcHlMIiwgIm9yaWdpbiI6ICJodHRwczpcL1wvc25hZ2FuZGxhLm9rdGFwcmV2aWV3LmNvbSIsICJ0eXAiOiAibmF2aWdhdG9yLmlkLmdldEFzc2VydGlvbiIgfQ==",
  "signatureData":"AQAAAAEwRQIgRDEdmXr_jh1bEHtoUs1l7mMd-eUDO0eKqXKkrK5hUi0CIQDaVX030GgxVPr4RX3c4XgugildmHwDLwKRL0aMS3Sbpw==",
}' "https://${org}.okta.com/api/v1/authn/factors/${factorId]/verify"
~~~

##### Response of U2F verification Example
{:.api .api-response .api-response-example}

~~~json
{
    "expiresAt":"2016-07-13T14:14:44.000Z",
    "status":"SUCCESS",
    "sessionToken":"201111XUk7La2gw5r5PV1IhU4WSd0fV6mvNYdlJoeqjuyej7S83x3Hr",
    "_embedded":{
        "user":{
            "id":"00ukv3jVTgRjDctlX0g3",
            "passwordChanged":"2016-07-13T13:29:58.000Z",
            "profile":{
                "login":"first.last@example.com",
                "firstName":"First",
                "lastName":"Last",
                "locale":"en_US",
                "timeZone":"America/Los_Angeles"
            }
        }
    }
}
~~~


#### Verify Call Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/factors/*:fid*/verify %}

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
fid          | `id` of factor                                      | URL        | String   | TRUE     |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
passCode     | OTP sent to device                                  | Body       | String   | FALSE    |

> If you omit `passCode` in the request a new OTP will be sent to the device, otherwise the request will attempt to verify the `passCode`

#### Response Parameters
{:.api .api-response .api-response-params}

[Authentication Transaction Object](#authentication-transaction-model) with the current [state](#transaction-state) for the authentication transaction.

If the `passCode` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oaei_IfXcpnTHit_YEKGInpFw",
  "errorCauses": [
    {
      "errorSummary": "Your answer doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Send Voice Call Challenge (OTP)

Omit `passCode` in the request to sent an OTP to the device.

###### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb"
}' "https://${org}.okta.com/api/v1/authn/factors/clf193zUBEROPBNZKPPE/verify"
~~~

###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "clf193zUBEROPBNZKPPE",
      "factorType": "call",
      "provider": "OKTA",
      "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
      }
    }
  },
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/clf193zUBEROPBNZKPPE/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

##### Verify Call Challenge (OTP)

Specify `passCode` in the request to verify the factor.

###### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "passCode": "65786"
}' "https://${org}.okta.com/api/v1/authn/factors/clf193zUBEROPBNZKPPE/verify"
~~~

###### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00t6IUQiVbWpMLgtmwSjMFzqykb5QcaBNtveiWlGeM",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}

~~~

## Recovery Operations

### Forgot Password
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/password %}

Starts a new password recovery transaction for a given user and issues a [recovery token](#recovery-token) that can be used to reset a user's password.

- [Forgot Password with Email Factor](#forgot-password-with-email-factor)
- [Forgot Password with SMS Factor](#forgot-password-with-sms-factor)
- [Forgot Password with Call Factor](#forgot-password-with-call-factor)
- [Forgot Password with Trusted Application](#forgot-password-with-trusted-application)

> Self-service password reset (forgot password) must be permitted via the user's assigned password policy to use this operation.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter   | Description                                                                                                      | Param Type | DataType                          | Required | MaxLength |
----------- | ---------------------------------------------------------------------------------------------------------------- | ---------- | --------------------------------- | -------- | --------- |
username    | User's non-qualified short-name (e.g. dade.murphy) or unique fully-qualified login (e.g dade.murphy@example.com) | Body       | String                            | TRUE     |           |
factorType  | Recovery factor to use for primary authentication                                                                | Body       | `EMAIL` or `SMS` or `Voice Call`  | FALSE    |           |
relayState  | Optional state value that is persisted for the lifetime of the recovery transaction                              | Body       | String                            | FALSE    |   2048    |

> A valid `factorType` is required for requests without an API token with admin privileges. (See [Forgot Password with Trusted Application](#forgot-password-with-trusted-application))

##### Response Parameters
{:.api .api-response .api-response-params}

###### Public Application

[Recovery Transaction Object](#recovery-transaction-model) with `RECOVERY_CHALLENGE` status for the new recovery transaction.

> You will always receive a [Recovery Transaction](#recovery-transaction-model) response even if the requested `username` is not a valid identifier to prevent information disclosure.

###### Trusted Application

[Recovery Transaction Object](#recovery-transaction-model) with an issued `recoveryToken` that can be distributed to the end-user.

You will receive a `403 Forbidden` status code if the `username` requested is not valid

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000095",
    "errorSummary": "Recovery not allowed for unknown user.",
    "errorLink": "E0000095",
    "errorId": "oaetVarN2dKS6ap08dq0k4n7A",
    "errorCauses": []
}
~~~

#### Forgot Password with Email Factor

Starts a new password recovery transaction for the email factor:

* You must specify a user identifier (`username`) but no password in the request.
* If the request is successful, Okta sends a recovery email asynchronously to the user's primary and secondary email address with a [recovery token](#recovery-token) so the user can complete the transaction.

Primary authentication of a user's recovery credential (e.g `EMAIL` or `SMS`) has not completed when this request is sent; the user is pending validation.

Okta provides security in the following ways:

* Since the recovery email is distributed out-of-band and may be viewed on a different user agent or device, this operation does not return a [state token](#state-token) and does not have a `next` link.
* Okta doesn't publish additional metadata about the user until primary authentication has successfully completed.
See the Response Example in this section for details.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "factorType": "EMAIL",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "status": "RECOVERY_CHALLENGE",
  "factorResult": "WAITING",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorType": "EMAIL",
  "recoveryType": "PASSWORD"
}
~~~


#### Forgot Password with SMS Factor

Starts a new password recovery transaction with a user identifier (`username`) and asynchronously sends a SMS OTP (challenge) to the user's mobile phone.  This operation will transition the recovery transaction to the `RECOVERY_CHALLENGE` state and wait for the user to [verify the OTP](#verify-sms-recovery-factor).

> Primary authentication of a user's recovery credential (e.g email or SMS) has not yet completed.
> Okta will not publish additional metadata about the user until primary authentication has successfully completed.

> SMS recovery factor must be enabled via the user's assigned password policy to use this operation.

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "factorType": "SMS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorType": "SMS",
  "recoveryType": "PASSWORD",
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": {
      "name": "sms",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/resend",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Forgot Password with Call Factor

Starts a new password recovery transaction with a user identifier (`username`) and asynchronously sends a Voice Call with OTP (challenge) to the user's phone.  This operation will transition the recovery transaction to the `RECOVERY_CHALLENGE` state and wait for user to [verify the OTP](#verify-call-recovery-factor).

> Primary authentication of a user's recovery credential (e.g email or SMS or Voice Call) has not yet completed.
> Okta will not publish additional metadata about the user until primary authentication has successfully completed.

> Voice Call recovery factor must be enabled via the user's assigned password policy to use this operation.

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "factorType": "call",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorType": "call",
  "recoveryType": "PASSWORD",
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/CALL/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": {
      "name": "call",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/CALL/resend",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Forgot Password with Trusted Application

Allows a [trusted application](#trusted-application) such as an external portal to implement its own primary authentication process and directly obtain a [recovery token](#recovery-token) for a user given just the user's identifier.

> Directly obtaining a `recoveryToken` is a highly privileged operation that requires an administrator API token and should be restricted to trusted web applications.  Anyone that obtains a `recoveryToken` for a user and knows the answer to a user's recovery question can reset their password or unlock their account.

> The **public IP address** of your [trusted application](#trusted-application) must be [whitelisted as a gateway IP address](../getting_started/design_principles.html#ip-address) to forward the user agent's original IP address with the `X-Forwarded-For` HTTP header

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36" \
-H "X-Forwarded-For: 23.235.46.133" \
-d '{
  "username": "dade.murphy@example.com",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "recoveryToken": "VBQ0gwBp5LyJJFdbmWCM",
  "recoveryType": "PASSWORD",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  },
  "_links": {
    "next": {
      "name": "recovery",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/token",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Unlock Account
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/unlock %}

Starts a new unlock recovery transaction for a given user and issues a [recovery token](#recovery-token) that can be used to unlock a user's account.

- [Unlock Account with Email Factor](#unlock-account-with-email-factor)
- [Unlock Account with SMS Factor](#unlock-account-with-sms-factor)
- [Unlock Account with Trusted Application](#unlock-account-with-trusted-application)

> Self-service unlock must be permitted via the user's assigned password policy to use this operation.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter   | Description                                                                                                      | Param Type | DataType                          | Required | Max Length |
----------- | ---------------------------------------------------------------------------------------------------------------- | ---------- | --------------------------------- | -------- | ---------- |
username    | User's non-qualified short-name (e.g. dade.murphy) or unique fully-qualified login (e.g dade.murphy@example.com) | Body       | String                            | TRUE     |            |
factorType  | Recovery factor to use for primary authentication                                                                | Body       | `EMAIL` or `SMS`                  | FALSE    |            |
relayState  | Optional state value that is persisted for the lifetime of the recovery transaction                              | Body       | String                            | FALSE    |  2048      |

> A valid `factoryType` is required for requests without an API token with admin privileges. (See [Unlock Account with Trusted Application](#unlock-account-with-trusted-application))

##### Response Parameters
{:.api .api-response .api-response-params}

###### Public Application

[Recovery Transaction Object](#recovery-transaction-model) with `RECOVERY_CHALLENGE` status for the new recovery transaction.

> You will always receive a [Recovery Transaction](#recovery-transaction-model) response even if the requested `username` is not a valid identifier to prevent information disclosure.

###### Trusted Application

[Recovery Transaction Object](#recovery-transaction-model) with an issued `recoveryToken` that can be distributed to the end-user.

You will receive a `403 Forbidden` status code if the `username` requested is not valid

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000095",
    "errorSummary": "Recovery not allowed for unknown user.",
    "errorLink": "E0000095",
    "errorId": "oaetVarN2dKS6ap08dq0k4n7A",
    "errorCauses": []
}
~~~

#### Unlock Account with Email Factor

Starts a new unlock recovery transaction with a user identifier (`username`) and asynchronously sends a recovery email to the user's primary and secondary email address with a [recovery token](#recovery-token) that can be used to complete the transaction.

Since the recovery email is distributed out-of-band and may be viewed on a different user agent or device, this operation does not return a [state token](#state-token) and does not have a `next` link.

> Primary authentication of a user's recovery credential (e.g `EMAIL` or `SMS`) has not yet completed.
> Okta will not publish additional metadata about the user until primary authentication has successfully completed.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "factorType": "EMAIL",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/unlock"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "status": "RECOVERY_CHALLENGE",
  "factorResult": "WAITING",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorType": "EMAIL",
  "recoveryType": "UNLOCK"
}
~~~


#### Unlock Account with SMS Factor

Starts a new unlock recovery transaction with a user identifier (`username`) and asynchronously sends a SMS OTP (challenge) to the user's mobile phone.  This operation will transition the recovery transaction to the `RECOVERY_CHALLENGE` state and wait for user to [verify the OTP](#verify-sms-recovery-factor).

> Primary authentication of a user's recovery credential (e.g email or SMS) has not yet completed.
> Okta will not publish additional metadata about the user until primary authentication has successfully completed.

> SMS recovery factor must be enabled via the user's assigned password policy to use this operation.

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "username": "dade.murphy@example.com",
  "factorType": "SMS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/unlock"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "factorType": "SMS",
  "recoveryType": "UNLOCK",
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": {
      "name": "sms",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/resend",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Unlock Account with Trusted Application

Allows a [trusted application](#trusted-application) such as an external portal to implement its own primary authentication process and directly obtain a [recovery token](#recovery-token) for a user given just the user's identifier.

> Directly obtaining a `recoveryToken` is a highly privileged operation that requires an administrator API token and should be restricted to [trusted web applications](#trusted-application).  Anyone that obtains a `recoveryToken` for a user and knows the answer to a user's recovery question can reset their password or unlock their account.

> The **public IP address** of your [trusted application](#trusted-application) must be [whitelisted as a gateway IP address](../getting_started/design_principles.html#ip-address) to forward the user agent's original IP address with the `X-Forwarded-For` HTTP header

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36" \
-H "X-Forwarded-For: 23.235.46.133" \
-d '{
  "username": "dade.murphy@example.com",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}' "https://${org}.okta.com/api/v1/authn/recovery/unlock"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "recoveryToken": "VBQ0gwBp5LyJJFdbmWCM",
  "recoveryType": "UNLOCK",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  },
  "_links": {
    "next": {
      "name": "recovery",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/token",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Verify Recovery Factor

#### Verify SMS Recovery Factor
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/factors/sms/verify %}

Verifies a SMS OTP (`passCode`) sent to the user's mobile phone for primary authentication for a recovery transaction with `RECOVERY_CHALLENGE` status.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                                  | Param Type | DataType | Required |
------------ | ------------------------------------------------------------ | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current recovery transaction | Body       | String   | TRUE     |
passCode     | OTP sent to device                                           | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

If the `passCode` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oae2uOmZcuzToCPEV2Pc_f5zw",
  "errorCauses": [
    {
      "errorSummary": "Your token doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "passCode": "657866"
}' "https://${org}.okta.com/api/v1/authn/factors/sms/verify"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY",
  "recoveryType": "PASSWORD",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      },
      "recovery_question": {
        "question": "Who's a major player in the cowboy scene?"
      }
    }
  },
  "_links": {
    "next": {
      "name": "answer",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/answer",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

###### Resend SMS Recovery Challenge
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/factors/sms/resend %}

Resends a SMS OTP (`passCode`) to the user's mobile phone

#### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                                  | Param Type | DataType | Required |
------------ | ------------------------------------------------------------ | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current recovery transaction | Body       | String   | TRUE     |

#### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

#### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh"
}' "https://${org}.okta.com/api/v1/authn/recovery/factors/sms/resend"
~~~

#### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY_CHALLENGE",
  "factorType": "SMS",
  "recoveryType": "PASSWORD",
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": {
      "name": "sms",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/SMS/resend",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

> The `factorType` and `recoveryType` properties vary depending on recovery transaction.

#### Verify Call Recovery Factor
{:.api .api-operation}

<span class="api-uri-template api-uri-post"><span class="api-label">POST</span> /api/v1/authn/recovery/factors/call/verify</span>

Verifies a Voice Call OTP (`passCode`) sent to the user's device for primary authentication for a recovery transaction with `RECOVERY_CHALLENGE` status.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                                  | Param Type | DataType | Required |
------------ | ------------------------------------------------------------ | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current recovery transaction | Body       | String   | TRUE     |
passCode     | Passcode received via the voice call                         | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

If the `passCode` is invalid you will receive a `403 Forbidden` status code with the following error:

~~~json
{
  "errorCode": "E0000068",
  "errorSummary": "Invalid Passcode/Answer",
  "errorLink": "E0000068",
  "errorId": "oae2uOmZcuzToCPEV2Pc_f5zw",
  "errorCauses": [
    {
      "errorSummary": "Your token doesn't match our records. Please try again."
    }
  ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "passCode": "65786"
}' "https://${org}.okta.com/api/v1/authn/factors/CALL/verify"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY",
  "recoveryType": "PASSWORD",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      },
      "recovery_question": {
        "question": "Who's a major player in the cowboy scene?"
      }
    }
  },
  "_links": {
    "next": {
      "name": "answer",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/answer",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
 }
~~~

###### Resend Call Recovery Challenge
{:.api .api-operation}

<span class="api-uri-template api-uri-post"><span class="api-label">POST</span> /api/v1/authn/recovery/factors/call/resend</span>

Resends a Voice Call with OTP (`passCode`) to the user's phone

#### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                                  | Param Type | DataType | Required |
------------ | ------------------------------------------------------------ | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current recovery transaction | Body       | String   | TRUE     |

#### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

#### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh"
}' "https://${org}.okta.com/api/v1/authn/recovery/factors/CALL/resend"
~~~

#### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00xdqXOE5qDXX8-PBR1bYv8AESqIEinDy3yul01tyh",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY_CHALLENGE",
  "factorType": "call",
  "recoveryType": "PASSWORD",
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/CALL/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "resend": {
      "name": "call",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/factors/CALL/resend",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

> The `factorType` and `recoveryType` properties will vary depending on recovery transaction


### Verify Recovery Token
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/token %}

Validates a [recovery token](#recovery-token) that was distributed to the end-user to continue the recovery transaction.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter     | Description                                                                                                | Param Type | DataType | Required |
------------- | ---------------------------------------------------------------------------------------------------------- | ---------- | -------- | -------- |
recoveryToken | [Recovery token](#recovery-token) that was distributed to end-user via out-of-band mechanism such as email | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with a `RECOVERY` status and an issued `stateToken` that must be used to complete the recovery transaction.

You will receive a `401 Unauthorized` status code if you attempt to use an expired or invalid [recovery token](#recovery-token).

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
    "errorCode": "E0000011",
    "errorSummary": "Invalid token provided",
    "errorLink": "E0000011",
    "errorId": "oaeY-4G_TBUTBSZAn9n7oZCfw",
    "errorCauses": []
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "recoveryToken": "00xdqXOE5qDZX8-PBR1bYv8AESqIFinDy3yul01tyh"
}' "https://${org}.okta.com/api/v1/authn/recovery/token"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "RECOVERY",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      },
      "recovery_question": {
        "question": "Who's a major player in the cowboy scene?"
      }
    }
  },
  "_links": {
    "next": {
      "name": "answer",
      "href": "https://your-domain.okta.com/api/v1/authn/recovery/answer",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Answer Recovery Question
{:.api .api-operation}

{% api_operation post /api/v1/authn/recovery/answer %}

Answers the user's recovery question to ensure only the end-user redeemed the [recovery token](#recovery-token) for recovery transaction with a `RECOVERY` [status](#transaction-state).

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
answer       | answer to user's recovery question                  | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

You will receive a `403 Forbidden` status code if the `answer` to the user's [recovery question](#recovery-question-object) is invalid

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000087",
    "errorSummary": "The recovery question answer did not match our records.",
    "errorLink": "E0000087",
    "errorId": "oaeGEiIPFfeR3a_XxpezUH9ug",
    "errorCauses": []
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb",
  "answer": "Annie Oakley"
}' "https://${org}.okta.com/api/v1/authn/recovery/answer"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "PASSWORD_RESET",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
   },
   "policy": {
    "expiration":{
      "passwordExpireDays": 0
      },
      "complexity": {
        "minLength": 8,
        "minLowerCase": 1,
        "minUpperCase": 1,
        "minNumber": 1,
        "minSymbol": 0,
        "excludeUsername": "true"
       },
       "age":{
         "minAgeMinutes":0,
         "historyCount":0
      }
    }
  },
  "_links": {
    "next": {
      "name": "password",
      "href": "https://your-domain.okta.com/api/v1/authn/credentials/reset_password",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Reset Password
{:.api .api-operation}

{% api_operation post /api/v1/authn/credentials/reset_password %}

Resets a user's password to complete a recovery transaction with a `PASSWORD_RESET` [state](#transaction-state).

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for current transaction | Body       | String   | TRUE     |
newPassword  | user's new password                                 | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Recovery Transaction Object](#recovery-transaction-model) with the current [state](#transaction-state) for the recovery transaction.

You will receive a `403 Forbidden` status code if the `answer` to the user's [recovery question](#recovery-question-object) is invalid.

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000087",
    "errorSummary": "The recovery question answer did not match our records.",
    "errorLink": "E0000087",
    "errorId": "oaeGEiIPFfeR3a_XxpezUH9ug",
    "errorCauses": []
}
~~~

You will also receive a `403 Forbidden` status code if the `newPassword` does not meet password policy requirements for the user.

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
    "errorCode": "E0000014",
    "errorSummary": "The password does meet the complexity requirements of the current password policy.",
    "errorLink": "E0000014",
    "errorId": "oaeS4O7BUp5Roefkk_y4Z2u8Q",
    "errorCauses": [
        {
            "errorSummary": "Passwords must have at least 8 characters, a lowercase letter, an uppercase letter, a number, no parts of your username"
        }
    ]
}
~~~

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb",
  "newPassword": "Ch-ch-ch-ch-Changes!"
}' "https://${org}.okta.com/api/v1/authn/credentials/reset_password"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00t6IUQiVbWpMLgtmwSjMFzqykb5QcaBNtveiWlGeM",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-11-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

## State Management Operations

### Get Transaction State
{:.api .api-operation}

{% api_operation post /api/v1/authn %}

Retrieves the current [transaction state](#transaction-state) for a [state token](#state-token).

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for a transaction       | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Transaction Object](#transaction-model) with the current [state](#transaction-state) for the authentication or recovery transaction.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb"
}' "https://${org}.okta.com/api/v1/authn"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_CHALLENGE",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factor": {
      "id": "sms193zUBEROPBNZKPPE",
      "factorType": "sms",
      "provider": "OKTA",
      "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
      }
    }
  },
  "_links": {
    "next": {
      "name": "verify",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/sms193zUBEROPBNZKPPE/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Previous Transaction State
{:.api .api-operation}

{% api_operation post /api/v1/authn/previous %}

Moves the current [transaction state](#transaction-state) back to the previous state.

For example, when changing state from the start of primary authentication to MFA_ENROLL > ENROLL_ACTIVATE > OTP,
the user's phone might stop working. Since the user can't see the QR code, the transaction must return to MFA_ENROLL.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for a transaction       | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Transaction Object](#transaction-model) with the current [state](#transaction-state) for the authentication or recovery transaction.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb"
}' "https://${org}.okta.com/api/v1/authn/previous"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "stateToken": "007ucIX7PATyn94hsHfOLVaXAmOBkKHWnOOLG43bsb",
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "MFA_ENROLL",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-09-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    },
    "factors": [
      {
        "factorType": "token",
        "provider": "RSA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "factorType": "token:software:totp",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      },
      {
        "factorType": "sms",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      }
      {
        "factorType": "push",
        "provider": "OKTA",
        "_links": {
          "enroll": {
            "href": "https://your-domain.okta.com/api/v1/authn/factors",
            "hints": {
              "allow": [
                "POST"
              ]
            }
          }
        }
      }
    ]
  },
  "_links": {
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Skip Transaction State
{:.api .api-operation}

{% api_operation post /api/v1/authn/skip %}

Send a skip link to skip the current [transaction state](#transaction-state) and advance to the next state.

If the response returns a skip link, then you can advance to the next state without completing the current state (such as changing the password).
For example, after being warned that a password will soon expire, the user can skip the change password prompt
by clicking a skip link.

Another example: a user has enrolled in multiple factors. After enrolling in one the user receives a skip link
to skip the other factors.

> This operation is only available for `MFA_ENROLL` or `PASSWORD_WARN` states when published as a link.

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for a transaction       | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

[Transaction Object](#transaction-model) with the current [state](#transaction-state) for the authentication or recovery transaction.

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb"
}' "https://${org}.okta.com/api/v1/authn/skip"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "expiresAt": "2015-11-03T10:15:57.000Z",
  "status": "SUCCESS",
  "relayState": "/myapp/some/deep/link/i/want/to/return/to",
  "sessionToken": "00t6IUQiVbWpMLgtmwSjMFzqykb5QcaBNtveiWlGeM",
  "_embedded": {
    "user": {
      "id": "00ub0oNGTSWTBKOLGLNR",
      "passwordChanged": "2015-11-08T20:14:45.000Z",
      "profile": {
        "login": "dade.murphy@example.com",
        "firstName": "Dade",
        "lastName": "Murphy",
        "locale": "en_US",
        "timeZone": "America/Los_Angeles"
      }
    }
  }
}
~~~

### Cancel Transaction
{:.api .api-operation}

{% api_operation post /api/v1/authn/cancel %}

Cancels the current transaction and revokes the [state token](#state-token).

##### Request Parameters
{:.api .api-request .api-request-params}

Parameter    | Description                                         | Param Type | DataType | Required |
------------ | --------------------------------------------------- | ---------- | -------- | -------- |
stateToken   | [state token](#state-token) for a transaction       | Body       | String   | TRUE     |

##### Response Parameters
{:.api .api-response .api-response-params}

Parameter    | Description                                                                            | Param Type | DataType | Required |
------------ | -------------------------------------------------------------------------------------- | ---------- | -------- | -------- |
relayState   | Optional state value that was persisted for the authentication or recovery transaction | Body       | String   | TRUE     |

##### Request Example
{:.api .api-request .api-request-example}

~~~sh
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-d '{
  "stateToken": "00lMJySRYNz3u_rKQrsLvLrzxiARgivP8FB_1gpmVb"
}' "https://${org}.okta.com/api/v1/authn/cancel"
~~~

##### Response Example
{:.api .api-response .api-response-example}

~~~json
{
  "relayState": "/myapp/some/deep/link/i/want/to/return/to"
}
~~~


## Transaction Model

The Authentication API is a *stateful* API that implements a finite state machine with [defined states](#transaction-state) and transitions.  Each initial authentication or recovery request is issued a unique [state token](#state-token) that must be passed with each subsequent request until the transaction is complete or canceled.

The Authentication API leverages the [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) format to publish `next` and `prev` links for the current transaction state which should be used to transition the state machine.

### Authentication Transaction Model

|---------------+--------------------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+----------+-----------|
| Property      | Description                                                                                            | DataType                                                       | Nullable | Readonly | MaxLength |
| ------------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- | -------- | -------- | --------- |
| stateToken    | ephemeral [token](#state-token) that encodes the current state of an authentication transaction        | String                                                         | TRUE     | TRUE     |           |
| sessionToken  | ephemeral [one-time token](#session-token) used to bootstrap an Okta session                           | String                                                         | TRUE     | TRUE     |           |
| expiresAt     | lifetime of the `stateToken` or `sessionToken` (See [Tokens](#tokens))                                 | Date                                                           | TRUE     | TRUE     |           |
| status        | current [state](#transaction-state) of the authentication transaction                                  | [Transaction State](#transaction-state)                        | FALSE    | TRUE     |           |
| relayState    | optional opaque value that is persisted for the lifetime of the authentication transaction             | String                                                         | TRUE     | TRUE     |   2048    |
| factorResult  | optional status of last verification attempt for a given factor                                        | [Factor Result](#factor-result)                                | TRUE     | TRUE     |           |
| _embedded     | [embedded resources](#embedded-resources) for the current `status`                                     | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | TRUE     |           |
| _links        | [link relations](#links-object) for the current `status`                                               | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | TRUE     |           |
|---------------+--------------------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+----------+-----------|

> The `relayState` paramater is an opaque value for the transaction and processed as untrusted data which is just echoed in a response.  It is the client's responsibility to escape/encode this value before displaying in a UI such as a HTML document using [best-practices](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)

### Recovery Transaction Model

|---------------+--------------------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+----------+-----------|
| Property      | Description                                                                                            | DataType                                                       | Nullable | Readonly | MaxLength |
| ------------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- | -------- | -------- | --------- |
| stateToken    | ephemeral [token](#state-token) that encodes the current state of a recovery transaction               | String                                                         | TRUE     | TRUE     |           |
| recoveryToken | ephemeral [one-time token](#recovery-token) for recovery transaction to be distributed to the end-user | String                                                         | TRUE     | TRUE     |           |
| expiresAt     | lifetime of the `stateToken` or `recoveryToken` (See [Tokens](#tokens))                                | Date                                                           | TRUE     | TRUE     |           |
| status        | current [state](#transaction-state) of the recovery transaction                                        | [Transaction State](#transaction-state)                        | FALSE    | TRUE     |           |
| relayState    | optional opaque value that is persisted for the lifetime of the recovery transaction                   | String                                                         | TRUE     | TRUE     |   2048    |
| factorType    | type of selected factor for the recovery transaction                                                   | `EMAIL` or `SMS`                                               | FALSE    | TRUE     |           |
| recoveryType  | type of recovery operation                                                                             | `PASSWORD` or `UNLOCK`                                         | FALSE    | TRUE     |           |
| factorResult  | optional status of last verification attempt for the `factorType`                                      | [Factor Result](#factor-result)                                | TRUE     | TRUE     |           |
| _embedded     | [embedded resources](#embedded-resources) for the current `status`                                     | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | TRUE     |           |
| _links        | [link relations](#links-object) for the current `status`                                               | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | TRUE     |           |
|---------------+--------------------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+----------+-----------|

> The `relayState` paramater is an opaque value for the transaction and processed as untrusted data which is just echoed in a response.  It is the client's responsibility to escape/encode this value before displaying in a UI such as a HTML document using [best-practices](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)

### Transaction State

{% img auth-state-model.png "State Model Diagram" alt:"State Model Diagram" %}

An authentication or recovery transaction has one of the following states:

|-----------------------+----------------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------|
| Value                 | Description                                                                                  | Next Action                                                                                                    |
| --------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------|
| `PASSWORD_WARN`       | The user's password was successfully validated but is about to expire and should be changed. | POST to the `next` link relation to [change the user's password](#change-password).                            |
| `PASSWORD_EXPIRED`    | The user's password was successfully validated but is expired.                               | POST to the `next` link relation to [change the user's expired password](#change-password).                    |
| `RECOVERY`            | The user has requested a recovery token to reset their password or unlock their account.     | POST to the `next` link relation to [answer the user's recovery question](#answer-recovery-question).          |
| `RECOVERY_CHALLENGE`  | The user must verify the factor-specific recovery challenge.                                 | POST to the `verify` link relation to [verify the recovery factor](#verify-recovery-factor).                   |
| `PASSWORD_RESET`      | The user successfully answered their recovery question and must to set a new password.       | POST to the `next` link relation to [reset the user's password](#reset-password).                              |
| `LOCKED_OUT`          | The user account is locked; self-service unlock or admin unlock is required.                 | POST to the `unlock` link relation to perform a [self-service unlock](#unlock-account).                        |
| `MFA_ENROLL`          | The user must select and enroll an available factor for additional verification.             | POST to the `enroll` link relation for a specific factor to [enroll the factor](#enroll-factor).               |
| `MFA_ENROLL_ACTIVATE` | The user must activate the factor to complete enrollment.                                    | POST to the `next` link relation to [activate the factor](#activate-factor).                                   |
| `MFA_REQUIRED`        | The user must provide additional verification with a previously enrolled factor.             | POST to the `verify` link relation for a specific factor to [provide additional verification](#verify-factor). |
| `MFA_CHALLENGE`       | The user must verify the factor-specific challenge.                                          | POST to the `verify` link relation to [verify the factor](#verify-factor).                                     |
| `SUCCESS`             | The transaction has completed successfully                                                   |                                                                                                                |
|-----------------------+----------------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------|

You advance the authentication or recovery transaction to the next state by posting a request with a valid [state token](#state-token) to the the `next` link relation published in the [JSON HAL links object](#links-object) for the response.

[Enrolling a factor](#enroll-factor) and [verifying a factor](#verify-factor) do not have `next` link relationships as the end-user must make a selection of which factor to enroll or verify.

> You should never assume a specific state transition or URL when navigating the [state model](#transaction-state).  Always inspect the response for `status` and dynamically follow the [published link relations](#links-object).

~~~json
{
  "_links": {
    "next": {
      "name": "activate",
      "href": "https://your-domain.okta.com/api/v1/authn/factors/ostf2xjtDKWFPZIKYDZV/lifecycle/activate",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "prev": {
      "href": "https://your-domain.okta.com/api/v1/authn/previous",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "skip": {
      "href": "https://your-domain.okta.com/api/v1/authn/skip",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    },
    "cancel": {
      "href": "https://your-domain.okta.com/api/v1/authn/cancel",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

### Tokens

Authentication API operations will return different token types depending on the [state](#transaction-state) of the authentication or recovery transaction.

#### State Token

Ephemeral token that encodes the current state of an authentication or recovery transaction.

- The `stateToken` must be passed with every request except when verifying a `recoveryToken` that was distributed out-of-band
- The `stateToken` is only intended to be used between the web application performing end-user authentication and the Okta API. It should never be distributed to the end-user via email or other out-of-band mechanisms.
- The lifetime of the `stateToken` uses a sliding scale expiration algorithm that extends with every request.  Always inspect the `expiresAt` property for the transaction when making decisions based on lifetime.

> All Authentication API operations will return `401 Unauthorized` status code when you attempt to use an expired state token.

~~~http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "errorCode": "E0000011",
  "errorSummary": "Invalid token provided",
  "errorLink": "E0000011",
  "errorId": "oaeY-4G_TBUTBSZAn9n7oZCfw",
  "errorCauses": []
}
~~~

> State transitions are strictly enforced for state tokens.  You will receive a `403 Forbidden` status code if you call an Authentication API operation with a `stateToken` with an invalid [state](#transaction-state).

~~~http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "errorCode": "E0000079",
  "errorSummary": "This operation is not allowed in the current authentication state.",
  "errorLink": "E0000079",
  "errorId": "oaen9Ly_ivHQJ-STb8KiADh9w",
  "errorCauses": [
    {
      "errorSummary": "This operation is not allowed in the current authentication state."
    }
  ]
}
~~~

#### Recovery Token

One-time token issued as `recoveryToken` response parameter when a recovery transaction transitions to the `RECOVERY` status.

- The token can be exchanged for a `stateToken` to recover a user's password or unlock their account.
- Unlike the `statusToken` the `recoveryToken` should be distributed out-of-band to a user such as via email.
- The lifetime of the `recoveryToken` is managed by the organization's security policy.

The `recoveryToken` is sent via an out-of-band channel to the end-user's verified email address or SMS phone number and acts as primary authentication for the recovery transaction.

> Directly obtaining a `recoveryToken` is a highly privileged operation and should be restricted to trusted web applications.  Anyone that obtains a `recoveryToken` for a user and knows the answer to a user's recovery question can reset their password or unlock their account.

#### Session Token

One-time token issued as `sessionToken` response parameter when an authentication transaction completes with the `SUCCESS` status.

- The token can be exchanged for a session with the [Session API](sessions.html#create-session-with-session-token) or converted to a [session cookie](/docs/examples/session_cookie.html).
- The lifetime of the `sessionToken` is the same as the lifetime of a user's session and managed by the organization's security policy.

### Factor Result

The `MFA_CHALLENGE` or `RECOVERY_CHALLENGE` state can return an additional property **factorResult** that provides additional context for the last factor verification attempt.

The following table shows the possible values for this property:

|------------------------+-------------------------------------------------------------------------------------------------------------------------------------|
| factorResult           | Description                                                                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------|
| `WAITING`              | Factor verification has started but not yet completed (e.g user hasn't answered phone call yet)                                     |
| `CANCELLED`            | Factor verification was canceled by user                                                                                            |
| `TIMEOUT`              | Unable to verify factor within the allowed time window                                                                              |
| `TIME_WINDOW_EXCEEDED` | Factor was successfully verified but outside of the computed time window.  Another verification is required in current time window. |
| `PASSCODE_REPLAYED`    | Factor was previously verified within the same time window.  User must wait another time window and retry with a new verification.  |
| `ERROR`                | Unexpected server error occurred verifying factor.                                                                                  |
|------------------------+-------------------------------------------------------------------------------------------------------------------------------------|

### Links Object

Specifies link relations (See [Web Linking](http://tools.ietf.org/html/rfc5988)) available for the current [transaction state](#transaction-state) using the [JSON](https://tools.ietf.org/html/rfc7159) specification.  These links are used to transition the [state machine](#transaction-state) of the authentication or recovery transaction.

|--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Link Relation Type | Description                                                                                                                                                               |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| next               | Transitions the  [state machine](#transaction-state) to the next state.  **Note: The `name` of the link relationship will provide a hint of the next operation required** |
| prev               | Transitions the  [state machine](#transaction-state) to the previous state.                                                                                               |
| cancel             | Cancels the current  transaction and revokes the [state token](#state-token).                                                                                             |
| skip               | Skips over the current  transaction state to the next valid [state](#transaction-state)                                                                                   |
| resend             | Resends a challenge or OTP to a device                                                                                                                                    |
|--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

> The Links Object is **read-only**

## Embedded Resources

### User Object

A subset of [user properties](users.html#user-model) published in an authentication or recovery transaction after the user successfully completes primary authentication.

|-------------------+---------------------------------------------+-------------------------------------------------------+----------+--------+----------|
| Property          | Description                                 | DataType                                              | Nullable | Unique | Readonly |
| ----------------- | ------------------------------------------- | ----------------------------------------------------- | -------- | ------ | -------- |
| id                | unique key for user                         | String                                                | FALSE    | TRUE   | TRUE     |
| passwordChanged   | timestamp when user's password last changed | Date                                                  | TRUE     | FALSE  | TRUE     |
| profile           | user's profile                              | [User Profile Object](#user-profile-object)           | FALSE    | FALSE  | TRUE     |
| recovery_question | user's recovery question                    | [Recovery Question Object](#recovery-question-object) | TRUE     | FALSE  | TRUE     |
|-------------------+---------------------------------------------+-------------------------------------------------------+----------+--------+----------|

~~~json
{
 "id": "00udnlQDVLVRIVXESCMZ",
 "passwordChanged": "2015-09-08T20:14:45.000Z",
 "profile": {
    "login": "dade.murphy@example.com",
    "firstName":"Dade",
    "lastName": "Murphy",
    "locale": "en_US",
    "timeZone": "America/Los_Angeles"
 },
 "recovery_question": {
    "question": "Who's a major player in the cowboy scene?"
  }
}
~~~

#### User Profile Object

Subset of [profile properties](users.html#profile-object) for a user

|-----------+------------------------------------------------------------------------------------------------------------------------------+----------+----------+--------+----------+-----------------------------------------------------------------------|
| Property  | Description                                                                                                                  | DataType | Nullable | Unique | Readonly | Validation                                                            |
| --------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------| -------- | ------ | -------- | --------------------------------------------------------------------- |
| login     | unique login for user                                                                                                        | String   | FALSE    | TRUE   | TRUE     |                                                                       |
| firstName | first name of user                                                                                                           | String   | FALSE    | FALSE  | TRUE     |                                                                       |
| lastName  | last name of user                                                                                                            | String   | FALSE    | FALSE  | TRUE     |                                                                       |
| locale    | user's default location for purposes of localizing items such as currency, date time format, numerical representations, etc. | String   | TRUE     | FALSE  | TRUE     | [RFC 5646](https://tools.ietf.org/html/rfc5646)                       |
| timeZone  | user's time zone                                                                                                             | String   | TRUE     | FALSE  | TRUE     | [IANA Time Zone database format](https://tools.ietf.org/html/rfc6557) |
|-----------+------------------------------------------------------------------------------------------------------------------------------+----------+----------+--------+----------+-----------------------------------------------------------------------|

#### Recovery Question Object

User's recovery question used for verification of a recovery transaction

|-------------------+--------------------------+ ---------+----------+--------+----------|
| Property          | Description              | DataType | Nullable | Unique | Readonly |
| ----------------- | ------------------------ | -------- | -------- | ------ | -------- |
| question          | user's recovery question | String   | FALSE    | TRUE   | TRUE     |
|-------------------+--------------------------+ ---------+----------+--------+----------|

### Password Policy Object

A subset of policy settings for the user's assigned password policy published during `PASSWORD_WARN`, `PASSWORD_EXPIRED`, or `PASSWORD_RESET` states

|------------+------------------------------+-----------------------------------------------------------+----------+--------+----------|
| Property   | Description                  | DataType                                                  | Nullable | Unique | Readonly |
| ---------- | ---------------------------- | --------------------------------------------------------- | -------- | ------ | -------- |
| expiration | password expiration settings | [Password Expiration Object](#password-expiration-object) | TRUE     | FALSE  | TRUE     |
| complexity | password complexity settings | [Password Complexity Object](#password-complexity-object) | FALSE    | FALSE  | TRUE     |
|------------+------------------------------+-----------------------------------------------------------+----------+--------+----------|

~~~json
{
  "expiration":{
    "passwordExpireDays": 0
  },
  "complexity": {
    "minLength": 8,
    "minLowerCase": 1,
    "minUpperCase": 1,
    "minNumber": 1,
    "minSymbol": 0,
    "excludeUsername": "true"
    },
   "age":{
     "minAgeMinutes":0,
     "historyCount":0
    }
}
~~~

#### Password Expiration Object

Specifies the password age requirements of the assigned password policy

|--------------------+-------------------------------------------+----------+----------+--------+----------|
| Property           | Description                               | DataType | Nullable | Unique | Readonly |
| ------------------ | ----------------------------------------- | -------- | -------- | ------ | -------- |
| passwordExpireDays | number of days before password is expired | Number   | FALSE    | FALSE  | TRUE     |
|--------------------+-------------------------------------------+----------+----------+--------+----------|

#### Password Complexity Object

Specifies the password complexity requirements of the assigned password policy

|--------------+------------------------------------------------------+----------+----------+--------+----------|
| Property     | Description                                          | DataType | Nullable | Unique | Readonly |
| ------------ | ---------------------------------------------------- | -------- | -------- | ------ | -------- |
| minLength    | minimum number of characters for password            | Number   | FALSE    | FALSE  | TRUE     |
| minLowerCase | minimum number of lower case characters for password | Number   | FALSE    | FALSE  | TRUE     |
| minUpperCase | minimum number of upper case characters for password | Number   | FALSE    | FALSE  | TRUE     |
| minNumber    | minimum number of numeric characters for password    | Number   | FALSE    | FALSE  | TRUE     |
| minSymbol    | minimum number of symbol characters for password     | Number   | FALSE    | FALSE  | TRUE     |
| excludeUsername    | Prevents username or domain from appearing in the password     | boolean   | FALSE    | FALSE  | TRUE     |
|--------------+------------------------------------------------------+----------+----------+--------+----------|

> Duplicate the minimum Active Directory requirements in these settings for AD-mastered users. No enforcement is triggered by Okta settings for AD-mastered users.

#### Password Age Object

Specifies the password requirements related to password age and history

|------------------+-------------------------------------------------------------------+----------+----------+--------+----------|
| Property         | Description                                                       | DataType | Nullable | Unique | Readonly |
| ---------------- | ----------------------------------------------------------------- | -------- | -------- | ------ | -------- |
| minAgeMinutes    | Minimum number of minutes required since the last password change | Number   | FALSE    | FALSE  | TRUE     |
| historyCount     |Number of previous passwords that the current password can't match | Number   | FALSE    | FALSE  | TRUE     |
|------------------+-------------------------------------------------------------------+----------+----------+--------+----------|

### Factor Object

A subset of [factor properties](factors.html#factor-model) published in an authentication transaction during `MFA_ENROLL`, `MFA_REQUIRED`, or `MFA_CHALLENGE` states

|----------------+------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+--------+----------|
| Property       | Description                                                                              | DataType                                                       | Nullable | Unique | Readonly |
| -------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------- | -------- | ------ | ---------|
| id             | unique key for factor                                                                    | String                                                         | TRUE     | TRUE   | TRUE     |
| factorType     | type of factor                                                                           | [Factor Type](factors.html#factor-type)                        | FALSE    | TRUE   | TRUE     |
| provider       | factor provider                                                                          | [Provider Type](factors.html#provider-type)                    | FALSE    | TRUE   | TRUE     |
| vendorName     | factor Vendor Name (Same as provider but for On Prem MFA it depends on Admin Settings)   | [Provider Type](/docs/api/resources/factors.html#provider-type)                      | FALSE    | TRUE   | TRUE     |
| profile        | profile of a [supported factor](factors.html#supported-factors-for-providers)            | [Factor Profile Object](factors.html#factor-profile-object)    | TRUE     | FALSE  | TRUE     |
| _embedded      | [embedded resources](#factor-embedded-resources) related to the factor                   | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | FALSE  | TRUE     |
| _links         | [discoverable resources](#factor-links-object) for the factor                            | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | FALSE  | TRUE     |
|----------------+------------------------------------------------------------------------------------------+----------------------------------------------------------------+----------+--------+----------|

~~~json
{
  "id": "ostfm3hPNYSOIOIVTQWY",
  "factorType": "token:software:totp",
  "provider": "OKTA",
  "profile": {
    "credentialId": "dade.murphy@example.com"
  },
  "_links": {
    "verify": {
      "href": "https://your-domain.okta.com/api/v1/authn/factors/ostfm3hPNYSOIOIVTQWY/verify",
      "hints": {
        "allow": [
          "POST"
        ]
      }
    }
  }
}
~~~

#### Factor Embedded Resources

##### TOTP Factor Activation Object

TOTP factors, when activated, have an embedded verification object which describes the [TOTP](http://tools.ietf.org/html/rfc6238) algorithm parameters.

|----------------+---------------------------------------------------+----------------------------------------------------------------+----------|--------|----------|
| Property       | Description                                       | DataType                                                       | Nullable | Unique | Readonly |
| -------------- | ------------------------------------------------- | -------------------------------------------------------------- | -------- | ------ | -------- |
| timeStep       | time-step size for TOTP                           | String                                                         | FALSE    | FALSE  | TRUE     |
| sharedSecret   | unique secret key for prover                      | String                                                         | FALSE    | FALSE  | TRUE     |
| encoding       | encoding of `sharedSecret`                        | `base32` or `base64`                                           | FALSE    | FALSE  | TRUE     |
| keyLength      | number of digits in an TOTP value                 | Number                                                         | FALSE    | FALSE  | TRUE     |
| _links         | discoverable resources related to the activation  | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | TRUE     | FALSE  | TRUE     |
|----------------+---------------------------------------------------+----------------------------------------------------------------+----------|--------|----------|

> This object implements [the TOTP standard](https://tools.ietf.org/html/rfc6238), which is used by apps like Okta Verify and Google Authenticator.

~~~ json
{
  "activation": {
    "timeStep": 30,
    "sharedSecret": "HE64TMLL2IUZW2ZLB",
    "encoding": "base32",
    "keyLength": 6
  }
}
~~~

###### TOTP Activation Links Object

Specifies link relations (See [Web Linking](http://tools.ietf.org/html/rfc5988)) available for the TOTP activation object using the [JSON Hypertext Application Language](http://tools.ietf.org/html/draft-kelly-json-hal-06) specification.  This object is used for dynamic discovery of related resources and operations.

|--------------------+--------------------------------------------------------------------------|
| Link Relation Type | Description                                                              |
| ------------------ | ------------------------------------------------------------------------ |
| qrcode             | QR code that encodes the TOTP parameters that can be used for enrollment |
|--------------------+--------------------------------------------------------------------------|

##### Phone Object

The phone object describes previously enrolled phone numbers for the `sms` factor.

|---------------+----------------------+-----------------------------------------------+----------+--------+----------|
| Property      | Description          | DataType                                      | Nullable | Unique | Readonly |
| ------------- | -------------------- | --------------------------------------------- | -------- | ------ | -------- |
| id            | unique key for phone | String                                        | FALSE    | TRUE   | TRUE     |
| profile       | profile of phone     | [Phone Profile Object](#phone-profile-object) | FALSE    | FALSE  | TRUE     |
| status        | status of phone      | `ACTIVE` or `INACTIVE`                        | FALSE    | FALSE  | TRUE     |
|---------------+----------------------+-----------------------------------------------+----------+--------+----------|

~~~json
{
    "id": "mbl198rKSEWOSKRIVIFT",
    "profile": {
        "phoneNumber": "+1 XXX-XXX-1337"
    },
    "status": "ACTIVE"
}
~~~

###### Phone Profile Object

|---------------+----------------------|----------+----------|--------|----------|
| Property      | Description          | DataType | Nullable | Unique | Readonly |
| ------------- | -------------------- | ---------| -------- | ------ | -------- |
| phoneNumber   | masked phone number  | String   | FALSE    | FALSE  | TRUE     |
|---------------+----------------------|----------+----------|--------|----------|

##### Push Factor Activation Object

Push factors must complete activation on the device by scanning the QR code or visiting activation link sent via email or sms.

|----------------+---------------------------------------------------+----------------------------------------------------------------+----------+--------+----------|
| Property       | Description                                       | DataType                                                       | Nullable | Unique | Readonly |
| -------------- | ------------------------------------------------- | -------------------------------------------------------------- | -------- | ------ | -------- |
| expiresAt      | lifetime of activation                            | Date                                                           | FALSE    | FALSE  | TRUE     |
| _links         | discoverable resources related to the activation  | [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-06) | FALSE    | FALSE  | TRUE     |
|----------------+---------------------------------------------------+----------------------------------------------------------------+----------+--------+----------|

~~~json
{
  "activation": {
    "expiresAt": "2015-11-13T07:44:22.000Z",
    "_links": {
      "send": [
        {
          "name": "email",
          "href": "https://your-domain.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors/opfbtzzrjgwauUsxO0g4/lifecycle/activate/email",
          "hints": {
            "allow": [
              "POST"
            ]
          }
        },
        {
          "name": "sms",
          "href": "https://your-domain.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors/opfbtzzrjgwauUsxO0g4/lifecycle/activate/sms",
          "hints": {
            "allow": [
              "POST"
            ]
          }
        }
      ],
      "qrcode": {
        "href": "https://your-domain.okta.com/api/v1/users/00u15s1KDETTQMQYABRL/factors/opfbtzzrjgwauUsxO0g4/qr/00Ji8qVBNJD4LmjYy1WZO2VbNqvvPdaCVua-1qjypa",
        "type": "image/png"
      }
    }
  }
}
~~~

###### Push Factor Activation Links Object

Specifies link relations (See [Web Linking](http://tools.ietf.org/html/rfc5988)) available for the push factor activation object using the [JSON Hypertext Application Language](http://tools.ietf.org/html/draft-kelly-json-hal-06) specification.  This object is used for dynamic discovery of related resources and operations.

|--------------------+------------------------------------------------------------------------------------|
| Link Relation Type | Description                                                                        |
| ------------------ | ---------------------------------------------------------------------------------- |
| qrcode             | QR code that encodes the push activation code needed for enrollment on the device  |
| send               | Sends an activation link via `email` or `sms` for users who can't scan the QR code |
|--------------------+------------------------------------------------------------------------------------|

##### Factor Links Object

Specifies link relations (See [Web Linking](http://tools.ietf.org/html/rfc5988)) available for the factor using the [JSON Hypertext Application Language](http://tools.ietf.org/html/draft-kelly-json-hal-06) specification.  This object is used for dynamic discovery of related resources and operations.

|--------------------+--------------------------------------------------------------|
| Link Relation Type | Description                                                  |
| ------------------ | -------------------------------------------------------------|
| enroll             | [Enrolls a factor](#enroll-factor)                           |
| verify             | [Verifies a factor](#verify-factor)                          |
| questions          | Lists all possible questions for the `question` factor type  |
| resend             | Resends a challenge or OTP to a device                       |
|--------------------+--------------------------------------------------------------|

> The Links Object is **read-only**
