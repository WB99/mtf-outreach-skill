# MTF school outreach skill

A SKILL.md for Claude Cowork that runs cold outbound to schools end to end: research each prospect, classify what it finds, write a personalised email off that classification, and move the CRM record along. Lark Base holds the data, Gmail sends. Both go through MCP connectors.

This is a GTM engineering proof of concept, not a product. Written for one campaign at one company, with IDs, names and target lists swapped for placeholders. It will not run out of the box. Read it for the method.

## What it is a proof of concept of

Personalised outbound at scale usually means mail merge with a first name in it (more like an EDM). The research is either skipped or painstakingly done by an SDR. This skill moves the research itself to the model.

Per school, it does five things a junior SDR would do:

1. Checks whether the school carries a Chinese-focus signal (SAP, CLEP, bicultural programme, clan affiliation).
2. Digs for real Mother Tongue Fortnight activity, in Chinese as well as English, on department subpages and social channels rather than the homepage.
3. Classifies the result as Strong, Usable or None, and records what it checked so the call can be audited.
4. Picks the matching email variant and fills the hook with a specific documented activity.
5. Writes the classification, the hook and the draft ID back to the CRM, and advances the stage.

The drafting is the easy part. Most of the file is guardrails: a gate that refuses to touch a colleague's account on the CRM, a research protocol that treats a fast null return as a failed search rather than an answer, and a human approval step the model cannot tick itself. An agent with write access to a shared CRM is one confident mistake away from erasing a coworker's pipeline.

## What is in the file

Sections 0 to 4 are safety and research. Sections 5 to 8 are copy and drafting mechanics. Sections 9 and 10 are the workflow and what every run has to report. The email template, the three hook variants and the voice rules are all included verbatim, so the personalisation logic is legible without access to the CRM.


## Using it

Take the structure, not the content. The parts worth stealing are the eligibility gate, the CRM access rules and guardrails, and the research protocol with a hard floor under it. The Chinese Focus Signal taxonomy and the hook copy are specific to this campaign and will not transfer, although the same propspecting and personalisation workflow can be applied to other industries and ICPs.
