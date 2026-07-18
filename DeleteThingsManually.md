1. Amazon Bedrock Console
Agents (Bedrock → Agents):

booking-agent — delete this (deletes its aliases TestAlias and the default TSTALIASID automatically, and its action group TableBookingsActionGroup with it)
Also check for any leftover test agents that may not have been cleaned up: booking-agent-test45eu, booking-agent-test — I gave you delete commands for these earlier, but double-check they're actually gone from the list

Knowledge Bases (Bedrock → Knowledge Bases):

booking-agent-kb — delete this (deletes its data source too)

2. Amazon OpenSearch Service Console → Serverless → Collections

Collection: bedrock-sample-rag-8814 — this is the one that bills hourly, don't leave it behind

Same page, tab for Security policies:

Encryption policy: bedrock-sample-rag-sp-8814
Network policy: bedrock-sample-rag-np-8814

Tab for Data access policies:

bedrock-sample-rag-ap-8814

3. AWS Lambda Console

Function: booking-agent-lambda

4. DynamoDB Console → Tables

Table: restaurant_bookings

5. Amazon S3 Console

Bucket: booking-agent-eu-central-1-ID — empty it first (select all objects → delete), then delete the bucket itself

6. IAM Console
Roles (IAM → Roles):

AmazonBedrockExecutionRoleForAgents_booking-agent
AmazonBedrockExecutionRoleForKnowledgeBase_8814
booking-agent-lambda-role

Policies (IAM → Policies → filter "Customer managed"):

booking-agent-ba
booking-agent-dynamodb-policy
AmazonBedrockOSSPolicyForKnowledgeBase_8814
AmazonBedrockS3PolicyForKnowledgeBase_8814
AmazonBedrockFoundationModelPolicyForKnowledgeBase_8814

(Deleting a role in the console will prompt you to detach its policies first, or auto-handle it — either way, delete these managed policies after the roles above are gone, or the console will just detach automatically.)
Your own user's inline policy — this one's easy to miss since it's not in the main Policies list:

IAM → Users → vmallya → Permissions tab → look under "Permissions policies" for an inline policy named AllowBedrockInvokeAgent → delete it (this was the one I had you add directly to your user earlier in this thread)
