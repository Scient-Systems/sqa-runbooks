# 05 — Creating & managing courses

This runbook has **two halves**, for two different readers.

| I want to… | Go to |
|---|---|
| Build and run a course myself, by clicking around | **Part 1** (§1–§7) — no coding, no terminal |
| Upload a course somebody handed me as a file | **Part 1, §7** |
| Generate a course from files, or script/automate it | **Part 2** (§8–§12) — developers |

If you're a course creator or manager, **Part 1 is your whole job**. You can stop at §7 and never
open Part 2.

---

# Part 1 — For course creators

## 1. The words you'll meet

Open edX has its own vocabulary. These are the ones you can't avoid:

| Word | What it means |
|---|---|
| **Studio** | The tool where you build courses. Point-and-click, like a website builder |
| **LMS** | The site your students see |
| **Section** | A big chunk of the course — e.g. "Week 1" |
| **Subsection** | One lesson inside a section |
| **Unit** | One page the student opens |
| **Component** | A piece of content on that page — text, video, a quiz question |
| **Run** | One intake of a course. The same course taught in spring and autumn = two runs |
| **Publish** | Making your work visible to students. Until you publish, only you can see it |

## 2. Where to log in

- **Live site:** `https://apps.lms.stemquestacademy.com/authoring/home`
- **Local test setup:** `http://studio.local.openedx.io:8001`

You need an account with Studio access — either an admin account, or you've been added to a
course's **Course Team** (§5).

## 3. Make a new course

1. Click **New Course**.
2. Fill in four boxes: **Course Name**, **Organization**, **Course Number**, **Course Run**.
3. Click **Create Course**.

🚨 **Get the last three right the first time.** Together they form the course's permanent ID, and
they **cannot be changed afterwards**. You can freely rename the course title later; you cannot
change the number or the run.

Example: Organization `SQA`, Number `SQA405`, Run `2026_T2` gives the ID
`course-v1:SQA+SQA405+2026_T2`. You'll see that string in links and in admin screens — it's just
those three values glued together.

**Naming tips:** use the run for the intake (`2026_T2`, `2026_Fall`), and keep the number stable
across runs of the same course. Avoid spaces and punctuation in all three.

## 4. Build the course

Everything nests, four levels deep:

```
Section          "Week 1: Meet the chatbot"
└── Subsection   "Lesson 1: What is a model?"
    └── Unit     "Video: how it works"      ← one page for the student
        └── Component   the video itself, plus a paragraph of text
```

**To add content:** open a Unit and use the **Add New Component** buttons at the bottom. The ones
you'll actually use:

| Component | Use it for |
|---|---|
| **Text** | Written lessons, instructions, images |
| **Video** | A video lesson |
| **Problem** | Quiz questions — multiple choice, dropdowns, text answers, and more |
| **Discussion** | A comment thread on that page |

**To reorder anything**, drag it by the handle on the left. Sections, subsections, units and
components can all be moved this way.

**To see it as a student would:** use **Preview** (shows unpublished work too) or **View Live**
(shows exactly what students currently get). Get in the habit of checking View Live — it's how you
catch things you forgot to publish.

## 5. Set dates, grading and who can edit

All under **Settings** in the top menu.

### Schedule & Details

Set the **Course Start Date**. 🚨 If the start date is in the future, students cannot see the
course at all — this is the single most common "why is my course missing?" cause.

Also here: end date, the course card image, and the short description shown in the catalog.

### Grading

This is where you decide how the final grade is worked out.

1. **Assignment types** — the buckets, e.g. "Homework", "Quizzes", "Final Project".
2. For each one, set the **weight** — how much of the final grade it's worth.
   🚨 **The weights must add up to exactly 100%.** If they don't, grades come out wrong.
3. Set the **pass mark** (the grade a student needs to pass, e.g. 60%).
4. Optionally **drop the lowest N** scores in a bucket, and set a **grace period** for late work.

Then tell each graded subsection which bucket it belongs to: open the subsection, and set its
assignment type to one of the types you created above.

🚨 Two things that silently produce wrong grades:
- A subsection's assignment type doesn't exactly match a type you defined.
- You defined more assignments of a type than actually exist in the course — the missing ones are
  counted as zeros.

### Course Team

Add other people who may edit this course. **Staff** can edit content; **Admin** can also add and
remove team members.

## 6. Publish — and what it actually means

Your edits are invisible to students until you publish them. Studio shows each unit's state:
draft, published, or published-with-unpublished-changes.

