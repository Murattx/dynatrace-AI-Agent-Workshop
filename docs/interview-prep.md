# SRE Interview Preparation: Agent Operations Workshop

Use this page to prepare to discuss the workshop in SRE, platform engineering,
DevOps, and AI infrastructure interviews. It is a question bank with answer
frameworks, not a script to memorize. Practice explaining the design in your own
words, draw the data path, and be precise about what you verified versus what you
would change before production.

## The 60-Second Project Story

Adapt this outline to your own experience:

> I built a Kubernetes lab to study how to operate AI agents, not to claim that
> this configuration is production-ready. It runs a kagent SRE investigator and
> a six-role BMAD software-building crew. The agents call a model and tools; an
> OpenTelemetry gateway enriches and normalizes their signals and sends traces to
> Phoenix, with Dynatrace as an optional backend. I tested the SRE agent against
> injected Kubernetes faults and compared its findings with scenario ground
> truth. The most useful operational lesson was that an accepted agent resource
> is not necessarily a ready workload: I diagnosed a missing model Secret from
> pod events, traced it to a bootstrap failure, fixed the provisioning path, and
> verified the agents became Ready and produced Phoenix spans. Before an
> enterprise pilot, I would replace the lab's unauthenticated runtime posture,
> tighten identity and tool permissions, and define data and model governance.

Only say that you personally performed work you actually did. In particular,
the current Codespace was verified with the agents and Phoenix, but it did not
have Dynatrace tenant credentials configured. Describe Dynatrace as an optional
integration in this run unless you configure it and verify tenant data yourself.

## Know These Facts

| Topic | Accurate workshop description |
|---|---|
| Cluster | Kind cluster named `workshop`, with a control-plane node and a worker node. It is a local lab, not a production Kubernetes service. |
| Agent count | Seven kagent Agents: one SRE agent plus analyst, PM, architect, developer, QA, and orchestrator roles in the BMAD crew. |
| Runtime | kagent is installed via pinned OCI Helm charts; agents are Kubernetes resources reconciled by the kagent controller. |
| Tools | The SRE agent uses the kagent Kubernetes/networking tool server. BMAD uses a GitHub MCP server for repository actions. |
| Model | The default is `deepseek-chat` through an OpenAI-compatible endpoint. The workshop proxy path dispenses a per-attendee key; self-hosting can use a provider key and endpoint. |
| Telemetry | Agent traces go by OTLP to the OpenTelemetry gateway. It adds Kubernetes/resource attributes and normalizes GenAI fields, then exports to Phoenix and optionally Dynatrace. |
| Local trace UI | Phoenix is available on port 6006. It is useful without a Dynatrace tenant, though its own LLM evaluation judge still needs a model endpoint. |
| Dynatrace | Optional. The full path needs tenant ingest credentials; Kubernetes topology and `dtctl`/`dt-evals` have additional token scopes. |
| Security lab | The repo includes a restricted Pod Security Admission and default-deny NetworkPolicy baseline, plus optional Tetragon runtime monitoring. These do not by themselves establish enterprise compliance. |
| Sample app | `sympozium-todo-demo` is a BMAD GitHub target, not a ready-to-run workload: it currently lacks an `npm start` script and Kubernetes deployment assets. |

## Architecture and Kubernetes

### 1. Why run agents on Kubernetes?

Explain the operational advantages: standard scheduling and rollout controls,
resource requests/limits, service discovery, policy enforcement, and a shared
platform for logs/metrics/traces. Balance that with the costs: controllers,
CRDs, resource pressure, network and identity complexity, and more failure
modes than a single process. Kubernetes is not automatically the right runtime
for every agent workload.

### 2. What is the difference between an Agent resource and an agent pod?

An Agent is a declarative custom resource describing the agent. The kagent
controller reconciles it into runtime Kubernetes resources, including a
Deployment and pod. `ACCEPTED=True` means the controller accepted the spec;
`READY=True` and a `1/1 Running` pod are stronger evidence that the workload is
serving. Always inspect the Deployment, pod events, and logs too.

### 3. What do Helm and an Operator each contribute here?

Helm installs versioned charts and their packaged Kubernetes resources. The
OpenTelemetry and Dynatrace Operators watch custom resources and reconcile
ongoing desired state, such as a Collector or DynaKube. Helm release success
does not prove every custom resource or downstream workload is healthy.

