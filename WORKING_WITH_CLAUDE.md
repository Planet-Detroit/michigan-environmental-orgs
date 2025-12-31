# 🎯 Working with Claude - Best Practices for This Project

## The Problem You're Facing

✅ You identified it perfectly:
- Conversations get cut off → lose context
- I don't have persistent file access
- You have to re-explain the project every time
- Documentation is scattered

---

## 🚀 Solution: The "Context Package" Approach

### Before Starting ANY New Claude Conversation:

**Upload these 3-4 files immediately:**

1. **README.md** (project overview - create this tonight!)
2. **package.json** (shows dependencies, what's installed)
3. **src/pages/Organizations.jsx** (main component)
4. **.env.example** (shows required env vars, no secrets)

**Say something like:**
> "I'm working on the Michigan Environmental Orgs directory. Here are the key files. Quick summary: It's a React + Supabase app that shows environmental orgs on a map. We have sidebar navigation, form submissions go to pending status, and it's deployed on GitHub Pages."

---

## 📋 Create These Documentation Files Tonight (5 minutes)

### 1. README.md (Most Important!)

```markdown
# Michigan Environmental Organizations Directory

React app showing 605+ environmental orgs in Michigan. Users can browse on a map, filter by geography/focus areas, and submit new organizations.

## Tech Stack
- React 19 + Vite
- Supabase (database)
- Leaflet (maps)
- React Router
- Tailwind CSS

## Key Features
- Interactive map with 517 geocoded orgs
- Sidebar filters (geography, focus areas)
- Submission form → pending review
- Admin panel for approving submissions

## Quick Start
```bash
npm install
cp .env.example .env  # Add your Supabase keys
npm run dev
```

## Deploy
```bash
npm run deploy  # Deploys to GitHub Pages
```

## Project Structure
- `/src/pages/Organizations.jsx` - Main browse page with sidebar
- `/src/pages/SubmissionForm.jsx` - Submit new org form
- `/src/components/OrganizationMap.jsx` - Leaflet map
- `/src/components/Header.jsx` - Site header
- Admin panel: `org-admin-FINAL-with-pending.html`

## Database (Supabase)
- Table: `organizations` (605 rows)
- Key fields: name, city, focus[], latitude, longitude, status
- Status values: approved, pending, rejected

## Deployment
- Repo: https://github.com/Planet-Detroit/michigan-environmental-orgs
- Live: https://planet-detroit.github.io/michigan-environmental-orgs/
- Deploy from: `/frontend/org-directory` subfolder

## Environment Variables
See `.env.example` for required vars.
```

---

### 2. .env.example (No Secrets!)

```bash
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key-here
```

---

### 3. CHANGELOG.md (Track Major Changes)

```markdown
# Changelog

## 2024-12-29
- ✅ Moved navigation from top bar to sidebar
- ✅ Added submission form with dynamic focus areas from Supabase
- ✅ Fixed main.jsx to import App.css
- ✅ Created admin panel with pending submissions tab
- ✅ Deployed to GitHub Pages
- ✅ Added auto-slug generation
- ✅ Map-first view (default to map instead of list)

## Known Issues
- 88 orgs missing addresses (can't geocode)
- Admin panel loads slowly with 605 orgs

## TODO
- Add geocoding tracking tab to admin
- Pagination for org list
- Better mobile responsive design
```

---

### 4. COMMON_TASKS.md (Quick Reference)

```markdown
# Common Tasks

## Start Development
```bash
cd /Users/user/projects/michigan-environmental-orgs
npm run dev
# Opens at http://localhost:5173
```

## Deploy to GitHub Pages
```bash
npm run deploy
# Wait 2-3 minutes
# Visit: https://planet-detroit.github.io/michigan-environmental-orgs/
```

## Update Styles
Edit: `src/App.css`
Changes auto-refresh in dev mode

## Add New Page
1. Create file in `src/pages/`
2. Import in `src/App.jsx`
3. Add route: `<Route path="/page" element={<Page />} />`

## Database Changes
- Go to: https://supabase.com/dashboard/project/zocaxurjikmwskmwfsfv
- SQL Editor for queries
- Table Editor for quick edits

## Geocode Missing Addresses
```bash
python3 geocode_addresses.py --run
# Uses Google Maps API (~$2.50 per 500)
```

## Supabase Credentials
- Project: zocaxurjikmwskmwfsfv
- URL: https://zocaxurjikmwskmwfsfv.supabase.co
- Keys in: Settings → API
```

---

## 🎯 Your Workflow for Next Time

### Starting a New Conversation with Claude:

**Step 1: Upload Context (30 seconds)**
```
[Upload these files]
- README.md
- package.json  
- CHANGELOG.md (if asking about recent changes)
- The specific file you're working on
```

**Step 2: One-Sentence Project Intro**
```
"Michigan Environmental Orgs directory - React + Supabase app 
with map view, filters, and submission form. See README for details."
```

**Step 3: State Your Goal**
```
"I need to [specific task]. Here's the current file: [upload file]"
```

**Example:**
```
[Upload: README.md, Organizations.jsx]

"Michigan Env Orgs directory - React/Supabase map app (see README). 

I want to add pagination to the org list because it's slow with 605 orgs. 
Here's the current Organizations component."
```

---

## 📁 File Organization on Your Mac

**Keep these reference files at project root:**

```
michigan-environmental-orgs/
├── README.md               ← Project overview
├── CHANGELOG.md           ← What changed when
├── COMMON_TASKS.md        ← Quick commands
├── .env.example           ← Required env vars
├── .env                   ← Your actual keys (gitignored)
├── package.json           ← Dependencies
├── vite.config.js         ← Build config
├── src/
│   ├── pages/
│   ├── components/
│   └── utils/
└── scripts/               ← Python geocoding, etc.
    └── geocode_addresses.py
```

---

## 🎨 For Design/Style Changes

**Always share:**
1. Screenshot of current state
2. App.css (or the CSS file)
3. The component JSX
4. What you want different

**Example:**
```
[Upload: App.css, Organizations.jsx, screenshot]

"The sidebar nav buttons look too plain. I want them 
rounded with a subtle shadow. Here's current code + screenshot."
```

---

## 🐛 For Bug Fixes

**Share:**
1. Error message (console screenshot)
2. The file causing the error
3. What you were trying to do

**Example:**
```
[Upload: screenshot of console error, SubmissionForm.jsx]

"Getting 'supabase.from is not a function' when submitting form. 
See error screenshot. Here's the form component."
```

---

## 💾 Git Commit Best Practices

**Make commits after each working feature:**

```bash
git add .
git commit -m "Add: sidebar navigation to replace top nav bar"
git push

# Next feature
git add .
git commit -m "Fix: form focus areas now load from Supabase"
git push
```

**Benefits:**
- Easy to see what changed when
- Easy to revert if something breaks
- Shows Claude your progress in CHANGELOG

---

## 🎯 Tonight Before Bed (5 minutes)

```bash
cd /Users/user/projects/michigan-environmental-orgs

# 1. Create README.md (copy content above)
nano README.md
# Paste, save

# 2. Create .env.example
nano .env.example
# Add template (no real keys!)

# 3. Create CHANGELOG.md
nano CHANGELOG.md
# List what we did today

# 4. Commit everything
git add .
git commit -m "Add project documentation (README, CHANGELOG)"
git push

# Done! Sleep! 😴
```

---

## 🚀 Advanced: Create a Custom Skill

**For future sessions, create a Claude Skill:**

1. Go to claude.ai settings
2. Create new skill: "Michigan Env Orgs Project"
3. Add this:

```
This is the Michigan Environmental Organizations Directory project.

Tech: React 19, Supabase, Leaflet maps, React Router, Tailwind
Database: 605 orgs in Supabase, table name: organizations
Deployment: GitHub Pages at planet-detroit.github.io/michigan-environmental-orgs

Key files:
- src/pages/Organizations.jsx - main browse page with sidebar filters
- src/pages/SubmissionForm.jsx - form for submitting new orgs
- src/components/OrganizationMap.jsx - Leaflet map component

Recent changes: Moved nav to sidebar, added submission workflow with 
pending status, deployed to GitHub Pages.

Common tasks in COMMON_TASKS.md. See README.md for full details.
```

Then Claude will have this context automatically!

---

## 🎓 Summary

**Bad Approach (What We Did Today):**
- Long conversation
- Files scattered
- Lost context multiple times
- Had to re-explain project

**Good Approach (For Next Time):**
- Upload README.md + key files at start
- One-sentence project intro
- Clear goal statement
- Reference docs in repo

**Best Approach (Once Set Up):**
- Custom Claude skill with project context
- README + CHANGELOG always updated
- Upload only the files you're working on
- Git commits show progress

---

## 💡 Final Tips

1. **Start small conversations** - One feature at a time
2. **Share screenshots** - Show don't tell when possible
3. **Update CHANGELOG** - After each session
4. **Commit often** - Safety net
5. **Upload context first** - Before asking questions

---

**Tonight: Create README.md (most important!)**
**Next session: Upload README.md + specific file → instant context!**

You'll save SO much time. Promise. 🎯

Now go to bed! 😴
