# Customer Support Chatbot with Amazon Bedrock AgentCore

My submission for the "Customer Support Chatbot with Amazon Bedrock AgentCore"
project. The chatbot handles customer messages for a fictional online shop and
routes each one to one of three behaviors: filing a bug report, answering from
the shop's FAQ, or handing off to human support.

All of the routing lives in a single system prompt. There is no classifier
resource and no separate agent - the AgentCore managed harness runs the model,
and the prompt decides what happens.

## Architecture

```
chat.py
  -> AgentCore managed harness (support_chatbot)
       -> Amazon Nova Pro (us.amazon.nova-pro-v1:0)
       -> system_prompt.txt  (FAQ embedded via {{FAQ}})
       -> AgentCore Gateway
            -> create_bug_report Lambda
                 -> DynamoDB (bug-report-tool-stack-bug-reports)
```

Everything runs in `us-east-1`.

## The three routes

**Bug reports.** The prompt requires all three fields - description, steps to
reproduce, environment - before the tool can be called. If something is missing,
the assistant asks for exactly one field and waits. Once all three are known it
calls `bugreports___create_bug_report` and gives the customer the returned
ticket ID.

**Platform questions.** Answered only from `online_shop_faq.md`, which gets
embedded into the prompt at harness-creation time. If the FAQ doesn't cover the
question, the assistant says so and points to the support line.

**Everything else.** Not answered. Redirected to the support line.

The prompt also has rules for the cases that sit between categories - "Where is
my order?" is a platform question, "the tracking page crashes when I try to find
my order" is a bug - and instructions to ignore attempts to override the system
prompt.

## Running it

```bash
cd project/starter
source venv/bin/activate

aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

python setup_gateway.py
python create_harness.py
python chat.py
```

## Testing

`harness-tests.json` has six cases: one per route, plus two edge cases the
project suggested - an ambiguous message that mentions an order but isn't a bug,
and a prompt-injection attempt.

```bash
python generate-eval-dataset.py --tests-json harness-tests.json
```

Each case runs in a fresh session, so the bug-report test expects a follow-up
question rather than a finished ticket. One message can't produce a ticket when
two of the three required fields are still missing, and a test that expected one
would fail by design.

The output went to S3 and through a Bedrock Evaluation job using
`Builtin.Correctness` with Nova Pro as the judge.

**Correctness: 1.00 across all six prompts.**

**See [`EVALUATION.md`](EVALUATION.md) for the full review of the evaluation
results** — per-case scores, the judge's reasoning for each, and my observations,
including one case where the 1.00 hides a partial match. Raw judge output is in
`eval-results.jsonl`.

## Notes from building this

**The reasoning traces are deliberate.** Nova Pro emits `<thinking>` blocks
before it answers, and they show up in the customer-facing output. I tried
several ways to suppress them, and every instruction strong enough to stop the
thinking also stopped the tool calls - the assistant would collect all three
fields correctly and then invent a ticket ID like `TIX-123456` instead of
calling `create_bug_report`. A DynamoDB scan confirmed nothing had been written.
The tool call matters much more than tidy output, so I removed the output-format
instructions and kept the traces. They turned out to be useful anyway: each one
names the FAQ entry it is relying on, which makes the grounding visible.

**Responses vary between runs.** Same prompt, same harness, `temperature: 0.0`
and `topK: 1`, and the results still differ. Occasionally a response comes back
containing only the thinking block with nothing after it. Running the same
prompt five times on its own was clean every time, so it isn't specific to any
one input - it seems to happen within a batch of successive calls. Adding a
delay between calls didn't help either, so it isn't simple rate limiting. I
couldn't find a prompt-side fix for this one.

**Persistent memory had to be turned off.** Early on, a brand-new `chat.py`
session claimed a bug had already been reported and quoted a ticket from a
previous conversation. Clearing DynamoDB didn't help, because the two are
separate: the table holds ticket records, while AgentCore's managed memory holds
conversation state across sessions. Disabling persistent memory fixed it.
Same-session state still works, since `chat.py` reuses one `runtimeSessionId`
for a whole conversation.

**The FAQ had no phone number.** The project asks for a hand-off to a human
support phone line, but `online_shop_faq.md` only listed a contact form and
email. Rather than have the model invent a number, I added one to the FAQ
(1-800-555-0142) so the answer stays grounded in the document.

## Where to review this submission

This project was built on the **AgentCore managed harness with prompt-based
routing**, not Bedrock Agents Classic / Flows. The rubric refers to flow
diagrams, Condition nodes, Output nodes and a FAQ Prompt node, none of which
exist in this version of the project. Here is where to find the equivalent
evidence for each criterion:

| Rubric criterion | Where to review it |
|---|---|
| Classification and routing | `project/starter/system_prompt.txt` — the three routes and routing rules. Behaviour shown in `evidence/06-chat-platform-and-other-routes.png` and `evidence/07-chat-bug-report-tool-call.png`. There is no flow diagram or Condition node; routing is entirely prompt-based. |
| Bug report path | `system_prompt.txt` (BUG_REPORT section and collection gate), `evidence/07-chat-bug-report-tool-call.png` for the `[tool call] bugreports___create_bug_report` line and returned ticket ID, `evidence/09-dynamodb-table-items.png` and `evidence/10-dynamodb-item-detail.png` for the stored records. |
| Platform question and other paths | `evidence/06-chat-platform-and-other-routes.png` — FAQ-covered answer, FAQ-uncovered hand-off, and out-of-scope hand-off, both hand-offs giving the support phone number. The FAQ is embedded via the `{{FAQ}}` placeholder in `system_prompt.txt`, substituted by `create_harness.py`; there is no Prompt node to screenshot. |
| Testing and evaluation | `project/starter/harness-tests.json` (the harness-era equivalent of `flow-tests.json`), `project/starter/output_eval_dataset.jsonl`, `evidence/08-bedrock-evaluation-correctness.png`, the per-case judge output in `evidence/11-bedrock-evaluation-prompt-details-1.png` and `evidence/12-bedrock-evaluation-prompt-details-2.png`, and the written observation in [`EVALUATION.md`](EVALUATION.md). |

## A note on the rubric

The rubric I was given describes a Bedrock Flow - a flow diagram, Condition node
expressions, Output nodes, a FAQ Prompt node, and `flow-tests.json`. None of
those exist in the AgentCore version of this project. There is no flow to
screenshot and no condition node to configure; the routing is entirely in the
system prompt, which is what the project instructions ask for.

So for the classification and routing criterion, the evidence is
`system_prompt.txt` and the `chat.py` transcripts showing each route behaving
differently, rather than a flow diagram. The test suite is `harness-tests.json`,
the name both `generate-eval-dataset.py` and the project README use.

## Evidence

- `project/starter/system_prompt.txt` - the main deliverable
- `project/starter/harness-tests.json` - test suite, all three routes plus edge cases
- `project/starter/output_eval_dataset.jsonl` - generated evaluation dataset
- Screenshots: CloudFormation stack resources, Lambda, AgentCore Gateway and
  harness, DynamoDB table and item, `chat.py` transcripts for all routes, and
  the Bedrock Evaluation results page

## Cleanup

```bash
python cleanup_agentcore.py
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
aws s3 rm s3://udacity-agentic-engineer-c1-eval-739067736634 --recursive
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```
