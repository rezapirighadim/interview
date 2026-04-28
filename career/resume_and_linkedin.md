# Resume & LinkedIn for Developers — International Job Search

This guide is written for developers who are serious about landing a remote or international role. No fluff, no "tailor your resume to each job" advice without showing you how. Real templates, real rewrites, honest talk about the things most guides skip.

---

## 1. Resume Fundamentals

### Length

| Experience level | Length |
|---|---|
| 0–5 years | 1 page, always |
| 5–10 years | 1 page preferred; 2 pages acceptable if genuinely dense |
| 10+ years | 2 pages max; cut anything older than 10 years unless it's remarkable |

One page forces you to prioritize. Two-page resumes usually just have more filler. When in doubt, cut.

### ATS-Friendly Format

Applicant Tracking Systems parse your resume before a human sees it. Most ATS tools struggle with:
- Tables (often parsed as garbled text)
- Headers and footers (often skipped)
- Text boxes and graphics
- Multi-column layouts (some ATS read column 1 then column 2, not left-to-right)
- PDFs with embedded fonts they can't extract

**Safe format:**
- Single column
- Standard section headers: **Summary**, **Experience**, **Skills**, **Education**
- Simple fonts: Calibri, Garamond, Georgia, Arial, or Liberation Serif — all render well everywhere
- Font size: 10–12pt body, 14–16pt name, don't go smaller than 10
- Margins: 0.5–0.75 inches
- Export as PDF (preserves layout) unless the job posting specifically asks for .docx

### Section Order

```
[Your Name]
[City, Country | email | GitHub | LinkedIn | phone (optional)]

SUMMARY (2–3 sentences, optional but useful for career changers or senior folks)

EXPERIENCE
  Company Name — Job Title                    Month Year – Month Year
  • bullet
  • bullet

SKILLS
  Languages: Python, Go, TypeScript
  Frameworks: FastAPI, React, gRPC
  Infrastructure: AWS, Kubernetes, Terraform, PostgreSQL

EDUCATION
  Degree, University Name, Year
```

Put Education last unless you're a new graduate. Your skills section should be scannable in 5 seconds — use horizontal groupings, not a wall of tags.

---

## 2. Writing Bullet Points That Work

### The Formula

```
[Action verb] + [what you did] + [measurable result]
```

The action verb is what you did. The result is why it matters. Most people write only the first part.

### Before / After Rewrites

**1. Infrastructure**
- Weak: "Managed the company's AWS infrastructure"
- Strong: "Redesigned AWS infrastructure for a 2M-user platform, cutting monthly cloud costs by 35% ($8K/month) by right-sizing EC2 instances and migrating static assets to CloudFront"

**2. API Development**
- Weak: "Built REST APIs using Django"
- Strong: "Built and maintained 12 REST API endpoints in Django serving 500K daily requests, achieving 99.95% uptime over 18 months"

**3. Performance**
- Weak: "Improved application performance"
- Strong: "Reduced p99 API response time from 2.1s to 180ms by introducing Redis caching for hot database queries, improving checkout completion rate by 12%"

**4. Team / Leadership**
- Weak: "Led a team of developers"
- Strong: "Led a team of 4 backend engineers, delivered a real-time bidding system 2 weeks ahead of schedule, adopted by 3 enterprise clients in the first month"

**5. Database**
- Weak: "Worked with databases"
- Strong: "Migrated 50M+ records from MySQL to PostgreSQL with zero downtime using a dual-write strategy and a 3-week phased rollout"

**6. Testing**
- Weak: "Wrote unit tests"
- Strong: "Increased backend test coverage from 24% to 78%, reducing production bug rate by ~60% over the following quarter"

**7. Feature delivery**
- Weak: "Developed new features for the product"
- Strong: "Designed and shipped a multi-currency payment feature supporting 8 currencies, directly enabling expansion into 3 new markets and contributing to a 20% increase in international revenue"

**8. On-call / reliability**
- Weak: "Was responsible for on-call duties"
- Strong: "Reduced mean time to resolution (MTTR) from 45 minutes to 12 minutes by building a centralized alerting dashboard and runbooks for the 10 most common incident types"

