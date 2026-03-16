# movingcircle.work

Research portfolio for Dong-Won Lee — paper reviews and implementation notes oriented toward geometric vision, 3D scene representation, teleoperation, and robot manipulation.

Built with [Hugo](https://gohugo.io) and the [Blowfish](https://blowfish.page) theme.

---

## Prerequisites

### Install Hugo (extended version required)

**macOS**
```bash
brew install hugo
```

**Linux (Debian/Ubuntu)**
```bash
# Replace VERSION with the version below (currently 0.147.2)
HUGO_VERSION=0.147.2
wget -O /tmp/hugo.deb \
  https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
sudo dpkg -i /tmp/hugo.deb
```

**Windows**
```powershell
winget install Hugo.Hugo.Extended
```

Verify:
```bash
hugo version
# Should show: hugo v0.147.2+extended ...
```

---

## Install the Blowfish Theme

Blowfish is included as a Git submodule. Initialize it after cloning:

```bash
git clone https://github.com/moovingsloth/movingcircle.work.git
cd movingcircle.work
git submodule update --init --recursive
```

This populates `themes/blowfish/`.

To add the submodule for the first time (already done if you cloned this repo):
```bash
git submodule add -b main https://github.com/nunocoracao/blowfish.git themes/blowfish
```

To update Blowfish to the latest version:
```bash
git submodule update --remote themes/blowfish
```

---

## Run Locally

```bash
hugo server --buildDrafts
```

Open [http://localhost:1313](http://localhost:1313). The server reloads automatically on file changes.

For a production-accurate preview (minified, no drafts):
```bash
hugo server --environment production
```

---

## Build for Production

```bash
hugo --minify
```

Output is written to `public/`. This directory is what gets deployed.

---

## Project Structure

```
.
├── .github/workflows/deploy.yml   # GitHub Actions deployment
├── archetypes/
│   ├── reviews.md                 # Template for new paper reviews
│   └── projects.md                # Template for new project entries
├── assets/css/custom.css          # Custom styles for the homepage
├── config/_default/
│   ├── hugo.toml                  # Site configuration
│   ├── params.toml                # Blowfish theme parameters
│   ├── languages.en.toml          # Language + author settings
│   └── menus.en.toml              # Navigation menu
├── content/
│   ├── _index.md                  # Homepage metadata
│   ├── about/index.md             # About page
│   ├── reviews/                   # Paper review entries
│   │   └── */index.md
│   └── projects/                  # Project entries
│       └── */index.md
├── layouts/index.html             # Custom homepage template
├── static/
│   ├── CNAME                      # Custom domain for GitHub Pages
│   └── files/                     # Place cv.pdf here
└── themes/blowfish/               # Theme submodule (git submodule)
```

---

## How to Add a Paper Review

```bash
hugo new reviews/paper-short-name/index.md
```

This generates a new file from `archetypes/reviews.md`. Fill in:

- `title` — full descriptive title (not just the paper title; name it like an essay)
- `date` — review date
- `tags` — choose from: `3DGS`, `Teleoperation`, `Manipulation`, `Digital-Twin`, `Humanoid`, `Segmentation`, `Tracking`, `Optimization`, `Geometric-Vision`
- `featured: true` — set this to show the card on the home page (keep to 6 or fewer)
- `weight` — lower number = shown earlier on list and home pages
- `insight` — one sentence distilling the key technical contribution

Write the review following the section structure in the archetype.

---

## How to Add a Project Entry

```bash
hugo new projects/project-short-name/index.md
```

Fill in `summary_line` for the home page card, set `featured: true` for home page display, and follow the archetype structure.

---

## Editing Contact Information

Edit in two places:

1. **`layouts/index.html`** — the contact block at the bottom of the homepage (email, GitHub URL, CV link)
2. **`config/_default/languages.en.toml`** — the `[author]` block (used by Blowfish in headers/footers)

Replace all instances of `moovingsloth` with your actual GitHub username.

---

## Editing Site Author / Description

`config/_default/languages.en.toml`:
```toml
[author]
  name = "Dong-Won Lee"
  headline = "Undergraduate Researcher"
  bio = "..."
  links = [
    { email = "mailto:dlehddnjs245@khu.ac.kr" },
    { github = "https://github.com/moovingsloth" },
  ]
```

---

## Deploying to GitHub Pages with movingcircle.work

### Step 1: Create the GitHub repository

Create a new repository at `https://github.com/moovingsloth/movingcircle.work` (or any name you prefer — the domain is configured via CNAME, not the repo name).

### Step 2: Push the project

```bash
cd movingcircle.work
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/moovingsloth/movingcircle.work.git
git push -u origin main
```

### Step 3: Enable GitHub Pages

1. Go to the repository on GitHub → **Settings** → **Pages**
2. Under **Build and deployment**, set **Source** to **GitHub Actions**
3. The `deploy.yml` workflow will build and publish the site automatically on every push to `main`

### Step 4: Configure the custom domain on GitHub Pages

1. In **Settings** → **Pages** → **Custom domain**, enter: `movingcircle.work`
2. Check **Enforce HTTPS** (enable after DNS propagation, usually within 24 hours)

The `static/CNAME` file already contains `movingcircle.work`, so GitHub Pages will pick it up automatically after the first deployment.

### Step 5: Configure DNS for movingcircle.work

In your domain registrar's DNS settings, add the following records:

**Apex domain (movingcircle.work) — A records pointing to GitHub Pages:**
```
A   @   185.199.108.153
A   @   185.199.109.153
A   @   185.199.110.153
A   @   185.199.111.153
```

**Optional: www redirect — CNAME record:**
```
CNAME   www   moovingsloth.github.io
```

DNS propagation typically takes 10 minutes to 48 hours. Once propagated, HTTPS will be automatically provisioned by GitHub via Let's Encrypt.

### Verify deployment

After the GitHub Actions workflow completes (check the **Actions** tab in your repository), the site should be live at:
- `https://movingcircle.work`

---

## Changing the Color Scheme

Edit `config/_default/params.toml`:

```toml
colorScheme = "zinc"     # Options: slate, zinc, neutral, stone, gray
defaultAppearance = "light"   # Options: light, dark
autoSwitchAppearance = true   # Follow system preference
```

Available Blowfish color schemes: `slate`, `zinc`, `neutral`, `stone`, `gray`, `rose`, `pink`, `fuchsia`, `purple`, `indigo`, `blue`, `sky`, `cyan`, `teal`, `emerald`, `green`, `lime`, `yellow`, `amber`, `orange`, `red`.

For a research site, `zinc`, `neutral`, or `slate` are most appropriate.

---

## Notes

- Do not add a `/contact/` page. Contact information is intentionally only in the homepage contact block.
- Do not create a `/posts/` section. This site is not a general blog.
- Keep the number of featured items controlled: ≤ 6 reviews and ≤ 3 projects on the home page.
- The `layouts/index.html` overrides Blowfish's default homepage. If you want to change the homepage layout, edit that file.