### 4. Why Kind rather than a managed Kubernetes cluster?

Kind gives a repeatable, inexpensive local lab using Docker containers as
Kubernetes nodes. It is good for learning and CI, but it does not reproduce a
managed control plane, cloud IAM, production networking, storage, scale, or
availability. A real enterprise deployment needs a separate target design and
validation plan.

### 5. How would you diagnose a pod stuck in `CreateContainerConfigError`?

Start with `kubectl describe pod` and recent namespace events. Identify the
specific missing config, Secret, volume, or service account. Verify resource
existence without printing Secret data, correct the source-of-truth provisioning
path, then watch the Deployment roll out and confirm readiness. Do not start by
deleting pods or dumping credentials.

### 6. What is the difference between readiness and liveness probes?

Readiness controls whether a pod should receive traffic; liveness determines
whether Kubernetes should restart it. A bad liveness probe can create restart
loops, while an overly permissive readiness probe can route traffic to a broken
container. Startup probes can protect slow initialization from premature
liveness failures.

### 7. How do you respond to `CrashLoopBackOff`?

Check restart count, current and previous container logs, `lastState`, exit code,
and events. Determine whether the process exits, is killed, or fails a probe;
then verify dependencies, configuration, resource limits, and rollout history.
Use the scenario answer key as ground truth in the lab rather than assuming the
agent's diagnosis is correct.

### 8. How would you respond to `ImagePullBackOff` or `OOMKilled`?

For `ImagePullBackOff`, inspect pod events, image name/tag, registry reachability,
and pull credentials; there may be no useful container logs because it never
started. For `OOMKilled`, inspect the terminated reason, memory usage/limits,
recent events, and workload behavior. These are different evidence channels;
avoid diagnosing one from symptoms that only support another.

### 9. What is the fault-injection safety boundary in this workshop?

Scenarios are applied into the dedicated `workshop-faults` namespace and have
explicit cleanup. The runner validates namespaces to limit blast radius. Explain
why an allowlist is safer than a blocklist and why cleanup is part of the test,
not an optional afterthought. Never inject a fault into a production cluster.

### 10. What would you monitor for the platform itself?

Cluster and node health, pod readiness/restarts, CPU/memory, queue/backpressure,
Collector exporter failures, model request latency/errors, agent task completion,
tool failures, and evaluation quality. Tie alerts to user impact and SLOs rather
than alerting on every internal metric.

## SRE Incident and Troubleshooting Questions

### 11. Tell me about an operational issue you debugged in this lab.

Use a truthful STAR story: agent resources were accepted but not ready; pod
events showed the missing `kagent-deepseek` Secret; the key existed in the
generated proxy environment, but the secret provisioning script exited before
creating it when prefix normalization returned a nonzero status under `set -e`;
you made the invalid-candidate path return successfully, reran provisioning,
then verified Secret existence, seven Ready agents, a completed SRE request, and
Phoenix spans. Keep it explicitly framed as a lab/bootstrap defect, not a
production incident.

### 12. How did you know the agent itself was ready?

Use multiple signals: accepted/ready conditions on all seven Agent resources,
each generated pod `1/1 Running`, a successful read-only A2A request, and
agent/LLM/tool spans in Phoenix. A green Helm release alone would not have been
sufficient.

### 13. What would you do if the kagent UI is reachable but an agent is not?

Separate UI/controller health from per-agent runtime health. Check Agent status,
Deployment, pod events, logs, model Secret references, MCP discovery, tool
server health, and model endpoint reachability. A healthy UI only proves the UI
path works.

### 14. What would you do if an agent gives an incorrect root cause?

Treat the answer as a hypothesis. Ask for the exact supporting events, logs,
status fields, or traces; compare them with known ground truth; distinguish
missing evidence from contradictory evidence; and avoid remediation until the
cause is established. Record the result as an evaluation example and improve
the prompt, tools, or workflow only after reproducing the failure.

### 15. How would you safely let an agent remediate an incident?

Begin read-only. Add narrowly scoped, reversible operations only after evaluation
and threat review. Require explicit human approval for high-impact actions,
bound the target namespace/resources, log the proposed and executed changes, and
verify the result with independent probes. Maintain a kill switch and a manual
runbook.

### 16. How do you distinguish a model failure from a tool or cluster failure?

