# Marketing Funnel Analytics — A From-Scratch Curriculum

**Who this is for:** an MBA-level learner with strong business sense and
**no technical background**. You will not be assumed to know SQL, Python,
Postgres, or the terminal. Every technical step is explained.

**What you will have when you finish:** a real PostgreSQL database you
built yourself, a synthetic-but-realistic marketing funnel dataset, a set
of analytical SQL queries, and a published interactive "console" that
presents the findings — plus the ability to **defend every line of it in
an interview.**

**Time budget:** ~1.5 focused days. A suggested schedule is at the end.

---

## How to use this document

This is a teaching document, not a copy-paste script. Each build step
follows the same rhythm:

1. **Concept** — the business idea, explained plainly.
2. **Your turn** — a prompt. Try it *before* reading further. This is
   where the learning happens.
3. **Checkpoint** — "you should now be able to…" so you know you're on
   track.
4. **Reference** — a worked answer inside a collapsed `▸ Reveal` block,
   with a **line-by-line explanation**. Open it only after you've
   attempted the step. The explanation is the point; the code is just the
   evidence.

> ⚠️ **The one rule that makes this worth doing:** if you copy the
> Reference blocks without attempting "Your turn" first, you will have a
> project you cannot explain. The entire value here is that you can sit
> across from an interviewer and justify every column and every `WHERE`
> clause. Attempt first. Always.

---

## Table of contents

