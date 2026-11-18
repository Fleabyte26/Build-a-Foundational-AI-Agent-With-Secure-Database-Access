# OCBL466 - Build a Foundational AI Agent With Secure Database Access

Three tiers: AlloyDB with vector search (data), MCP Toolbox on Cloud Run (secure API), ADK agent (consumer). The agent never touches the database directly, only the two tools you define in tools.yaml.

## Known lab traps/bugs (read first)

1. The lab's tools.yaml and agent.py steps use an open `cat << EOF` that waits for you to type EOF by hand. Paste errors here are the number one way this lab breaks. The blocks below write the files in one paste.
2. The lab reads the Toolbox URL and the database password from shell variables. Open a new tab or lose your session and they are empty: the agent crashes on startup or the Toolbox fails to log in to AlloyDB. This guide saves every value to a file in your home folder instead.
3. The model names are filled in per lab instance. You must copy two of them from your own lab page (Setup below says where).
4. IAM changes take a few minutes to reach AlloyDB. A `Permission denied on the resource` error on the embedding query means wait 2-3 minutes and run it again.
5. Enabling public IP on AlloyDB is a Console step, not a gcloud one. Doing it with `gcloud alloydb instances update` fails with INVALID_ARGUMENT because public IP also requires the password.enforce_complexity database flag, which the Console sets for you.
6. Creating the VPC connector takes several minutes. Wait for the prompt to come back before pasting the next block.
7. Task 5 talks about ScaNN but the query builds an ivfflat index. Run it as given; the checker accepts it.
8. The checker lags. If a check fails, wait 30-60 seconds and click again.

## Setup (Cloud Shell)

Wait until cymbal-cluster and cymbal-instance both show Ready in AlloyDB, then paste:

```bash
cd ~
gcloud config get-value project > ~/project
gcloud config get-value account > ~/student
gcloud projects describe "`cat ~/project`" --format='value(projectNumber)' > ~/projnum
gcloud alloydb clusters list --format='value(name)' | head -1 | cut -d/ -f4 > ~/region
echo "text-embedding-005" > ~/embmodel
echo "gemini-2.5-flash" > ~/agentmodel
head ~/project ~/student ~/projnum ~/region ~/embmodel ~/agentmodel
```

You should see six values, none empty. The region must match the one on your lab page.

Now check the two model names against your lab page and fix them if they differ:

- Embedding model: Task 3, the first value inside `embedding('...', ...)`. Fix with `echo "NAME" > ~/embmodel`
- Agent model: Task 8, the `model=` line in the agent.py script. Fix with `echo "NAME" > ~/agentmodel`

## Task 1. Configure AlloyDB for vector search

Grant the role first (Cloud Shell) so it has time to propagate while you do the SQL:

```bash
gcloud projects add-iam-policy-binding "`cat ~/project`" \
  --member="serviceAccount:service-`cat ~/projnum`@gcp-sa-alloydb.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user" --condition=None --quiet
gcloud projects get-iam-policy "`cat ~/project`" --format='value(bindings.members)' | tr ';' '\n' | grep -c gcp-sa-alloydb
```

You should see a number of 1 or more. If the grant fails with 403, wait a few minutes and paste again.

Check my progress: Grant Agent Platform User role to AlloyDB service account

Then in the Console: AlloyDB > Clusters > cymbal-instance > AlloyDB Studio. Sign in with Database postgres, User postgres, Password changeme. Open a new SQL editor tab. Run these one at a time (Clear between each):

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

```sql
GRANT EXECUTE ON FUNCTION embedding TO postgres;
```

Each should say Statement executed successfully.

Check my progress: Enable extensions and permissions in AlloyDB for vector search

## Tasks 2-5 (AlloyDB Studio)

Everything in this section runs in the AlloyDB Studio query editor, not Cloud Shell. Click Clear between every query.

Four of these queries call the embedding model by name. Your lab page fills that name in; this file writes it as MODEL_NAME. Before you paste one of those, replace MODEL_NAME with the model from your own lab page (Task 3, inside `embedding(...)`). It is the same value you saved to ~/embmodel in Setup.

### Task 2a. Create the table

```sql
CREATE TABLE customer_records_data (
    id VARCHAR(25),
    type VARCHAR(25),
    number VARCHAR(20),
    country VARCHAR(2),
    date VARCHAR(20),
    abstract VARCHAR(300000),
    title VARCHAR(100000),
    kind VARCHAR(6),
    num_claims BIGINT,
    filename VARCHAR(100),
    withdrawn BIGINT,
    abstract_embeddings vector(768)
);
```

