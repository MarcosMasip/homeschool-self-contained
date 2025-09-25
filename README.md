# homeschool

An app for homeschool planning

## Self‑contained local run (Docker) — step by step

This app runs fully locally with Docker Desktop (macOS/Windows/Linux). No real emails or payments occur. Follow these exact steps.

Prerequisite: Install Docker Desktop and ensure `docker compose` works.

1) Clone and enter the repo

```bash
git clone https://github.com/MarcosMasip/homeschool-self-contained.git
cd homeschool-self-contained
```

Expected outcome:
- The repository is on your machine and your shell is in the project directory.

2) Create your local env file

```bash
cp .env.example .env
```

Windows (PowerShell) alternative:

```powershell
Copy-Item .env.example .env
```

Expected outcome:
- A new `.env` file exists with safe local defaults (console email, no live services, Stripe in test mode with dummy keys).

3) Build and start the web service

```bash
COMPOSE_PARALLEL_LIMIT=1 docker compose up --build web
```

Windows (PowerShell) alternative:

```powershell
$env:COMPOSE_PARALLEL_LIMIT=1; docker compose up --build web
```

Expected outcome:
- Image builds (includes frontend CSS, docs, and blog) without parallel-export issues.
- Migrations run automatically on first start.
- Logs end with Gunicorn listening: `Listening at: http://0.0.0.0:8000`.
- App is reachable at http://localhost:8000.

Tip: Run in background if you prefer

```bash
COMPOSE_PARALLEL_LIMIT=1 docker compose up -d web
```

4) Start the background worker (new terminal)

```bash
docker compose up worker
```

Expected outcome:
- Huey worker starts and waits for jobs. Keep this terminal open; it will print emails (magic links) to the console.

5) Enable the sign‑in form (feature flag) and set the Site domain (one‑time)

```bash
docker compose exec web ./manage.py shell -c "from homeschool.core.models import Flag; Flag.objects.update_or_create(name='signup_flag', defaults={'everyone': True})"
docker compose exec web ./manage.py shell -c "from django.contrib.sites.models import Site; Site.objects.update_or_create(id=1, defaults={'domain':'localhost:8000','name':'localhost'})"
```

Expected outcome:
- First command creates/enables the `signup_flag` so the email form appears.
- Second command sets the Sites framework to `localhost:8000` so magic links open your local app.

6) Sign in via magic link (fully local)

```text
Open http://localhost:8000/signin in your browser
Enter your email and submit
```

Expected outcome:
- The page says “Please Check Your Email!”.
- The worker terminal prints the email contents with a URL like:
  `http://localhost:8000/login?token=…`
- Click that URL to be logged in.

Alternate (no worker): Generate your magic link directly

```bash
docker compose exec web ./manage.py shell -c "from homeschool.accounts.models import User; from sesame.utils import get_query_string; from homeschool.core.site import full_url_reverse; u=User.objects.get(email='YOUR_EMAIL_HERE'); print(full_url_reverse('sesame-login') + get_query_string(u))"
```

Expected outcome:
- The command prints a full login URL. Open it to sign in.

7) Create an admin user (optional, for /admin)

```bash
docker compose exec web ./manage.py createsuperuser
```

Expected outcome:
- You’re prompted for email/password. “Superuser created successfully.” then you can log in at http://localhost:8000/admin.

8) Stop services when you’re done

```bash
docker compose down
```

Expected outcome:
- Containers and network are removed; your local SQLite DB file remains in the project folder.

What stays local and safe:
- Files: local filesystem (no S3). 
- Email: printed to console (no real email sent).
- Payments: Stripe live mode is OFF with dummy test keys; nothing real is charged. The subscriptions page loads Stripe.js only if you navigate to it—feel free to avoid that page for 100% offline usage.