Use the trace hierarchy and timestamps: model request/error spans, tool-call
spans, agent orchestration spans, Kubernetes events, and application logs. Check
whether the tool was invoked, what it returned, and whether the model completed
afterward. Correlation is useful, but verify the causal sequence rather than
inferring it from a single error count.

### 17. What would you do if an agent rollout never completes?

Inspect rollout status, old/new ReplicaSets, pod events, readiness probes, image
pulls, Secret/config references, resource pressure, and controller logs. Avoid
serial long waits that hide the underlying pod error; inspect one representative
pod and the Deployment's events first.

### 18. How should the team handle a model-provider outage?

Define the user-facing degraded behavior: bounded retries with backoff, request
timeouts, concurrency and rate limits, clear errors, queue limits, and fallback
only to an approved model with compatible behavior. Avoid retry storms. Decide
whether non-AI functionality can continue and communicate the incident through
normal service operations.

## Observability and OpenTelemetry

### 19. Why OpenTelemetry instead of instrumenting each backend directly?

OpenTelemetry provides a vendor-neutral instrumentation and transport layer.
The gateway can enrich and route signals to Phoenix and Dynatrace without
embedding backend-specific exporters in every agent. The trade-off is another
service to operate and configure; collector availability, queueing, retries,
memory, and exporter errors need monitoring.

### 20. Trace, metric, and log: when do you use each?

Traces explain an individual request's distributed path and latency. Metrics
provide aggregated, low-cardinality trends and alerting. Logs provide detailed
event context. Link them with trace/request identifiers and keep attributes
bounded so telemetry stays queryable and affordable.

### 21. What is OTLP and where does it flow here?

OTLP is the OpenTelemetry protocol. kagent emits spans to the in-cluster
Collector over gRPC or HTTP. The Collector enriches them with Kubernetes
metadata, normalizes selected GenAI attributes, and exports to Phoenix and, if
configured, Dynatrace.

### 22. What does the Collector add to the agent spans?

The gateway uses processors such as Kubernetes attribute enrichment, resource
detection, resource attributes, GenAI normalization, batching, and memory
limiting. It stamps workshop identity attributes and maps framework attributes
onto fields useful for querying. These transforms are workshop-specific; don't
present them as automatic OpenTelemetry behavior.

### 23. What are `gen_ai.*` attributes for?

They standardize model/provider, operation, token usage, and related GenAI
information across instrumentation. They let an operator group model calls,
measure latency and token consumption, and compare agent workflows. Verify which
fields the actual SDK emits; semantic conventions and vendor mappings can
change.

### 24. How do you reduce telemetry cost without losing diagnostic value?

Start from questions the team needs to answer. Use appropriate sampling,
retention, aggregation, and attribute limits; avoid high-cardinality or sensitive
fields. Preserve error and representative slow traces, and validate that
sampling still captures enough LLM/tool context for diagnosis and evaluation.

### 25. What is dangerous about capturing prompts and completions?

They can contain customer data, source code, secrets, or regulated information.
Treat span bodies as sensitive: redact before export, restrict read access, set
retention and deletion controls, and document processor/region boundaries. Token
counts and model metadata may still be useful when content capture is disabled.

### 26. How would you troubleshoot “Phoenix has traces but Dynatrace does not”?

Check the Collector's Dynatrace exporter and logs, tenant `.live.` endpoint,
ingest token scopes, Secret references, authentication responses, network egress,
and exporter queue/retry state. Phoenix success proves receiver and one exporter
path work; it does not prove the Dynatrace endpoint or token is valid.

## Agent, MCP, and Evaluation Questions

### 27. What is an AI agent in this design?

An agent combines instructions, a model, and access to tools, then uses a loop to
decide and act. That loop introduces non-determinism, model/provider dependency,
tool risk, and potentially unbounded cost. Operational controls must cover both
the model call and the tool's authority.

### 28. What is MCP, and why is it used for GitHub?

Model Context Protocol standardizes how an agent discovers and invokes tools.
Here it separates GitHub tool serving from agent manifests and exposes repo,
issue, and PR operations. It still needs explicit authentication, tool
allowlisting, permission scoping, and audit; MCP is a protocol, not a security
boundary by itself.

### 29. What is A2A in the workshop?

