# SQL Injection Attack Lab - System Security TP (CyberRange CZ)

CyberRange CZ scenario built from the **SEED Labs "SQL Injection Attack
Lab"** (copyright (c) 2006-2020 Wenliang Du, CC BY-NC-SA 4.0). See
`ATTRIBUTION.txt`.

It follows the same file layout as the `library-junior-hacker` template,
adapted as described below.

## What this scenario does

- One **seed** host (image `seed`, flavor `m1.medium`, `mgmt_user: ubuntu`
  - confirmed against two working scenarios on the same platform, the
  `csrf-seedlab` and `library-seed-prng` topologies), connected through a
  router to the internet.
- Students log in as **`seed` / `dees`**, the standard SEED VM account,
  with its admin rights (sudo and similar) explicitly stripped by
  provisioning - `docker` group membership is kept, since Tasks 1 and 4
  require `docker exec` (see "Anti-cheat" below for why this matters).
- Provisioning installs Docker, deploys the **SEED `Labsetup`** (bundled
  in `provisioning/roles/seed_lab/files/Labsetup/`, originally fetched
  from the public `seed-labs/seed-labs` GitHub repository; one file,
  `defense/getinfo.php`, has one small documented addition - see
  `ATTRIBUTION.txt`), and runs `docker-compose build && up -d` to start
  the vulnerable web application (`www-10.9.0.5`) and its MySQL database
  (`mysql-10.9.0.6`). The build/start steps retry automatically on
  transient network failures (observed once: a mid-download TCP reset
  pulling the `mysql:8.0.22` base image).
- Provisioning then plants a **different secret per sandbox** into three
  normally-empty database columns, and writes a small file,
  `~/task3_targets.txt`, with three more per-sandbox target values. This
  is what makes every student's flags different (see "Anti-cheat" below).
- No `provisioning/requirements.yml` is used. The only external role
  used, `user-access`, is bundled locally under
  `provisioning/roles/user-access/`, so no GitHub access is required at
  provisioning time (even though it has been confirmed available on this
  platform, via the `library-seed-prng` scenario's use of
  `requirements.yml`).

## Levels and scoring (200 points = 20/20)

CyberRange CZ's backend only accepts **integer** scores (confirmed by the
`library-seed-prng` scenario on this same platform). This definition
therefore scores everything **x10**: 200 points total instead of 20, so
that a 0.5-point hint penalty becomes an integer (5). Divide the final
score by 10 to get the grade out of 20.

| # | Level | Type | Points (x10) | Flag |
|---|---|---|---|---|
| 0 | Introduction | Info | - | - |
| 1 | Get Access | Access | - | passkey `start` |
| 2 | Task 1 - SQL basics | Training | 15 | unique per student |
| 3 | Task 2.1 - SELECT injection (webpage) | Training | 25 | unique per student |
| 4 | Task 2.2 - SELECT injection (curl) | Training | 25 | unique per student |
| 5 | Task 2.3 - stacked queries / countermeasure | Training | 20 | same for all (conceptual answer) |
| 6 | Task 3.1 - UPDATE injection, own salary | Training | 30 | unique per student |
| 7 | Task 3.2 - UPDATE injection, Boby's salary | Training | 25 | unique per student |
| 8 | Task 3.3 - UPDATE injection, Boby's password | Training | 30 | unique per student |
| 9 | Task 4 - Prepared statement fix | Training | 30 | same for all (fixed message) |
| | **Total** | | **200** (= 20/20) | |

Hint penalties are between **5 and 15 points** (0.5-1.5 on the /20
scale), set per hint in `training.json` (`hint_penalty` field).
Revealing a level's full `solution` is also penalized
(`solution_penalized: true`) on every training level except Task 1.

## Anti-cheat: per-student flags (APG)

This definition uses CyberRange CZ's **APG (Automatic Problem
Generation)** mechanism:

- `variables.yml` declares 6 variables (`secret_task1`, `secret_task2a`,
  `secret_task2b`, `target_salary_self`, `target_salary_boby`,
  `target_password_boby`). Each is generated once per sandbox.
- The matching training levels have `"variant_answers": true` and
  `"answer_variable_name"` set to the variable name, instead of a static
  `"answer"`.
- `training.json` has `"variant_sandboxes": true` at the top level, which
  is required for APG trainings.
