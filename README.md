# Michigan Environmental Organizations Directory

Interactive directory of 605+ environmental organizations in Michigan. Browse organizations on a map, filter by location and focus areas, and submit new organizations for review.

## 🚀 Quick Start

```bash
# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Add your Supabase credentials to .env

# Start development server
npm run dev
```

Visit http://localhost:5173

## 🛠️ Tech Stack

- **React 19** - UI framework
- **Vite** - Build tool
- **Supabase** - Database and backend
- **Leaflet** - Interactive maps
- **React Router** - Navigation
- **Tailwind CSS** - Styling
- **Lucide React** - Icons

## ✨ Key Features

- **Interactive Map** - Browse 517 geocoded organizations on a Leaflet map
- **Advanced Filtering** - Filter by geography (region, metro, local area) and focus areas
- **Submission Form** - Public form for submitting new organizations
- **Admin Panel** - Review and approve pending submissions
- **Responsive Design** - Works on desktop and mobile

## 📁 Project Structure

```
michigan-environmental-orgs/
├── src/
│   ├── pages/
│   │   ├── Organizations.jsx      # Main browse page with sidebar
│   │   └── SubmissionForm.jsx     # Submit new org form
│   ├── components/
│   │   ├── Header.jsx             # Site header
│   │   ├── OrganizationMap.jsx    # Leaflet map component
│   │   └── RegionTable.jsx        # Regional view table
│   ├── utils/
│   │   └── geographicHierarchy.js # Geography data & filters
│   ├── App.jsx                    # Main app component
│   ├── App.css                    # Global styles
│   └── main.jsx                   # Entry point
├── public/
│   └── org-admin-FINAL-with-pending.html  # Admin panel
└── scripts/
    └── geocode_addresses.py       # Geocoding utility
```

## 🗄️ Database

**Supabase Table:** `organizations`

**Key Fields:**
- `id` - UUID
- `name` - Organization name
- `slug` - URL-friendly name
- `city` - City location
- `address` - Street address
- `latitude`, `longitude` - Coordinates
- `focus` - Array of focus areas
- `region`, `metro_area`, `geo_location` - Geographic hierarchy
- `status` - approved | pending | rejected
- `submitted_by_email` - Submitter contact
- `org_type` - Organization type (501c3, LLC, etc.)

**Row Count:** 605 approved organizations, 517 with coordinates

## 🎯 Focus Areas

Organizations are tagged with focus areas like:
- Air Quality
- Biodiversity & Habitat
- Climate Action
- Energy
- Environmental Justice
- Food & Agriculture
- Water Quality
- Wildlife & Habitat
- And more...

## 🌍 Geographic Hierarchy

**3 Levels:**
1. **Regions** (7) - Southeast Michigan, West Michigan, etc.
2. **Metro Areas** (27) - Detroit Metro, Grand Rapids Metro, etc.
3. **Local Areas** (100+) - Specific cities and counties

## 🚀 Deployment

Deployed to GitHub Pages:
**Live Site:** https://planet-detroit.github.io/michigan-environmental-orgs/

**Deploy Command:**
```bash
npm run deploy
```

This builds the app and pushes to the `gh-pages` branch.

## 🔧 Environment Variables

Create a `.env` file:

```bash
VITE_SUPABASE_URL=your-supabase-url
VITE_SUPABASE_ANON_KEY=your-anon-key
```

See `.env.example` for template.

## 📝 Common Tasks

### Start Development
```bash
npm run dev
```

### Build for Production
```bash
npm run build
```

### Deploy to GitHub Pages
```bash
npm run deploy
```

### Geocode Missing Addresses
```bash
cd scripts
python3 geocode_addresses.py --run
```

## 🔐 Admin Panel

Located at: `public/org-admin-FINAL-with-pending.html`

**Features:**
- View all organizations
- Approve/reject pending submissions
- Edit organization details
- Bulk operations
- Data quality checks

**Access:** Open the HTML file directly in a browser and enter Supabase credentials.

## 📊 Submission Workflow

1. User fills out form at `/submit`
2. Submission saved with `status='pending'`
3. Admin reviews in admin panel
4. Admin approves or rejects
5. Approved orgs appear in live directory

## 🗺️ Geocoding

**Current Status:** 517/605 organizations have coordinates

**To geocode more:**
```bash
cd scripts
python3 geocode_addresses.py --run
```

Uses Google Maps Geocoding API (~$2.50 per 500 addresses)

## 🎨 Customization

### Colors
Main brand color (teal): `#14b8a6`
Edit in `App.css` and component styles

### Focus Areas
Focus areas are loaded dynamically from Supabase
Edit directly in database to add new categories

### Geography
Edit `src/utils/geographicHierarchy.js` to modify regions/metro areas

## 🐛 Known Issues

- 88 organizations missing addresses (can't be geocoded)
- Admin panel can be slow with 605+ orgs
- Map only shows orgs with coordinates

## 📋 TODO

- [ ] Add pagination to organization list
- [ ] Add geocoding status tracking in admin
- [ ] Improve mobile responsiveness
- [ ] Add org logo uploads
- [ ] Add user accounts for org owners
- [ ] Export filtered results to CSV

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📄 License

[Add your license here]

## 📧 Contact

**Planet Detroit**
- Website: https://planetdetroit.org
- Email: [your email]
- GitHub: https://github.com/Planet-Detroit

## 🙏 Acknowledgments

Built for Planet Detroit's environmental journalism and community engagement mission.

---

**Last Updated:** December 29, 2024