Statement executed successfully. The last column holds the vectors you generate in Task 3.

### Task 2b. Load the data

This one is 56 long insert statements, so use the lab's own copy button rather than retyping it: in the lab's Task 2, find the block that begins `insert into customer_records_data(id, type, number, ...)`, click its copy button, paste the whole thing into Studio and click Run once.

Verify with:

```sql
SELECT count(*) FROM customer_records_data;
```

You should see 56.

Check my progress: Create a new table and load customer data

### Task 3a. Test the embedding model

```sql
SELECT embedding('MODEL_NAME', 'AlloyDB is a managed, cloud-hosted SQL database service.');
```

You should get a long list of numbers. `Permission denied on the resource` means the IAM grant from Task 1 has not propagated: wait 2-3 minutes and run it again (trap 4).

### Task 3b. Generate embeddings for every row

```sql
UPDATE customer_records_data
SET abstract_embeddings = embedding('MODEL_NAME', abstract);
```

This calls the model once per row, so give it up to a minute. Verify with:

```sql
SELECT count(*) FROM customer_records_data WHERE abstract_embeddings IS NOT NULL;
```

You should see 56. If it is lower, run the UPDATE again.

Check my progress: Generate and store text embeddings for the customer data

### Task 4. Vector search

```sql
SELECT id, title, abstract
FROM customer_records_data
ORDER BY abstract_embeddings <=> embedding('MODEL_NAME', 'what should I do about water damage in my home?')::vector
LIMIT 10;
```

ART-10326507, the water damage mitigation article, should be at or near the top, followed by the wind-versus-water bulletin and the HO-3 policy. No checkpoint here, but look at what came back: the search phrase shares almost no keywords with those rows. They were matched by meaning.

### Task 5. Index

```sql
CREATE INDEX ON customer_records_data
USING ivfflat (abstract_embeddings vector_l2_ops)
WITH (lists = 100);
```

The lab text calls this ScaNN but the query builds an ivfflat index. Run it as written; the checker accepts it (trap 7).

Check my progress: Optimize vector search with an index

## Task 6. Build and deploy the secure API service

### 6a. Public IP on AlloyDB (Console, takes several minutes)

Do this one in the Console, as the lab does. Enabling public IP through gcloud also requires the password.enforce_complexity database flag, which the Console sets for you; without it the command fails with INVALID_ARGUMENT.

First get the network to authorize:

```bash
ip -4 -o addr show eth0 | tr -s ' ' | cut -d' ' -f4 | cut -d. -f1-2 > ~/netprefix
echo "`cat ~/netprefix`.0.0/16"
```

You should see something like 10.88.0.0/16. That is your Cloud Shell network, already in the form the next step wants.

Then, in the Console: AlloyDB > cymbal-instance > Connectivity > Edit.

1. Under Public IP connectivity, tick Enable public IP.
2. Under Authorized external networks, in the Networks box, paste the value the command above printed.
3. Click Update, and wait for it to finish.

Confirm from Cloud Shell:

```bash
gcloud alloydb instances describe cymbal-instance --cluster=cymbal-cluster \
  --region="`cat ~/region`" --format='value(publicIpAddress)'
```

You should see a public IP address. An empty line means the update has not finished yet.

### 6b. Service account, roles and the password secret

```bash
gcloud iam service-accounts describe "toolbox-identity@`cat ~/project`.iam.gserviceaccount.com" || \
gcloud iam service-accounts create toolbox-identity --display-name="MCP Toolbox Service"
gcloud projects add-iam-policy-binding "`cat ~/project`" \
  --member="serviceAccount:toolbox-identity@`cat ~/project`.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" --condition=None --quiet
gcloud projects add-iam-policy-binding "`cat ~/project`" \
  --member="serviceAccount:toolbox-identity@`cat ~/project`.iam.gserviceaccount.com" \
  --role="roles/alloydb.client" --condition=None --quiet
gcloud projects add-iam-policy-binding "`cat ~/project`" \
  --member="serviceAccount:toolbox-identity@`cat ~/project`.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageConsumer" --condition=None --quiet
gcloud iam service-accounts add-iam-policy-binding \
  "toolbox-identity@`cat ~/project`.iam.gserviceaccount.com" \
  --member="user:`cat ~/student`" --role="roles/iam.serviceAccountUser" --quiet
printf '%s' 'changeme' | gcloud secrets create alloydb-password --data-file=-
gcloud projects get-iam-policy "`cat ~/project`" --flatten='bindings[].members' \
  --filter='bindings.members:toolbox-identity' --format='value(bindings.role)'
gcloud secrets list --format='value(name)'
```

