# SQL Injection Attack Lab - System Security TP (CyberRange CZ)

CyberRange CZ scenario built from the **SEED Labs "SQL Injection Attack
Lab"** (copyright (c) 2006-2020 Wenliang Du, CC BY-NC-SA 4.0). See
`ATTRIBUTION.txt`.

It follows the same file layout as the `library-junior-hacker` template,
adapted as described below.

## What this scenario does

- One **seed** host (image: your "Ubuntu 20.04 VM", flavor `m1.medium`,
  `mgmt_user: ubuntu` - **check/adjust this**, see Assumptions below),
  connected through a router to the internet.
- Provisioning installs Docker, deploys the **official, unmodified SEED
  `Labsetup`** (bundled in `provisioning/roles/seed_lab/files/Labsetup/`,
  fetched from the public `seed-labs/seed-labs` GitHub repository), and
  runs `docker-compose build && up -d` to start the vulnerable web
  application (`www-10.9.0.5`) and its MySQL database (`mysql-10.9.0.6`).
- Provisioning then plants a **different secret per sandbox** into three
  normally-empty database columns, and writes a small file,
  `~/task3_targets.txt`, with three more per-sandbox target values. This
  is what makes every student's flags different (see "Anti-cheat" below).
- No `provisioning/requirements.yml` is used. The four Ansible roles the
  original template downloaded from GitHub (`user-access`, `chrony`,
  `sandbox-logging`, `hosts-aliases`) are **not needed here**: only
  `user-access` is used, and it is bundled locally under
  `provisioning/roles/user-access/`. `chrony` and `sandbox-logging` were
  dropped, as agreed.

## Levels and scoring (20 points total)

| # | Level | Type | Points | Flag |
|---|---|---|---|---|
| 0 | Introduction | Info | - | - |
| 1 | Get Access | Access | - | passkey `start` |
| 2 | Task 1 - SQL basics | Training | 1.5 | unique per student |
| 3 | Task 2.1 - SELECT injection (webpage) | Training | 2.5 | unique per student |
| 4 | Task 2.2 - SELECT injection (curl) | Training | 2.5 | unique per student |
| 5 | Task 2.3 - stacked queries / countermeasure | Training | 2.0 | same for all (conceptual answer) |
| 6 | Task 3.1 - UPDATE injection, own salary | Training | 3.0 | unique per student |
| 7 | Task 3.2 - UPDATE injection, Boby's salary | Training | 2.5 | unique per student |
| 8 | Task 3.3 - UPDATE injection, Boby's password | Training | 3.0 | unique per student |
| 9 | Task 4 - Prepared statement fix | Training | 3.0 | same for all (fixed error message) |
| | **Total** | | **20.0** | |

Hint penalties are between **0.5 and 2 points**, set per hint in
`training.json` (`hint_penalty` field). Revealing a level's full
`solution` is also penalized (`solution_penalized: true`) on every
training level except Task 1.

**Check with your platform before import**: this definition uses decimal
`max_score` and `hint_penalty` values (for example `1.5`, `0.5`). If your
CyberRange CZ version rejects decimals in the JSON schema, the simplest
fix is to multiply every `max_score` and `hint_penalty` by 5 (giving a
total of 100 instead of 20) and divide the final score by 5 yourself when
grading.

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
│           ├── tasks/main.yml
│           └── files/Labsetup/ # official, unmodified SEED lab files
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
   through all levels once (fastest way to confirm the image name,
   `mgmt_user`, Docker build time, and that decimal scores import
   correctly - see "Assumptions to double check" below).

## Assumptions to double check

- **Image name / mgmt_user**: `topology.yml` currently uses
  `image: "Ubuntu 20.04 VM"` with `mgmt_user: ubuntu`, assuming a plain
  cloud image with the default Ubuntu cloud-init account. If your image
  catalog uses a different exact name or a different default account,
  update both fields.
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
