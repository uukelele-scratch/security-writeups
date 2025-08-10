# security-writeups
A collection of my security vulnerability disclosures and write-ups.

**Leaks**
- [World of Walker data leak](world-of-walker-data-leak.md)
  - Merely minutes after launch, I discovered a critical flaw in the world of walker site, that could leak any user's full name, email address, country, and date of birth - simply from their walker ID.
  - **TL;DR:** The World of Walker API exposed personal user data including full names, emails, country, and D.O.B. via unauthenticated queries using Walker IDs or Clerk IDs. This was discovered minutes after launch and responsibly disclosed, with the issue fixed within hours.
