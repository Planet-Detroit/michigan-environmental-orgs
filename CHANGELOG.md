# Changelog

All notable changes to the Michigan Environmental Organizations Directory.

## [Unreleased]

### Planned
- Pagination for organization list
- Geocoding status tracking in admin panel
- Better mobile responsive design
- CSV export functionality

---

## [1.0.0] - 2024-12-29

### Added
- **Sidebar Navigation** - Moved nav from top bar into sidebar for cleaner layout
- **Submission Form** - Public form for submitting new organizations
  - Dynamic focus areas loaded from Supabase
  - Auto-generates slug from org name
  - Saves with status='pending'
- **Admin Panel** - Complete admin tool with 4 tabs
  - Organizations tab - view, edit, delete
  - Bulk Operations - assign categories to multiple orgs
  - Data Quality - fix missing data
  - **Pending Submissions** - approve/reject new submissions
- **GitHub Pages Deployment** - Live site deployed
- **Map-First View** - Default view changed from list to map
- **Geographic Hierarchy** - 3-level system (Region → Metro → Local)
- **Smart Icon Matching** - Automatic icon selection based on focus area keywords
- **ProPublica 990 Links** - Direct links to tax forms for orgs with EINs

### Fixed
- **App.css Import** - Added missing import to main.jsx
- **Supabase Variable Names** - Fixed supabase vs supabaseClient inconsistency
- **Form Focus Areas** - Changed from hardcoded to database-driven
- **Navigation Styling** - Removed obtrusive top nav bar

### Changed
- **Project Structure** - Moved from ask-planet-detroit subdirectory to standalone repo
- **Repository** - New dedicated repo: michigan-environmental-orgs
- **Deployment URL** - Now at planet-detroit.github.io/michigan-environmental-orgs/
- **Default View** - Map view instead of list view
- **Map Zoom** - Adjusted to show full Michigan state
- **Marker Size** - Reduced for cleaner map appearance

### Database
- Added fields: `status`, `submitted_by_name`, `submitted_by_email`, `org_type`, `rejection_reason`
- Created indexes on status and org_type for performance
- Total organizations: 605 approved
- Geocoded organizations: 517 (85%)

### Technical
- React 19 + Vite
- Supabase for database
- Leaflet for maps
- React Router for navigation
- Tailwind CSS for styling
- Deployed on GitHub Pages

---

## Known Issues

### Critical
- None currently

### Medium Priority
- 88 organizations missing addresses (cannot be geocoded)
- Admin panel loads slowly with 605 organizations
- No pagination on organization list

### Low Priority
- Some focus area names need standardization
- Mobile view could be more optimized

---

## Technical Debt

- Consider adding pagination to improve performance
- Refactor admin panel into React component
- Add loading states for map
- Improve error handling in forms

---

## Migration Notes

### From ask-planet-detroit to standalone repo
- Moved files from `ask-planet-detroit/frontend/org-directory/` to `michigan-environmental-orgs/`
- Updated vite.config.js base path
- Created new GitHub repository
- Deployed to new GitHub Pages URL

### Database Schema Updates
```sql
-- Added 2024-12-29
ALTER TABLE organizations ADD COLUMN status TEXT DEFAULT 'approved';
ALTER TABLE organizations ADD COLUMN submitted_by_name TEXT;
ALTER TABLE organizations ADD COLUMN submitted_by_email TEXT;
ALTER TABLE organizations ADD COLUMN org_type TEXT;
ALTER TABLE organizations ADD COLUMN rejection_reason TEXT;
```

---

## Statistics

- **Organizations:** 605
- **Geocoded:** 517 (85%)
- **Focus Areas:** 42 unique
- **Cities:** 200+
- **Code Files:** ~20 components
- **Lines of Code:** ~8,000

---

**Format:** This changelog follows [Keep a Changelog](https://keepachangelog.com/) format.