Troubleshooting quick refs:
- “Environment variable ... not set”: ensure `.env` exists (copy from `.env.example`) and re-run `docker compose up`.
- “signup form not visible”: run step 5 to enable `signup_flag`.
- Magic link points to example.com: run step 5 to set Site domain to `localhost:8000`.
- Docker build/export error on macOS like “parent snapshot ... does not exist”: build/run sequentially with `COMPOSE_PARALLEL_LIMIT=1` as shown in step 3. If it still appears, run `docker builder prune -af` then try again, or temporarily disable BuildKit with `DOCKER_BUILDKIT=0 docker compose up --build web`.

## Native development (optional)

If you prefer not to use Docker:

### Python

`uv` is required.

### JavaScript

Node.js is required (23.x preferred; Tailwind works on most recent LTS as well).

Install JS packages for Tailwind CSS.

```bash
npm --prefix frontend i
```

Install Python deps and run

```bash
cp .env.example .env
uv sync
uv run manage.py migrate
uv run honcho start -f Procfile
```

Create a superuser (optional for /admin):

```bash
uv run manage.py createsuperuser
```

## Docker Compose

Analyzing image contents:

```
alias dive="docker run -ti --rm  -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive"
dive klakegg/hugo:0.101.0
```

### uv

If for some reason I need to work with uv inside the container,
there are some settings needed to get around the fact that image doesn't install
packages in a place writeable by the app user.

Here's an example:

```
UV_PROJECT_ENVIRONMENT=/tmp/uv-venv uv add -n --dev 'types-toml==0.10.8.20240310'
```

## Server config

1. Add ssh keys.
2. Turn off passworth auth

```
/etc/ssh/sshd_config
PasswordAuthentication no
systemctl restart ssh
```

3. Firewall stuff.

```
ufw allow OpenSSH
ufw default deny incoming
ufw enable
```

## Market Research

This is my analysis of the market
to assess the features and positioning
of what is available.

Research plan:
Sign up for a service trial
to test out the account.
Share findings on forums
to solicit feedback.

Measurement criteria:

* Price
* Features
* Quality

