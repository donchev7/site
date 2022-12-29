---
date: 2022-12-28
image: 'post/2022/github_app.png'
title: 'How to authenticate using a GitHub App'
slug: how-to-authenticate-using-a-github-app
toc: false
tags:
  - github
---

## Introduction

Recently I was working on moving a bitbucket bot to GitHub. With bitbucket, I can use a personal access token to authenticate with the API. It seems that this personal access token was valid forever? Which can be a good thing, but also a bad thing. With GitHub you can create a bot account and issue an access token for that account. But this token is only valid for max 1 year. After a brief research I encountered GitHub apps. Which are basically webhooks and bot accoun with granular permissions installed on a GitHub organization. However, getting the authentication right was a bit tricky. In this post I will try to explain how to authenticate with a GitHub app.

## Webhook secret, private key, JWT and access token

When I first started to look into GitHub apps I must admit that I was a bit confused. There are a lot of different keys and tokens. Let me try to explain what they are and how they are used.


### Webhook secret

Whenever a GitHub app needs to subscribe to events from one ore multiple repositores I must create a webhook secret. This secret is used to verify that the webhook request is coming from GitHub. This is a good thing, because it prevents other people from sending fake webhook requests to my app. I can create a webhook secret in the GitHub app settings. Lets quickly go over how to verify the webhook request.

I created a simple decorator that I can use to verify the webhook request:


```python
import functools
import hmac
import hashlib
import os

API_SECRET = os.environ["GITHUB_WEBHOOK_SECRET"]

def auth(f):
    @functools.wraps(f)
    def wrapper(req, *args, **kwargs):
        logging.info('Verifying request')
        got = req.headers.get('X-Hub-Signature-256')
        if not got:
            raise VerifyError('No signature found')

        payload = req.get_body()
        want = 'sha256=' + hmac.new(bytes(API_SECRET, 'latin-1'), msg = payload, digestmod = hashlib.sha256).hexdigest()
        isCorrect = hmac.compare_digest(got, want)
        if not isCorrect:
            raise VerifyError()
        logging.info('Request verified')
        
        return f(req, *args, **kwargs)
    return wrapper
```

The decorator is pretty simple. It gets the `X-Hub-Signature-256` header from the request and compares it with the signature that we computed using the webhook secret and payload. 

If the signatures match we know that the request is coming from GitHub. If the signatures don't match we raise an (custom) exception.

Note that the decorator was created for Azure Functions, but it can be easily adapted to other frameworks. This is how I use the decorator in my azure function handler:

```python
@auth
def main(req: func.HttpRequest) -> func.HttpResponse:
    return handler(req)
```

### Private key, JWT and access token

Now that I have verified the origin (GitHub) I can start calling the GitHub API with the APP. In order for me to do this I need to create a JWT (JSON Web Token) and use that to authenticate with the GitHub API. The JWT is created using the private key that I downloaded from the GitHub app settings. After I create a JWT I **must** use that JWT to get an access token. I can then use this access token to call the GitHub API.

Lets see how we can create a JWT and use that to get an access token.

First you'll need to install a dependency `cryptography`:

```bash
pip install cryptography
```

```python
import hmac
import hashlib
import requests
from datetime import datetime, timedelta
from cryptography.hazmat.primitives import serialization

APP_ID='YOUR_APPLICATION_ID'
APP_INSTALLATION_ID='YOUR_APPLICATION_INSTALLATION_ID'


class Auth(object):
    def __init__(self, private_key):
        self.private_key = private_key

    def get_jwt(self):
        due_date = datetime.now() + timedelta(minutes=10) # 10 minutes from now
        expiry = int(due_date.timestamp())
        payload = {
            'iat': int(datetime.now().timestamp() - 60), # 1 minute ago
            'exp': expiry,
            'iss': APP_ID
        }
        priv_rsakey = serialization.load_pem_private_key(self.private_key.encode('utf8'), password=None)

        return jwt.encode(payload, priv_rsakey, algorithm='RS256')

    def get_accesstoken(self):
        token = self.get_jwt()
        resp = requests.post(self.url, headers={'Authorization': f'Bearer {token}'})
        if not resp.ok:
            raise Exception('Failed to get access token')

        return resp.json()['token']
```

Note that we can only create a JWT that is valid for 10 minutes. Also, because of clock skew, I created the JWT 1 minute ago. This way I can be sure that the JWT is valid when I exchange it for an access token.


The private key that I pass to the Auth class can come from the environment:

```python

import os

private_key = os.environ['GITHUB_PRIVATE_KEY']
auth = Auth(private_key)
access_token = auth.get_accesstoken()

# Use the access token to call the GitHub API

```

I have stored my private key in KeyVault and load it into the environment variable `GITHUB_PRIVATE_KEY`. Note that if you store your private key in a `.env` file or KeyVault you need to add newline chars .e.g `\n` in order for it to work:

```.env
GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
.....
.....
"
```

or

```.env
GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n....\n....\n"
```


## Conclusion

As you can see there are a lot of steps involved in authenticating with a GitHub app. I am also only scratching the surface here. Theoretically you can also create an Oauth GitHub app and let users log into your app using their GitHub account and issue API calls on their behalf. But that is a topic for another post.


What surprised me the most and what took me the longest to figure out was the ACCESS TOKEN. I tried using the JWT to call the GitHub API, but that did not work. I had to use the access token. I hope this post will help you get started with GitHub apps. If you have any questions or comments you can reach me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev).
