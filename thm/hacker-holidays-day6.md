# THM - Hacker Holidays Day 6: Overheard at Breakfast
**Category:** OSINT
**Difficulty:** Easy

## Tools Used
- Image viewer
- md5sum
- Browser
- base64

## Steps
1. Downloaded and extracted task files — found `conversation.png`
2. Analyzed the screenshot of a conversation between Ponzi and Lambo
3. Identified key clues:
   - Email address: `lambobytelotushotel@gmail.com`
   - Mentioned using a free profile linking tool "starting with a G"
   - Said she wiped everything but the account might still exist
4. Identified the service as **Gravatar** — a profile service starting with G that links social accounts via email
5. Gravatar uses MD5 hash of email as profile URL — hashed the email:
   `echo -n "lambobytelotushotel@gmail.com" | md5sum`
6. Got hash: `d4a5fc5d3128890778667e24617d7cc0`
7. Visited: `https://gravatar.com/d4a5fc5d3128890778667e24617d7cc0`
8. Found Base64 encoded string in the profile description
9. Decoded it: `echo "<base64_string>" | base64 -d`
10. Retrieved the flag

## What I Learned
- OSINT involves extracting clues from conversations, images, and public profiles
- Gravatar links profiles to email addresses using MD5 hashes
- Even "wiped" accounts can leave traces on third-party services
- Profile descriptions and metadata can contain hidden information
- Base64 is commonly used to encode flags and data in CTF challenges
