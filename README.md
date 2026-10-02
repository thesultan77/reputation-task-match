# ReputationTaskMatch

**Skill-Verified Freelance Matching with On-Chain Reputation**

A task-matching escrow contract where candidate ranking isn't based on
semantic fit alone — it's weighted by a worker's track record of genuinely
AI-verified completed work, built entirely from this contract's own history
rather than imported from anywhere else.

---

## What it does

1. Workers **register a skill description**, which is embedded into a
   vector and stored in a `VecDB` alongside their address.
2. A poster calls **`find_best_match`** with a task description. Candidates
   are ranked not just by semantic similarity to the task, but by a combined
   score: `similarity + (completed_tasks * 0.05)`. A worker with a strong
   completion history will outrank a worker with marginally higher raw
   similarity but no track record.
3. The poster **funds a task** (`post_task`, payable) assigned to a specific
   worker.
4. The worker **submits a deliverable URL**. The contract fetches the page
   content and asks an LLM whether it satisfies the task description,
   reaching validator consensus (`gl.eq_principle.prompt_comparative`) on
   the categorical `SATISFIED`/`UNSATISFIED` outcome before paying out.
5. On `SATISFIED`, the worker is paid **and their completion count
   increments** — directly feeding back into `find_best_match`'s ranking
   for every future task. On `UNSATISFIED`, the poster is refunded and no
   reputation is gained.
6. `cancel_task` and `claim_timeout_refund` (48h) provide fund-safety exits
   if a worker never delivers.

## Why this is different

Most on-chain task-matching contracts treat every match request as if every
candidate were a stranger — pure semantic similarity, no memory of who has
actually delivered before. ReputationTaskMatch closes that loop: every
successfully judged deliverable strengthens that worker's future ranking,
so the matching engine gets smarter about who to recommend the more the
contract is used — without needing any off-chain reputation system or
oracle. The reputation signal is generated, verified, and consumed entirely
within the contract itself.

## Architecture

| Concern | Mechanism |
|---|---|
| Skill registry & search | `VecDB[float32, 384, WorkerProfile]` — vector search over registered skills |
| Reputation tracking | `TreeMap[str, u256]` keyed by worker address string, incremented only on AI-verified `SATISFIED` completions |
| Ranking | `find_best_match` combines vector similarity with a reputation bonus (`completed_tasks * 0.05`), re-sorted and truncated to `top_n` |
| Embeddings | `SentenceTransformer("all-MiniLM-L6-v2")`, called via `get_embedding_generator()(text)` |
| AI judgment | `gl.nondet.exec_prompt` + `gl.nondet.web.render` inside a closure, reconciled across validators via `gl.eq_principle.prompt_comparative` |
| Fund safety | Stakes only move on `SATISFIED` (worker paid), `UNSATISFIED` (poster refunded), `cancel_task` (poster refunded pre-delivery), or `claim_timeout_refund` (poster refunded post-deadline) |

Address-taking parameters (`worker_address`) are typed as `str` and
converted with `Address(worker_address)` rather than typed `Address`
directly — a GenVM `Address`-typed parameter arrives inside the contract as
a raw integer rather than a proper `Address` object, which complicates
storage; accepting the hex string and converting explicitly avoids that
entirely.

## Contract methods

### Write

- **`register_skill(skill_description: str)`**
  Registers the caller as a worker with the given skill description.
- **`post_task(task_description: str, worker_address: str)`** *(payable)*
  Poster funds and assigns a task to a specific worker address.
- **`submit_deliverable(url: str)`**
  Assigned worker submits their deliverable URL, triggering AI judgment,
  payout/refund, and (on success) a reputation increment.
- **`cancel_task()`**
  Poster reclaims their stake if no deliverable has been submitted yet.
- **`claim_timeout_refund()`**
  Anyone can trigger a refund to the poster once the 48-hour deadline has
  passed with no deliverable submitted.

### View

- **`find_best_match(task_desc: str, top_n: int) -> list`**
  Returns the top-N workers ranked by `similarity + reputation bonus`, each
  with their worker address, skill text, similarity, completed-task count,
  and combined match score.
- **`get_worker_reputation(worker_address: str) -> str`**
- **`get_current_task() -> str`**
- **`get_last_result() -> str`**
- **`get_total_workers() -> str`**
- **`get_total_completed() -> str`**

## Tested end-to-end in GenLayer Studio

- Registered two worker skill profiles (Python backend, React frontend).
- Baseline `find_best_match` confirmed `match_score == similarity` exactly
  when `completed_tasks` is `0` for all candidates (similarity `0.7955852`
  observed for both profiles pre-completion).
- `post_task` → `submit_deliverable` (a clean, self-contained Gist raw-text
  URL as the deliverable) → `SATISFIED`, validator consensus reached,
  worker paid.
- `get_worker_reputation` confirmed the completion count incrementing from
  `"0"` to `"1"` after the successful completion.
- Re-running `find_best_match` with identical inputs after the completion
  showed `match_score` rising to `0.8455852` — exactly `similarity + 0.05`
  — confirming the reputation bonus correctly shifts ranking as designed.
- `cancel_task` safety path confirmed `SUCCESS`/`FINALIZED` with correct
  refund behavior.
- All test transactions reached `FINALIZED`/`SUCCESS` with supermajority
  validator agreement.

## Deployment

```
# v0.2.16
# { "Seq": [
#     { "Depends": "py-lib-genlayer-embeddings:09h0i209wrzh4xzq86f79c60x0ifs7xcjwl53ysrnw06i54ddxyi" },
#     { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }
# ] }
```

Constructor takes no arguments. Deploy directly in GenLayer Studio or via
`genlayer-js` against Studionet / Bradbury testnet.

## Notes on GenVM quirks discovered while building this

- `gle.SentenceTransformer("all-MiniLM-L6-v2")` returns a **callable**, not
  an object with `.encode()`. Call it directly: `model(text)`, not
  `model.encode(text)`. The reliable pattern is an instance method
  (`get_embedding_generator`) that returns the model, called via
  `self.get_embedding_generator()(txt)`.
- Prefer typing address-taking method parameters as `str` and converting
  with `Address(the_string)`, rather than typing the parameter `Address`
  directly — the latter arrives inside the contract as a raw integer, not a
  proper `Address` object, which requires extra byte-conversion handling to
  store safely.
- When reading a `TreeMap` key that may not exist yet (e.g. a worker who
  hasn't completed any tasks), wrap the read in `try/except` and default to
  zero rather than assuming the key is always present.

## License

MIT
