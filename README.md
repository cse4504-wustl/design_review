# Application Design Review Studio

In this studio, your team will audit the request objects from your original design, discover the abstract interfaces your use cases need, and assign every concrete implementation to a named teammate. This corresponds to the "In class design in a feature branch" step of the workflow; all changes go into the `cli` branch of your team project repository.

By the end of class, you should be able to:

1. Tell apart *user input* (what the controller can get from the person at the keyboard) from *system state* (what the application has stored), and keep system state out of request objects.
2. Derive the interfaces a use case needs by walking through its steps, and define them from the use case's point of view rather than a technology's.
3. Plan concrete implementations of those interfaces so that no teammate is blocked waiting on another.

**GenAI policy.** Same as the application design assignment: all design ideas and decisions come from the team. No GenAI for auditing, interface design or task splitting.

## Overview and Schedule

| Time (min) | Segment | Format | Output |
| --- | --- | --- | --- |
| 20 | Finish application design | Team | Required `DESIGN.md` sections complete (checklist below) |
| 10 | Part 1a: Trace every request field | Individual, own use case | Annotated request object in notes file |
| 10 | Part 1b: Team audit | Team | Audit notes, revised requests |
| 12 | Part 2a: Walk through your use case | Individual | Step list with EXTERNAL needs |
| 13 | Part 2b: Merge and sketch interfaces | Team | Interface files in `interfaces/` |
| 5 | Part 3: Assign implementation owners | Team | Implementation list with owners |
| 3 | Commit and push | Team | Commit on `cli` |

After the first 20 minutes, every team starts Part 1 wherever it is. Before then, `DESIGN.md` must have:

- **Use Cases**: request and response objects for all three user stories
- **Use Case Response**: how use cases deliver the response
- **Error Handling**: bad input and other failures, with examples

Coding standards, the `.gitignore` and entity files can be finished after class if needed; the studio does not depend on them.

## Part 1: Audit your request objects (20 min)

A request object may contain only what the controller can get from the user: values they type, choices they make, and data parsed from a file they point to. Anything the application already has stored is *system state*, and fetching it is the use case's job, not the controller's.

### 1a. Trace every field (individual, 10 min)

Create a notes file (e.g. `notes.md`) **outside** your repo folder, so it can't be committed by accident. All of your individual work in 1a and 2a goes there.

For the use case assigned to you, copy your request object from `DESIGN.md` into your notes file and annotate each field with its source in brackets, e.g. `tournamentId: string [typed by user]`. Use one of these sources:

- [typed by user]
- [chosen from a menu]
- [parsed from a user-supplied file]
- **[stored in the system]**
- **[computed from stored data]**

Any field whose source is "stored in the system" or "computed from stored data" is a red flag. Replace that field with the smallest piece of user input that lets the use case find it (usually an identifier or name), and note what the use case must now look up. Add these to your notes file as a "Lookups" list: it feeds Part 2. Bring your notes to the team audit in 1b.

Also check that no field is a file name, path, or other external format (carried over from the original assignment).

### Worked example: Withdraw a player

**Before** (controller would need the database):

- `tournament`: a full Tournament entity, including all players and rounds
- `player`: a full Player entity, including current score
- `remainingRounds`: number of rounds not yet paired

**After** (controller needs only the keyboard):

- `tournamentId`: string the user types or selects
- `playerId`: string the user types or selects
- `effectiveRound`: number the user types

The use case now looks up the tournament and player itself, and computes remaining rounds from the tournament entity. That lookup is the first hint of an interface.

Parsing a CSV of new registrations is still fine: the user supplied that file, so the controller converts it to a list of raw player records. What changed is *who knows the data*, not *where it came from on disk*.

### 1b. Team audit (10 min)

Present your work from 1a to the rest of the team. Team: review critically and flag errors, using these questions:

1. Could a controller build this request object with no access to stored data? If not, which field breaks the rule?
2. Does any field contain an entity object, or a list of things the system already knows?
3. Is any field derived (a count, a total, a standing) rather than entered?
4. Is any field an external format (file name, JSON string, raw command-line text)?
5. Is anything the use case clearly needs *missing* from the request?

As a team, revise the request objects in the Use Cases section of `DESIGN.md` based on the audit. One scribe makes all the edits on their laptop; nobody else changes files in the repo.

When the edits are done, the scribe pushes and everyone else pulls:

1. Scribe: `git add DESIGN.md`, then `git commit -m "Revise request objects after audit"`, then `git push origin cli`
2. Everyone else: `git checkout cli`, then `git pull origin cli`

