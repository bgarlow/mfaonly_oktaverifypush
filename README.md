# mfaonly_oktaverifypush
Example API calls to create a new user and enroll in Okta Verify with Push for MFA only. This repo includes a Postman Collection for the flow described below, as well as a few related calls (different options for creating the user, resetting factors, etc.)

## Use Case (abstracted)

The user creation and MFA enrollment flow described below was developed for a customer who wanted to add Multi-Factor Authentication (push notification in this case) to a service that brokers orders between the application used by the end user and a 3rd party service. The user may work in one of several supported front-end apps. When the user places an order, it is received by the broker service which performs some validation and forwards the order to one of several 3rd party systems for fulfilment. Orders for certain restricted materials require a second form of authentication.

The organization that manages the broker service has an enrollment process whereby a user A at company X is given an account to use company X's front-end application to place orders through the broker service. The end user is assigned a unique ID number that identifies the end user to the broker service. The end user may be enrolled to use front-end systems at more than one company, but will always use the same ID (let's call it their BrokerUserID...that's right BUD). User records are maintained by the broker service. The broker service will leverage Okta for MFA only.

Push notification is the preferred factor for restricted materials, so the broker service will leverage Okta Verify with Push. In order to enroll an end user in a factor in Okta, an Okta user must be created. We will create Okta users with minimal profile information and without the ability to authenticate with Okta. Their Okta username will be their BrokerUserID.

The sequence diagrams below describes the flow user enrollment in MFA (which includes creating the user in Okta and enrolling them in Okta Verify with Push), as well as the flow for challenging the user for MFA.


```sequence
user->>app: Login
user->>app: Click to enroll in broker
app->>user: Display instructions for downloading Okta Verify app
Note over user: Download Okta Verify from app store and install 
app->>broker: Enroll (BUD)
broker->>Okta: Create new user with userName = BUD
Okta->>broker: Return new user's Okta UUID
broker->>Okta: Enroll new user in Okta Verify with Push
Okta->>broker: factorId and activation links
broker->>app: URL of QR code 
broker->>app: (ALT) SMS or Email activation links
app->>user: Display QR code
loop Poll to see if user has scanned QR code
broker->>Okta: /api/v1/users/{BUD}/factors/{factorId}/lifecycle/activate/poll
Okta->>broker: factorStatus: "WAITING"
Note over user: Scans QR code with \nOkta Verify app
end
Okta->>broker: status: "ACTIVE"
broker->>app: status
Note over app: indicate success/failure of factor enrollment
```
#### Enrollment

