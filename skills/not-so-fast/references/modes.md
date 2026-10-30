# Mode playbooks

Read this file before building the setup card. Each section has three parts:

- **Probe**: what to add to the interview frontier once the mode is confirmed. A mode the request names counts as confirmed, so its probes go on the first card.
- **Run**: how to run the mode during the critique.
- **Deliver**: what the mode adds to the critique, on top of the rated findings and verdict every critique carries.

Sections follow the menu order, which is also the run order.

## Frame

### Question the Problem

- **Probe**: who has the problem, how they cope today, what it costs them, how the user knows, and why now.
- **Run**: separate the stated problem from the underlying need, since the request may describe a solution rather than the problem it serves. Ask "why" until the goal stops changing. Test the do-nothing option and the cheapest non-solution: a process change, a policy, or an existing tool.
- **Deliver**: the problem restated in one sentence, whether the idea actually solves it, and the cost of doing nothing.

### Question Me

- **Probe**: what the user already believes, how confident they are, and what would change their mind.
- **Run**: this mode's critique is itself interactive. Run it in question rounds rather than conclusions: the evidence behind each belief, the strongest counterexample, and what the user would expect to see if they were wrong. Follow each answer to the next weak point, then close with the rated findings and verdict as usual.
- **Deliver**: the beliefs that survived questioning, the ones that did not, and the conclusions the user reached, in their words where possible.

## Attack

### Challenge My Idea

- **Probe**: goals, success criteria, constraints, and what the user is least sure about.
- **Run**: sweep the stress-test lens, going deep only where a finding could change the verdict.
- **Deliver**: the standard critique; this mode adds no separate artifact.

### Find How It Breaks

- **Probe**: who uses it, who might attack or misuse it, what it depends on, and what "broken" means here.
- **Run**: red team the thing itself, whether a system, product, policy, or argument. Push inputs, load, dependencies, and incentives to their edges. Think like an adversary, a careless user, and a competitor, and find the strongest counterargument. Imagine It Failed covers the project around the thing.
- **Deliver**: failure modes ranked by likelihood and blast radius, each with its trigger, impact, and mitigation.

### Imagine It Failed

- **Probe**: the timeline, who is accountable, and what success looks like at the deadline.
- **Run**: a pre-mortem. State "It is [date] and this failed," write the most plausible failure story, then list causes across people, market, timing, execution, and dependencies. Look back from the failed project; Find How It Breaks covers the thing itself.
- **Deliver**: the failure story in a short paragraph, the top causes, the earliest warning sign for each, and the preventive action.

### Argue the Other Side

- **Probe**: who opposes or must approve this, what they care about, and what they stand to lose.
- **Run**: steelman the strongest opposing position, then argue it in the first person as each relevant stakeholder, such as a CFO, an on-call engineer, a customer, a regulator, or a competitor. Make the case they would actually make, at full strength.
- **Deliver**: each opposing case in its strongest form, which points the idea answers and which it does not, and what change or concession would win over the holdouts.

## Ground

### Check the Evidence

- **Probe**: which claims the idea rests on and where each came from.
- **Run**: list the load-bearing claims and verify them with web search and the provided material. Compare the plan with base rates for similar efforts (the outside view). Name sunk cost, anchoring, confirmation bias, survivorship bias, or motivated reasoning where the evidence shows them.
- **Deliver**: each claim marked **Supported**, **Contradicted**, or **Unverified**, with sources; the base-rate comparison; and each bias named with the evidence for it.

### Run the Numbers

- **Probe**: the inputs the user knows, such as volumes, prices, costs, and timelines, and their confidence in each.
- **Run**: build a back-of-the-envelope model with code execution. State each input with its source or assumption. Vary the uncertain inputs to find which ones flip the answer, and compute break-even where it applies.
- **Deliver**: the bottom line, the inputs it is most sensitive to, the break-even point, and the assumptions to verify first.

### Design the Cheapest Test

- **Probe**: the budget and time available, and what result would actually change the user's plan.
- **Run**: pick the riskiest load-bearing assumption and design the cheapest, fastest experiment that could prove it false, such as a fake door, a concierge trial, a prototype, a spike, a survey, or a data pull. Set success and kill criteria before the test runs.
- **Deliver**: the assumption under test, the experiment, its cost and duration, and the results that mean go, change course, or stop.

## Decide

### Explore Better Alternatives

- **Probe**: what has already been considered and rejected, and which constraints are real rather than assumed.
- **Run**: generate materially different approaches: a different mechanism, buying instead of building, a non-technical fix, or doing nothing. Then simplify: find the smallest version that tests the core bet, and what to cut.
- **Deliver**: three to five genuinely different alternatives, each with its key advantage and the condition under which it beats the user's idea, plus the smallest viable version of the idea itself.

### Compare My Options

- **Probe**: the options on the table, the decision deadline, the criteria that matter, and how much each criterion weighs.
- **Run**: score each option against the agreed criteria and weights, and check whether the winner holds when the weights shift.
- **Deliver**: a comparison table, the recommended option, the tradeoff it accepts, and what would flip the recommendation.

## Ship

### Review the Design

- **Probe**: the requirements, non-functional constraints such as scale, latency, and compliance, the team's skills, and who will own it.
- **Run**: evaluate complexity, coupling, ownership, maintainability, scalability, security, operability, and implementation risk. Inspect any provided diagrams, documents, or code directly.
- **Deliver**: the simplest design that meets the requirements, and the decisions that are expensive to reverse.

### Test Execution Readiness

- **Probe**: the timeline, team, budget, dependencies, approvals, and how success will be measured.
- **Run**: walk the plan for dependencies, sequencing, resources, ownership, adoption, governance, operations, rollout, and rollback. Check that success metrics are measurable and that kill criteria exist: the signals that mean stop or roll back.
- **Deliver**: a readiness check with each area marked **Ready**, **At risk**, or **Missing**; the blocking gaps; and the kill criteria.

### Prep Me for the Room

- **Probe**: the audience, their priorities and known objections, the format and time available, and the decision the user needs from them.
- **Run**: play the audience. Anticipate the hardest questions they would ask, including the one the user hopes they won't, and draft strong, honest answers. Flag the questions the user cannot answer yet.
- **Deliver**: the questions ranked by likelihood and difficulty, a draft answer for each, the weak spots to shore up before the meeting, and the single ask to make in the room.