- Provisioning (`provisioning/roles/seed_lab/tasks/main.yml`) reads these
  generated values as Ansible variables of the same name and uses them to
  (a) edit three otherwise-unused database columns (Alice's `NickName`,
  Admin's `NickName` and `Email`) and (b) write the three UPDATE-task
  targets to `~/task3_targets.txt` on the seed host.

Levels 5 (stacked queries) and 9 (prepared statement) use a **static**
answer instead, because their flag is a conceptual/fixed outcome
(the name of the countermeasure, and the application's fixed error
message) rather than data that can be planted per student. If you need
those to vary too, tell me and I can add two more APG variables, for
instance by appending a random marker to the error-page text.

### Closing the seed/dees admin backdoor

The `seed` image ships the standard SEED VM account, `seed` / `dees`,
which is public, documented knowledge (anyone who has used a SEED VM
before knows it) and is typically an administrator on the base image.
Left alone, **any student could log in with those credentials and get
root**, bypassing the whole exercise - reading the database directly,
editing any file, etc. Provisioning now explicitly strips `sudo` and
other admin-group memberships (`admin`, `wheel`, `adm`, `disk`, `lxd`,
`libvirt`, `systemd-journal`) from that account, removes any matching
`sudoers` rules, and asserts afterwards that `sudo -l -U seed` reports no
rights - the provisioning run fails loudly if that assertion doesn't
hold, rather than silently leaving a hole open.

### A limitation this cannot fully close

Tasks 1 and 4 of the official lab explicitly require `docker exec`
access (into the mysql and www containers respectively), so the `seed`
account keeps `docker` group membership. Being in the `docker` group is
root-equivalent for the **entire host**, including both containers - so
a student could, after Task 1, simply run
`docker exec mysql-10.9.0.6 mysql -uroot -pdees sqllab_users -e "SELECT * FROM credential;"`
and read every planted secret directly, skipping the intended web-based
injection for Tasks 2 and 3 entirely on **their own** sandbox. Per-student
unique values stop students from sharing answers with each other; they
do not stop an individual student from shortcutting their own exercise
this way, because the access the official lab itself grants in Task 1
is the shortcut. There is no way to close this without breaking Task 1
or Task 4 as specified in the PDF.

Two mitigations, consistent with how the original lab was actually meant
to be graded:
- The PDF's own "Submission" section (section 5) requires a written
  report with screenshots. That has always been the real check on
  *method* - keep requiring it; treat the platform's automatic flags as
  a fast completion/partial-credit layer on top, not a replacement.
- Command logging can be added (the `sandbox-logging` role, confirmed as
  legitimate first-party CyberRange CZ infrastructure by its use in both
  the original `library-junior-hacker` template and the
  `library-seed-prng` scenario) so you can review each student's actual
  command history and spot anyone who bypassed the web app. This isn't
  included here yet - ask if you want it added.

## Repository structure

```
sqli-seed-lab/
├── topology.yml              # Sandbox Definition: hosts/router/networks
├── variables.yml              # Sandbox Definition: APG variables
├── provisioning/               # Sandbox Definition: provisioning
│   ├── playbook.yml
│   ├── host_vars/seed.yml
│   └── roles/
│       ├── user-access/        # local copy, no GitHub download needed
│       └── seed_lab/
│           ├── tasks/main.yml  # deploy, plant flags, strip admin rights
│           └── files/Labsetup/ # official SEED lab files (1 file patched)
├── training/
│   └── training.json          # Training Definition, imported separately
├── README.md
├── LICENSE                    # MIT, code only
├── ATTRIBUTION.txt            # CC BY-NC-SA notice for the SEED content
└── .gitignore
```

This mirrors how CyberRange CZ expects things: the **Sandbox Definition**
is everything CyberRange CZ clones straight from Git -
`topology.yml`, `variables.yml` and `provisioning/` at the repository
root (this is the "simplest directory structure" the platform docs
describe). The **Training Definition** (`training/training.json`) is a
separate object in the platform, imported as a JSON file through the
Training Agenda UI or API - it is not read from Git, so it lives in its
own subfolder here purely so it's versioned alongside everything else.

## Deploying

1. **Push this repository to GitHub** (public, or private with a deploy
   key/token your CyberRange CZ instance can use - ask your platform
   admin which it needs):

   ```bash
   cd sqli-seed-lab
   git init
   git add .
   git commit -m "SQL Injection Attack Lab - initial scenario"
   git branch -M main
   git remote add origin https://github.com/<your-org>/sqli-seed-lab.git
   git push -u origin main
   ```

2. **Register the Sandbox Definition** in CyberRange CZ: Sandbox Agenda ->
   Sandbox Definitions -> create new -> give it the repository URL above
   and the revision to use (branch `main`, or pin a tag/commit once you
   are happy with it).

3. **Create a sandbox pool** from that definition, sized for your ~30
   students (one sandbox per student, since each needs its own unique
   variable values from `variables.yml`).

4. **Import the Training Definition**: Training Agenda -> Training
   Definitions -> create/import -> upload `training/training.json`.
   Make sure **"Generate variant sandboxes"** is enabled (this is what
   `variant_sandboxes: true` reflects in the JSON).

5. **Create a Training Instance**, attach it to the sandbox pool from
   step 3, and open enrollment for your students.

6. Before the real session, **allocate one sandbox yourself** and walk
   through all levels once (fastest way to confirm Docker build time,
   that `seed`/`dees` really has no sudo left after provisioning, and
   that Task 4's flag text matches - see "Assumptions to double check"
   below).

## Assumptions to double check

- **Image name / mgmt_user**: `topology.yml` uses `image: seed` with
  `mgmt_user: ubuntu`, matching two working scenarios on the same
  platform (`csrf-seedlab` and `library-seed-prng`). If your catalog's
  SEED image is named differently, update both fields.
- **seed/dees not already locked**: if your image's `seed` account is
  disabled, has a different password, or doesn't exist at all, the
  `user-access` role will create/reset it as described - but double
  check the hardening block in `seed_lab/tasks/main.yml` still finds and
  strips the right groups on your actual image.
- **Internet access during provisioning**: confirmed available. Docker,
  docker-compose and their apt dependencies, plus the Docker Hub base
  images the lab's own Dockerfiles pull (`mysql:8.0.22` and
  `handsonsecurity/seed-server:apache-php`), are downloaded at
  provisioning time. No GitHub access is required (see above).
- **Single host, no router/attacker/client split**: as requested, the
  whole lab (web + database) runs as Docker containers on one `seed`
  host; there is no separate attacker machine.

## License

- The scenario code (topology, provisioning) follows the template's MIT
  license.
- The lab content is based on SEED Labs material (CC BY-NC-SA 4.0) - see
  `ATTRIBUTION.txt`. Keep this notice if you redistribute the scenario.