- [Module 0 — Setup (≈45 min)](#module-0--setup-45-min)
- [Module 1 — The mental model (≈45 min)](#module-1--the-mental-model-45-min)
- [Module 2 — Design the three tables (≈90 min)](#module-2--design-the-three-tables-90-min)
- [Module 3 — Generate the data (≈90 min)](#module-3--generate-the-data-90-min)
- [Module 4 — Load the data into Postgres (≈45 min)](#module-4--load-the-data-into-postgres-45-min)
- [Module 5 — First funnel queries (≈90 min)](#module-5--first-funnel-queries-90-min)
- [Module 6 — The judgment traps (≈2.5 hrs)](#module-6--the-judgment-traps-25-hrs)
- [Module 7 — Cost & spend analysis (≈90 min)](#module-7--cost--spend-analysis-90-min)
- [Module 8 — Build the console (≈2 hrs)](#module-8--build-the-console-2-hrs)
- [Module 9 — Interview defense drills](#module-9--interview-defense-drills)
- [Suggested 1.5-day schedule](#suggested-15-day-schedule)
- [Glossary](#glossary)

---

## Module 0 — Setup (≈45 min)

### 0.1 What each piece of software does

You will touch four things. Here is what each one *is*, in plain terms:

| Tool | What it is | Why you need it |
|---|---|---|
| **PostgreSQL** ("Postgres") | A database — a program that stores tables and answers questions written in SQL. | It holds your data and runs your analysis. |
| **psql** | A text-based way to talk to Postgres from the terminal. | The quickest way to run a query. |
| **A GUI client** (TablePlus or DBeaver) | A point-and-click window into the database. | Far friendlier than the terminal for a non-technical learner. Recommended. |
| **Python** | A general programming language. | You'll use it once, to *generate* fake-but-realistic data. |

### 0.2 The terminal — the absolute minimum

The terminal is a window where you type commands instead of clicking.
Open it: press `Cmd + Space`, type `Terminal`, press Enter.

You only need three ideas:
- You type a command and press **Enter** to run it.
- A command that finishes silently usually **worked**. Errors are loud.
- `$` (or `%`) at the start of a line is the prompt — you don't type it.

### 0.3 Confirm Postgres is installed and start it

Postgres is already on this machine. Confirm and start the server (the
"server" is just the Postgres program running in the background so it can
answer questions):

```bash
psql --version
brew services start postgresql@16
```

If `psql --version` prints a version number, you're set. If
`brew services start` names a different version, use the name it suggests.
`brew services list` shows what's running.

> **Concept — client vs. server.** Postgres has two halves: a *server*
> that runs continuously and holds the data, and a *client* (like `psql`)
> that connects to it to ask questions. Starting the service starts the
> server. This client/server split is worth understanding — it's how
> nearly every real database works, and interviewers ask about it.

### 0.4 Create your database

A single Postgres server can hold many independent databases. Create one
for this project:

```bash
createdb marketing_funnel
```

Connect to it:

```bash
psql marketing_funnel
```

Your prompt changes to `marketing_funnel=#`. You are now "inside" the
database. Type `\q` and Enter to leave. Type `\dt` to list tables (none
yet). Commands starting with `\` are psql shortcuts, not SQL.

### 0.5 Install a GUI client (strongly recommended)

Typing long queries into a terminal is painful. Install one of these — a
window where you write queries in a proper editor and see results as a
grid:

- **TablePlus** — https://tableplus.com (free tier is enough). Easiest.
- **DBeaver** — https://dbeaver.io (fully free, open source).

In the client, create a new **PostgreSQL connection** with:
host `localhost`, port `5432`, database `marketing_funnel`, user = your
Mac username (leave password blank). Click Connect.

### 0.6 Git — save your progress

This is a git repository, which means every change can be saved as a
snapshot you can return to. You don't need to master git. You need three
commands, run from the project folder, whenever you finish something:

```bash
git add -A
git commit -m "Describe what you just did"
git push
```

**Checkpoint 0.** You can start Postgres, connect to `marketing_funnel`
in a GUI, run `SELECT 1;` and see a result, and commit to git.

---

## Module 1 — The mental model (≈45 min)

Do not skip this. Every mistake in funnel analytics is a modeling mistake
wearing a SQL costume. Get the model right and the SQL becomes easy.

### 1.1 The story we're modeling

A company runs marketing campaigns on several **channels** (say Google
Ads, LinkedIn, a webinar program, organic search). Those campaigns cost
money and produce **leads** — people who raised a hand. Leads then move
through a **funnel**: some become marketing-qualified, some become
sales-qualified, some turn into opportunities, and some finally close as
won (a paying customer) or lost.

The business questions we ultimately want to answer:

- How many leads make it through each stage? Where do they leak?
- What does a lead cost? What does a *customer* cost?
- Which channel is actually worth more money, and which is tapped out?

### 1.2 Four words that are not synonyms

Interviewers test this immediately. Say them out loud with the
distinction:

- **Lead** — a raw hand-raise. Might be junk, might be a real buyer. May
  be a duplicate of someone you already know.
- **Contact** — a known person, usually deduplicated, that you can
  actually reach.
- **Customer** — someone who has paid. A *state a contact reaches*, not a
  separate species.
- **Account** — the *company*, not the person. One account can contain
  many contacts (a buying committee).

> **Your turn (say it back).** In one sentence each, explain to an
> imaginary interviewer why "we got 500 leads" and "we got 500 customers"
> are wildly different claims, and why "cost per account" and "cost per
> lead" can point to opposite decisions.

### 1.3 Grain — the single most important idea in this project

**Grain = what one row of a table represents.** Before you create any
table, you must be able to finish the sentence "one row in this table is
exactly one ______."

Why it dominates everything: if you're fuzzy on the grain, every count,
average, and ratio you compute later is quietly wrong, and **no error
message will ever tell you.** The query runs. The number looks
reasonable. It's garbage.

Two grain sins you will be tempted to commit, and must not:

- **Fan-out.** Joining a table where one row = one lead to a table where
  one lead has *many* rows multiplies your rows. Sum anything afterward
  and it's inflated. No error is raised.
- **Mixing grains in one calculation.** You cannot divide a
  per-channel-per-day spend number by a per-lead table directly. You must
  first roll both sides up to a **common grain**.

### 1.4 Store events, not current state

Tempting design: give each lead a column `current_stage` and update it as
they move. **Resist this.** The moment you overwrite `mql` with `sql`,
you have destroyed the *when* — and every interesting funnel question
("how long does MQL→SQL take?", "how did the March cohort do?") is a
question about *when*.

The rule: **store the events you cannot rebuild; derive the state you
can.** An event ("lead 401 entered stage SQL at 09:14 on March 3") can
never be reconstructed once lost. "What stage is lead 401 in *now*?" can
always be derived from the events (it's the latest one). So store events.

> **Concept — funnel stages are values, not tables.** Beginners make one
> table per stage (`mqls`, `sqls`…). Don't. A stage is a *value in a
> column* of one events table. This keeps the funnel flexible and the
> queries sane.

### 1.5 Censoring — the trap that makes smart people look dumb

A **censored** record is one where the event you're measuring *hasn't
happened yet.* A lead sitting in "SQL" today has no "closed" event.

Here's the trap: if you measure "average time to close" by subtracting
timestamps, the still-open leads have nothing to subtract, so they
silently **drop out of the average** — which means your average is
computed only over leads that already closed, i.e. it **excludes exactly
the stuck leads that made you ask the question.** Your metric looks
precise and is deeply misleading.

Hold onto this. We will deliberately build censored leads into the data
so you can fall into the trap and then climb out of it.

**Checkpoint 1.** Without looking, you can define *grain*, explain
*fan-out*, argue why *events* beat a `current_stage` column, and describe
how *censoring* corrupts a naïve average.

---

## Module 2 — Design the three tables (≈90 min)

Now you design the schema. **This is the heart of the project.** Resist
the urge to peek at the Reference. Sit with the "Your turn" prompts — the
struggle is the education.

### 2.1 What we need to capture

Reread the story in 1.1. Three kinds of fact live in it:

1. **Who the leads are** and where each came from (fixed facts about a
   lead that don't change).
2. **What happened to each lead over time** (their movement through the
   funnel — events).
3. **What we spent** to acquire them.

That maps to three tables. Your job is to decide the **grain** of each
and the **columns** each should hold.

### 2.2 Your turn — design before you read

For **each** of the three tables, on paper, answer:

1. "One row in this table is exactly one ______." (the grain)
2. Which columns belong here — and, crucially, *why each one belongs on
   this table and not another?*
3. Which column uniquely identifies a row (the **primary key**)?
4. Which column points at another table (a **foreign key**)?

Two design tests to apply to every column you propose:

- **The change test:** does this fact ever change for a given row? Facts
  about *what a lead is* (its source) don't change → they belong on the
  lead. Facts about *what happened* change over time → they're events.
- **The grain test:** if I put this column here, does it stay true for
  every single row at this grain, or does it force me to repeat/duplicate
  data?

Write your three designs down **before** opening anything below.

### 2.3 Guided nudges (use only if stuck)

- Table 1 (leads): a lead's *source channel* — does it change after the
  lead is created? So where does it live? What about the lead's *current
  stage* — is that a fact that changes? (Recall 1.4.)
- Table 2 (events): if one lead can enter five stages, how many rows does
  that lead have in this table? So what's the grain — per lead, or per
  lead-per-stage?
- Table 3 (spend): you spend money on a *channel over time*, not on an
  individual lead. So what's the natural grain — per lead? per channel?
  per channel per day? Which one lets you say "we spent $400 on LinkedIn
  on March 3"?

### 2.4 A note on the source-casing trap (design it in on purpose)

Real CRM data is filthy. The same channel arrives as `Google`,
`google`, `Google Ads`, `GOOGLE`. If you `GROUP BY` that raw column,
Postgres treats each spelling as a different channel and your report
fractures. We will **deliberately dirty** the source column in the data
generator so you learn to normalize it. For now, just make sure your
leads table stores the *raw* source as it "arrived."

### 2.5 Reference schema — reveal only after designing

<details>
<summary>▸ Reveal the reference three-table design + line-by-line reasoning</summary>

This is *a* strong design, not the only one. If yours differs but passes
the change test and the grain test, yours may be just as defensible —
that's the point. Compare your reasoning to this reasoning.

**Table 1 — `leads` (a dimension: the fixed facts about each lead)**

- **Grain:** one row = one lead. Full stop.
- **Columns and why:**
  - `lead_id` — the primary key. Uniquely names the lead so events can
    point back to it.
  - `created_at` — when the lead first appeared. A fixed fact about the
    lead → belongs here, not in events.
  - `source_raw` — the channel as it arrived, *un-cleaned* (the dirt from
    2.4). A fixed fact → belongs here.
  - `campaign` — which campaign produced the lead. Fixed → here.
  - `region` — geography of the lead. Fixed → here. (Optional; include it
    if you want a second dimension to slice by.)
- **Why `current_stage` is deliberately absent:** stage changes over time
  (fails the change test) → it lives in events, not here. (Module 1.4.)

**Table 2 — `lead_stage_events` (a fact table: the funnel movements)**

- **Grain:** one row = one lead entering one stage at one moment. A lead
  that reaches four stages has four rows here.
- **Columns and why:**
  - `event_id` — primary key; names each event.
  - `lead_id` — foreign key pointing at `leads.lead_id`. This is how the
    two tables connect.
  - `stage` — the funnel stage as a *value* (`created`, `mql`, `sql`,
    `opportunity`, `closed_won`, `closed_lost`). Stages are values in a
    column, never separate tables. (Module 1.4.)
  - `event_at` — the timestamp of *this* stage entry. This column is what
    makes duration and cohort analysis possible.
- **How censoring shows up here:** a lead that's stuck in `sql` simply has
  **no** `closed_won`/`closed_lost` row. Its absence *is* the censoring.
  (Module 1.5.)

**Table 3 — `spend` (a fact table: money over time)**

- **Grain:** one row = one channel on one day.
- **Columns and why:**
  - `spend_id` — primary key.
  - `channel` — the channel the money was spent on (stored clean here, so
    you'll later reconcile it against the *dirty* `source_raw` on leads —
    a realistic, teachable friction).
  - `spend_date` — the day. Spend is a per-day fact.
  - `amount` — dollars spent that day on that channel.
  - `impressions`, `clicks` — optional, but they let you compute
    click-through and richer efficiency metrics.
- **Why spend is NOT on the leads table:** you spend on a *channel over
  time*, not on an individual lead. Putting a spend column on `leads`
  would force you to invent a per-lead number that doesn't exist, and it
  would fail the grain test the instant a channel had spend on a day it
  produced zero leads. Cost-per-lead is a **ratio computed by rolling
  both tables up to a common grain**, never a stored column. (Module
  1.3.)

**The relationships, in one breath:** `lead_stage_events.lead_id` →
`leads.lead_id` (many events per lead). `spend.channel` relates to
`leads.source_raw` only *after* you normalize the casing and aggregate —
they are **different grains** and must never be joined row-to-row.

</details>

### 2.6 Your turn — write the `CREATE TABLE` statements

A `CREATE TABLE` statement tells Postgres the table's name, its columns,
and each column's **type** (`INT` for whole numbers, `TEXT` for strings,
`TIMESTAMP` for dates+times, `NUMERIC` for money, `DATE` for a day).

Write all three `CREATE TABLE` statements yourself. If you hit a syntax
wall, ask me to explain the specific piece of syntax — don't ask me to
write the statement.

<details>
<summary>▸ Reveal reference DDL (only after attempting) — with an explanation of every clause</summary>

```sql
-- "DDL" = Data Definition Language: statements that define structure.

CREATE TABLE leads (
    lead_id     INTEGER PRIMARY KEY,   -- unique id; PRIMARY KEY = unique + not null
    created_at  TIMESTAMP NOT NULL,    -- when the lead appeared; NOT NULL = required
    source_raw  TEXT NOT NULL,         -- channel as it arrived, un-cleaned
    campaign    TEXT,                  -- nullable: some leads have no campaign
    region      TEXT
);

CREATE TABLE lead_stage_events (
    event_id  INTEGER PRIMARY KEY,
    lead_id   INTEGER NOT NULL REFERENCES leads(lead_id),  -- FK: must match a real lead
    stage     TEXT NOT NULL,           -- 'created','mql','sql','opportunity','closed_won','closed_lost'
    event_at  TIMESTAMP NOT NULL
);

CREATE TABLE spend (
    spend_id     INTEGER PRIMARY KEY,
    channel      TEXT NOT NULL,
    spend_date   DATE NOT NULL,
    amount       NUMERIC(10,2) NOT NULL,  -- money: 10 digits, 2 after the decimal
    impressions  INTEGER,
    clicks       INTEGER
);
```

Clause-by-clause:
- `PRIMARY KEY` — guarantees the column is unique and never null; it's how
  a single row is addressed.
- `NOT NULL` — the column is required; refuse rows that leave it blank.
  Used for facts that must always exist (a lead must have a created time).
- `REFERENCES leads(lead_id)` — a **foreign key**. Postgres will now
  *refuse* to insert an event whose `lead_id` doesn't exist in `leads`.
  This enforces the relationship at the database level — a lead's events
  can't point at a ghost.
- `NUMERIC(10,2)` — exact decimal for money. Never store money as a
  floating-point type; rounding errors compound. `(10,2)` = up to 10
  total digits, 2 after the decimal point.
- Why `campaign`/`region` are *not* `NOT NULL` — real leads sometimes
  lack a campaign or region; forcing them would mean fabricating data.

Run these in your GUI client. Then `\dt` in psql (or refresh the client)
should show three tables.

</details>

**Checkpoint 2.** Three empty tables exist. You can state the grain of
each in one sentence and justify why each column sits where it sits — and
why `current_stage` and a per-lead `spend` column do **not** exist.

Commit: `git commit -m "Design and create the three tables"`.

---

## Module 3 — Generate the data (≈90 min)

Empty tables teach nothing. You need realistic data — and building the
generator yourself teaches you *exactly* what's in the data, which is what
makes your later analysis defensible.

### 3.1 Why synthetic data, and why generate it yourself

You don't have a real CRM, and even if you did, you couldn't share it.
Synthetic data lets you *control the truth* — you decide the real
conversion rates, then check whether your queries recover them. If your
query says LinkedIn converts at 4% and you *built it* to convert at 4%,
your query is trustworthy. That's a rare luxury; use it.

### 3.2 What "realistic" must include (design the realism on purpose)

Your generator must bake in the phenomena the analysis depends on.
Decide, before coding, how you'll produce each:

1. **A leaky funnel.** Not every lead advances. Each stage passes only a
   *fraction* forward. (This creates the drop-off you'll measure.)
2. **Channel-varying quality.** Different channels convert at different
   rates (so cost-per-customer differs from cost-per-lead).
3. **Time spread.** Leads arrive across a date range (so cohorts exist).
4. **Realistic delays.** Moving from one stage to the next takes days or
   weeks — later stages take longer (so duration analysis is meaningful).
5. **Censoring, on purpose.** Recent leads should *not* have finished the
   funnel yet — they're still in flight. (So the censoring trap is real.)
6. **Dirty source casing.** Emit the same channel under several spellings
   in `source_raw`. (So normalization is required.)
7. **Spend that roughly tracks leads** but is set independently per
   channel per day (so cost ratios are non-trivial).

### 3.3 Your turn — plan the generator's logic in words

Before writing any Python, write the *algorithm* in plain English. For
example: "For each of N leads, pick a created date; pick a channel
(weighted); pick a messy spelling of that channel; emit a `created`
event; then with probability p_mql advance to MQL after a random delay;
if MQL, with probability p_sql advance to SQL; …; if a computed event
date is in the future relative to 'today', **don't emit it** (that's the
censoring)." Get the logic right on paper first.

### 3.4 Python you actually need — a 10-minute primer

You'll use only these ideas. Ask me to expand any of them:

- A **variable** stores a value: `n_leads = 2000`.
- A **list** holds many values: `channels = ["google", "linkedin"]`.
- A **loop** repeats: `for i in range(n_leads):` does something 2000×.
- **`random`** makes randomness: `random.random()` gives a number in
  [0,1) (compare it to a probability to decide yes/no);
  `random.choices(list, weights=...)` picks with weights.
- **`datetime`/`timedelta`** do date math: a start date `+ timedelta(days=k)`.
- **`csv`** writes spreadsheet files your database can read.
- Python uses **indentation** (spaces) to show what's inside a loop. This
  matters — misaligned spaces are the #1 beginner error.

### 3.5 Your turn — write `generate_data.py`

Write a Python script that produces three CSV files — `leads.csv`,
`lead_stage_events.csv`, `spend.csv` — matching your table grains. Attempt
the whole thing. When it errors (it will), read the error's last line —
it usually names the file, the line number, and what went wrong — and fix
that line. Bring me the error message, not a request to write it.

Run it with:

```bash
python3 generate_data.py
```

<details>
<summary>▸ Reveal reference generator (only after a real attempt) — heavily commented</summary>

Study the comments; they explain *why*, which is what you'll be asked
about. Tune the probabilities and counts to your taste — owning those
numbers is part of owning the project.

```python
import csv
import random
from datetime import datetime, timedelta

random.seed(42)  # makes the "random" data identical every run → reproducible.
                 # An interviewer likes reproducibility; it means your
                 # results aren't a fluke of one lucky run.

N_LEADS = 2000
START = datetime(2025, 1, 1)          # first possible lead date
TODAY = datetime(2025, 7, 24)         # our "now" — anything after this is the future
DAYS_RANGE = (TODAY - START).days     # how wide the arrival window is

# Channels differ in VOLUME (weights) and QUALITY (conversion multipliers).
# quality > 1 means this channel's leads advance more readily.
channels = {
    "google":   {"weight": 40, "quality": 1.0, "spellings": ["google", "Google", "Google Ads", "GOOGLE"]},
    "linkedin": {"weight": 25, "quality": 1.4, "spellings": ["linkedin", "LinkedIn", "Linkedin"]},
    "webinar":  {"weight": 15, "quality": 1.8, "spellings": ["webinar", "Webinar", "WEBINAR"]},
    "organic":  {"weight": 20, "quality": 0.7, "spellings": ["organic", "Organic", "organic search"]},
}
channel_names = list(channels.keys())
channel_weights = [channels[c]["weight"] for c in channel_names]

# Base pass-through rates for the funnel, applied stage to stage.
# These are the "truth" your queries should later recover.
BASE = {"mql": 0.55, "sql": 0.45, "opportunity": 0.40, "won": 0.30}

# Typical delay (in days) to reach each stage — later stages take longer.
DELAY = {"mql": 3, "sql": 10, "opportunity": 25, "closed": 45}

leads_rows, event_rows = [], []
event_id = 1

for lead_id in range(1, N_LEADS + 1):
    # --- pick when this lead arrived ---
    created = START + timedelta(days=random.randint(0, DAYS_RANGE),
                                hours=random.randint(0, 23))

    # --- pick a channel by weight, then a messy spelling of it ---
    chan = random.choices(channel_names, weights=channel_weights)[0]
    source_raw = random.choice(channels[chan]["spellings"])   # the dirt (2.4)
    quality = channels[chan]["quality"]

    leads_rows.append([lead_id, created.isoformat(), source_raw,
                       f"campaign_{random.randint(1,6)}",
                       random.choice(["NA", "EMEA", "APAC"])])

    # --- every lead has a 'created' event at its created time ---
    def emit(stage, when):
        global event_id
        # CENSORING: if the computed date is in the future, the event
        # hasn't happened yet — do NOT emit it. This is the whole trap.
        if when > TODAY:
            return False
        event_rows.append([event_id, lead_id, stage, when.isoformat()])
        event_id += 1
        return True

    emit("created", created)

    # --- walk the funnel; each stage gated by a probability × channel quality ---
    t = created
    # MQL
    if random.random() < min(BASE["mql"] * quality, 0.95):
        t = t + timedelta(days=random.expovariate(1/DELAY["mql"]))
        if not emit("mql", t):
            continue          # future → lead is censored here; stop
        # SQL
        if random.random() < min(BASE["sql"] * quality, 0.95):
            t = t + timedelta(days=random.expovariate(1/DELAY["sql"]))
            if not emit("sql", t):
                continue
            # Opportunity
            if random.random() < min(BASE["opportunity"] * quality, 0.95):
                t = t + timedelta(days=random.expovariate(1/DELAY["opportunity"]))
                if not emit("opportunity", t):
                    continue
                # Closed won or lost
                t = t + timedelta(days=random.expovariate(1/DELAY["closed"]))
                if random.random() < min(BASE["won"] * quality, 0.95):
                    emit("closed_won", t)
                else:
                    emit("closed_lost", t)

# --- SPEND: one row per channel per day, set independently of leads ---
spend_rows = []
spend_id = 1
day = START
while day <= TODAY:
    for chan in channel_names:
        # rough daily budget per channel, with noise
        base_budget = {"google": 500, "linkedin": 350, "webinar": 150, "organic": 40}[chan]
        amount = round(base_budget * random.uniform(0.7, 1.3), 2)
        impressions = int(amount * random.uniform(80, 120))
        clicks = int(impressions * random.uniform(0.01, 0.04))
        spend_rows.append([spend_id, chan, day.date().isoformat(), amount, impressions, clicks])
        spend_id += 1
    day += timedelta(days=1)

# --- write the three CSVs ---
def write_csv(name, header, rows):
    with open(name, "w", newline="") as f:
        w = csv.writer(f)
        w.writerow(header)
        w.writerows(rows)

write_csv("leads.csv", ["lead_id","created_at","source_raw","campaign","region"], leads_rows)
write_csv("lead_stage_events.csv", ["event_id","lead_id","stage","event_at"], event_rows)
write_csv("spend.csv", ["spend_id","channel","spend_date","amount","impressions","clicks"], spend_rows)

print(f"Wrote {len(leads_rows)} leads, {len(event_rows)} events, {len(spend_rows)} spend rows.")
```

Things worth being able to explain about this generator:
- **`random.expovariate`** produces delays that cluster small with a long
  tail — more realistic than a fixed delay, and it's *why* your duration
  averages will be skewed (a good thing to notice later).
- The **censoring `if when > TODAY`** is the single most important line.
  Recent leads get cut off mid-funnel exactly like real in-flight leads.
- Spend is generated in a **separate loop** with its **own grain** — this
  is the physical embodiment of "different grains" from Module 1.

</details>

**Checkpoint 3.** Three CSV files exist in the project folder. You can
open `leads.csv` in a spreadsheet and see messy source spellings, and you
can explain which line of your generator creates the censoring.

Commit your generator (but consider not committing the CSVs — ask me
about a `.gitignore` if you want to keep generated files out of git).

---

## Module 4 — Load the data into Postgres (≈45 min)

### 4.1 Concept — `COPY` reads a CSV straight into a table

Postgres has a `COPY` command that bulk-loads a CSV into a table, matching
columns by position. The CSV's column order must match the table's column
order (yours do, by design).

### 4.2 Your turn — load all three CSVs

Look up the `COPY … FROM … WITH (FORMAT csv, HEADER)` syntax and load each
file. You'll need the **absolute path** to each CSV (the full path from
`/`). `pwd` in the terminal prints your current folder.

<details>
<summary>▸ Reveal reference load commands + explanation</summary>

```sql
-- Run inside psql, connected to marketing_funnel.
-- Replace /FULL/PATH/ with the output of `pwd` in your project folder.

COPY leads (lead_id, created_at, source_raw, campaign, region)
FROM '/FULL/PATH/leads.csv' WITH (FORMAT csv, HEADER true);

COPY lead_stage_events (event_id, lead_id, stage, event_at)
FROM '/FULL/PATH/lead_stage_events.csv' WITH (FORMAT csv, HEADER true);

COPY spend (spend_id, channel, spend_date, amount, impressions, clicks)
FROM '/FULL/PATH/spend.csv' WITH (FORMAT csv, HEADER true);
```

- `HEADER true` tells Postgres the first CSV row is column names, not
  data — so it skips it.
- If you get a **permissions** error, it's because server-side `COPY`
  needs Postgres to be able to read the file. The fix that always works:
  use psql's client-side version, `\copy` (same syntax, no `WITH`
  keyword capitalization change needed), which reads the file *as you*:

  ```
  \copy leads FROM 'leads.csv' WITH (FORMAT csv, HEADER true)
  ```

  Run from the folder that contains the CSVs so the relative path works.
- **Load order matters:** load `leads` **before** `lead_stage_events`.
  The foreign key means an event can't reference a lead that isn't there
  yet.

</details>

### 4.3 Your turn — verify the load

Write three quick `SELECT COUNT(*) FROM …;` queries and confirm the counts
match your generator's printout. Then eyeball ten rows of each with
`SELECT * FROM leads LIMIT 10;`.

**Checkpoint 4.** All three tables are populated, counts match, and a
`SELECT` returns real rows. Commit.

---

## Module 5 — First funnel queries (≈90 min)

Now you learn SQL by answering real questions. Each sub-section teaches
one or two new pieces of syntax. **Predict the answer before you run
each query** — that habit is how you catch wrong-but-plausible results.

### 5.1 SQL's shape

Every query you'll write has this skeleton. Learn the reading order:

```
SELECT   columns / calculations        -- 5. what to show
FROM     table                          -- 1. start here
JOIN     other_table ON ...             -- 2. attach related rows
WHERE    row filter                     -- 3. keep some rows
GROUP BY columns                        -- 4. collapse into groups
HAVING   group filter                   -- 4b. keep some groups
ORDER BY columns                        -- 6. sort
```

You write it top-to-bottom, but Postgres *executes* roughly `FROM → WHERE
→ GROUP BY → HAVING → SELECT → ORDER BY`. Knowing that order explains most
"why doesn't my query work" confusion.

### 5.2 Your turn — how many leads do we have, by channel?

The trap is built in: the raw source is dirty. First run it the naïve way
and *see* the fracture; then fix it. Concepts you'll need: `COUNT(*)`,
`GROUP BY`, and a normalizing function (`LOWER()` collapses casing;
you'll still need to decide how to unify `google` vs `Google Ads`).

<details>
<summary>▸ Reveal — the naïve version, why it's wrong, and the fix</summary>

Naïve (wrong):
```sql
SELECT source_raw, COUNT(*) AS lead_count
FROM leads
GROUP BY source_raw
ORDER BY lead_count DESC;
```
This returns a *separate* row for `google`, `Google`, `Google Ads`,
`GOOGLE` — the same channel split four ways. Every downstream per-channel
number would be wrong. **This is the casing trap from Module 2.4, live.**

The fix is to normalize into a clean channel first. `LOWER()` handles case
but not `google` vs `google ads`; you need a deliberate mapping. A `CASE`
expression is the readable way:
```sql
SELECT
  CASE
    WHEN LOWER(source_raw) LIKE 'google%'   THEN 'google'
    WHEN LOWER(source_raw) LIKE 'linkedin%' THEN 'linkedin'
    WHEN LOWER(source_raw) LIKE 'webinar%'  THEN 'webinar'
    WHEN LOWER(source_raw) LIKE 'organic%'  THEN 'organic'
    ELSE 'other'
  END AS channel,
  COUNT(*) AS lead_count
FROM leads
GROUP BY 1                 -- "1" = group by the first SELECT expression
ORDER BY lead_count DESC;
```
- `CASE … END` is SQL's if/else; it maps every dirty spelling to one clean
  channel.
- `LIKE 'google%'` matches anything starting with "google" (`%` = "any
  characters"), catching `google ads` too.
- `GROUP BY 1` groups by the computed `channel`. (You can't reference the
  alias `channel` in `GROUP BY` in standard SQL, but you can reference its
  position.)

> **Interview note:** the fact that you *normalize source before grouping*
> is exactly the kind of judgment that separates an analyst from someone
> who can type SQL. Do it every time and say why.

</details>

### 5.3 Your turn — the funnel counts (how many leads reach each stage?)

You want one number per stage: how many distinct leads ever entered it.
The subtlety: you're counting **leads**, not **events**, and you must not
double-count. Concept you'll need: `COUNT(DISTINCT …)`.

<details>
<summary>▸ Reveal — stage counts, and the DISTINCT reasoning</summary>

```sql
SELECT stage, COUNT(DISTINCT lead_id) AS leads_reaching_stage
FROM lead_stage_events
GROUP BY stage
ORDER BY leads_reaching_stage DESC;
```
- `COUNT(DISTINCT lead_id)` counts *unique leads*, so if a lead somehow
  had two `mql` rows it still counts once. At this grain it's a safety
  belt; the habit matters more than this dataset needing it.
- Order will roughly follow the funnel: `created` > `mql` > `sql` >
  `opportunity` > (`closed_won` + `closed_lost`).

> **Predict-first drill:** before running, write down your expected order
> of the stages. If `sql` outnumbers `mql`, something is wrong — you can't
> reach SQL without passing MQL. Catching that impossibility is the skill.

</details>

### 5.4 Your turn — stage-to-stage conversion rates

A count is raw; a *rate* is the insight. Compute the percentage of leads
that pass from each stage to the next. This needs the counts from 5.3 and
a bit of arithmetic. Concept you'll need: dividing one aggregate by
another (and the integer-division gotcha — `5/2` is `2` in SQL unless you
make one side decimal).

<details>
<summary>▸ Reveal — conversion rates with the integer-division fix</summary>

One approachable way — pull the stage counts into a small set of values
and divide. A compact version using conditional aggregation:

```sql
SELECT
  COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'mql')::numeric
    / COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'created') AS created_to_mql,
  COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'sql')::numeric
    / COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'mql')       AS mql_to_sql,
  COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'opportunity')::numeric
    / COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'sql')       AS sql_to_opp,
  COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'closed_won')::numeric
    / COUNT(DISTINCT lead_id) FILTER (WHERE stage = 'opportunity') AS opp_to_won
FROM lead_stage_events;
```
- `COUNT(...) FILTER (WHERE …)` counts only rows meeting a condition —
  conditional aggregation. It lets you compute several stage counts in one
  pass.
- `::numeric` casts the top of the fraction to a decimal so the division
  isn't integer division. Without it, `380 / 2000` would be `0`.
- These recovered rates should land near the `BASE` rates you *built into*
  the generator (times channel-quality mix). If they don't, either your
  query or your understanding of the data is off — investigate.

</details>

**Checkpoint 5.** You can produce clean per-channel lead counts, the
funnel stage counts, and stage-to-stage conversion rates, and you can
explain the normalization and the `::numeric` cast. Commit.

---

## Module 6 — The judgment traps (≈2.5 hrs)

This module is the reason the project is worth doing. Anyone can `GROUP
BY`. The analyst is the person who *doesn't* report the naïve number.
Each trap below: you'll compute the tempting-but-wrong version, see why
it's wrong, then compute the honest version.

### 6.1 Censored averages — "how long does it take to close?"

**Your turn (the wrong way first).** Compute average days from `created`
to `closed_won` by subtracting timestamps. To do this you'll join the
events table to itself (the `created` row and the `closed_won` row for the
same lead). Concept: a **self-join**, and date subtraction.

Now the critical question — **before you trust the number, ask: which
leads are in this average, and which silently fell out?**

<details>
<summary>▸ Reveal — the naïve duration, why it lies, and the honest framing</summary>

Naïve:
```sql
SELECT AVG(EXTRACT(EPOCH FROM (won.event_at - c.event_at)) / 86400) AS avg_days_to_won
FROM lead_stage_events c
JOIN lead_stage_events won
  ON won.lead_id = c.lead_id AND won.stage = 'closed_won'
WHERE c.stage = 'created';
```
- The `JOIN … ON same lead_id` with different stages is the self-join: it
  pairs each lead's `created` row with its `closed_won` row.
- `EXTRACT(EPOCH FROM interval)/86400` converts the time difference to
  days.

**Why it lies:** the `JOIN` only keeps leads that *have* a `closed_won`
row. Every lead still working through the funnel — every censored lead —
has no such row and is silently excluded. So this "average time to close"
is computed **only over leads that already closed**, which are
disproportionately the *fast* ones and the *old* ones. It systematically
**understates** reality and ignores exactly the stuck leads that motivate
the question. (This is Module 1.5, made concrete.)

**The honest response is not a different `AVG` — it's a reframe:**
- State the metric's population explicitly: "mean time to close *among
  leads that have closed*," and report **how many leads that is vs. how
  many are still open** (the censored count). A number without its
  denominator is a half-truth.
- To compare fairly over time, fix a **measurement window** (see 6.3):
  e.g. "of leads created in January, what share closed *within 90 days*?"
  — every lead gets the same 90-day chance, so censoring is handled by
  design, not by deletion.

Being able to *say this out loud* is the single most valuable sentence in
your interview.

</details>

### 6.2 Fan-out — the join that inflates every total

**Your turn.** Try to answer "total spend and total leads per channel" by
joining `leads` to `spend` on channel and summing. Predict the spend total
first. Then run it and compare to the true total spend
(`SELECT SUM(amount) FROM spend;`).

<details>
<summary>▸ Reveal — watch the total explode, and why</summary>

The tempting query:
```sql
SELECT s.channel, SUM(s.amount) AS total_spend, COUNT(*) AS n
FROM spend s
JOIN leads l ON l.source_raw ILIKE s.channel || '%'   -- rough channel match
GROUP BY s.channel;
```
Compare `SUM(s.amount)` here to the real `SELECT SUM(amount) FROM spend;`.
It will be **massively larger** — often by hundreds of times.

**Why:** `spend` has one row per channel per day. `leads` has many leads
per channel. Joining them pairs *every* spend row with *every* lead of
that channel, so each spend row is duplicated once per matching lead. Sum
the amount and you've counted each dollar hundreds of times. **No error is
raised. The number just looks big and wrong.** (Module 1.3.)

**The fix is a mindset:** never join two fact tables of different grain
and then sum. Instead, **aggregate each side to a common grain first**,
*then* join the summaries. You'll do exactly that in Module 7. Write on a
sticky note: *aggregate, then join — never join, then aggregate across
grains.*

</details>

### 6.3 Cohort attribution & cohort maturity

**Concept.** "What's our conversion rate?" hides a choice:
- **Conversion-date attribution:** count a win in the month it *closed*.
  Good for "what did we book this month," bad for judging a channel,
  because the win's cost was incurred months earlier.
- **Cohort (created-date) attribution:** credit a win back to the month
  the lead was *created*. This is what you want to judge acquisition — it
  lines the win up with the spend that caused it.

The catch is **maturity**: a cohort created last week can't have closed
yet (deals take ~45+ days here). Comparing last week's cohort to
January's raw conversion makes recent cohorts look terrible purely because
they're *young*, not worse.

**Your turn.** Build a cohort conversion table: group leads by their
created-month, and for each cohort compute the share that closed won. Then
identify the flaw and fix it by fixing the window.

<details>
<summary>▸ Reveal — cohort conversion, the maturity flaw, and the fixed-window fix</summary>

Naïve cohort table:
```sql
SELECT date_trunc('month', l.created_at) AS cohort_month,
       COUNT(DISTINCT l.lead_id) AS cohort_size,
       COUNT(DISTINCT won.lead_id) AS won,
       COUNT(DISTINCT won.lead_id)::numeric / COUNT(DISTINCT l.lead_id) AS won_rate
FROM leads l
LEFT JOIN lead_stage_events won
  ON won.lead_id = l.lead_id AND won.stage = 'closed_won'
GROUP BY 1
ORDER BY 1;
```
- `date_trunc('month', …)` buckets timestamps into their month — that's
  the cohort.
- `LEFT JOIN` keeps every lead even if it never won (an inner join would
  drop the non-winners and destroy the denominator — a subtle,
  devastating bug). The non-winners contribute `NULL` for `won.lead_id`,
  which `COUNT(DISTINCT won.lead_id)` correctly ignores.

**The flaw:** the most recent cohorts show a collapsing `won_rate` — not
because they're bad, but because they haven't had *time* to close. You're
comparing a 6-month-old cohort to a 2-week-old one on the same yardstick.

**The fix — fix the window, don't freeze the number.** Measure every
cohort at the *same age*: "share of the cohort that closed won **within 90
days of creation**." Add a time condition to the win:
```sql
... LEFT JOIN lead_stage_events won
      ON won.lead_id = l.lead_id
     AND won.stage = 'closed_won'
     AND won.event_at <= l.created_at + INTERVAL '90 days'
-- and only include cohorts old enough to HAVE 90 days of data:
WHERE l.created_at <= (DATE '2025-07-24' - INTERVAL '90 days')
```
Now every included cohort has had a full 90-day chance, so differences are
real signal, not maturity artifacts. State the window every time you quote
the rate: "90-day closed-won rate," never a bare "conversion rate."

</details>

### 6.4 Predictive metrics — estimating what hasn't happened yet

**Concept.** Because young cohorts are censored, you can *estimate* their
eventual outcome: take a **mature** cohort's 90-day conversion rate and
apply it to a young cohort's *current volume*. That's a forecast — clearly
labeled as one. **Your turn:** in words, describe how you'd project the
number of customers the June cohort will eventually yield, and state the
one assumption that makes it valid.

<details>
<summary>▸ Reveal — the projection logic and its load-bearing assumption</summary>

Logic: `projected_wins(June) = size(June cohort) × mature_90day_won_rate`.
The **assumption that must hold: no drift** — that the conditions
producing the mature rate (audience quality, product, pricing, sales
motion) still hold for June. If you changed channel mix or ran a
discount, the borrowed rate is invalid. Always name the assumption when
you present a projection; an unlabeled forecast presented as fact is how
analysts lose credibility.

</details>

**Checkpoint 6.** You can *demonstrate* each trap — run the wrong query,
explain precisely what it miscounts, and produce the honest version. This
is your interview centerpiece. Commit.

---

## Module 7 — Cost & spend analysis (≈90 min)

Here the grain lessons pay off. Every cost metric is a **ratio across two
tables of different grain**, so every one requires aggregate-then-join.

### 7.1 Cost per lead, done correctly

**Concept.** Cost per lead by channel = (total spend on that channel) ÷
(leads from that channel). Two different grains → you must roll each to
`channel` first, *then* divide.

**Your turn.** Using a **CTE** (a named temporary result — `WITH name AS
(…)`), build one summary of spend-per-channel and one of leads-per-channel
(normalizing the dirty source!), then join those two summaries and divide.
This directly avoids the fan-out from 6.2.

<details>
<summary>▸ Reveal — cost per lead via aggregate-then-join</summary>

```sql
WITH spend_by_channel AS (          -- grain: one row per channel
    SELECT channel, SUM(amount) AS total_spend
    FROM spend
    GROUP BY channel
),
leads_by_channel AS (               -- grain: one row per channel (normalized!)
    SELECT
      CASE
        WHEN LOWER(source_raw) LIKE 'google%'   THEN 'google'
        WHEN LOWER(source_raw) LIKE 'linkedin%' THEN 'linkedin'
        WHEN LOWER(source_raw) LIKE 'webinar%'  THEN 'webinar'
        WHEN LOWER(source_raw) LIKE 'organic%'  THEN 'organic'
        ELSE 'other'
      END AS channel,
      COUNT(*) AS lead_count
    FROM leads
    GROUP BY 1
)
SELECT s.channel,
       s.total_spend,
       l.lead_count,
       ROUND(s.total_spend / l.lead_count, 2) AS cost_per_lead
FROM spend_by_channel s
JOIN leads_by_channel l ON l.channel = s.channel
ORDER BY cost_per_lead DESC;
```
- Each CTE collapses its table to **one row per channel** — a shared
  grain. Only *then* do we join, so one spend total meets one lead count.
  No fan-out. This is 6.2's lesson applied.
- `ROUND(…, 2)` presents cents.
- Note we normalize `source_raw` *inside* the leads CTE — the casing trap
  never sleeps.

</details>

### 7.2 Cost per customer — and why it differs from cost per lead

**Your turn.** Replace "leads per channel" with "closed-won *customers*
per channel" and recompute. Predict, before running: will webinar's cost
*per customer* be relatively better or worse than its cost *per lead*?
Why? (Recall you built webinar to convert at higher quality.)

<details>
<summary>▸ Reveal — cost per customer and the interpretation</summary>

Swap the second CTE to count wins by channel:
```sql
customers_by_channel AS (
    SELECT
      CASE  -- same normalization block as before
        WHEN LOWER(l.source_raw) LIKE 'google%'   THEN 'google'
        WHEN LOWER(l.source_raw) LIKE 'linkedin%' THEN 'linkedin'
        WHEN LOWER(l.source_raw) LIKE 'webinar%'  THEN 'webinar'
        WHEN LOWER(l.source_raw) LIKE 'organic%'  THEN 'organic'
        ELSE 'other'
      END AS channel,
      COUNT(DISTINCT won.lead_id) AS customers
    FROM leads l
    JOIN lead_stage_events won
      ON won.lead_id = l.lead_id AND won.stage = 'closed_won'
    GROUP BY 1
)
```
then `total_spend / customers` = cost per customer.

**Interpretation:** a channel can have a *high* cost per lead but a *low*
cost per customer if its leads convert well (webinar, by construction).
The reverse (cheap leads, expensive customers) is the classic trap of
optimizing the wrong metric — buying lots of cheap junk leads that never
close. **Cost per customer is closer to the decision** because customers,
not leads, pay you.

</details>

### 7.3 Marginal vs. average cost — the metric that actually drives spend

**Concept.** Average cost per customer tells you *how you did*. It does
**not** tell you what to do next, because spending *more* on a channel
reaches a *worse* audience — the eager buyers came cheap, the next dollar
finds fence-sitters. So the **marginal** cost of the next customer is
higher than the average. The decision rule: **spend up to where marginal
cost per customer ≈ the value of a customer**, not where average cost
looks good.

**Your turn (analysis, not just SQL).** Split each channel's history into
an early period and a later period (or low-spend days vs high-spend days),
compute cost per customer in each, and observe whether the later/higher
period costs more per customer — evidence of diminishing returns. Then
write the recommendation *in business terms*.

<details>
<summary>▸ Reveal — a diminishing-returns cut and how to read it</summary>

One simple cut: compare the first half of the date range to the second.
```sql
-- Sketch: compute customers and spend per channel within each half of
-- the calendar, using WHERE spend_date / created_at < or >= the midpoint,
-- then cost_per_customer for each half, per channel.
```
Reading it:
- If cost per customer **rises** in the higher-spend half, you're seeing
  diminishing returns — the audience got worse as you pushed volume.
- **The recommendation is rarely "kill the channel."** A rising *average*
  usually means *spend less* (pull back to where the margin is healthy),
  not zero. Killing a channel throws away the cheap early customers too.
- Say it in business language: "LinkedIn's marginal cost per customer in
  the high-spend period ($X) exceeds our ~$Y customer value, so I'd cap
  LinkedIn spend near the level where it was still under $Y, and reallocate
  the excess to webinar, whose marginal cost is still below value."

That sentence — average for scoring, margin for deciding, reallocate
rather than kill — is a senior-analyst answer.

</details>

**Checkpoint 7.** You can compute cost per lead and cost per customer by
channel without fan-out, and you can articulate the marginal-vs-average
distinction and turn it into a spend recommendation. Commit.

---

## Module 8 — Build the console (≈2 hrs)

The deliverable that makes this portfolio-ready: a single interactive page
that presents the funnel story. This is where the analysis becomes
something you can *show*.

### 8.1 What the console should contain

Design it as a narrative, top to bottom:
1. **The funnel** — stage counts and conversion rates, as a funnel chart.
2. **Channel scorecard** — leads, customers, cost per lead, cost per
   customer per channel, in a table.
3. **Cohort view** — the fixed-window (90-day) conversion by cohort month,
   with young/immature cohorts clearly marked.
4. **The judgment callouts** — short written notes stating the censoring
   caveat, the normalization, and the marginal-cost recommendation. *These
   text notes are what prove you understand the numbers.*

### 8.2 How to build it (two honest options)

- **Option A — static numbers, hand-placed (recommended for the
  deadline).** Run your Module 5–7 queries, copy the resulting numbers
  into a self-contained HTML page. Simple, fast, fully in your control,
  and you can explain every figure. The numbers don't auto-update, which
  is fine for a portfolio piece.
- **Option B — export-driven.** Have Postgres write query results to CSV,
  and have the page read those CSVs. More "live," more moving parts. Only
  if you have time to spare.

Go with **A** unless you're ahead of schedule.

### 8.3 Your turn — assemble the numbers, then the page

First, produce a small results table for each of the four sections by
running your queries and recording the outputs (a scratch text file is
fine). *Then* build the page. When you're ready to publish it as a shareable
interactive artifact, tell me — I can help you turn your assembled HTML
into a published console (like your existing Accountability Console),
without writing your analysis for you. The words and numbers must be
yours; I'll help with the presentation scaffolding and publishing.

> **Design help is fair game.** Per your working agreement, I won't write
> your SQL or your analytical conclusions — but layout, chart scaffolding,
> and publishing are tooling, and I'll help with those freely. Ask.

**Checkpoint 8.** A single page exists that a stranger could read and
understand your funnel, your channel economics, and — critically — the
caveats. Commit and (when ready) publish.

---

## Module 9 — Interview defense drills

Do not skip this. The project's whole purpose is that you can defend it.
For each question below, answer **out loud** without looking. If you
stumble, that's your signal to re-read the relevant module — *before* the
interview, not during.

**On modeling**
1. What is the grain of each of your three tables? Why those grains?
2. Why is there no `current_stage` column on `leads`? What did that buy
   you?
3. Why are funnel stages values in a column instead of separate tables?
4. Why does `spend` live in its own table at its own grain instead of a
   column on `leads`?

**On the traps**
5. Your "average days to close" — which leads does it include, which does
   it silently exclude, and in which direction is it biased?
6. Show me a join that would inflate total spend. Why does it inflate?
   How do you prevent it?
7. Two cohorts, one from January and one from last week, show very
   different conversion. How much of that is real and how much is
   maturity? How do you separate them?
8. You group leads by `source` and Google appears four times. What
   happened and what's your fix?

**On the economics**
9. Cost per lead vs. cost per customer — which drives a channel decision,
   and why can they disagree?
10. Average cost per customer says $180 and rising. Do you kill the
    channel? What number would you actually base the decision on?
11. You want to forecast the June cohort's eventual customers. How? What
    single assumption makes your forecast valid or worthless?

**On the craft**
12. Walk me through one query line by line. (Pick any. You should be able
    to do this for *every* query in the project. If you can't, you copied
    it — go back and rebuild it.)

---

## Suggested 1.5-day schedule

**Day 1 — morning (Setup + model + design)**
- Module 0 — Setup (45 min)
- Module 1 — Mental model (45 min)
- Module 2 — Design the three tables (90 min) ← the intellectual core

**Day 1 — afternoon (Data + first queries)**
- Module 3 — Generate the data (90 min)
- Module 4 — Load into Postgres (45 min)
- Module 5 — First funnel queries (90 min)

**Day 1 — evening OR Day 2 — early morning (Judgment)**
- Module 6 — The judgment traps (2.5 hrs) ← the interview centerpiece

**Day 2 — morning/afternoon (Economics + deliverable)**
- Module 7 — Cost & spend analysis (90 min)
- Module 8 — Build the console (2 hrs)
- Module 9 — Interview drills (ongoing; do them as you go)

If time gets tight, protect Modules 2, 6, and 9 — the design, the traps,
and the defense. They are what an interviewer probes. Everything else is
plumbing.

---

## Glossary

- **Grain** — what one row of a table represents.
- **Primary key** — the column that uniquely identifies each row.
- **Foreign key** — a column pointing at another table's primary key.
- **Fan-out** — row multiplication from joining different grains; inflates
  sums silently.
- **Censoring** — the measured event hasn't happened yet; naïve averages
  drop these rows and mislead.
- **Cohort** — a group of leads bucketed by when they were created.
- **Maturity** — how much time a cohort has had to convert; must be
  equalized before comparing cohorts.
- **CTE** (`WITH … AS`) — a named temporary result you can build on; used
  to aggregate-then-join.
- **Conditional aggregation** — `COUNT(...) FILTER (WHERE …)`; several
  filtered counts in one pass.
- **Marginal cost** — the cost of the *next* customer; drives decisions.
  Distinct from **average cost**, which scores the past.
- **Drift** — the conditions behind a borrowed rate no longer hold, so the
  rate no longer applies.

---

*Built for interview-defensibility. If you ever can't explain a line,
that line isn't done — rebuild it until you can.*
