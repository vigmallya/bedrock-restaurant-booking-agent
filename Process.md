# Building a Bedrock Agent with Knowledge Base & Action Groups
## Deep-Dive Code Explanation, Pipeline & Purpose

---

## 1. What Is this code About?

This code teaches how to build a **fully functional Amazon Bedrock Agent** that combines two key capabilities:

1. **Action Groups** — The agent can *do* things (create/get/delete restaurant bookings via AWS Lambda + DynamoDB)
2. **Knowledge Base (RAG)** — The agent can *know* things (answer questions about menus using documents stored in S3 → indexed in OpenSearch Serverless)

This is the classic **RAG + Tool Use** pattern for agentic AI systems.

---

## 2. File Overview

| File | Role |
|---|---|
| `agent.py` | Orchestration script — creates IAM roles, Lambda function, DynamoDB table, and the Bedrock Agent |
| `lambda_function.py` | The actual business logic that runs when the agent calls a tool (create/get/delete bookings) |
| `knowledge_base.py` | Infrastructure-as-code to set up the full RAG pipeline (S3 → OpenSearch Serverless → Bedrock KB) |
| `dataset/` | PDF menus the agent learns from (Children's Menu, Dinner Menu, Weekly Specials) |
| `bedrock-agent-tutorial.ipynb` | Jupyter notebook walkthrough tying everything together |
| `requirements.txt` | Python dependencies |

---

## 3. The Full Pipeline (Step by Step)

```
User Query
    │
    ▼
Amazon Bedrock Agent (Claude Foundation Model)
    │
    ├─── [Tool Call] ──▶ Action Group ──▶ AWS Lambda ──▶ DynamoDB
    │                   (booking CRUD)     lambda_function.py   restaurant_bookings table
    │
    └─── [RAG Query] ──▶ Knowledge Base ──▶ OpenSearch Serverless
                        (menu questions)    Vector Index (HNSW/faiss)
                                                │
                                           ▲ Embeddings
                                                │
                                           Titan Embed Model
                                                │
                                           S3 Bucket (PDFs)
```

---

## 4. Deep Dive: `knowledge_base.py`

### Purpose
This file handles the **entire RAG infrastructure** setup. It is a self-contained class `BedrockKnowledgeBase` that provisions all AWS resources needed to build a Knowledge Base.

### Class: `BedrockKnowledgeBase`

**Initialization (`__init__`)**
```python
BedrockKnowledgeBase(
    kb_name,
    kb_description,
    data_bucket_name,
    embedding_model="amazon.titan-embed-text-v1"
)
```
When initialized, it immediately runs **6 automated steps**:

---

### Step 1 — Create S3 Bucket (`create_s3_bucket`)
- **Why?** The Knowledge Base needs a place to store source documents (PDFs, text files).
- **How?** Calls `s3_client.create_bucket()`. Handles region-specific bucket creation (us-east-1 is special — no `LocationConstraint` needed).
- **What?** Creates a bucket like `restaurant-kb-1234` where the PDF menus will be uploaded.

---

### Step 2 — Create IAM Execution Role (`create_bedrock_kb_execution_role`)
- **Why?** Bedrock needs permission to read from S3 and invoke the embedding model.
- **How?** Creates 3 IAM policies:
  - `AmazonBedrockFoundationModelPolicy` — allows `bedrock:InvokeModel` on the embedding model ARN
  - `AmazonBedrockS3Policy` — allows `s3:GetObject` and `s3:ListBucket` on the data bucket
  - `AmazonBedrockOSSPolicy` — allows `aoss:APIAccessAll` on the OpenSearch collection (created later)
- Creates a role with a **trust policy** allowing `bedrock.amazonaws.com` to assume it.
- Attaches all 3 policies to this role.

---

### Step 3 — Create OpenSearch Serverless Security Policies (`create_policies_in_oss`)
- **Why?** OpenSearch Serverless (AOSS) requires explicit encryption, network, and data access policies before you can create a collection.
- **How?** Three policy types are created:
  - **Encryption policy** — uses AWS-owned KMS key to encrypt the collection at rest
  - **Network policy** — allows public access to the collection endpoint
  - **Data access policy** — grants the Bedrock execution role (and your current identity) full CRUD permissions on both the collection and its indexes

---

### Step 4 — Create OpenSearch Serverless Collection (`create_oss`)
- **Why?** This is the **vector database** where document embeddings will be stored and searched.
- **How?** Creates a `VECTORSEARCH` type collection. Waits (polling every 30 seconds) until the status changes from `CREATING` to `ACTIVE`. Then attaches the OSS access policy to the Bedrock execution role.
- **Result?** A collection endpoint URL like `<collection-id>.us-east-1.aoss.amazonaws.com`

---

### Step 5 — Create Vector Index (`create_vector_index`)
- **Why?** The collection needs a specific index with a vector field that stores document embeddings.
- **How?** Creates an OpenSearch index with:
  - `index.knn: true` — enables K-Nearest-Neighbor search
  - `knn_vector` field of **dimension 1536** (the output size of `amazon.titan-embed-text-v1`)
  - Uses the **HNSW algorithm** (Hierarchical Navigable Small World) with the **faiss** engine and **L2 (Euclidean) distance**
  - `number_of_shards: 1, number_of_replicas: 0` — lightweight single-node setup
  - Also stores `text` and `text-metadata` as text fields

---

### Step 6 — Create Knowledge Base & Data Source (`create_knowledge_base`)
- **Why?** This registers the entire setup with Bedrock so it knows *where* the vector store is and *which* embedding model to use.
- **How?**
  - Calls `bedrock_agent_client.create_knowledge_base()` with:
    - `VECTOR` type knowledge base
    - OpenSearch Serverless as the storage backend
    - Amazon Titan as the embedding model
  - Then calls `create_data_source()` which tells Bedrock:
    - Source is S3 (the bucket created in Step 1)
    - Chunking strategy: **FIXED_SIZE** — splits documents into 512-token chunks with 20% overlap (so context isn't lost at chunk boundaries)

---

### `start_ingestion_job` — Sync Documents to the KB
- **Why?** After uploading PDFs to S3, you must trigger an ingestion job so Bedrock reads them, embeds them using Titan, and writes the vectors into OpenSearch.
- **How?** Calls `start_ingestion_job()`, then polls `get_ingestion_job()` until status is `COMPLETE`.
- **What happens internally?** Bedrock reads each PDF → splits into 512-token chunks → calls Titan Embed → gets a 1536-dimensional vector → writes to OpenSearch.

---

## 5. Deep Dive: `agent.py`

### Purpose
This script wires together all the infrastructure: DynamoDB table, Lambda function, IAM roles, and the Bedrock Agent itself.

### Key Functions

#### `create_dynamodb(table_name)`
- Creates a DynamoDB table named `restaurant_bookings`
- **Partition key:** `booking_id` (String) — uniquely identifies each reservation
- Uses **PAY_PER_REQUEST** billing — no capacity planning needed, scales automatically
- Waits until the table is `ACTIVE` before proceeding

#### `create_lambda_role(agent_name, dynamodb_table_name)`
- Creates an IAM role for the Lambda function with a trust policy for `lambda.amazonaws.com`
- Attaches `AWSLambdaBasicExecutionRole` (for CloudWatch Logs)
- Creates a custom policy granting `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:DeleteItem` on the specific table ARN
- **Why?** Lambda needs these permissions to read and write booking records

#### `create_lambda(lambda_function_name, lambda_iam_role)`
- Packages `lambda_function.py` into a ZIP archive in memory using `BytesIO` + `zipfile`
- Deploys it to AWS Lambda with:
  - Runtime: **Python 3.12**
  - Timeout: **60 seconds**
  - Handler: `lambda_function.lambda_handler`
- **Why?** The Bedrock Agent needs a Lambda endpoint to execute its tools

#### `create_agent_role_and_policies(agent_name, agent_foundation_model, kb_id)`
- Creates an IAM role that Bedrock Agent will assume during execution
- Role name follows the convention: `AmazonBedrockExecutionRoleForAgents_<agent_name>`
- Trust policy: allows `bedrock.amazonaws.com` to assume the role
- Base policy: `bedrock:InvokeModel` on the specified foundation model
- If `kb_id` is provided: also grants `bedrock:Retrieve` and `bedrock:RetrieveAndGenerate` on the Knowledge Base ARN
- **Why?** This enforces least-privilege — the agent can ONLY invoke the specific model and KB it's attached to

#### `delete_agent_roles_and_policies` / `clean_up_resources`
- Cleanup functions that detach and delete all IAM policies, roles, Lambda, DynamoDB table, and Bedrock Agent resources
- **Why?** AWS charges for running resources. Clean teardown prevents billing accumulation.

---

## 6. Deep Dive: `lambda_function.py`

### Purpose
This is the **tool implementation** — the code that actually runs when the Bedrock Agent decides to call a tool.

### Tool Functions

#### `get_booking_details(booking_id)`
- **What:** Looks up a booking in DynamoDB by `booking_id`
- **How:** Uses `table.get_item(Key={'booking_id': booking_id})`
- **Returns:** The booking record `{booking_id, date, name, hour, num_guests}` or a "not found" message

#### `create_booking(date, name, hour, num_guests)`
- **What:** Creates a new restaurant reservation
- **How:**
  - Generates a random 8-character UUID as `booking_id`
  - Calls `table.put_item()` to write the record
- **Returns:** `{'booking_id': '<uuid>'}` — the caller (agent) can share this with the user

#### `delete_booking(booking_id)`
- **What:** Cancels an existing booking
- **How:** Calls `table.delete_item()` and checks the HTTP status code
- **Returns:** Success or failure message

#### `lambda_handler(event, context)` — The Dispatcher
- This is the entry point AWS Lambda calls
- Reads three things from the event:
  - `actionGroup` — which action group triggered this call
  - `function` — which specific function the agent wants to invoke
  - `parameters` — the arguments the agent extracted from the user's message
- Routes to the correct business function using `if/elif`
- Wraps the result in Bedrock's expected response format:
```json
{
  "response": {
    "actionGroup": "...",
    "function": "...",
    "functionResponse": {
      "responseBody": {
        "TEXT": {"body": "..."}
      }
    }
  },
  "messageVersion": "1.0"
}
```
- **Why this format?** This is the contract Bedrock expects from Lambda-backed action groups. The agent reads the `TEXT.body` and incorporates it into its final response to the user.

### `get_named_parameter(event, name)`
Helper function that extracts a named parameter from the Bedrock event's parameter list:
```python
event['parameters'] = [{"name": "booking_id", "value": "abc123"}, ...]
```

---

## 7. The RAG Flow in Detail

When a user asks *"What's on the dinner menu?"*:

1. **User sends query** to the Bedrock Agent
2. **Agent decides** this is a knowledge question (not a booking action)
3. **Agent embeds the query** using the same Titan model used for indexing
4. **OpenSearch KNN search** finds the top-k most similar document chunks
5. **Retrieved chunks** (from the dinner menu PDF) are injected into the agent's context
6. **Agent generates a response** grounded in the retrieved menu content
7. **User receives** an accurate answer about the menu

---

## 8. The Action Group Flow in Detail

When a user says *"Book a table for 4 people this Friday at 7 PM under John"*:

1. **User sends request** to Bedrock Agent
2. **Claude (the LLM)** identifies the intent: `create_booking`
3. **Agent extracts parameters:**
   - `date`: Friday's date
   - `name`: "John"
   - `hour`: "19:00"
   - `num_guests`: "4"
4. **Agent invokes Lambda** via the Action Group
5. **Lambda runs `create_booking()`** → writes to DynamoDB → returns booking ID
6. **Agent incorporates the result** and responds: *"Your table is booked! Your reservation ID is abc12345."*

---

## 9. Why This Architecture?

| Design Choice | Reason |
|---|---|
| **DynamoDB for bookings** | Schemaless, serverless, scales from 0 to millions of requests. Perfect for simple CRUD. |
| **Lambda for tool execution** | Serverless, event-driven. Bedrock invokes it on demand. No servers to manage. |
| **OpenSearch Serverless** | Fully managed vector search. No cluster tuning. AWS handles scaling and availability. |
| **Fixed-size chunking (512 tokens, 20% overlap)** | Balances retrieval precision (small chunks = focused context) vs. coherence (overlap = continuity across chunk boundaries). |
| **HNSW algorithm** | Industry-standard approximate nearest-neighbor search — extremely fast even at scale. |
| **Titan Embed for both indexing and retrieval** | Same model must be used for both — the vector space must be consistent. |
| **IAM least privilege** | Agent role can only call its own model and KB. Lambda role can only access its own DynamoDB table. |

---

## 10. Summary of the Complete Setup Pipeline

```
SETUP PHASE
┌─────────────────────────────────────────────────────────────────┐
│  1. Create S3 Bucket (stores PDFs)                              │
│  2. Create KB IAM Role + Policies                               │
│  3. Create AOSS Encryption/Network/Data Access Policies         │
│  4. Create AOSS Collection (vector DB)                          │
│  5. Create Vector Index (knn_vector, dim=1536, HNSW/faiss)      │
│  6. Create Bedrock Knowledge Base + Data Source (S3→AOSS)       │
│  7. Upload PDFs to S3                                           │
│  8. Start Ingestion Job (PDF→chunks→embed→AOSS)                 │
│  9. Create DynamoDB Table (restaurant_bookings)                 │
│  10. Create Lambda IAM Role + DynamoDB access policy            │
│  11. Deploy Lambda function (lambda_function.py)                │
│  12. Create Agent IAM Role + Bedrock model + KB permissions     │
│  13. Create Bedrock Agent (Claude model + system prompt)        │
│  14. Create Action Group (Lambda-backed, 3 booking functions)   │
│  15. Associate Knowledge Base with Agent                        │
│  16. Create Agent Alias (deploy agent for invocation)           │
└─────────────────────────────────────────────────────────────────┘

RUNTIME PHASE
┌─────────────────────────────────────────────────────────────────┐
│  User Query → Bedrock Agent                                     │
│     ├── Menu question → KB (RAG) → OpenSearch → Titan → Answer  │
│     └── Booking action → Lambda → DynamoDB → Confirmation       │
└─────────────────────────────────────────────────────────────────┘
```
