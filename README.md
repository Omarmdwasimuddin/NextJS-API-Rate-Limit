## NextJS API: Rate limit

#### Visit koro: https://upstash.com/ ---> login koro (account agee theke thakle click koro: https://console.upstash.com/) ---> Create Database e click koro---> value set koro--->Next click koro---> abar Next click koro---> Create click koro.
![](https://imgur.com/xS7IDVR.png)
![](https://imgur.com/Sb8UDQw.png)

---

#### Copy koro & .env te paste koro.
![](https://imgur.com/4WYqOJQ.png)

---

#### Package Install
```bash
npm install @upstash/ratelimit @upstash/redis
```
---

### lib/rate-limit.ts
```bash
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";
import { env } from "./validations/env";


const redis = new Redis({
    url: env.UPSTASH_REDIS_REST_URL,
    token: env.UPSTASH_REDIS_REST_TOKEN,
});

// IP based: একটা IP থেকে 10 বার/1min এর বেশি না
export const ipRateLimit = new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(10, "1 m"),
    analytics: true,
    prefix: "ratelimit:login:ip",
});

// Email based: একটা নির্দিষ্ট email এ 5 বার/1min এর বেশি না (stricter, কারণ এটাই মূল brute-force target)
export const emailRateLimit = new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(5, "1 m"),
    analytics: true,
    prefix: "ratelimit:login:email",
});
```
---

###
```bash

```
---
