# The Four Kinds of Orchestrator

*Orchestration is one word for four different jobs. Here is how to tell them apart before you pick a tool.*

Strip orchestration down and only three questions remain: what runs, in what order, and what happens when a step fails. The UI, the DAG renderer, the retry decorators, the alerting integrations are all decoration on those three answers.

The first two questions have similar answers everywhere. The third does not, and that is where the categories come from. A pipeline that fails by losing a day of data needs checkpointing and backfill. A workflow that fails by leaving a customer refund half-issued needs compensation and a human. A conversation that fails by producing a wrong number needs validation and a retry with different context. No single failure model serves all three without becoming so general it stops being useful.

So the categories are not a taxonomy someone invented. They fall out of the questions.

## First, name your backbone

Before comparing tools, answer one question about the system you already have: what starts work?

**Event-driven**: work starts because something happened. A message lands, a webhook fires, a row changes. The system reacts. **Poll-based**: work starts because a clock said so, or because a loop looked at a queue and found something waiting. The system acts on its own schedule.

Neither is superior, and anyone selling one as universally correct is selling you something. But the backbone constrains which orchestrators drop in cleanly, because an orchestrator is fundamentally an opinion about **when work starts**. Force a scheduled DAG engine onto a purely reactive system and you spend your life writing sensors — tasks whose only job is to wait for something that already announced itself. Force a reactive engine onto batch ETL and you spend your life writing fake events to trigger work a cron line would have started for free.

![Event-driven arrivals are bursty and irregular; poll-based ticks are evenly spaced.](../assets/orch-backbone.png)

Be clear about what this question does: it narrows the tools that fit without friction. It does not pick your category. Half of all orchestrator disagreements are two people describing different backbones.

The two can also be bridged on purpose. A common and deliberate pattern is a Kestra JDBC trigger polling a Postgres queue table every one to five minutes, running alongside ordinary cron schedules in the same engine. The claim is a transaction that selects candidate rows with `FOR UPDATE SKIP LOCKED` so concurrent pollers never collide on the same row, updates a status column, and commits. `SKIP LOCKED` is the concurrency mechanism; the status update is the claim. That is poll-based machinery wrapped around an event-shaped workload, chosen rather than inherited: a claim table is boring, observable and replayable, and a lost broker message is none of those three. When the cost of a dropped event is high and the cost of a minute of latency is low, polling a durable table beats reacting to an ephemeral message. You are trading latency for inspectability. Make that trade knowingly.

## The line that actually matters

The instinct is to split orchestrators into AI and not-AI. That line is wrong, and it will send you to the wrong tool. The real one is **who authors the topology, and when.**

Camunda, Airflow and Kestra all execute a graph a human wrote before the run started. Airflow's dynamic task mapping looks like a counterexample and is not. `.expand()` changes how *wide* the graph is, never what *shape* it is. Cardinality is data-determined; topology is not. Fan out to three tenants or three hundred, and you still have the same nodes in the same order with the same edges.

State the claim precisely, because there are two obvious objections. DAG files are Python evaluated at parse time, so structure can vary between parses; and `BranchPythonOperator` picks a path at run time. Both are true and neither breaks the rule: for a given run, the node set and the edges are fixed before that run begins. `.expand()` decides how many instances of an already-declared node exist. Branching selects among already-declared paths. That constraint is what lets you reason about failure. You can point at a node and say what happens when it dies, because the node existed before the run did.

A planner breaks that. Topology becomes an output of execution rather than an input to it, and every guarantee that depended on knowing the graph in advance has to be rebuilt.

![A fixed forward state machine against an agent-built graph with a node added at runtime.](../assets/orch-divide.png)

AI is incidental to the line. You can build an entirely LLM-powered system whose graph is fixed in YAML, and it belongs on the declared-topology side. You could build a non-AI planner that emits task graphs from a solver, and it belongs on the other.

## The four categories

**Business process.** The unit of work is a business process, and the hard part is time and people. A refund approval idles for three days waiting on a human. The failure mode is not a crash; it is a process stranded halfway through with real-world side effects already committed. The test is sharper than "human in the loop." Not *does a human start the run*. Not *does a human read the output*. Does the run **suspend mid-execution, wait on a person for days, and resume where it left off**? Data tools model that badly — not because a sleeping task holds a worker slot (deferrable operators and reschedule-mode sensors exist precisely to free it) but because task inboxes, assignment, escalation, multi-week durable state, and an audit trail a compliance team reads are not primitives they have. Compensation rather than retry is the failure response. BPMN engines like Camunda live here, and so do finance back offices.

**Data ETL/ELT.** The unit of work is a dataset. Steps are idempotent and re-runnable, so failure is survivable as long as one bad input does not take down everything else. The hard part is blast radius. Consider a platform running roughly 28 DAGs across 13 families on CeleryExecutor with three workers, Redis as broker, Postgres as result backend. The parts that earn their keep are not the tutorial parts. Dynamic task mapping via `.expand()` gives per-tenant fan-out, so one tenant's bad payload fails its own mapped task and never blocks other tenants' work. Watermark and per-day checkpointing makes incremental ingestion resumable, which is what matters when a backfill dies at hour nine. Custom pools cap concurrency so an ML training job with a 24-hour execution timeout cannot starve the ingestion lane. Nearly all of it is containment: isolate failure, checkpoint progress, bound concurrency. None of it is clever code. It is the accumulated shape of things that went wrong once, and it is the actual reason to adopt a mature data orchestrator rather than write one. Ask what the blast radius of one bad record is. If the honest answer is "the whole run," you need containment primitives, not a better DSL.

