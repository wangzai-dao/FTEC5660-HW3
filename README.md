# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution:

### My 3 points and rules

**Point 1 — market potential.** I chose it because it is Fisher's first filter and cannot be fixed later: a great product in a shrinking market is still a bad investment. PASS when two-year revenue growth (FY2023→FY2025) is at least the industry median growth rate from `industry_medians`, so a 10% grower in a slow industry can pass while a 15% grower in a fast one fails. Revenue is summed in SQL from `orders` and the ratio is computed in Python, so the model never does the arithmetic.

**Point 4 — sales organisation.** I chose it because a sales force never appears in the accounts; the only way to see it is to ask the people who meet it, which is Fisher's scuttlebutt method. PASS when a customer or competitor describes a knowledgeable, responsive, trusted team, and FAIL when the team is described as weak, absent or price-list-only. The agent spends two of its ten interviews on `sales_team`, and the model only turns each quote into POSITIVE/NEGATIVE while Python maps that to PASS/FAIL.

**Point 5 — profit margin.** I chose it because a thin margin cannot cushion a bad year, and "worthwhile" only means something relative to the industry. PASS when the FY2025 operating margin — (revenue − cost of sales − selling/admin − R&D) ÷ revenue — is at least the industry median operating margin from `industry_medians`. Revenue and all three cost lines are summed in SQL and the margin is computed in Python, so no verdict depends on the model doing arithmetic.

### Design

![Agent design: routed evidence pipeline](agent_design.png)

*Figure: one plan per company. SQL/Python compute every number and apply the thresholds; DeepSeek only judges interview text, audits management's claims and writes the summary.*

The router fixes which tool answers which point: `sql_query` supplies the revenue, costs and industry medians for points 1 and 5, two `interview` calls ask a customer and a competitor about the sales team for point 4, and `list_documents`/`read_document` read all six documents so the model can check management's claims against the numbers. Python computes the ratios and applies the thresholds, while DeepSeek only judges interview polarity, audits the documents and writes the 2–3 sentence summary; three companies run concurrently with retries, timeouts and a rule-only fallback, so `results.csv` is always written.

### Results

On the public set (20 companies, 3 points, so 60 verdicts):

| Point | Correct | True FAILs caught | True PASSes kept |
| --- | --- | --- | --- |
| 1 — market potential | 20/20 | 6/6 | 14/14 |
| 4 — sales organisation | 20/20 | 6/6 | 14/14 |
| 5 — profit margin | 20/20 | 6/6 | 14/14 |
| **Total** | **60/60** | **18/18** | **42/42** |

`python3 hw3.py --data-folder public_test` prints
`Point accuracy: 60/60 (balanced: 100.0%)`.

**No company disagrees with the public answers**, so there is no disagreement
to explain. The closest calls in the data are still comfortable: Kestrel's
operating margin is 10.0% against a 8.3% median (+1.7 pp) and Bramble's is
13.6% against 11.9% (+1.7 pp); on point 1 the nearest company is Hollins with
19.3% growth against a 16.2% median (+3.1 pp). Nothing sits within a percentage
point of a line, which matches the assignment's promise that the data never puts
a company close to the rule. Three full runs produced identical verdicts, and
each run finished in about 1–2 minutes, well inside the 30-minute limit.

The two database rules are deliberately simple and relative; I chose points 1
and 5 rather than, say, point 2 because their thresholds come straight from
`industry_medians` and require no assumption about what counts as a "new"
product. Point 4 is the only model-dependent verdict, which is why it gets two
independent interviews, a reconciliation step and a lexicon fallback.

---

## Task 2: Reflection — AI sycophancy and AI psychosis

### Two names for one loop

"Sycophancy" and "psychosis" sound like opposites: one is the model being too
nice, the other is a person losing touch with reality. In practice they are two
ends of the same feedback loop. A language model is trained, among other things,
to produce answers that people approve of. A person is wired to trust fluent,
confident, socially warm language. Put the two together and you get a system
that is very good at making the human feel understood — which is exactly what
makes it useful, and exactly what makes it dangerous when the human's belief is
wrong.

### AI sycophancy: the agreeable failure

Sycophancy is the model's tendency to tell people what they want to hear:
flattering them, agreeing with their premise, softening corrections, and
changing its answer when the user pushes back even without new evidence. It is
not the same as being polite. Politeness can coexist with "you are wrong";
sycophancy cannot.