You should see the three roles (alloydb.client, secretmanager.secretAccessor, serviceusage.serviceUsageConsumer) and the secret alloydb-password. This is least privilege in practice: the Toolbox identity can read secrets and connect to AlloyDB, nothing else.

Check my progress: Prepare AlloyDB for the toolbox interaction and grant the necessary roles to the service account

### 6c. Install the Toolbox binary

```bash
mkdir -p ~/mcp-toolbox && cd ~/mcp-toolbox
curl -O https://storage.googleapis.com/genai-toolbox/v0.12.0/linux/amd64/toolbox
chmod +x toolbox
ls -l toolbox
```

You should see an executable file of roughly 100 MB or more.

### 6d. Write tools.yaml

Two tools, each a fixed SQL statement with numbered placeholders. The agent can only fill the placeholders, never write SQL. That is the SQL injection defence.

```bash
cd ~/mcp-toolbox
cat > tools.yaml <<'EOF'
sources:
  customer_records_data:
    kind: "alloydb-postgres"
    project: "PROJECT_X"
    region: "REGION_X"
    cluster: "cymbal-cluster"
    instance: "cymbal-instance"
    database: "postgres"
    user: "postgres"
    password: "DBPASS_X"

tools:
  find_similar_customer_records:
    kind: postgres-sql
    source: customer_records_data
    description: "Finds customer records with abstracts similar to a given query text. Use this for semantic search on the contents of policies and articles."
    parameters:
      - name: query_text
        type: string
        description: "The text to search for similar abstracts."
      - name: limit
        type: integer
        description: "The maximum number of similar records to return. Defaults to 5."
    statement: |
      SELECT id, title, abstract, 1 - (embedding('MODEL_X', PARAM1)::vector <=> abstract_embeddings) as similarity
      FROM customer_records_data
      ORDER BY embedding('MODEL_X', PARAM1)::vector <=> abstract_embeddings
      LIMIT PARAM2;

EOF
wc -l tools.yaml
```

You should see 29 tools.yaml. Second half (note the double arrow, it appends):

```bash
cd ~/mcp-toolbox
cat >> tools.yaml <<'EOF'
  get_record_by_id:
    kind: postgres-sql
    source: customer_records_data
    description: "Retrieves the full details of a specific insurance policy or article when the exact record ID is known. Use this for precise lookups."
    parameters:
      - name: record_id
        type: string
        description: "The unique identifier of the policy or article to retrieve (e.g., 'POL-10326103')."
    statement: |
      SELECT id, type, number, country, date, title, abstract, kind, num_claims, filename
      FROM customer_records_data
      WHERE id = PARAM1;

toolsets:
  customer_data_tools:
    - find_similar_customer_records
    - get_record_by_id
EOF
wc -l tools.yaml
tail -4 tools.yaml
```

You should see 46 tools.yaml and the toolset lines. Now fill in the placeholders:

```bash
cd ~/mcp-toolbox
sed -i "s|PROJECT_X|`cat ~/project`|; s|REGION_X|`cat ~/region`|" tools.yaml
sed -i "s|MODEL_X|`cat ~/embmodel`|g" tools.yaml
sed -i "s|DBPASS_X|`gcloud secrets versions access latest --secret=alloydb-password`|" tools.yaml
sed -i 's/PARAM\([0-9]\)/\x24\1/g' tools.yaml
grep -n 'project:\|region:\|password:\|embedding\|LIMIT\|WHERE' tools.yaml
grep -c '_X\|PARAM' tools.yaml
```

You should see your project, your region, the password changeme, your embedding model, and the SQL placeholders written as a dollar sign plus a number (dollar-1, dollar-2). The last line must print 0.

### 6e. Optional: run the Toolbox locally for 25 seconds

```bash
cd ~/mcp-toolbox
timeout 25 ./toolbox --tools-file "tools.yaml" --port=8080
echo "local test finished"
```