### Quantifying Without Exact Numbers

You don't need the exact number. You need a credible approximation.

- "approximately 200K daily active users" is fine
- "reduced latency by roughly 40%" is fine
- "a team of ~6 engineers" is fine
- "saved around $3K/month in infrastructure costs" is fine

If you genuinely have no numbers: describe scope instead.
- "Core service in a monolith serving all B2B clients"
- "Sole backend engineer responsible for the billing system"
- "High-traffic notification service (~1M messages/day)"

Numbers you can usually get from memory or back-of-envelope: team size, rough user count, database size, number of services you owned, deploy frequency, incident count.

---

## 3. Framing Experience from Regional Companies

### The core challenge

A hiring manager in Berlin, Toronto, or San Francisco may have never heard of your company. That's fine — but you need to give them context quickly. They're not going to Google it during resume review.

### Add a one-line company descriptor

```
Snapp! — Senior Backend Engineer                          2021 – 2023
Iran's largest ride-hailing platform (~20M users, ~300 engineers)
```

```
Digikala — Software Engineer                              2020 – 2022
Iran's leading e-commerce marketplace, comparable to Amazon.com in the region
```

```
CafeBazaar — Platform Engineer                            2019 – 2021
Android app store with 50M+ users across the Middle East and Central Asia
```

This gives context without requiring the reader to have prior knowledge. Use analogies sparingly and accurately — "comparable to X" sets expectations, so don't oversell.

### Describing scale in internationally understood terms

| Instead of | Say |
|---|---|
| "High traffic system" | "~500K daily active users, ~50M requests/day" |
| "Large database" | "PostgreSQL cluster with ~800GB of data, 200M+ records" |
| "Worked on a big team" | "Engineering organization of ~150, team of 8" |
| "Popular product" | "Ranked #1 in its category on Google Play in Iran, 4M installs" |

Absolute numbers are almost always more convincing than relative claims.

### Handling Employment Gaps

Be straightforward. A gap is not disqualifying — a fumbled explanation is.

- 6 months or less: you don't need to explain it on your resume. Gaps under 6 months are normal.
- 6 months to 2 years: add a brief line if the gap was for something real: "Career break: family care", "Independent study: systems programming + Rust", "Personal project: built and launched X"
- Longer gaps: address it directly in your cover letter or LinkedIn, not your resume

In interviews: "I took time off to [reason]. During that time I [what you did or learned]. I'm ready to go full-time now and here's what I've been doing to stay current."

Never apologize for a gap. Own it.

### Addressing Location and Remote Work Preference

On your resume, in the header:
```
Tehran, Iran | Open to remote | reza@email.com | github.com/reza
```

Or if you're relocating:
```
Tehran, Iran (relocating to Berlin, Q3 2026) | reza@email.com
```

If you're open to either:
```
Tehran, Iran | Remote or relocation | reza@email.com
```

Don't hide your location. Companies that discriminate based on location alone are not companies you want to work for — better to filter them out early.

---

## 4. LinkedIn Optimization

### Headline Formula

Your headline should not be your job title. That's the default and nobody reads it.

Formula:
```
[What you do] | [specialization or notable tool/stack] | [who you help or what you're after]
```

Examples:
- "Backend Engineer | Go & Distributed Systems | Open to remote roles"
- "Senior Software Engineer | Python, Kubernetes, AWS | Building scalable data platforms"
- "Full Stack Developer | React + Node.js | Fintech & Payments"
- "Platform Engineer | Infrastructure & SRE | Helping teams ship faster"

Keep it under 120 characters. Avoid buzzwords like "passionate" or "results-driven" — show, don't tell.

### About Section Structure

Three paragraphs. No more. Each has a job:

**Paragraph 1: Who you are**
3–4 sentences. Your professional identity, years of experience, what kind of engineer you are.

> "I'm a backend engineer with 6 years of experience building distributed systems in Go and Python. I've spent most of my career in high-growth startups where I owned services end-to-end — from design to production monitoring. I care a lot about correctness, observability, and making things simple enough that the next engineer doesn't curse your name."

