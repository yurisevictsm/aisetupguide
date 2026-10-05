#  Drupal 11 Project Setup Guide (Pantheon + DDEV + Claude) 

Step-by-step guide to start a new Drupal 11 project: create it on Pantheon first, bring it down to a local DDEV environment, wire up the `ddev-ai-workspace` AI tooling, then have Claude install the contrib modules and scaffold a new Bootstrap 5 subtheme in a single prompt.

**How this guide is organised**

- **Part A — Developer steps (1–7):** everything you do yourself, in order, top to bottom. Follow it as written.
- **Part B — Claude build specification:** the contrib modules, theme structure, `package.json`, `gulpfile.js`, the build/enable commands and the cleanup that the Step 6 prompt tells Claude to use. You do not need to follow Part B by hand; it is at the end so Part A reads straight through.
- **Part C — Background:** optional reading on what the `ddev-ai-workspace` add-on installed in Step 4 includes.

Assumes: DDEV is already installed locally. A Pantheon account with permission to create sites.

---

# Part A — Developer steps

## 1. Create the site on Pantheon (source of truth first)

Pantheon is the system of record — create the site there before anything exists locally.

In the Pantheon dashboard: New Site → choose the **Drupal 11** upstream → name the site → wait for the "dev" environment to finish initializing.

Note the site machine name — it should match (or closely match) your project/theme name used in Step 6.

---

## 2. Pull code and database down locally

Get the git URL from the dashboard's **Connection Info** panel on the `dev` environment, then clone the repo:

```bash
git clone <git_url> <project-name>
cd <project-name>
```

Pull the database and files from the dashboard: on the `dev` environment open **Backups**, create a fresh **Database** backup and a **Files** backup (so the export is current), then download both. Save them in the project root as `db-backup.sql.gz` and `files-backup.tar.gz` — Step 3 imports them by those names.

---

## 3. Bring the project up in DDEV

```bash
ddev config --project-type=drupal11 --docroot=web --php-version=8.4
ddev start
ddev composer install
ddev import-db --file=db-backup.sql.gz
ddev import-files --source=files-backup.tar.gz
ddev drush cr
```