A2A is the agent-to-agent/request interface used by the kagent controller for
invocations. The sample smoke request sends a `message/send` JSON-RPC call to an
agent endpoint. Be ready to describe where orchestration ends, where a delegated
agent starts, and how tracing correlates the hops.

### 30. How does the BMAD crew divide work?

The analyst clarifies needs, PM plans, architect decomposes/designs, developer
implements, QA reviews, and orchestrator coordinates. The benefit is explicit
roles and handoffs; the costs are more model calls, context transfer, latency,
and failure opportunities. Measure whether each role improves quality enough to
justify its cost.

### 31. How do you evaluate an agent beyond “the demo looked good”?

Use representative tasks, a rubric, a ground-truth set where possible, and
repeatable scoring. Track correctness/grounding, tool selection, task success,
latency, tokens/cost, safety violations, and failure modes. Include human review
for ambiguous or high-impact outputs and test regression cases before rollout.

### 32. What do relevance, faithfulness, and hallucination mean here?

Relevance asks whether the answer addresses the request. Faithfulness asks
whether it is supported by supplied evidence. Hallucination scoring looks for
unsupported claims. Define rubrics and known examples; LLM-as-judge scores are
signals, not objective truth, and should be calibrated against human review.

### 33. How would you evaluate a diagnosis agent?

Inject isolated, known faults; give a symptom-only prompt; require evidence and
root cause; score required signals and disallowed claims against an answer key;
then clean up. Test distinct evidence sources such as logs, events, and
termination reasons so the agent cannot pass by guessing a common diagnosis.

### 34. How can an agent loop become expensive or stuck?

Repeated tool calls, retries, large context, excessive delegation, or a failure
to recognize completion can multiply tokens and latency. Track task duration,
tool/model call counts, retries, context size, and cost per successful task. Add
step/time/token budgets, cancellation, and loop detection; evaluate quality after
each optimization.

### 35. How do you prevent tool descriptions or retrieved content from becoming
prompt-injection paths?

Assume all external text can be adversarial. Separate instructions from data,
constrain tools and inputs, validate outputs, keep secrets out of context, limit
network egress, and require human approval for sensitive actions. A prompt alone
is not an authorization control.

## Security, Privacy, and Regulation

### 36. Is this workshop production-ready for a regulated enterprise?

No. It is a learning lab. The deployed kagent controller reported
`AUTH_MODE=unsecure`; the GitHub MCP receives a token in its workload
environment; the local Kind cluster is not a high-availability production
platform; and model/trace data boundaries require organization-specific review.
State the gaps and describe how you would close them before a pilot.

### 37. What is the least-privilege model for the GitHub integration?

Use a fine-grained token scoped to one repository with only the required content,
issue, and PR permissions. Assign tools per agent, avoid organization/admin
permissions, store the credential in an approved secret manager, rotate it, and
audit usage. For high-assurance use, prefer short-lived workload identity if the
provider supports it.

### 38. How would you secure Kubernetes tools exposed to an agent?

Use a dedicated service account and minimal RBAC verbs/resources, isolate
namespaces, constrain network egress, and deny privileged workloads. Separate
read-only investigation from remediation capabilities. Add approval gates and
audit logs for writes; verify permissions with negative tests, not only a
successful read.

### 39. What do Pod Security Admission and NetworkPolicy protect against?

Restricted Pod Security Admission limits unsafe pod settings such as root,
privilege escalation, and capabilities. A default-deny NetworkPolicy constrains
network paths, with explicit exceptions. Neither protects against every kernel,
identity, application, or control-plane threat; confirm that the CNI enforces
policies and test the actual boundaries.

### 40. What does Tetragon add?

Tetragon uses eBPF to observe process and kernel events and can apply policies.
In this lab it can detect sensitive-file access. Detection gives runtime
evidence; enforcement has kernel and operational implications. Validate loaded
policies and actions carefully, and do not assume that installing the DaemonSet
means every policy is active.

### 41. What data-governance questions would you ask before enabling a model?

Where is inference processed? Is data retained or used for training? What data
classification is permitted? How are prompts and outputs logged? Who can access
the traces? How long are they retained? Are subprocessors and regions approved?
Can content capture be disabled or redacted? Record the answers in the service's
data-flow and risk assessment.

### 42. How do you manage model and tool credentials?