- Click the green **Publish** button on a unit to push it live.
- **Editing an already-published unit un-publishes those changes** — you have to publish again.
  This catches everyone out.

To hide something deliberately, don't leave it unpublished — use the visibility settings on the
section or subsection instead, so you don't confuse "not finished" with "not meant to be seen".

## 7. Before you launch — checklist

Run through this before you tell students the course is open:

- [ ] **Course start date** is today or earlier (§5).
- [ ] **Every unit is published** — check View Live, not Preview (§6).
- [ ] **Grading weights add up to 100%**, and every graded subsection has an assignment type (§5).
- [ ] **Course Team** has everyone who needs access (§5).
- [ ] You've **clicked through the course as a student** would, at least once.
- [ ] If the course needs a paid membership, the level is set — ask a developer, or see §12.

## 8. Uploading a course somebody sent you

Courses are shared as a single `.tar.gz` file. You may be handed one to load into the platform.

**In Studio:** open the course → **Tools → Import** → choose the file → start the import.

🚨 **Import wipes out whatever is already in that course** and replaces it. If the existing content
matters, create a fresh course run and import into that instead.

**If you were given a folder rather than a `.tar.gz`**, you need one command to package it. Open
PowerShell (Windows) or a terminal (Mac/Linux) in the folder that *contains* the course folder:

```bash
tar -czf course.tar.gz folder_name
```

Replace `folder_name` with the actual course folder's name. That produces `course.tar.gz`, ready to
upload.

**To check it's packaged right** before uploading — the listing should show `folder_name/course.xml`
near the top, not something buried several folders deep:

```bash
tar tzf course.tar.gz | head
```

**Going the other way** — to get a copy of a course out of the platform: **Tools → Export**
downloads the whole course as a `.tar.gz`. Good for backups and for moving a course between sites.

---

# Part 2 — For developers

Everything below is the file format, the command line, and how courses wire into our plugin.
Course creators don't need any of it.

Worked example throughout: the "Silly Bot" course (`course-v1:SQA+SQA405+2026_T2`), built as OLX in
`~/silly_bot_course_project`.

## 9. OLX — a course as a folder of files

OLX is what a course looks like on disk: a folder of XML files. Export writes it, import reads it.
Use it to generate or bulk-edit a course instead of clicking through Studio.

| Folder / file | What it holds |
|---|---|
| `course.xml` | The entry point — points at everything else |
| `chapter/` | Sections |
| `sequential/` | Subsections |
| `vertical/` | Units (the pages) |
| `html/` | Text lessons |
| `video/` | Videos |
| `problem/` | Quizzes and graded questions |
| `lti/` | External learning tools plugged into the course |
| `static/` | Images, PDFs and other assets |
| `policies/` | Course settings and the grading rules |

Note the naming mismatch with the Studio UI: a Section is a `chapter`, a Subsection is a
`sequential`, a Unit is a `vertical`.

**How the files point at each other:**

```
course/                      ← the importable root
├── course.xml               ← top pointer: <course org="SQA" course="SQA405" url_name="2026_T2"/>
├── course/2026_T2.xml       ← course run: references chapters in order, holds display_name/start
├── chapter/<id>.xml         ← a Section → references sequentials
├── sequential/<id>.xml      ← a Subsection → references verticals; graded ones carry
│                              graded="true" format="<Assignment Type>"
├── vertical/<id>.xml        ← a Unit → references leaf components
├── html/<id>.xml + .html    ← an HTML component (pointer + body)
├── problem/<id>.xml         ← a Problem (MCQ, customresponse, etc.)
├── policies/<run>/
│   ├── policy.json          ← tabs, discussion settings, display_name
│   └── grading_policy.json  ← assignment types, weights, GRADE_CUTOFFS
└── about/overview.html      ← marketing description
```

**Content file examples** (validated against our Open edX version — see
`silly_bot_course_project/SILLY_BOT_COURSE_BUILD_PLAN.md` §7 for the full set, and the demo course
for worked examples of every question type: single-select, multi-select, dropdowns, formula and
math-expression input, multi-part problems, Python-evaluated input, polls, surveys, Staff Graded
Assignments and LTI):