![Enrollment Sequence Diagram](https://github.com/bgarlow/mfaonly_oktaverifypush/blob/master/enroll_sequence.png)


#### Challenge
![Challenge Sequence Diagram](https://github.com/bgarlow/mfaonly_oktaverifypush/blob/master/challenge_sequence.png)


## Create User with Authentication Provider
This will create a new user with no password, but who's status is Active. The _provider_ attribute on the user's _credentials_ object will be set to "FEDERATION". In the future, this user could be converted to an Okta-mastered user (with Okta-controlled password) via the Reset Password API endpoint. See the example at the bottom of this document.

```bash
curl -X POST \
  'https://dev-123456.oktapreview.com/api/v1/users?provider=true' \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 54de3b7c-4290-48b1-a7f8-0f86c3d121d6' \
  -H 'cache-control: no-cache' \
  -d '{
  "profile": {
    "login": "TPR1539702348",
    "email": "TPR1539702348@dummy.com"
  },
  "credentials": {
    "provider": {
      "type": "FEDERATION",
      "name": "FEDERATION"
    }
  }
}'
```
```json
{
    "id": "00ugntsq8a24jn8Nw0h7",
    "status": "ACTIVE",
    "created": "2018-10-16T15:06:15.000Z",
    "activated": "2018-10-16T15:06:15.000Z",
    "statusChanged": "2018-10-16T15:06:15.000Z",
    "lastLogin": null,
    "lastUpdated": "2018-10-16T15:06:15.000Z",
    "passwordChanged": null,
    "profile": {
        "firstName": null,
        "lastName": null,
        "mobilePhone": null,
        "secondEmail": null,
        "login": "TPR1539702375",
        "email": "TPR1539702375@dummy.com"
    },
    "credentials": {
        "emails": [
            {
                "value": "TPR1539702375@dummy.com",
                "status": "VERIFIED",
                "type": "PRIMARY"
            }
        ],
        "provider": {
            "type": "FEDERATION",
            "name": "FEDERATION"
        }
    },
    "_links": {
        "suspend": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntsq8a24jn8Nw0h7/lifecycle/suspend",
            "method": "POST"
        },
        "resetPassword": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntsq8a24jn8Nw0h7/lifecycle/reset_password",
            "method": "POST"
        },
        "changeRecoveryQuestion": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntsq8a24jn8Nw0h7/credentials/change_recovery_question",
            "method": "POST"
        },
        "self": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntsq8a24jn8Nw0h7"
        },
        "deactivate": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntsq8a24jn8Nw0h7/lifecycle/deactivate",
            "method": "POST"
        }
    }
}
```
## Enroll Okta Verify Push Factor
This will begin the enrollment process and result in a status of "PENDING_ACTIVATION". The response will contain a "poll" url that you will need to poll to determine when the user has clicked "Accept" in Okta Verify. See [Verify Results Object](https://developer.okta.com/docs/api/resources/factors#factor-result).
```bash
curl -X POST \
  https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: a2as4c1c-zzz-49c9-zzzz-d1bb3884ffzzzz174' \
  -H 'cache-control: no-cache' \
  -d '{
  "factorType": "push",
  "provider": "OKTA"
}'
```
```json
{
    "id": "opfgntwrfgEajTaxD0h7",
    "factorType": "push",
    "provider": "OKTA",
    "vendorName": "OKTA",
    "status": "PENDING_ACTIVATION",
    "created": "2018-10-16T15:11:10.000Z",
    "lastUpdated": "2018-10-16T15:11:10.000Z",
    "_links": {
        "self": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        },
        "poll": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/poll",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "user": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    },
    "_embedded": {
        "activation": {
            "expiresAt": "2018-10-16T15:21:10.000Z",
            "factorResult": "WAITING",
            "_links": {
                "qrcode": {
                    "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/qr/20111_1m5LbMgotFosg1J4zjWMb1h4K51g30eb_niPbBHJhjP72hI7H",
                    "type": "image/png"
                },
                "send": [
                    {
                        "name": "email",
                        "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/email",
                        "hints": {
                            "allow": [
                                "POST"
                            ]
                        }
                    },
                    {
                        "name": "sms",
                        "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/sms",
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
```
### Poll to see if user has accepted the enrollment
```bash
curl -X POST \
  https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/poll \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Postman-Token: d2fc48d1-6501-4478-baf9-19a0da2743b5' \
  -H 'cache-control: no-cache'
```
```json
{
    "expiresAt": "2018-10-16T15:21:10.000Z",
    "factorResult": "WAITING",
    "_links": {
        "send": [
            {
                "name": "email",
                "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/email",
                "hints": {
                    "allow": [
                        "POST"
                    ]
                }
            },
            {
                "name": "sms",
                "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/sms",
                "hints": {
                    "allow": [
                        "POST"
                    ]
                }
            }
        ]
    }
}
```
### Activate Okta Verify with Push by sending an SMS
The factor enrollment response includes 3 options for activating Okta Verify with Push. QR code, email with an activation link, and SMS with an activation link. 
```bash
curl -X POST \
  https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/sms \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: e251436a-ec60-48ac-84b7-7f6eaf94fbba' \
  -H 'cache-control: no-cache' \
  -d '{
	"profile": {
		"phoneNumber": "5052694259"
	}
}'
```
The user will receive an SMS message at the specified phoneNumber: "click here to enroll Okta Verify: [activation URL]". Upon clicking the link, the browswer will open to an Okta landing page. If the user does not already have the Okta Verify application installed on their mobile device, they will be presented with button to download the app from the apps store. If they already have it installed, they will see the "Open in Okta Verify?" dialog.
```bash
{
    "id": "opfgntwrfgEajTaxD0h7",
    "factorType": "push",
    "provider": "OKTA",
    "vendorName": "OKTA",
    "status": "PENDING_ACTIVATION",
    "created": "2018-10-16T15:11:10.000Z",
    "lastUpdated": "2018-10-16T15:36:23.000Z",
    "_links": {
        "self": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        },
        "poll": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/poll",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "user": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    },
    "_embedded": {
        "activation": {
            "expiresAt": "2018-10-16T15:46:23.000Z",
            "factorResult": "WAITING",
            "_links": {
                "send": [
                    {
                        "name": "email",
                        "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/email",
                        "hints": {
                            "allow": [
                                "POST"
                            ]
                        }
                    },
                    {
                        "name": "sms",
                        "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/sms",
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
```
Again, you'll need to poll the _poll_ URL in the response until the status changes from "PENDING_ACTIVATION" to another state.
```bash
curl -X POST \
  https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/lifecycle/activate/poll \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Postman-Token: 0a1f6e99-bcf8-4629-bb3b-e621ea2b5565' \
  -H 'cache-control: no-cache'
```
```json
{
    "id": "opfgntwrfgEajTaxD0h7",
    "factorType": "push",
    "provider": "OKTA",
    "vendorName": "OKTA",
    "status": "ACTIVE",
    "created": "2018-10-16T15:11:10.000Z",
    "lastUpdated": "2018-10-16T15:36:56.000Z",
    "profile": {
        "credentialId": "TPR1539701808",
        "deviceType": "SmartPhone_IPhone",
        "keys": [
            {
                "kty": "PKIX",
                "use": "sig",
                "kid": "default",
                "x5c": [
                    "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3zdmobWWcCn0w1Oi/fBxCpoJLYLBJLNZWlY6iVfIxKZHsr+AxGtsEusL/dim4akQxh/98Zrb7YkNOUPRQ+mem7uy64PS1zFPaJkFJHzB3uermqbzPwFaAe45hcW7TUZTKucDmmmflw2ZeLKKoLN+BAZ1iLl9hRE2tbsd/49Z5Uub3aJBbHsncfhI2+tSwpfXURYHxStf1k/p2D77X3aY26eTTW+PBb3xCbAXyTtT9vC3HqBaXeIAIisOqpmMpd352KrfgnpkBZZH8qiYTNqndEKJnEjWkUKkk+G6k+4RbDEanx2YJZTwxrdzANnKItNBx1oVCyIV53JmEDLJG56kcQIDAQAB"
                ]
            }
        ],
        "name": "Brent’s iPhone",
        "platform": "IOS",
        "version": "12.0"
    },
    "_links": {
        "self": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7",
            "hints": {
                "allow": [
                    "GET",
                    "DELETE"
                ]
            }
        },
        "verify": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/factors/opfgntwrfgEajTaxD0h7/verify",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "user": {
            "href": "https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    }
}
```
## Convert User from FEDERATION to OKTA provider
The user we created using the Create User with Authorization Provider is an active user, but it does not have a password and cannot authenticate with Okta. In the future, if you want to leverage the Okta platform for funcitonality beyond MFA, you can convert the user to a regular Okta-mastered user type with the _Reset Password_ API, which will change the users provider attribute from FEDERATION to OKTA. You can then either use the _resetPasswordUrl_ in the response to redirect the user to the Okta self-service password flow, or you can set the user's password programatically by calling the _Set Password_ API.
```bash
curl -X POST \
  'https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7/lifecycle/reset_password?sendEmail=false' \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: acab2c5b-eb85-4b61-84c4-e6ad556e5d56' \
  -H 'cache-control: no-cache'
```
### Redirect user to self-service password setup
```json
{
    "resetPasswordUrl": "https://dev-123456.oktapreview.com/reset_password/hh4HFNsgK4ukIpEJFyY4"
}
```
### Or, set the user's password programatically
```bash
curl -X PUT \
  https://dev-123456.oktapreview.com/api/v1/users/00ugntseevo3Te3zZ0h7 \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 00ma8D_zzzzzz_PFR5ete' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 2c30a76d-0b54-4b03-88b9-0c45bb0587ec' \
  -H 'cache-control: no-cache' \
  -d '{
  "credentials": {
    "password" : { "value": "D3m0demo1" }
  }
}'
```