Use an approved secret manager or workload identity, inject credentials at
runtime, limit scope and lifetime, rotate/revoke, and prevent values from
appearing in Git, logs, images, traces, or shell history. Detect accidental
exposure and define a revocation playbook. A Kubernetes Secret is encoded at
rest by default unless cluster encryption and access controls are configured.

### 43. What would a regulated pilot look like?

Start with synthetic or explicitly approved data in an isolated environment.
Use an approved private model endpoint, no mutating tools, authenticated access,
short-lived credentials, restricted egress, redacted telemetry, audit trails,
human review, an owner, and an exit/rollback plan. Establish acceptance criteria
with security, privacy, legal, and platform teams before expanding.

### 44. How would you respond to a prompt-injection incident?

Contain the agent by disabling or narrowing tools and credentials; preserve audit
and trace evidence without distributing sensitive content; determine which tools
were invoked and what data/actions were affected; revoke exposed credentials;
notify the incident/security owners; remediate the boundary; and add a regression
test. Do not assume changing the system prompt alone resolves the vulnerability.

## Reliability, Capacity, and Cost

### 45. What are the main dependencies in the request path?

The UI/API, kagent controller/runtime, model endpoint, tool servers, Kubernetes
API, Collector for telemetry, and optional trace backends. Identify which are
critical to answering a request versus only to observing it. Decide how each
failure degrades, times out, retries, or blocks execution.

### 46. How would you define an SLO for an agent service?

Choose a user-visible outcome, such as successful completion of an approved
task within a latency budget and without policy violations. Define the eligible
request population and measurement window. Pair success/latency with quality and
safety guardrails; a fast but incorrect answer is not an SLO success.

### 47. How do you prevent retries from amplifying an outage?

Use bounded retries with exponential backoff and jitter, request deadlines,
concurrency limits, circuit breaking where appropriate, and cancellation
propagation. Distinguish retryable transport errors from invalid requests or
policy denials. Measure retry volume and ensure one orchestrator does not cause
multiple agents to retry independently without a shared budget.

### 48. How do you estimate and control model cost?

Measure input/output tokens and calls per successful task by model and agent;
apply current provider pricing to estimate cost. Set per-user or per-workflow
budgets, cap steps and context, remove unused tools, and choose smaller models
only after quality evaluation. Reconcile estimates against provider billing.

### 49. What resource limits matter for this lab?

The full stack is recommended on a 4-core/16-GB Codespace. Operators, Tetragon,
Phoenix, the Collector, kagent, and agent pods all consume resources. Set
requests/limits, watch node pressure and scheduling events, and opt into the
additional demo application only when capacity allows. The lab size is not a
production capacity model.

### 50. What is your disaster-recovery story?

The Kind cluster is disposable and its local persistent state is not a production
backup strategy. Recreate it from versioned manifests and pinned dependencies;
protect external configuration and credentials separately. For production,
design backups for required state, test restore, define RPO/RTO, and ensure the
model, MCP, and telemetry dependencies can be restored or replaced.

## Dynatrace and Vendor-Neutral Design

### 51. How does the design avoid hard dependence on Dynatrace?

Agents emit OTLP through an OpenTelemetry gateway; Phoenix is a local backend,
and Dynatrace is a parallel optional exporter. The core agent, Kubernetes, and
Phoenix learning path works without tenant credentials. Dynatrace-specific
Operator, DQL, `dtctl`, and `dt-evals` assets remain optional integrations.

### 52. What is the difference between the Dynatrace URLs and tokens?

The `.live.` tenant endpoint is used for OTLP ingest and the Operator API. The
`.apps.` platform endpoint is used by platform tooling such as `dtctl`. Ingest,
operator/evaluation, and platform tokens have different scope requirements.
Keep them separate and follow the current secret contract rather than reusing an
ingest-only token for Kubernetes monitoring.

### 53. How would you prove the Dynatrace path works?

Verify the OTLP Secret and endpoint, Collector exporter logs, accepted/exported
signals, and a known trace in the tenant. Separately verify DynaKube/ActiveGate
conditions and Kubernetes entities in Smartscape. A healthy Collector pod alone
does not prove tenant ingestion or topology integration.

### 54. How do you avoid locking the design to one observability vendor?

Keep instrumentation and resource attributes in OpenTelemetry, avoid embedding
backend-specific APIs in agents, and isolate vendor-specific queries/dashboards
behind separate assets. Document semantic-convention mappings and test them
against each backend. Vendor-neutral transport does not make every query or
evaluation artifact portable.

