---
title: "Unleash custom authentication with Google Identity Aware Proxy"
date: 06-01-2023
tags: "typescript,feature flag,google identity aware proxy"
---

# Unleash custom authentication with Google Identity Aware Proxy

I would be sharing notes from my spike in enabling custom authentication for open source feature flag software named [Unleash](https://getunleash.io). 

Since  we are internally using [Google Identity Aware Proxy (IAP)](https://cloud.google.com/iap) in our organization, we gonna need to perform some verification to ensure that the JWT token generated on IAP is valid when it receives the backend service. Google provides documentation on [how we can secure our app with its signed header](https://cloud.google.com/iap/docs/signed-headers-howto). What you will be seeing on this note later on is how we use google [google-auth-library](https://www.npmjs.com/package/google-auth-library) npm package to do verification required for user authentication in Unleash server.

If we decided to use unleash custom authentication, we will need to build our custom unleash server, package and build it as docker image (if necessary), and do the deployment by ourself. To provide custom authentication, we are going to specify the custom auth handler by providing it to `authentication.customAuthHandler`. For more detail, read [Implementing Custom Authentication](https://docs.getunleash.io/reference/deploy/securing-unleash#implementing-custom-authentication)

```typescript
import unleash, { IAuthType, LogLevel} from 'unleash-server';
import { googleIapAuthentication } from './google-iap';

export function start(): void {
    unleash.start({
      authentication: {
        type: IAuthType.CUSTOM,
        customAuthHandler: googleIapAuthentication,
      },
    })
    .then((unleash: any) => {
      logger.info(`Unleash started on http://localhost:${unleash.app.get('port')}`);      
    });
}

start()
```

IAP will always sends HTTP header with key `x-goog-iap-jwt-assertion` when forwarding request to the backend service. We will use this to verify whether this JWT token is valid by using `google-auth-library`. 

In order to verify, you will need to provide:
* IAP public keys by using `oAuth2Client.getIapPublicKeys()`.
* IAP target audience that you can retrieve from IAP dashboard for the related backend service.
* Issuer: `'https://cloud.google.com/iap'`.

Here is how we verify the JWT sent by Google IAP.

```typescript
import { Response, NextFunction, Express } from 'express';
import { OAuth2Client, LoginTicket } from "google-auth-library"
import config from './config';

export const verify = async (iapJwt: string): Promise<LoginTicket> => {  
  const oAuth2Client = new OAuth2Client()  
  const response = await oAuth2Client.getIapPublicKeys()
  const ticket: LoginTicket = await oAuth2Client.verifySignedJwtWithCertsAsync(
    iapJwt,
    response.pubkeys,
    "/projects/xxxx/global/backendServices/yyyyyyy", // IAP Target Audience
    ['https://cloud.google.com/iap']
  )
  return ticket
}
```

When writing custom authentication, you can decide about what to do for any particular route by providing nodejs middleware. In my case, I'm protecting all routes with prefix `/api/admin` with Google IAP. Other route such as `/health` which is meant for health check is not protected since it might be used internally by kubernetes service to perform health check in the cluster.

Once the JWT is verified, we are going to read the email from JWT payload and use `userService.loginUserWithoutPassword()` method to sign the user in. This method is provided by the services passed as parameters on the custom auth handler signature. Once we get the user, we simply add the user into the request session so that it can be used later by the handler to perform authorization and others.

```typescript
export const googleIapAuthentication = (app: Express, config, services) => {
  const { userService } = services;

  app.use('/health', async (req: any, res: Response, next: NextFunction) => {
    return next();
  });

  app.use('/api/admin', async (req: any, res: Response, next: NextFunction) => {
    const iapJwt: string = req.headers['x-goog-iap-jwt-assertion'] ? req.headers['x-goog-iap-jwt-assertion']?.toString() : ''
    try {
      let ticket = await verify(iapJwt);
      const userEmail: string | undefined = ticket.getPayload()?.email;
      if (!userEmail) {
        throw new Error("access denied")
      }
      const user = await userService.loginUserWithoutPassword(userEmail!, true);
      req.user = user;
      req.session.user = user;
      return next();
    } catch(error) {
      return res.status(401).send("access denied");
    }
  }); 
}
```

That's it. Thanks for reading my first note about javascript/typescript! lol