```xml
<!-- Multiple choice -->
<problem display_name="Quiz" max_attempts="" rerandomize="never" showanswer="finished" weight="1.0">
  <multiplechoiceresponse>
    <label>What is the model?</label>
    <choicegroup type="MultipleChoice">
      <choice correct="true">The AI brain that writes a reply</choice>
      <choice correct="false">A password</choice>
    </choicegroup>
  </multiplechoiceresponse>
</problem>
```
```xml
<!-- Python grader that runs IN the sandbox (no network — see §11) -->
<problem display_name="Your First Reply" showanswer="finished" weight="1.0">
  <script type="loncapa/python"><![CDATA[
def check_reply(expect, ans):
    ns = {}; exec(compile(ans, '<student>', 'exec'), ns)
    return {'ok': True, 'msg': 'Nice!'}
  ]]></script>
  <customresponse cfn="check_reply" expect="ok">
    <label>Write a function reply() that returns something.</label>
    <textbox rows="5" cols="65" mode="python" tabsize="4"/>
  </customresponse>
</problem>
```

**Validate before importing.** Two checks:

1. **Every file is well-formed XML:**

    ```bash
    xmllint --noout course/**/*.xml
    ```

2. **Referential integrity** — every `url_name`/`filename` a parent references must resolve to a
   real file, and there must be no orphan files. (The Silly Bot build enforced this as ACs
   G-a / G-b.)

## 10. Import & export from the command line

```bash
# dev:
tutor dev exec cms ./manage.py cms import /openedx/data /openedx/data/<course-dir>
# k8s:
tutor k8s exec cms -- ./manage.py cms import ../data <course-dir>
```

`manage.py cms import <data_dir> <course_dir>` — first arg is the data root, second the extracted
course folder. 🚨 The CLI import tends to create a **new run** rather than overwrite; for a clean
overwrite of an existing run, the Studio UI import (§8) is more predictable.

## 11. 🚨 The sandbox has no internet (shapes course design)

Open edX's Python grader (`loncapa/python` / `customresponse`) runs in **CodeJail**, a sandbox with
**no network**. A grader cannot call an external API (Gemini, etc.). Design consequences:

- **Inside edX, graded:** MCQs, prompt-writing submissions, pure-Python warm-ups (no network), and
  Open Response Assessments (ORA, peer/self graded).
- **Outside edX, not autograded:** anything that needs the internet (the real Gemini calls, the
  Streamlit app). Verified via an ORA submission (URL + screenshot) and class showcase.

CodeJail must be enabled in the deployment for `customresponse` graders to run (it is on this
project). If a Python problem errors about the sandbox, CodeJail isn't enabled.

## 12. Grading policy as a file

The UI in §5 writes this; in OLX you author it directly.
`policies/<run>/grading_policy.json` defines assignment types and the pass cutoff:

```json
{
  "GRADER": [
    { "type": "Concept Checks", "short_label": "CC", "min_count": 3, "drop_count": 0, "weight": 0.20 },
    { "type": "Final Project",  "short_label": "Final", "min_count": 1, "drop_count": 0, "weight": 0.40 }
  ],
  "GRADE_CUTOFFS": { "Pass": 0.60 }
}
```

🚨 Rules: weights must sum to **exactly 1.0**; each graded `<sequential>`'s `format="…"` must match a
GRADER `type` **exactly**; and `min_count` must equal the actual number of graded subsections of that
type, or edX pads the grade with phantom zeros.

## 13. How courses connect to our membership plugin

A course can require a membership level two ways:

1. **Django admin** → create a `CourseMembershipRequirement` row directly (course key → required level).
2. **Studio Advanced Settings** → set `membership_required_level` to a level slug (the field is exposed
   by `ENABLE_OTHER_COURSE_SETTINGS`, runbook 04 §6). On every **publish**, the CMS signal handler
   `on_course_catalog_info_changed` reads that value and upserts the `CourseMembershipRequirement` for
   you — so authors never touch Django admin. (This sync only runs when `gate_enrollment` is ON.)

When `gate_enrollment` is ON, an enrollment attempt by a user below the required level is **blocked
before the enrollment row is written** (an openedx-filters step, all sources — runbook 06 §4). It does
not retroactively unenroll already-enrolled students. Don't enable in prod until verified (runbook 04
§4). For AI courses, a `CourseIntegrationGrant` ties a course to an AI provider via the broker
(runbook 13).

🚨 Course-key fields in Django admin are strict — a trailing comma
(`course-v1:SQA+SQA405+2026_T2,`) raises `InvalidKeyError` → 500. Enter the clean key only.

---

## What's next

- The plugin that gates these courses → **06-installing-a-django-plugin-app.md**
- Wiring an AI course to the broker → **13-deploying-companion-services.md**