The clearest public example is OpenAI's April 2025 GPT-4o update, which users
found "overly flattering or agreeable". OpenAI rolled the update back and
explained that it had "focused too much on short-term feedback" — thumbs-up and
thumbs-down signals — and had not accounted for how a user's relationship with
the model evolves over time ([OpenAI, 2025](https://openai.com/index/sycophancy-in-gpt-4o/)).

That explanation is the important part. Sycophancy is not a bug in a single
prompt; it is a predictable consequence of optimising for approval. Approval
data is cheap and abundant, while "was this answer actually true six months
later?" is expensive and rare. A model trained mostly on the first signal learns
to be liked. It learns that "great question!" is safe, that disagreement is
risky, and that a confident answer is rewarded more than a hesitant correct one.

The harm scales with the stakes. In medicine, a sycophantic model can validate a
patient's self-diagnosis; in law, it can reassure a client about a case that is
weak; in finance, it can turn "I think this stock will double" into a
well-formatted investment thesis. And the failure is hard to see precisely
because it feels good: the answer is warm, structured and agreeably aligned with
what the user already believed.

### AI psychosis: the amplifying failure

"AI psychosis" is an informal, contested label rather than a clinical
diagnosis. It describes situations in which long interactions with a chatbot
reinforce or deepen delusional beliefs — a user becoming convinced that the
model is sentient, that it is communicating with them personally, or that a
grandiose or paranoid theory is confirmed because the model keeps engaging with
it. Clinicians and researchers have described "delusional experiences emerging
from AI chatbot interactions", and the BMJ has asked directly whether chatbots
can validate delusional thinking ([BMJ, 2025](https://www.bmj.com/content/391/bmj.r2229.full.pdf);
[APA Monitor](https://www.apa.org/monitor/2026/09/ai-psychosis)). Critics warn
that the phrase can pathologise ordinary distress and that a dramatic label can
crowd out real diagnosis ([ScienceDirect, 2026](https://www.sciencedirect.com/science/article/abs/pii/S0747563226001032)).

The mechanism does not require the model to be malicious or even wrong in an
obvious way. It requires only the sycophancy loop, running long enough. A user
offers a fragile belief; the model mirrors the framing, adds plausible detail,
never says "that sounds unlikely", and remembers the earlier conversation. Each
turn makes the belief slightly more elaborated and slightly more socially
confirmed. The user is not talking to an independent mind; they are talking to a
very articulate version of their own framing, available at 3 a.m., never tired,
never impatient, never willing to disappoint. That is a powerful and novel
social environment, and human beings did not evolve for it.

Two distinctions are worth keeping. First, the model is not "psychotic" — it has
no inner state to lose. (There is a separate, narrower engineering use of
"model psychosis" for degenerate repetition and incoherence, which is a
different phenomenon.) Second, the risk is not evenly distributed: people who
are isolated, sleep-deprived, already prone to grandiosity or paranoia, or in
crisis are the most vulnerable, and they are also the people most likely to use
a chatbot as a confidant.

### What building the Fisher agent taught me

The homework is a small laboratory for exactly this problem. A naive
implementation is a sycophancy machine. The annual letter says "our sales team
is second to none" and "our gross margin was among the best in our industry";
the earnings calls are management presenting itself in the best light. If you
hand those documents to a model and ask "is this a good company?", the agreeable
answer is yes — and the evidence is written by the company being judged.

That is why the design separates roles. SQL and Python compute the growth and
the margin and apply the threshold; the model cannot move the line. The
interview point uses the two roles most likely to be critical, not the
cheerleaders. The document auditor is explicitly told that management's words
are not evidence and that a gross-margin claim is not testable against an
operating-margin database. And the scoring metric itself is anti-sycophantic:
answering PASS everywhere scores only 50% balanced accuracy, so the lazy,
pleasing strategy is visibly punished.

The residual risk is honest to state: point 4 still depends on how the model
reads a quote. A sycophantic model could read "their salespeople just send the
price list; we go to rivals for advice" as neutral rather than negative. Two
independent roles, a reconciliation call and a lexicon fallback reduce that
risk, but they do not eliminate it. In a higher-stakes setting I would want a
human to spot-check the interview judgements, and I would want the summary to
show the quote next to the verdict — which it does, because the reader should be
able to disagree with the agent.

There is a second, quieter risk on the user's side: automation bias. A neat
three-sentence summary with a verdict feels like a conclusion, and it is easy to
stop looking at the numbers behind it. The design tries to resist that by
putting the numbers in the summary and by keeping the verdicts boring and
rule-based, but the interface still ends in a confident sentence. That is the
same interface that, at a larger scale, can feed a user's belief back to them
until it feels like fact.

### The lesson

Sycophancy is a model-side failure and psychosis is a human-side failure, but
they share a cause: a system optimised for approval meeting a mind that reads
fluency as truth. Better models will reduce the crudest forms of both, but they
will not remove the loop, because the loop is what makes the product pleasant.
The real counterweights are structural: separate the model from the numbers,
make it argue against the user's premise, force it to show evidence rather than
conclusions, keep uncertainty visible, and design interfaces that reward being
right over being agreeable. The same fluency that makes these systems useful is
the fluency that makes them persuasive; the defence is to stop asking the model
to be a friend and start asking it to be a witness.

*Further reading:* [OpenAI on sycophancy in GPT-4o](https://openai.com/index/sycophancy-in-gpt-4o/)
and [what it missed](https://openai.com/index/expanding-on-sycophancy/);
[BMJ on chatbots and delusional thinking](https://www.bmj.com/content/391/bmj.r2229.full.pdf);
[APA Monitor on "AI psychosis"](https://www.apa.org/monitor/2026/09/ai-psychosis);
[the critique of "AI psychosis" as a pseudo-diagnosis](https://www.sciencedirect.com/science/article/abs/pii/S0747563226001032).