Adjust `--docroot` if the Pantheon upstream uses a different docroot, and `--php-version` to match the version set on Pantheon (check the dashboard's **Settings → PHP Version**).

Verify `web/sites/default/settings.php` includes the DDEV-generated settings file (DDEV normally appends this automatically on first `ddev start` if the file is writable):

```php
if (file_exists($app_root . '/sites/default/settings.ddev.php')) {
  include $app_root . '/sites/default/settings.ddev.php';
}
```

---

## 4. Install the `ddev-ai-workspace` add-on

[`ddev-ai-workspace`](https://github.com/trebormc/ddev-ai-workspace) is a DDEV add-on that sets up an AI-assisted development environment inside your project: Claude Code and related tooling, plus a shared set of Drupal skills. It does not scaffold any Drupal code by itself; the project is built in Step 6. See Part C at the end of this guide for what it installs.

```bash
ddev add-on get trebormc/ddev-ai-workspace
ddev restart
```

Configure your Anthropic API key as instructed by the add-on's README (an environment variable or `.ddev/.env` entry), then start the Claude Code session:

```bash
ddev claude-code   # alias: ddev cc
```

---

## 5. Add this guide to the project root

Claude builds from the specification in Part B of this guide, so the file has to be in the project. Download it ([here](https://github.com/yurisevictsm/aisetupguide/tree/main)) and save it in the project root — the same folder as `composer.json` and `.ddev/` — with exactly this name, because the Step 6 prompt refers to it by name:

```
DRUPAL_11_SETUP_GUIDE.md
```

Confirm it is in place:

```bash
ls DRUPAL_11_SETUP_GUIDE.md
```

If this prints the filename, continue. If it says "No such file", you saved it in the wrong folder or under a different name.

---

## 6. Have Claude set up local files, modules and the subtheme

Inside the `ddev claude-code` session, give Claude a single instruction covering the local project bootstrap, the contrib modules, the new subtheme, the theme build and enable, and the final cleanup. Replace `<project-name>` with the actual project/theme machine name (the subtheme name must match it):

```
Set up this Drupal 11 project locally: run composer install if needed, confirm
settings.php includes settings.ddev.php, and rebuild caches. Then install and
enable the contrib modules and the bootstrap5 base theme defined in
DRUPAL_11_SETUP_GUIDE.md (Part B, B1). Then scaffold a new Bootstrap 5 subtheme
named <project-name> at web/themes/custom/<project-name>, using the folder
structure, package.json and gulpfile.js defined in the same guide (Part B, B2–B4)
— copy them in as-is, replacing <project-name> with the real machine name
throughout. Then build the theme assets, enable the theme and rebuild caches as
described in Part B, B5. Finally clean up the files created during setup as
described in Part B, B6.
```

Claude will route the backend/file bootstrap and theme scaffolding to the project's specialist agents automatically. Wait for it to finish before moving on; the theme is built, enabled and cleaned up when it does.

---

## 7. Export configuration once initial setup is confirmed

```bash
ddev drush cex -y
```

Commit the result (project files + new theme) once you've reviewed the diff — git write access from inside the Claude session follows this project's `git-workflow` policy, so by default you commit manually.

---

# Part B — Claude build specification

Reference material for the Step 6 prompt. Claude reads this section to scaffold the subtheme; you only need it if you want to review or change what gets built. `<project-name>` is replaced with the real machine name throughout.

## B1. Contrib modules and base theme

Claude installs and enables these as part of the Step 6 prompt. Commands are shown with `ddev`; inside the Claude session run them in the web container instead (`ssh web composer ...`, `ssh web drush ...`).

```bash
ddev composer require \
  drush/drush \
  drupal/bootstrap5 \
  drupal/module_filter \
  drupal/block_class \
  drupal/admin_toolbar \
  drupal/paragraphs \
  drupal/menu_link_attributes \
  drupal/focal_point \
  drupal/media_library_edit \
  drupal/metatag \
  'drupal/viewsreference:^2.0@beta' \
  drupal/extlink \
  drupal/twig_tweak \
  drupal/svg_image \
  drupal/paragraphs_browser

ddev drush en -y module_filter block_class admin_toolbar paragraphs \
  menu_link_attributes focal_point media_library_edit metatag \
  viewsreference extlink twig_tweak svg_image paragraphs_browser
```

`viewsreference` is pinned to `^2.0@beta` on purpose: its stable 1.x releases only support Drupal 8/9, and the Drupal 11 versions are 2.0 betas that the project's `stable` minimum-stability would otherwise block (the whole `composer require` fails and rolls back).

`drush` is a Composer dev dependency, not an enabled module. `bootstrap5` is a **base theme**, not a module — it has nothing to enable via `drush en`; it's set as the new subtheme's `base theme` (see B2).

## B2. Theme folder structure

This is the specification Claude follows in Step 6. Define the new subtheme's structure directly — do not copy it from another theme in this codebase. Machine name must match `<project-name>`.

```
web/themes/custom/<project-name>/
├── <project-name>.info.yml
├── <project-name>.libraries.yml
├── <project-name>.breakpoints.yml
├── <project-name>.theme          # preprocess hooks
├── package.json
├── gulpfile.js
├── scss/
│   ├── style.scss                # main entry point, imports everything below
│   ├── _variables.scss           # project color/type/spacing tokens
│   ├── _variables_bootstrap.scss # Bootstrap 5 variable overrides
│   ├── _mixins.scss
│   ├── layout/                   # page-level layout partials (header, footer, containers)
│   └── components/               # reusable UI piece partials (cards, buttons, nav, forms)
├── css/                          # compiled output (git-ignored or committed, team's choice)
├── js/
│   ├── scripts.js                # entry point, Drupal behaviors
│   └── scripts.min.js            # built output
├── images/
├── fonts/
├── templates/
│   ├── layout/                   # page.html.twig, region templates
│   ├── content/                  # node--*.html.twig
│   ├── paragraphs/               # paragraph--*.html.twig
│   ├── field/                    # field--*.html.twig overrides
│   ├── block/                    # block--*.html.twig
│   ├── navigation/               # menu--*.html.twig
│   ├── views/                    # views-view--*.html.twig, exposed forms
│   └── partials/                 # includable fragments (e.g. {% include %} snippets)
└── config/
    ├── install/                  # <project-name>.settings.yml (Bootstrap 5 theme settings)
    └── optional/                 # default block placements
```

`<project-name>.info.yml` must declare `base theme: bootstrap5`, override `libraries-override: {bootstrap5/global-styling: false}`, and declare the regions the project needs (start from Bootstrap 5's defaults and add only what the design requires — don't carry over a region set from another project).

## B3. package.json

```json
{
  "name": "<project-name>-theme",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "gulp",
    "build": "gulp build"
  },
  "devDependencies": {
    "autoprefixer": "^10.4.19",
    "gulp": "^4.0.2",
    "gulp-concat": "^2.6.1",
    "gulp-dart-sass": "^1.1.0",
    "gulp-postcss": "^9.0.1",
    "gulp-sourcemaps": "^3.0.0",
    "gulp-terser": "^2.1.0",
    "sass": "^1.77.0"
  },
  "browserslist": [
    "last 2 versions",
    ">0.5%",
    "not dead"
  ]
}
```

## B4. gulpfile.js

```js
const gulp = require('gulp');
const sass = require('gulp-dart-sass');
const sourcemaps = require('gulp-sourcemaps');
const postcss = require('gulp-postcss');
const autoprefixer = require('autoprefixer');
const concat = require('gulp-concat');
const terser = require('gulp-terser');

const paths = {
  scssEntry: 'scss/style.scss',
  scssWatch: 'scss/**/*.scss',
  cssDest: 'css',
  jsEntry: 'js/scripts.js',
  jsWatch: 'js/**/*.js',
  jsDest: 'js',
};

function styles() {
  return gulp.src(paths.scssEntry)
    .pipe(sourcemaps.init())
    .pipe(sass().on('error', sass.logError))
    .pipe(postcss([autoprefixer()]))
    .pipe(sourcemaps.write('.'))
    .pipe(gulp.dest(paths.cssDest));
}

function scripts() {
  return gulp.src(paths.jsEntry)
    .pipe(terser())
    .pipe(concat('scripts.min.js'))
    .pipe(gulp.dest(paths.jsDest));
}

function watch() {
  gulp.watch(paths.scssWatch, styles);
  gulp.watch(paths.jsWatch, scripts);
}

exports.styles = styles;
exports.scripts = scripts;
exports.watch = watch;
exports.build = gulp.parallel(styles, scripts);
exports.default = gulp.series(gulp.parallel(styles, scripts), watch);
```

## B5. Build and enable the theme

Claude runs these after scaffolding the subtheme, in the web container. This compiles `css/style.css` and `js/scripts.min.js` (the theme's library references the latter) and enables the theme:

```bash
ssh web "cd $DDEV_DOCROOT/themes/custom/<project-name> && npm install && npm run build"
ssh web drush then <project-name>
ssh web drush cr
```

Claude does not run `npm run dev`: it starts a long-running Sass watcher. Start it yourself when you begin theming (inside the theme directory: `npm run dev`).

## B6. Clean up files created during setup

Claude runs this last, after the theme is built, so that only files worth committing remain. It removes working files from the setup session that don't belong in the repo. It never commits (see the project's `git-workflow` policy) and never deletes anything it doesn't recognise: unknown untracked files are listed in its summary for you to decide on.

First Claude makes sure `.gitignore` excludes Claude's working artifacts and theme build tooling output — it appends this block if it isn't already there:

```gitignore
# Local DDEV environment (regenerated by Steps 3 and 4; not deployed to Pantheon)
/.ddev/

# Claude / AI-assistant working artifacts (not part of the site)
/.mcp.json
/plan-*.md
/commit-msg.txt
/LESSONS_LEARNED.md

# Theme build tooling output that should not be committed
# (compiled css/ and js/scripts.min.js ARE committed: Pantheon does not run Gulp)
node_modules/
/web/themes/custom/*/css/*.map
```

Then it cleans up:

```bash
rm -f plan-*.md commit-msg.txt
git status --short        # read-only: report anything untracked it does not recognise
```

It also removes the `.gitkeep` placeholder from every theme folder that now holds real files (git only needs it to track empty folders):

```bash
find web/themes/custom/<project-name> -name .gitkeep | while read f; do
  [ "$(ls -A "$(dirname "$f")" | wc -l)" -gt 1 ] && rm "$f"
done
```

Claude keeps the compiled `css/style.css` and `js/scripts.min.js` — Pantheon does not run Gulp, so the built assets must be committed.

---

# Part C — Background: the ddev-ai-workspace add-on

Background reading for Step 4. You do not need it to complete the setup.

## C1. What the add-on installs

[`ddev-ai-workspace`](https://github.com/trebormc/ddev-ai-workspace) is a bundle: installing it pulls in these add-ons, each of which adds a piece of the tooling:

- **Claude Code** (`ddev claude-code`, alias `ddev cc`) and **OpenCode** (`ddev opencode`): AI coding assistants that run in their own container, with direct access to your project files. They run Drupal commands (Drush, Composer, PHPUnit) in the web container over SSH, so nothing extra has to be installed on your machine.
- **Playwright MCP:** a browser the assistant can drive to open pages and take screenshots of your local site.
- **Beads** (`ddev bd`): git-backed task tracking, so the assistant can record and resume work.
- **Ralph** (`ddev ralph`): runs longer tasks autonomously from a requirements file.
- **Agents sync:** installs the shared agents, skills and rules (Drupal development, theming, testing, code review) that Claude follows. `ddev agents-update` refreshes them.

## C2. Drupal skills installed by the add-on

The agents-sync component installs a set of skills: packaged instructions that Claude loads when your request matches them. The ones relevant to Drupal work:

**Building**

- `module-scaffold`: scaffold a custom module (info, services, routing, permissions, config schema).
- `drupal-code-patterns`: templates for forms, block plugins, routes, controllers, hooks, caching, Batch and Queue APIs, and AJAX.
- `drupal-migration`: Drupal 7 upgrades and custom migrations from CSV, JSON, APIs or SQL.
- `tailwind-drupal`: TailwindCSS in a Drupal theme.

**Configuration and site operations**

- `drush-commands`: cache clears, database updates, module management and cron.
- `config-management`: config export and import, config_split, schema validation.
- `drupal-update`: safe Composer update workflow for core and contrib, with rollback.
- `drupal-debugging`: inspect services, entities, cache, watchdog logs and queries.

**Quality, performance and security**

- `quality-checks`: PHPCS, PHPStan, Rector and PHPUnit (uses the Drupal Audit module when installed).
- `drupal-audit-setup`: install and configure the Drupal Audit module.
- `code-analysis`: prioritised review report separating blockers from suggestions.
- `phpstan-phpcs-conflict-resolution`: resolve conflicts between PHPCS and PHPStan.
- `performance-audit`: cache metadata, queries, lazy builders and N+1 problems.
- `drupal-render-pipeline`: render and cache semantics (bubbling, lazy builders, escaping).
- `twig-audit`: Twig anti-patterns and cache bubbling problems.
- `drupal-accessibility`: WCAG 2.2 AA the Drupal way.
- `xdebug-profiling`: Xdebug profiling.

**Testing**

- `drupal-testing`: picks the right test type and hands off to the skills below.
- `drupal-unit-test`, `drupal-kernel-test`, `drupal-functional-test`, `drupal-functionaljs-test`: PHPUnit tests.
- `drupal-behat-test` and `drupal-playwright-test`: Behat and Playwright end-to-end tests.
- `drupal-test-suite-audit`: review and simplify an existing test suite.
- `playwright-testing` and `screenshot-analysis`: drive a browser, capture screenshots and interpret them.

**Drupal.org contribution**

- `drupal-org-contribution-prep`, `drupal-security-coverage-prep` and `drupal-multi-version-compat`: prepare a module for drupal.org, its security review, and Drupal 10/11/12 compatibility.

**Workflow**

- `beads-task-tracking`, `commit-message`, `session-distill` and `skill-creator`: task tracking, commit messages, turning session lessons into guidance, and authoring new skills.

Skills are matched to what you ask for, so you do not call them by name.