**Business plus data hybrid.** Not pure ETL, not pure approval flow. Data pipelines with business decisions wired through the middle. A representative Kestra deployment runs a dozen declarative YAML flows across four namespaces, GitOps deployed — git is the source of truth and flows are upserted to the server over REST on push. Subflows with `ForEach` and a `concurrencyLimit` handle per-item fan-out. JDBC Postgres tasks, HTTP tasks, JS eval tasks and native AI agent tasks calling LLMs sit inside the same declarative flow. A flow-level `MAX_DURATION` SLA backstops hung runs, because something eventually hangs.

Note that these last two reach opposite conclusions about state and both are right. An Airflow cluster of that kind is typically stateful by design: on-box Postgres and Redis, `prevent_destroy` on the resources, paid for in operational care. A Kestra deployment can invert it — stateless by design, all state pushed off-box to managed Postgres and S3, so the compute node is disposable. Running a stateful engine on ephemeral infrastructure, or a stateless one with local-disk assumptions, produces failures that look like bugs and are actually mismatches. Evaluate the contract, not the noun.

**Agentic.** If deciding what to do next is itself part of the work, no author can write the graph in advance. LangChain and LangGraph serve this case.

| Domain | Typical tool | Who authors the topology | You are here if |
|---|---|---|---|
| Business process | Camunda | Human, ahead of the run | The run suspends for days on a person; failure means undo, not retry |
| Data ETL/ELT | Airflow | Human, ahead of the run | Scheduled DAGs and backfills; the question is blast radius |
| Business + data hybrid | Kestra | Human, ahead of the run | A pipeline that needs business decisions wired through its middle |
| Agentic | LangChain / LangGraph | Human declares nodes; a planner may emit the graph mid-run | The steps cannot be enumerated before the request arrives |

![Four domains and the orchestrator each one calls for.](../assets/orch-map.png)

## The selection sequence

0. **What starts work — an event or a clock?** Answer honestly, not aspirationally. This constrains which tools fit; it does not pick the category.
1. **Can you draw the graph before the run starts?** If no, you are agentic, and you should expect to own more. No amount of dynamic mapping will save you. If yes, continue.
2. **Does a run suspend and wait on a person for days, then resume?** If yes, business process engine.
3. **When a step fails, is the correct response to retry it, or to undo what already happened?** Retry points at data engines. Undo points at BPMN.
4. **Pure data movement, or data plus business decisions mid-pipeline?** The first points at Airflow, the second at a hybrid engine like Kestra.
5. **Can the compute node die?** Stateful-by-design and stateless-by-design are both valid. Choosing one accidentally is not.

## Build versus buy

Start by separating two things that get conflated constantly. LCEL is a composition library. It composes handlers — a `PromptTemplate` piped into a runnable model is composition. A system can use LCEL for every stage handler and still need a separate runtime to sequence stages, checkpoint them, and recover. Comparing a composition library to an orchestrator is comparing a function-call convention to a scheduler.

The difference is visible in what a run-time-authored system actually contains. A multi-stage agentic orchestrator typically decomposes into stages — validate and classify intent, plan a task DAG, select a model or agent per task, execute, respond, validate — where each stage handler may be an LCEL chain. The orchestration is everything around that. Execution runs the *generated* graph through a Kahn topological sort into dependency batches, concurrent under an `asyncio.Semaphore`, with a factory routing each task to a specialized agent. Pipeline state checkpoints per stage to Postgres so a session can resume mid-pipeline. State trees are recursive, because one analysis can spawn child analyses. Nothing in that topology existed before the run.

Building your own is defensible only for a capability **combination** you cannot get without rebuilding around someone else's state model: a topology generated mid-run from intermediate results, **and** recursive state trees where one analysis spawns children. Either alone is buyable. Only both together clear the bar.

LangGraph is the honest test, and understating it weakens the argument. Conditional edges route at run time. `Send` fans out at run time. `Command(goto=...)` redirects at run time. Checkpointers resume from a node and subgraphs nest. What it does not do is let a planner invent nodes that were never declared — the same width-not-shape invariant as `.expand()`, one layer up. That is the gap, and it is exactly why the category line is about authorship rather than about AI.

Two rules for migration evaluations generally. "Good engineering" is not a reason to migrate; a framework that solves no problem your system actually has is a rewrite with no numerator. And cross-language rewrites carry blockers that framework maturity does not offset — a pandas-based analyst agent has no clean JavaScript equivalent, and that alone can end an evaluation. The question is never "is this tool good." It is "does this tool solve a problem this system actually has."

Assume buy. Make build clear the bar: name the specific capability combination that forces it, then check that a fixed-topology engine truly cannot express it.

The economics push the same way. In event-driven microservice architectures the build case sounds especially reasonable, because half the machinery already exists — a broker, consumers, retries. What is left is "just" a state machine. The estimate covers the build. It never covers the next three years. Build cost was never the expensive part; maintenance, scalability and reliability are. AI writes the code now. It does not carry the pager.

Every tool named here is open source. The knowledge is free; only the reinvention is expensive. Most build-versus-buy calls go wrong long before anyone compares anything, because nobody in the room knows what already exists.

![Technical depth is mastery of how; cognitive depth is mastery of what and why.](../assets/orch-depth.png)

Technical depth is mastery of *how*: implementation, algorithms, the craft of construction. Cognitive depth is mastery of *what* and *why*: which tool fits this workload, what ownership costs in year three, what breaks at 10x, when not to build at all. It is knowing the shape of a problem well enough to recognize which already-solved problem it is. AI collapsed the cost of building. It did not collapse the cost of owning.