You should see a line containing Server ready to serve before it stops by itself. An error here means tools.yaml or the public IP step (6a) is wrong; fix it before deploying.

### 6f. Deploy to Cloud Run

```bash
cd ~/mcp-toolbox
gcloud secrets create tools --data-file=tools.yaml || gcloud secrets versions add tools --data-file=tools.yaml
gcloud compute networks vpc-access connectors create alloydb-connector \
  --region="`cat ~/region`" --network=peering-network --range=10.8.0.0/28
gcloud compute networks vpc-access connectors list --region="`cat ~/region`" --format='value(name,state)'
```

Takes 2-3 minutes. You should see alloydb-connector with state READY.

```bash
cd ~/mcp-toolbox
gcloud run deploy toolbox \
  --image=us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:latest \
  --vpc-connector=alloydb-connector \
  --region="`cat ~/region`" \
  --service-account=toolbox-identity \
  --allow-unauthenticated \
  --set-secrets="/app/tools.yaml=tools:latest" \
  --args="--tools-file=/app/tools.yaml","--address=0.0.0.0","--port=8080" \
  --ingress=all --min-instances=1 --quiet
gcloud run services describe toolbox --region="`cat ~/region`" --format='value(status.url)' > ~/toolboxurl
cat ~/toolboxurl
curl -s "`cat ~/toolboxurl`/api/toolset/customer_data_tools" | head -c 300; echo
```

You should see a URL ending in run.app, then JSON that names find_similar_customer_records. Note that tools.yaml lives in Secret Manager, not in the image: adding a tool later means a new secret version, not a rebuild. (In production this service would be deployed with no public access; the lab opens it only because Cloud Shell sits outside the VPC.)

Check my progress: Deploy MCP Toolbox for Database to Cloud Run

## Task 7 and 8. Set up and build the ADK agent

```bash
mkdir -p ~/mcp-toolbox/my-adk-agent && cd ~/mcp-toolbox/my-adk-agent
python3 -m venv .venv
source .venv/bin/activate
pip install --quiet google-adk toolbox-core
adk create multi_tool_agent --model="`cat ~/agentmodel`" \
  --project="`cat ~/project`" --region="`cat ~/region`"
cp ../tools.yaml multi_tool_agent/
ls -a multi_tool_agent
```

The flags answer the lab's four prompts for you (model, Vertex AI backend, project, region). You should see .env, __init__.py, agent.py and tools.yaml. If your lab page shows a different region for the agent than for the database, edit multi_tool_agent/.env to match.

Now replace agent.py:

```bash
cd ~/mcp-toolbox/my-adk-agent
cat > multi_tool_agent/agent.py <<'EOF'
from google.adk.agents.llm_agent import Agent
from toolbox_core import ToolboxSyncClient

# Load the toolset from the secure API layer (never from the database)
toolbox = ToolboxSyncClient("URL_X")
tools = toolbox.load_toolset('customer_data_tools')

root_agent = Agent(
    name='claims_assistant',
    model='MODEL_X',
    description='The Cymbal Claims Assistant is designed to help insurance adjusters at Cymbal Insurance find relevant articles or policies and find a specific policy or article by providing its unique ID.',
    instruction='You are an insurance claims assistant specifically helping insurance adjusters at Cymbal Insurance. Your primary function is to quickly and accurately retrieve information from a database of insurance policies and related knowledge base articles.',
    tools=tools,
)
EOF
sed -i "s|URL_X|`cat ~/toolboxurl`|; s|MODEL_X|`cat ~/agentmodel`|" multi_tool_agent/agent.py
grep -n 'ToolboxSyncClient("\|model=' multi_tool_agent/agent.py
```

You should see your run.app URL and your agent model. The URL is written into the file, so a new tab or a reconnect cannot break it (trap 2).

## Task 9. Run and test your agent

```bash
cd ~/mcp-toolbox/my-adk-agent
source .venv/bin/activate
adk run multi_tool_agent
```

At the user prompt, paste these one at a time and wait for each answer:

```
Find articles or policies that discuss water damage from broken pipes.
```

```
What are the details of the record with ID POL-11000139?
```

```
Search for any articles related to fraud or suspicious claims.
```

You should see: water damage and frozen pipe articles for the first (semantic search tool), the Workers Compensation Policy for the second (lookup by ID tool), and the Red Flags for Property Claim Fraud article for the third. Type exit to quit.

Check my progress: Test the ADK agent
