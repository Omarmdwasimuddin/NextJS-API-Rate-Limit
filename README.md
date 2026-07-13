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

### lib/get-client-ip.ts
```bash
import { NextRequest } from "next/server";

export function getClientIp(request: NextRequest): string {
  const forwardedFor = request.headers.get("x-forwarded-for");
  if (forwardedFor) {
    return forwardedFor.split(",")[0].trim();
  }
  const realIp = request.headers.get("x-real-ip");
  if (realIp) return realIp;
  return "unknown";
}
```
---

###
```bash
import { NextRequest, NextResponse } from "next/server";
import prisma from "@/lib/prisma";
import { log } from "@/lib/logger"
import { handlePrismaError } from "@/lib/errors/handlePrismaError";
import { loginSchema } from "@/lib/validations/user.schema";
import { verifyPassword } from "@/lib/auth/password";
import { signAccessToken, generateRefreshToken } from "@/lib/auth/tokens";
import { ipRateLimit, emailRateLimit } from "@/lib/rate-limit";
import { getClientIp } from "@/lib/get-client-ip";

export async function POST(request: NextRequest) {
    const requestId = crypto.randomUUID();
    try {
        log.info(
            {
                requestId,
                path: request.nextUrl.pathname,
                method: request.method,
            },
            "Login data received"
        )

        // ---- Step 1: IP rate limit check (body parse করার আগেই, কারণ এটা cheapest check) ----
        const clientIp = getClientIp(request);
        const ipResult = await ipRateLimit.limit(clientIp);

        if (!ipResult.success) {
            log.warn(
                {
                    requestId,
                    ip: clientIp,
                    remaining: ipResult.remaining,
                    reset: ipResult.reset,
                },
                "IP rate limit exceeded on login"
            )
            return NextResponse.json(
                {
                    status: "fail",
                    requestId,
                    message: "Too many login attempts. Please try again later.",
                },
                {
                    status: 429,
                    headers: {
                        "Retry-After": Math.ceil((ipResult.reset - Date.now()) / 1000).toString(),
                    },
                }
            )
        }

        const jsonBody = await request.json();
        const validation = loginSchema.safeParse(jsonBody);

        if (!validation.success) {
            log.warn(
                {
                    requestId,
                    errors: validation.error.flatten().fieldErrors,
                },
                "Invalid Input"
            )
            return NextResponse.json(
                {status: "fail", requestId, message: "Invalid Input", errors: validation.error.flatten().fieldErrors},
                {status: 400}
            )
        }

        const { email, password } = validation.data;

        // ---- Step 2: Email rate limit check (body validate হওয়ার পর, কারণ email লাগবে) ----
        const emailResult = await emailRateLimit.limit(email);

        if (!emailResult.success) {
            log.warn(
                {
                    requestId,
                    email,
                    ip: clientIp,
                    reset: emailResult.reset,
                },
                "Email rate limit exceeded on login"
            )
            return NextResponse.json(
                {
                    status: "fail",
                    requestId,
                    message: "Too many attempts for this account. Please try again later.",
                },
                {
                    status: 429,
                    headers: {
                        "Retry-After": Math.ceil((emailResult.reset - Date.now()) / 1000).toString(),
                    },
                }
            )
        }

        const user = await prisma.user.findUnique({
            where: { email },
        });

        if (!user || !(await verifyPassword(user.passwordHash, password))) {
            log.warn(
                {
                    requestId,
                    email,
                },
                "Failed login attempt"
            )
            return NextResponse.json(
                {status: "fail", requestId, message: "Invalid email or password"},
                {status: 401}
            )
        }

        const accessToken = await signAccessToken(user.id, user.role);
        const { raw: refreshRaw, hashed: refreshHashed } = generateRefreshToken();

        await prisma.refreshToken.create({
            data: {
                tokenHash: refreshHashed,
                userId: user.id,
                expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
            },
        });

        const response = NextResponse.json({
            status: "success",
            user: {
                id: user.id,
                name: user.name,
                email: user.email,
            },
        });

       response.cookies.set("access_token", accessToken, {
        httpOnly: true,
        secure: process.env.NODE_ENV === "production",
        sameSite: "lax",
        path: "/",
        maxAge: 60 * 15,
       }) 

       response.cookies.set("refresh_token", refreshRaw, {
        httpOnly: true,
        secure: process.env.NODE_ENV === "production",
        sameSite: "lax",
        path: "/api/auth",
        maxAge: 60 * 60 * 24 * 7,
       })

       log.info(
            {
                requestId,
                userId: user.id,
                email: user.email,
            },
            "User logged in"
        )
        
        return response;
        
    } catch (error) {
        const { status, message } = handlePrismaError(error);
        log.error(
        { 
            requestId,
            err: error instanceof Error 
                ? { message: error.message, stack: error.stack, name: error.name }
                : String(error),
        },
        message
    )
    return NextResponse.json(
        { status: "fail", requestId, message },
        { status }
    )
    }
}
```
---