**Paragraph 2: What you've done**
2–3 sentences. Highlight 2–3 concrete achievements.

> "At Snapp, I rebuilt the real-time dispatch engine handling 200K daily rides, reducing latency by 60% and cutting infrastructure cost by 30%. I also led a team of 5 engineers to migrate our monolith to a service-based architecture over 18 months with no major incidents."

**Paragraph 3: What you're looking for**
Direct and specific. Don't be vague.

> "I'm currently looking for remote backend or platform engineering roles, ideally at a company working on developer tools, fintech, or infrastructure. I'm timezone-flexible and have been working async-first for the last 3 years. If you think there's a fit, I'd love to chat."

### Getting Recruiter Attention

**Open to Work settings:**
- Go to your profile → "Open to Work" → "Recruiters only" or "Everyone"
- Fill in: job titles (be specific: "Backend Engineer", "Software Engineer", "Platform Engineer" — not just "Software Developer")
- Start date: "Immediately" or "In 1-2 months"
- Work type: Remote, On-site, Hybrid — check all that apply
- Locations: add cities you'd consider plus "Remote"

**Keywords that matter:**
LinkedIn's search uses your headline, current title, skills section, and About text. Use the exact terms recruiters search for:
- Language names: Go, Python, TypeScript (not "Golang" — recruiters search "Go")
- Cloud: AWS, GCP, Azure — spell them out and use abbreviations
- Infrastructure: Kubernetes, Docker, Terraform, CI/CD
- Database: PostgreSQL, Redis, MongoDB, Elasticsearch
- Architecture: microservices, distributed systems, event-driven

Add skills in the Skills section — they're searchable and endorsable.

**Engagement:**
Post 1–2 times a month about technical problems you've solved, projects you're working on, or things you've learned. Comment thoughtfully on posts by engineers at companies you want to work at. This is optional but it compounds — recruiters see active profiles.

### Connection Strategy

- Connect with engineers (not just recruiters) at companies you want to work at
- Connect with people who attended the same conferences or studied the same topics
- Personalize connection requests — 1 sentence is enough: "I saw your post on Go concurrency and found it useful. I'm a backend engineer interested in similar problems."
- 500+ connections is a credibility signal — get there by connecting with colleagues, classmates, and people you meet online
- Don't connect-and-immediately-pitch. Build first, ask later.

---

## 5. Cold Outreach Templates

These are short on purpose. Nobody reads long cold messages.

### LinkedIn Message to a Recruiter

```
Hi [Name],

I'm a backend engineer with 6 years of experience in Go and Python, 
currently looking for remote roles. I noticed [Company] is hiring for 
a Senior Backend Engineer — I'd love to learn more about the team and 
whether my background might be a fit.

Happy to share my resume or answer any questions. Thanks for your time.

— [Your name]
```

Keep it under 5 sentences. Mention the specific role. Don't attach your resume in the first message unless they ask.

### LinkedIn Message Asking for a Referral

```
Hi [Name],

I hope this isn't too forward — I'm a backend engineer with 5+ years 
in distributed systems (Go, Kubernetes) and I've been following 
[Company]'s engineering blog for a while. I'm applying for the 
[role name] position and noticed you work there.

I'd genuinely appreciate a 15-minute call to hear what it's like 
working there and whether you think I'd be a good fit. No pressure 
if you're not in a position to help — I just thought it was worth asking.

Thanks,
[Your name]
[LinkedIn URL or portfolio]
```

Note: this message doesn't ask directly for a referral — it asks for a conversation. The referral often comes naturally if the conversation goes well. Asking for a referral in the first message is off-putting.

### Follow-Up After No Response

Wait 5–7 business days. Send one follow-up. No more.

```
Hi [Name],

Just following up on my message from [date]. Totally understand if 
the timing isn't right or if it's not a fit — no worries at all.

If there's ever a better time, I'd still love to connect.

— [Your name]
```

One follow-up is professional. Two is pushy. Three is why people turn off connection requests.

---

## 6. Handling the Location / Visa / Remote Question

