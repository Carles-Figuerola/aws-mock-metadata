aws-mock-metadata
=================

Mock EC2 metadata service that can run on a developer machine.

It is possible in AWS to protect [API access with
MFA](http://docs.aws.amazon.com/IAM/latest/UserGuide/MFAProtectedAPI.html).
However, this is not convenient to setup or use since you must regularly refresh
your credentials.

Amazon EC2 instances have a metadata service that can be accessed to
retrieve the instance profile credentials. Those credentials are
rotated regularly and have an associated expiration. Most AWS SDKs
check this service to use the instance profile credentials.

This service runs on a developer machine and masquerades as the
Amazon metadata service. This allows it to integrate with the SDKs
and provide session credentials when needed. When the metadata
service is queried for security credentials, this service can prompt
the user to enter the MFA device token or generate the OTP code (if
the virtual MFA device secret is provided). Those credentials are cached
until they expire and the user is prompted again to provide an updated
token.

# Dependencies

This application is still beta and may change in breaking ways in the
future.

* python 2
  * python 3 has not been tested
* boto python library
  * this may change to [botocore](https://github.com/boto/botocore), which
    will impact config file format

# Installation

Currently only works on OSX, but should be trivial to make it work on
other platforms.

This should be changed to run as a launchd daemon at some point.

1. Clone the repo
2. Add AWS access key and secret key to *server.conf* in the repo
   directory
  * See Configuration section
  * These permanent keys are used to generate the session keys using the MFA token
  * There must be an MFA device associated with the account
3. Run `bin/server-macos`
  * Prompts for password to setup an IP alias and firewall forwarding rule.
  * You can examine the script and run the commands separately. The
    script is only provided for convenience.

# Configuration

*SERVICE_HOME/server.conf*<br>
*~/.aws-mock-metadata/config*

```
[aws]
access_key=...   # Optional. Access key to generate temp keys with.
                 # If not provided, the service falls back on boto's
                 # credential discovery.
secret_key=...   # Optional. Secret key to generate temp keys with.
                 # If not provided, the service falls back on boto's
                 # credential discovery.
region=us-east-1 # Optional. Not generally necessary since IAM and STS
                 # are not region specific. Default is us-east-1.
mfa_secret=...   # Optional. Base32 encoded virtual MFA device secret.
                 # If set, the metadata server will generate OTP codes
                 # internally instead of showing a prompt. The secret
                 # is provided when setting up the virtual mfa device.

[metadata]
host=169.254.169.254 # Optional. Interface to bind to. Default is
                     # 169.254.169.254.
port=45000           # Optional. Port to bind to. Default is 45000.
token_duration=43200 # Optional. Timeout, in seconds, for the generated
                     # keys. Minimum is 15 minutes.
                     # Default is 12 hours for sts:GetSessionToken and
                     # 1 hour for sts:AssumeRole. Maximum is 36 hours
                     # for sts:GetSessionToken and 1 hour for
                     # sts:AssumeRole. If you specify a value higher
                     # than the maximum, the maximum is used.
role_arn=arn:aws:iam::123456789012:role/${aws:username}
                     # Optional. ARN of a role to assume. If specified,
                     # the metadata server uses sts:AssumeRole to create
                     # the temporary credentials. Otherwise, the
                     # metadata server uses sts:GetSessionToken.
                     # The string '${aws:username}' will be replaced
                     # with the name of the user requesting the
                     # credentials. No other variables are currently
                     # supported.
profile=...          # Name of the initial profile to select. Default is
                     # "default".

# Define a profile. A profile consists of a set of options
# used to create a session.
# Multiple profiles can be defined and switched on-the-fly.
# Default option values are taken from the [aws] and [metadata]
# sections.
[profile:NAME]  # NAME is a string used to identify the profile
access_key=...
secret_key=...
region=...
token_duration=...
role_arn=...
```

## AWS CLI

If you don't provide the MFA secret and use a separate device to generate
OTP codes, it is recommended to update the local metadata service timeout
for the AWS command line interface. This ensures that you have enough time
to enter the MFA token before the request expires and your script can
continue without interruption.

*~/.aws/config*

```
[default]
metadata_service_timeout = 15.0  # 15 second timeout to enter MFA token
```

# Mock Endpoints

The following EC2 metadata service endpoints are implemented.

```
169.254.169.254/latest/meta-data/iam/security-credentials/
169.254.169.254/latest/meta-data/iam/security-credentials/local-credentials
```

If the MFA device secret is not provided and 
`/latest/meta-data/iam/security-credentials/local-credentials` is
requested and there are no session credentials available, a dialog pops
up prompting for the MFA token. The dialog blocks the request until the
correct token is entered. Once the token is provided, the session
credentials are generated and cached until they expire. Once they
expire, a new token prompt will appear on the next request for
credentials.

# User Interface

A simple UI to view, reset, and update the session credentials is
available by loading *http://169.254.169.254/manage* in your
browser.

# API

There is an API available to query and update the MFA session
credentials.

## Get Profiles

`GET 169.254.169.254/manage/profiles`

### Response: 200

```json
{
    "profiles": [
        {
            "name": "...",
            "accessKey": "...",
            "region": "...",
            "tokenDuration": 123, // Optional
            "roleArn": "...",     // Optional
            "session": {  // Optional, if there is no active session for the profile
                "accessKey": "....",
                "secretKey": "....",
                "sessionToken": "...",
                "expiration": "2014-04-09T09:00:44Z"
            },
        }
    ]
}
```

## Get Profile

`GET 169.254.169.254/manage/profiles/NAME`

### Response: 404

* Profile does not exist.

### Response: 200

```json
{
    "profile": {
        "name": "...",
        "accessKey": "...",
        "region": "...",
        "tokenDuration": 123, // Optional
        "roleArn": "...",     // Optional
        "session": {  // Optional, if there is no active session for the profile
            "accessKey": "....",
            "secretKey": "....",
            "sessionToken": "...",
            "expiration": "2014-04-09T09:00:44Z"
        },
    }
}
```

## Get Credentials

`GET 169.254.169.254/manage/session`

### Response: 200

Returns current session information.

```json
{
    "session": {  // Optional, if there is no active session for the profile
        "accessKey": "....",
        "secretKey": "....",
        "sessionToken": "...",
        "expiration": "2014-04-09T09:00:44Z"
    },
    "profile": {
        "accessKey": "...",
        "region": "...",
        "name": "...",
        "tokenDuration": 123, // Optional
        "roleArn": "..."      // Optional
    }
}
```

## Clear Credentials

`DELETE 169.254.169.254/manage/session`

### Response: 200

Session credentials were cleared or no session exists.

## Create Credentials

`POST 169.254.169.254/manage/session`

Create a new session, change the current profile, or both.

Examples:
```bash
curl -X POST -d token=123456 169.254.169.254/manage/session
curl -X POST -d session=other 169.254.169.254/manage/session
curl -X POST -d 'token=123456&session=other' 169.254.169.254/manage/session
```

### Body

*Content-Type: application/x-www-form-urlencoded*

```
token=123456&session=other
```

*Content-Type: application/json*

```json
{
    "token": "123456",
    "session": "other"
}
```

### Response: 400

* Invalid MFA token string format. Must be 6 digits.
* The specified profile name does not exist.
* Neither profile nor token was specified.

### Response: 403

* Specified MFA token does not match expected token for the MFA device.

### Response: 200

Session was created successfully and/or profile the was changed. Returns the
session information. The response will contain a session if a valid token was
provided. The response will not contain a session if only a new profile name was
specified and the profile does not have an existing session.

```json
{
    "session": {  // Optional, if there is no active session for the profile
        "accessKey": "....",
        "secretKey": "....",
        "sessionToken": "...",
        "expiration": "2014-04-09T09:00:44Z"
    },
    "profile": {
        "accessKey": "...",
        "region": "...",
        "name": "...",
        "tokenDuration": 123, // Optional
        "roleArn": "..."      // Optional
    }
}
```
# License

The MIT License (MIT)
Copyright (c) 2014 Cory Thomas

See [LICENSE](LICENSE)
