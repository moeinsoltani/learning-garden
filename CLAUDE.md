# CLAUDE.md — Linux Networking Learning Project

## Student Profile
- Experience level: very limited (beginner)
- Platform: Windows 11, with access to a Linux VM and WSL2
- Use the Linux VM for labs (WSL2 as fallback for early lessons)

## Teaching Methodology
For each lesson:
1. Create a lesson file at `lessons/lesson-NN-topic.md` before teaching
2. The file contains: mental model, mechanics, lab commands with expected output, and checkpoint questions with hidden answers
3. Student reads the file and asks questions in chat if needed
4. Student moves to the next lesson at their own pace — no checkpoint gate
5. At the end of each Phase, give a written test covering all lessons in that phase

## Site Theme
- Theme: **Just the Docs** (dark color scheme) via `remote_theme: just-the-docs/just-the-docs@v0.10.0`
- Config: `_config.yml` — do not change the theme without updating this file
- Phase parent pages live in `lessons/phase-NN-name.md` with `has_children: true`
- Lesson files live in `lessons/lesson-NN-topic.md` with `parent: "Phase N: Name"`

## Lesson File Format
Each lesson file must have this exact front matter:
```yaml
---
title: "Lesson NN — Title"
nav_order: N
parent: "Phase N: Name"
---
```

Then immediately after front matter:
- **Home button**: `[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }` (Just the Docs button style, uses Jekyll relative_url so it works under /learn-networking/ baseurl)
- **Concept** — mental model, analogy, ASCII diagram
- **How it works** — mechanics (brief, focused on "why")
- **Lab** — exact commands with expected output (`$` = run this, `#` = comment)
- **Checkpoint** — each question has:
  1. `**Your answer:**` blank field for the student
  2. A hidden model answer: `<details><summary>Show Answer</summary><br>Answer here.</details>`
- **Homework** — one harder task with its own hidden model answer

Model answers must always be written — never leave a checkpoint or homework without one.

## Handling Student Questions
When the student asks a question about a term or concept from a lesson:
1. Answer briefly in chat
2. Then update the lesson file to incorporate the answer — either as a `{: .note }` callout block inline with the relevant section, or as expanded text
3. Add reference links (Wikipedia, man7.org man pages) for any technical term introduced — inline on first mention AND in the lesson's **Further Reading** table
4. Every lesson must have a **Further Reading** table before the Checkpoint section

### Callout syntax (Just the Docs)
```markdown
{: .note }
> **Title**
> Body text here.
```

### Reference link sources to prefer
- Linux man pages: `https://man7.org/linux/man-pages/manN/name.N.html`
- Wikipedia: `https://en.wikipedia.org/wiki/Topic`
- Only include links you are certain exist — do not guess URLs

## Curriculum
The full lesson sequence is defined in `learning-plan.md`. **Always consult it before creating a new lesson** — it specifies the topic, goals, and key commands for every lesson across all 15 phases. Never invent a lesson topic; follow the plan in order.

## Lesson Index
- Lesson 01: `lessons/lesson-01-namespaces-intro.md` — What a network namespace is ✓
- Lesson 02: `lessons/lesson-02-host-as-namespace.md` — The host is just another namespace ✓
- Lesson 03: `lessons/lesson-03-build-namespaces.md` — Build two isolated namespaces ✓
- Lesson 04: `lessons/lesson-04-ip-addressing.md` — IP addressing fundamentals (addrs) ✓
- Lesson 05: `lessons/lesson-05-arp-neigh.md` — Neighbor tables & ARP (neigh) ✓

*(Update this index as lessons are created)*