Use the same routine at the end of 2b and Part 3 (the scribe also adds `interfaces/`).

## Part 2: Discover and sketch interfaces (25 min)

Every time a use case needs something from outside the boundary, it needs an interface. The use case owns that interface: it is named and shaped by what the use case needs, and it lives in `interfaces/`.

### 2a. Walk through your use case (individual, 12 min)

In your notes file, write your use case as numbered plain-English steps, from receiving the request to delivering the response. Tag each step:

- **[INTERNAL]** the use case or an entity does it (validation, business rules, building the response)
- **[EXTERNAL]** it needs something from outside the boundary

For every [EXTERNAL] step, write down what the use case needs in its own words, e.g. "get the tournament with this ID" or "save the updated tournament." Common [EXTERNAL] needs to check for:

- Loading or saving stored data (the lookups you moved out of the request in Part 1)
- Delivering the response, if your team chose the presenter approach in "Use Case Response"
- Generating unique IDs
- Calls to a 3rd party library or tool (this does not include standard libraries that the programming language provides)

**Worked example (Withdraw a player):**

1. [EXTERNAL] Get the tournament with `tournamentId`; if none, report a bad-input error.
2. [INTERNAL] Find the player in the tournament; if absent, report a bad-input error.
3. [INTERNAL] Check the business rule that a player cannot withdraw from a round already completed.
4. [INTERNAL] Mark the player withdrawn from `effectiveRound` onward.
5. [EXTERNAL] Save the updated tournament; if saving fails, report a system failure.
6. [EXTERNAL] Deliver the response to the presenter.

### 2b. Merge and sketch (team, 13 min)

Put the three [EXTERNAL] lists side by side. Many needs will overlap (all three use cases probably load a tournament). Then:

1. Group the needs into interfaces. One interface per kind of collaborator is a reasonable default (e.g. one for tournament storage, one presenter per use case).
2. Name each interface for its role, not its technology: `TournamentRepository`, not `JsonFileReader` or `SqliteDatabase`.
3. Write each method's signature in your chosen language, in a file in `interfaces/`. Parameters and return values may only be entities, DTOs, or primitive types; never file handles, JSON strings, SQL rows, or console output.
4. For each method, write a one-line comment stating what happens when it cannot do its job (not found, storage unavailable), consistent with your "Error Handling" section.
5. Remove any method no use case needs yet.

A sketch at this level is enough (shown language-neutral):

```
interface TournamentRepository
    findById(id: string) -> Tournament or nothing   // nothing if no such tournament
    save(tournament: Tournament)                     // signals a system failure if storage fails

interface WithdrawPlayerPresenter
    presentSuccess(response: WithdrawPlayerResponse)
    presentError(error: ErrorResponse)
```

## Part 3: Split up the implementations (5 min)

Every interface needs a concrete implementation for the CLI. In class, the team lists them and gives each one an owner.

**Step 1: List every implementation.** For each interface from Part 2, decide:

- The **CLI implementation**: e.g. a `TournamentRepository` that saves to a local file, or an `InMemoryTournamentRepository` that keeps tournaments in a list or map while the program runs.

Concrete implementations live outside the boundary. Presenters go in the existing `cli` directory. Every other implementation goes in a new, dedicated directory that the team creates and names for its role, such as `persistence/` for storage.

**Step 2: Assign owners.** Add a plain list like this to `DESIGN.md`:

```markdown
## Interface Implementations

- FileTournamentRepository (implements TournamentRepository)
  - Directory: persistence/
  - Used by: US-1, US-2, US-3
  - Owner: <name>
- ConsoleWithdrawPlayerPresenter (implements WithdrawPlayerPresenter)
  - Directory: cli/
  - Used by: US-3
  - Owner: <name>
```

**Rules for a fair split:**

1. Everyone owns at least one implementation that someone *else's* use case depends on.
2. Balance the load: integration with 3rd party tools is heavier than a console presenter, so its owner takes fewer other items.
3. Changing a shared interface after today requires a separate, dedicated pull request reviewed by every teammate.

## Deliverables

By the end of class, the `cli` branch of your team repo must have::

- **Use Cases** section of `DESIGN.md` updated with revised request objects, plus one line per use case on what changed after the audit (or "no change")
- New **Interfaces** section in `DESIGN.md`: each interface, its methods in one line each, and which use cases depend on it
- One file per interface in `interfaces/`, in the team's language, with the failure comment on each method
- New **Interface Implementations** section in `DESIGN.md` with the owner list from Part 3, including each implementation's directory
