---
date: 2021-03-17
dateName: '17 March 2021'
title: 'Switching from flask to gatsby'
slug: switching-from-flask-to-gatsby
tags:
  - Cloud
  - Firebase
---


In 2016 when I started this site I [wrote about](https://donchev.is/post/about-this-site) the tech stack I choose. 

Yesterday I switched from servers to serverless using [Gatsby](https://www.gatsbyjs.com/) and [firebase hosting](https://firebase.google.com/docs/hosting)


Feels good to leave the db and containers behind and have gatsby build a static website whenever I make changes.

With firebase I can review all my changes before promoting them to production (https://donchev.is) How? 

No, I dont have seperate projects for dev / prod in firebase. I just use their recently released feature "hosting channels" ;)


Here is how:

```
firebase --project donchev-is hosting:channel:deploy $(uuidgen) --expires 1h


✔  hosting:channel: Channel URL (donchev-is): https://donchev-is--a5894fa1-1982-49ce-9513-e3135fd2e866-nw2h141t.web.app [expires 2021-03-17 14:25:01]
```

Now I can view my changes on the newly created channel URL

*https://donchev-is--a5894fa1-1982-49ce-9513-e3135fd2e866-nw2h141t.web.app*

(will probably be expired by the time you read this)

I can even run UI tests on that link if I am too lazy doing the testing myself, here is how I do it:

====


```
"HEADLESS=false WEBSITE=https://donchev-is--a5894fa1-1982-49ce-9513-e3135fd2e866-nw2h141t.web.app yarn test:e2e",

```

this triggers the e2e tests written with [jest-puppeteer](https://github.com/smooth-code/jest-puppeteer)

```js
const puppeteer = require('puppeteer')

describe('donchev-is', () => {
  beforeAll(async () => {
    jest.setTimeout(15000)
    const website = process.env.WEBSITE || 'https://donchev.is'
    await page.goto(website)
    await page.setViewport({ width: 1712, height: 1283 })
  })

  it('should work :) ', async () => {
    // write your e2e tests here with 
  })
})
```

<br />

My new dev workflow goes something like this:

1. Check in code
2. Run **yarn build**
3. Run **yarn deploy:dev** (creates a firebase hosting channel that expires in 1h)
4. Run **WEBSITE="hostinChannelUrl" yarn test:e2e**
5. Run **yarn deploy:prod**



No more servers and no more ansible playbooks for patching my servers ;)