You will be asked. Have a clear, confident answer ready.

### In the Application

Most application forms ask for your location. Be honest. If the role is explicitly remote-friendly, say "Tehran, Iran — remote" or "Available remotely from [city]". Lying about location to get through ATS is a bad idea — it always comes out in the first call and breaks trust immediately.

If the form asks "Are you authorized to work in [country]?":
- If no, and the role requires it: some companies will sponsor, some won't. Apply anyway if the job is compelling — you can address it in the call.
- If it's a remote role with no work authorization required: you don't need to mention visas at all.

### In the Interview

Be direct and positive:

> "I'm based in Tehran. I've been working fully remote for the past 3 years and I'm comfortable with async communication, timezone overlap, and all the tooling that goes with it. For this role, I'd need to work as a contractor or you'd need to support an employer-of-record arrangement — I'm open to discussing how that would work on your end."

Or if you want to relocate:

> "I'm currently in Tehran and I'm actively looking to relocate to [city/country]. I'm in the process of [applying for visa / exploring my options] — I'd expect to be on the ground within [timeframe]. In the meantime, I can work remotely."

Two things make this go well: confidence and preparation. Know your own situation. Don't be vague about what you need.

### Common Scenarios

| Your situation | What to say |
|---|---|
| Fully remote, contractor OK | "I work remotely as a contractor. No visa issues — I invoice through [your setup]." |
| Want to relocate, need sponsorship | "I'm looking to relocate and would need visa sponsorship. I'm happy to discuss timeline and what that looks like on your end." |
| EU Blue Card / Global Talent visa in progress | "I'm currently applying for [visa]. Expected approval in [timeframe]. I can start remotely immediately." |
| Already have right to work | "I have [visa/status] — no sponsorship needed." |

---

## 7. Salary Negotiation for Remote / International Roles

### Know the market before you talk numbers

Research salaries for the role in the company's country, not yours. Use:
- levels.fyi (for big tech, very accurate)
- Glassdoor (noisy but useful for ranges)
- LinkedIn Salary
- Blind (brutal but honest)
- Asking people in the community directly (the most accurate)

Remote roles usually pay based on the company's location (US/EU company = US/EU salary) or on your location (less common but it happens). Ask early: "Is compensation location-dependent or is this a global rate?"

### Never give a number first

> "I'd love to understand the full compensation package before I share a number. Can you tell me the budgeted range for this role?"

If they push:

> "Based on my research for [role] at [company type/location], I'm targeting [range]. But I want to make sure we're aligned on the full picture — base, equity, benefits — before getting into specifics."

### The actual negotiation

Once you have an offer:

1. Never accept immediately. "Thank you — this is exciting. I'd like 48 hours to review it properly."
2. Anchor high. Ask for 15–20% more than the offer. Most people anchor too low.
3. If they can't move on base, negotiate equity, signing bonus, or extra PTO — these are often more flexible.
4. One counter is expected. Two counters is fine. Three is starting to feel adversarial.

**Script for the counter:**

> "I'm really excited about this opportunity — [company] is exactly the kind of place I want to work. Based on my experience and the market research I've done, I was hoping for something closer to [your number]. Is there flexibility on the base, or could we look at the signing bonus or equity?"

### Contractor vs Employee math

If you're paid as a contractor (common for international remote hires), remember:
- No employer-paid benefits (health insurance, pension)
- You handle your own taxes (typically higher effective rate)
- No paid leave unless you negotiate it

A contractor rate of $80K is not equivalent to an employee salary of $80K. A rough rule: contractor rate should be 20–30% higher than the equivalent employee salary to break even. Know this going in.

### A note on negotiating as an international candidate

Some people feel they have less leverage because they're "competing" against local candidates. That's not how good companies think. If they made you an offer, they want you. The candidate from the Bay Area who turns down the offer because it's too low costs them just as much. Negotiate.

The one thing that actually weakens your position: needing the job urgently. Build a pipeline of 3–5 active opportunities so you always have alternatives. Nothing helps your negotiation more than being able to genuinely say "I have another offer."