[Well Trained Mind reference analysis](https://docs.google.com/spreadsheets/d/1SroV3KdUshFIQMOEKfVDtv5zwmcylL6OHfewTuaeJRo/edit?usp=sharing)

### Product Matrix

| Product | Business Model | Price | Reviewed |
| --- | --- | --- | --- |
| CM Organizer | Freemium | $7.95 / month | 2/1/21 |
| Google Classroom | Free | $0 | 3/12/21 |
| Homeschool Manager | Subscription | $5.99 / month | 3/13/21 |
| Homeschool Minder | Subscription | $4.99 / month | 5/3/21 |
| Homeschool Planet | Subscription | $7.95 / month | 5/20/21 |

### Homeschool Planet

https://homeschoolplanet.com/

"Synchronize your home, school, and work into a single place"

* Landing page
  * Email list capture
* Pricing
  * 30 day trial
  * $69.95 / year
  * $7.95 / month
* Social Media
  * Heavy activity on Facebook (at least daily)
  * Pinterest
  * YouTube to host help docs
  * Instagram
  * Blog (infrequent posts - handful per year, guest posts)
* Marketplace
  * Other vendors provide lesson plans
* Features
  * Daily digest email
  * Text messaging
  * Widgets with the calendar
  * Assignment == School Desk Task
  * Grades have different types and calculations
  * Tracks attendance
  * Tracks resources
  * Profiles (pictures, emails for sending digests, phone number for text messages)
  * Lookup widget (embedded search of other services)
* Testimonials
  * User reviews (but they are old! 2017!)
  * Featured reviews (appear "fresh" from 2019, but are recycled old reviews),
    seems shady
  * Lots of undated testimonials (current user reviews?)

Forum topics:

* https://forums.welltrainedmind.com/topic/701712-best-family-calendarorganizer-app/ compat with Google Calendar
* https://forums.welltrainedmind.com/topic/683608-6th-grade-2019-2020/ winner in comparison
* https://forums.welltrainedmind.com/topic/574023-need-to-simplify-school/ student access
* https://forums.welltrainedmind.com/topic/670110-reviews-on-open-tent-academy-writing-classes/ mentioned putting in schedule
* https://forums.welltrainedmind.com/topic/658388-were-starting-to-hate-math-mammoth/ lesson plan
* https://forums.welltrainedmind.com/topic/539500-onenote-vs-homeschool-planet/ gushing for planet
* https://forums.welltrainedmind.com/topic/558061-homeschool-planet-users-help-a-newbie-out/ support question
* https://forums.welltrainedmind.com/topic/540597-homeschool-tracker-online-vs-homeschool-planet/ beating homeschool tracker
* https://forums.welltrainedmind.com/topic/481657-homeschool-planet-anyone-trying-it/ beta review 2013
* https://forums.welltrainedmind.com/topic/684121-homeschool-planet-coupon-code/ looking for coupon code
* https://forums.welltrainedmind.com/topic/618353-homeschool-planner-and-how-do-you-plan/ product plug
* https://forums.welltrainedmind.com/topic/666654-mid-year-shakeup/ trialing
* https://forums.welltrainedmind.com/topic/521718-digital-planner-for-looping-schedule/ looping schedule question w/ no response
* https://forums.welltrainedmind.com/topic/540799-which-online-planner-will-let-me-bump-two-ways/ bump work to another day
* https://forums.welltrainedmind.com/topic/681351-educents-shutting-down-changing-direction/ comparison

### Homeschool Reporting Online

https://homeschoolreporting.com/

Forum topics:

Nothing found.

### Homeschool Skedtrack

http://www.homeschoolskedtrack.com/HomeSchool/displayLogin.do

Forum topics:

* https://forums.welltrainedmind.com/topic/109604-anyone-use-homeschool-skedtrack/ discussion of site
* https://forums.welltrainedmind.com/topic/521718-digital-planner-for-looping-schedule/ looping questions
* https://forums.welltrainedmind.com/topic/540598-lets-share-how-we-set-up-onenote/ onenote replacing skedtrack
* https://forums.welltrainedmind.com/topic/265754-gradingrecord-keepingtranscripts/ looking for tools 2011
* https://forums.welltrainedmind.com/topic/109114-free-curriculum-list/ curriculum list 2009
* https://forums.welltrainedmind.com/topic/337281-if-you-have-a-long-loop-schedule-for-your-homeschool-can-you-share/ looping schedule topic
* https://forums.welltrainedmind.com/topic/397457-scholaric-or-homeschool-tracker-plus/ tool comparison
* https://forums.welltrainedmind.com/topic/401214-which-homeschool-planners-allow-you-to-print-like-this/ printing schedules
* https://forums.welltrainedmind.com/topic/338659-if-youve-done-both-mfw-and-hod/ preparing schedules
* https://forums.welltrainedmind.com/topic/586966-how-do-you-schedule-your-180-days/ planning many days in advance
* https://forums.welltrainedmind.com/topic/449025-is-there-has-planner-app/?tab=comments looking for ipad app
* https://forums.welltrainedmind.com/topic/665813-opinions-on-memoria-press-core-curriculum/ product plug

### Homeschool Tracker

https://www.homeschooltracker.com/

Forum topics:

* https://forums.welltrainedmind.com/topic/383203-edu-track-vs-homeschool-tracker/ vs EduTrack 2012
* https://forums.welltrainedmind.com/topic/23954-homeschool-tracker-questions/ HST vs HST+ 2008
* https://forums.welltrainedmind.com/topic/540597-homeschool-tracker-online-vs-homeschool-planet/ vs planet 2015
* https://forums.welltrainedmind.com/topic/254395-record-keeping-homeschool-tracker-and-ipad/ ipad
* https://forums.welltrainedmind.com/topic/54161-homeschool-tracker-question/ questions about product 2008
* https://forums.welltrainedmind.com/topic/397457-scholaric-or-homeschool-tracker-plus/ vs Scholaric 2012
* https://forums.welltrainedmind.com/topic/186110-whats-a-good-homeschool-planner/ product plug 2010
* https://forums.welltrainedmind.com/topic/5826-do-you-find-lesson-planning-software-useful-any-for-mac-users-other-suggestions/ product plug
* https://forums.welltrainedmind.com/topic/265754-gradingrecord-keepingtranscripts/ product plug
* https://forums.welltrainedmind.com/topic/401214-which-homeschool-planners-allow-you-to-print-like-this/ print question
* https://forums.welltrainedmind.com/topic/649011-do-you-lesson-plan/ product plug
* https://forums.welltrainedmind.com/topic/249069-transcript-for-7th-and-8th-grade/ passing reference
* https://forums.welltrainedmind.com/topic/109114-free-curriculum-list/ product plug 2009
* https://forums.welltrainedmind.com/topic/683608-6th-grade-2019-2020/ tried but planet won 2019
* https://forums.welltrainedmind.com/topic/637551-is-classical-conversations-a-cultor-productor/ for transcripts 2017
* https://forums.welltrainedmind.com/topic/8909-how-much-is-too-much-or-not-enough/ product plug 2008
* https://forums.welltrainedmind.com/topic/129844-homeschool-tracker-question/?tab=comments#comment-1222284 question about product
* https://forums.welltrainedmind.com/topic/203459-terribly-embarrassed-herewhyhow-do-you-create-lesson-plans/ product plug
* https://forums.welltrainedmind.com/topic/222511-ambleside-online/ for printing
* https://forums.welltrainedmind.com/topic/341507-wondering-about-tearing-up-workbooks-vs-leaving-them-whole/ product plug
* https://forums.welltrainedmind.com/topic/111220-baffled-by-michael-clay-thompson-la/ lesson plans
* https://forums.welltrainedmind.com/topic/100480-ambleside-online-users/ product plug
* https://forums.welltrainedmind.com/topic/84644-need-ideas-for-tracking/?tab=comments#comment-828678 product plug

### Homeschooling Records

https://homeschoolingrecords.com/default.aspx

Forum topics:

Nothing found.

### Lessontrek

https://lessontrek.com/

Forum topics:

* https://forums.welltrainedmind.com/topic/618353-homeschool-planner-and-how-do-you-plan/ former user 2016
* https://forums.welltrainedmind.com/topic/688779-how-do-i-make-a-custom-planner/?tab=comments#comment-8402441 asked about usage

### My School Year

https://www.myschoolyear.com/

Forum topics:

Nothing found.

### Scholaric

https://www.scholaric.com/marketing

Forum topics:

* https://forums.welltrainedmind.com/topic/397457-scholaric-or-homeschool-tracker-plus/ vs HST+ 2012
* https://forums.welltrainedmind.com/topic/423847-scholaric-review-and-giveaway/ product plug 2012
* https://forums.welltrainedmind.com/topic/675595-grading-software/ product plug grading software 2018
* https://forums.welltrainedmind.com/topic/540799-which-online-planner-will-let-me-bump-two-ways/ product plug 2015
* https://forums.welltrainedmind.com/topic/401214-which-homeschool-planners-allow-you-to-print-like-this/ for printing 2012
* https://forums.welltrainedmind.com/topic/297229-scholaric-vs-skedtrack-for-planning/?tab=comments#comment-2986890 vs skedtrack 2011
* https://forums.welltrainedmind.com/topic/481657-homeschool-planet-anyone-trying-it/ vs planet 2013
* https://forums.welltrainedmind.com/topic/521718-digital-planner-for-looping-schedule/ looping question
* https://forums.welltrainedmind.com/topic/640723-do-you-use-a-planner-of-some-sort/ planner question 2017
* https://forums.welltrainedmind.com/topic/618353-homeschool-planner-and-how-do-you-plan/ product comparison 2016
* https://forums.welltrainedmind.com/topic/624024-record-keeping-for-relaxed-homeschooling-with-larger-families/ large family 2016
* https://forums.welltrainedmind.com/topic/611151-truthquest-vs-veritas-press-self-paced-history/ product plug 2016
* https://forums.welltrainedmind.com/topic/477446-little-known-secret-curric-to-try/ product plug 2013
* https://forums.welltrainedmind.com/topic/358659-ipads-for-homeschooling/ product plug 2012
* https://forums.welltrainedmind.com/topic/343523-well-my-paces-curriculum-came-today/ product plug 2012
* https://forums.welltrainedmind.com/topic/449025-is-there-has-planner-app/?tab=comments ipad 2012
* https://forums.welltrainedmind.com/topic/376514-curious-about-winter-promise/ product plug 2012

### Well Planned Gal

https://shop.wellplannedgal.com/index.php/shop/well-planned-day-online.html

* https://forums.welltrainedmind.com/topic/393417-ultimate-homeschool-planner-vs-well-planned-day/ product comparison 2012
* https://forums.welltrainedmind.com/topic/195695-anyone-try-the-well-planned-day-planners/ paper? 2010
* https://forums.welltrainedmind.com/topic/186110-whats-a-good-homeschool-planner/ product comparison 2010
* https://forums.welltrainedmind.com/topic/618353-homeschool-planner-and-how-do-you-plan/ product comparison 2016
* https://forums.welltrainedmind.com/topic/481657-homeschool-planet-anyone-trying-it/ vs planet 2013
* https://forums.welltrainedmind.com/topic/415058-ipad-users-homeschool-helper-app/ ipad 2012

### Audience

Things to try:

* Google Adwords keyword planner
* followerwonk.com
* Topsy Analytics to check social mentions?
* Ahrefs Site Explorer to see referral traffic for other sites
* BuzzSumo for Twitter analysis
* Homeschool magazines? Is that a thing?
* Amazon reviews of homeschool stuff
* Reddit has a homeschool subreddit
* Popular homeschool blogs to contact?
* Homeschool podcasts?
* Monthly webinars?

### Future

* Dunning - Don't make customer sign in to update card info
* Dunning - Don't email until the card has actually failed (wait until 3rd reattempt or 5 days)
* Dunning - Email multiple times. It's ok. People miss stuff.
* Churn Buster if crazy popular by some remote chance
* Make Annual <-> Monthly up/downgrade not hard

### Persona

This is an exercise to build a reasonable persona
of the ideal customer for School Desk
as suggested in SaaS Marketing Essentials.

Hannah Homeschooler:

* Hannah is a 30 - 45 year old woman.
* She is educated and a self-starter (fiercely independent).
* Hannah has two children.
* Hannah lives in a suburban area or small to mid-size city.
* Her time is constrained while balancing home obligations with child education.
* Hannah has a fondness for paper for keeping records.
  Before finding School Desk,
  she was starting to hit the limits of her process
  and was looking for a way to save time.
* Hannah wants full control of *what* she teaches her children,
  but would like some tools that can take out the drudgery
  of building weekly schedules for school.
* After using School Desk,
  Hannah benefits from the automatically generated schedules
  that show *her* material
  in a timeframe that fits the constraints
  that she specified to School Desk.
* School Desk makes it painless to stay on top of homeschool
  so that she focus her attention on other parts of her life.

### Product Position

What's the point? Why would Hannah want to use School Desk?
Is it control? Is it time saving?

What is the benefit?

* "Take Control of Your Homeschool Plan"
* "Save Time Building Your Homeschool Plan"
* "Focus on Your School, Let Us Handle the Schedule"
* Stop swimming in spreadsheets
* Not an airplane cockpit

Customer answer: The key benefit is simplicity.
