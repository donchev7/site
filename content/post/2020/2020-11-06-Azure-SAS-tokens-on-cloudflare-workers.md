---
date: 2020-05-23
title: 'Azure SAS tokens on cloudflare workers'
slug: azure-sas-token-on-cloudflare
toc: false
tags:
  - Cloud
  - Azure
  - TypeScript
---

I have been playing with cloudflare workers a lot recently. I fell in love with the tech. Although you need a different attitude when using cloudflare workers than when building NodeJS apps. For starters, cloudflare workers run as service workers on the edge, which means you can't use all NodeJS libraries out there.

In my opinion this is also gives cloudflare workers an advantage. We are so used to doing npm install in NodeJS world that we almost forgot how to write a left padding algorithm ourself :)

With cloudflare workers we have more constraints and have to think about the libraries we include carefully. These constraints made me start writing the libraries my applications needed myself instead of relying on npm for everything. And it has been refreshing and fun :)

Just recently I wrote a library that generates Azure SAS tokens. 

> Azure SAS token is a string that gives your clients temporaily access to Azure storage


![azure_sas_token](https://github.com/NeuroCode-io/azure-sas-token/raw/master/sas-storage-uri.png)


Microsoft provides a javascript SDK that can generate SAS tokens but unfortunately it only works for NodeJS and not in the service workers and browsers.

I decided to write a library that can generate SAS tokens in service workers and is compatible with browsers and cloudflare edge workers. For this I used the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API). You can view the code on [GitHub](https://github.com/neurocode-io/azure-sas-token/blob/master/src/index.ts). I wrote it in typescript, so you can use it with javascript or typescript.

The API of the library is also pretty simple. Just call createBlobSas and use the returned string to store / view files in Azure.


For example, Imagine you have someBlob.txt file stored in azure storage and you want to give somebody access to view the file for five minutes. You could use the following code snippet:

<!--more-->

```javascript
import { createBlobSas } from '@neurocode.io/azure-sas-token'

const expireInMin = 5

const { blobSasUrl } = await createBlobSas({
      accountKey: 'yourStorageAccountKey',
      accountName: 'yourAccountName',
      containerName: 'youStorageContainerName',
      blobName: 'someBlob.txt',
      permissions: 'r',
      expiresOn: new Date(new Date().valueOf() + expireInMin * 60 * 1000)
})
```

As of now the library only issues Blob SAS tokens. If you need the capability to issue container and account SAS token. Let me know! Or event better help out by contributing a PR on [GitHub](https://github.com/neurocode-io/azure-sas-token).

