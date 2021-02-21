---
title: "What happened to the accounts system ?"
---

The old account system has been nuked. It was pretty terrible, used PHP, usernames instead of UUIDs, used poor quality hashing by today's standards, and generally is not a good fit for the project. The reason this system was centralized in the first place was the game did not start out as FOSS software, and this was a holdover ever since.