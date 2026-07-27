# 05 — Creating & managing courses

Two ways to build a course in Open edX: **Studio** (the visual authoring tool, best for most work)
and **OLX** (the XML file format, best for bulk/scripted/AI-generated content). This runbook covers
both, plus import/export and the platform features courses depend on.

Worked example throughout: the "Silly Bot" course (`course-v1:SQA+SQA405+2026_T2`), built as OLX in
`~/silly_bot_course_project`.

---

## 1. Course identity (the key)

Every course has an opaque key: `course-v1:<ORG>+<NUMBER>+<RUN>`, e.g.
`course-v1:SQA+SQA405+2026_T2` (Org `SQA`, number `SQA405`, run `2026_T2`). You choose these when
you create the course; they're permanent.

## 2. Create a course in Studio (the visual way)

Studio is the course-building tool. It's all point-and-click — no files, no XML.

**Where to log in** (with an account that has Studio access — a superuser, or someone added to the
course's **Course Team**):

- Production: `https://apps.lms.stemquestacademy.com/authoring/home`
- Local dev: `http://studio.local.openedx.io:8001`

**Step 1 — make the course.** Click **New Course**, fill in the four fields (Course Name,
Organization, Course Number, Course Run), click **Create Course**. 🚨 The last three become the
permanent course key (§1). You can rename the course later; you cannot change these.

**Step 2 — build the structure.** Four levels, each one living inside the one above it:

| Level | What it is |
|---|---|
| **Section** | A big chunk — e.g. "Week 1" |
| **Subsection** | One lesson inside that week |
| **Unit** | One page the student actually opens |
| **Component** | The content on that page — text, video, quiz question, discussion |

**Step 3 — set the course up.** Under **Settings**:

- **Schedule & Details** → the **Start date**. 🚨 A course starting in the future is invisible to
  students.
- **Grading** → assignment types and the pass mark (§6).
- **Course Team** → who else is allowed to edit.

**Step 4 — publish.** Hit the green **Publish** button on each unit. 🚨 Unpublished work doesn't
exist as far as students are concerned, even though you can see it fine.

## 3. OLX — a course as a folder of files

OLX is what a course looks like on disk: a folder full of XML files. Export writes it out, import
reads it back in. Use it when you want to generate or bulk-edit a course instead of clicking
through Studio.

**What's in the folder**

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

Note the naming: the folders use the *internal* names for the three structure levels — a Section is
a `chapter`, a Subsection is a `sequential`, a Unit is a `vertical`.

**The same thing, showing how the files point at each other:**

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

**Examples of the actual content files** (validated against our Open edX version — see
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
<!-- Python grader that runs IN the sandbox (no network — see §5) -->
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

**Check the files before importing.** Two things:

1. **Every file is valid XML:**

    ```bash
    xmllint --noout course/**/*.xml
    ```

2. **Every link resolves** — if a parent file points at a `url_name`, that file has to actually
   exist, and there should be no leftover files that nothing points at. (The Silly Bot build
   enforced this as ACs G-a / G-b.)

## 4. Import & export

### Through Studio (the normal way)

**To download a course:** open it → **Tools → Export** → you get a `.tar.gz` file of the OLX.

**To upload a course:** open the course → **Tools → Import** → choose your `.tar.gz` → start the
import. 🚨 Import **replaces everything** in that course. If the existing content matters, import
into a throwaway course run first.

**Making the `.tar.gz`.** Put the course folder somewhere, open a terminal there, and zip it up:

```powershell
# Windows (PowerShell)
tar -czf course.tar.gz folder_name        # folder_name = the course folder
```
```bash
# Linux / Mac / WSL
tar -czf course.tar.gz folder_name
```

🚨 `course.xml` has to sit at the top of the archive. Check before uploading — the first lines
should show `folder_name/course.xml`, not something buried three folders deep:

```bash
tar tzf course.tar.gz | head
```

### From the command line (for automation)
```bash
# dev:
tutor dev exec cms ./manage.py cms import /openedx/data /openedx/data/<course-dir>
# k8s:
tutor k8s exec cms -- ./manage.py cms import ../data <course-dir>
```
`manage.py cms import <data_dir> <course_dir>` — first arg is the data root, second the extracted
course folder. 🚨 The CLI import tends to create a **new run** rather than overwrite; for a clean
overwrite of an existing run, the Studio UI import is more predictable.

## 5. 🚨 The sandbox has no internet (shapes course design)

Open edX's Python grader (`loncapa/python` / `customresponse`) runs in **CodeJail**, a sandbox with
**no network**. A grader cannot call an external API (Gemini, etc.). Design consequences:

- **Inside edX, graded:** MCQs, prompt-writing submissions, pure-Python warm-ups (no network), and
  Open Response Assessments (ORA, peer/self graded).
- **Outside edX, not autograded:** anything that needs the internet (the real Gemini calls, the
  Streamlit app). Verified via an ORA submission (URL + screenshot) and class showcase.

CodeJail must be enabled in the deployment for `customresponse` graders to run (it is on this
project). If a Python problem errors about the sandbox, CodeJail isn't enabled.

## 6. Grading policy

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

## 7. How courses connect to our membership plugin

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