## Behavioral and Trade-Off Questions

### 55. Why did you build this workshop?

Explain the learning objective in terms of operating agents: make tool/model
activity observable, test diagnosis against known faults, and understand cost
and safety controls. Avoid presenting “AI” as the goal by itself; connect it to
an operational problem and what evidence you wanted to gather.

### 56. What was your most important design trade-off?

Possible examples: Kind simplicity versus production realism; a shared Collector
versus per-agent exporters; multi-agent specialization versus added token cost;
trace content versus privacy; hosted proxy convenience versus provider control.
State the alternatives, the chosen constraint, and what evidence would make you
change the decision.

### 57. What would you improve first?

Prioritize by risk, not novelty. A credible answer is: replace unauthenticated
runtime access; integrate workload identity/secret management; restrict and test
tool RBAC; document data residency/retention; add policy gates; then define
quality and operational SLOs. Explain which one you would do first for the
specific organization.

### 58. How did you know the fix worked?

Describe the before/after evidence and a regression check: the missing Secret was
reproduced from an unset prefix; the shell behavior was corrected; the secret
script completed with no explicit prefix; Kubernetes showed the Secret; all
seven agent resources became Ready; a smoke call completed; Phoenix contained
agent and LLM spans. Distinguish these checks from the unconfigured Dynatrace
path.

### 59. What would you do if compliance rejects the model provider?

Do not route around the decision. Determine the approved deployment options
(private endpoint, on-prem model, or no model for the workload), data and
retention constraints, and whether the use case remains viable. Demonstrate the
vendor-neutral telemetry path with synthetic data while the approved model
integration is reviewed.

### 60. How do you communicate uncertainty in an agent's answer?

Require evidence references and explicit unknowns. Separate observed facts from
hypotheses, ask the agent to state confidence and missing evidence, and route
high-impact decisions to a human. Measure calibration and false confidence in
evaluations rather than treating fluent wording as reliability.

### 61. How would you convince a skeptical SRE team to try this?

Propose a bounded, opt-in experiment with synthetic or low-risk tasks, read-only
permissions, a human operator in the loop, explicit success criteria, and a
rollback plan. Compare with the existing runbook process and publish failures as
well as successes. Do not ask the team to trust an opaque agent or replace
on-call judgment.

## A Repeatable Answer Structure

For architecture or incident questions, organize answers in this order:

1. **Goal and scope:** What user or operational problem are you addressing?
2. **Evidence:** What did you observe (status, event, log, span, metric)?
3. **Reasoning:** How did the evidence narrow the cause or design choice?
4. **Action and safety:** What did you change, and what was the blast radius?
5. **Verification:** Which independent checks proved the outcome?
6. **Trade-off and next step:** What remains unproven, and what would you improve?

For behavioral questions, use STAR (Situation, Task, Action, Result), then add
what you learned. Keep the Result measurable where possible and label lab results
as lab results.

## Questions to Ask the Interviewer

- Which agent or automation workloads are currently approved, and which remain
  experimental?
- What are the organization's model-provider, data-residency, and prompt-retention
  requirements?
- How are workload identity, tool authorization, and human approvals handled for
  automation that can mutate systems?
- What reliability and quality measures would need to pass before a pilot could
  expand?
- Which observability platform is standard, and are OpenTelemetry pipelines
  already available?
- Who owns model, prompt, evaluation, and incident-response changes after launch?

These questions demonstrate that you understand enterprise adoption as a joint
security, platform, SRE, and governance problem rather than a chart installation.

## Practice Plan

1. Deliver the project story in 60 seconds, then again in 3 minutes with an
   architecture sketch.
2. Reproduce the missing-Secret incident from evidence, without relying on notes.
3. Answer questions 19–26 by drawing the trace path and naming failure points.
4. Practice questions 36–44 as a security review; explicitly say what the lab
   does not provide.
5. Pick one scenario and explain its symptom, evidence channel, diagnosis, safe
   remediation, verification, and cleanup.
6. Ask a peer to challenge one design decision and defend it with evidence, not
   preference.

Continue with the [replication and self-study guide](replicate-workshop.md) for
the architecture, source repository map, setup instructions, and operational
checklists.