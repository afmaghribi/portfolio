# AI Agent Guide for lychnobyte.my.id

This document serves as a technical handover for AI agents tasked with maintaining or extending this portfolio.

## 🏗 Architecture
- **Framework**: [Hugo](https://gohugo.io/) (Extended version required).
- **Theme**: [Hugo Profile](https://github.com/gurusabarish/hugo-profile).
- **Deployment**: GitHub Pages via GitHub Actions.
- **CDN/DNS**: Cloudflare (Manual cache purge required after deployment).

## 📁 Key File Structure
- `hugo.yaml`: The primary configuration file. Contains site parameters, Hero section, Experience entries, and Professional Training data.
- `content/certifications/`: individual markdown files for each certification.
- `content/blogs/`: Blog posts in Markdown.
- `layouts/`: Custom template overrides.
  - `layouts/index.html`: Controls the homepage section order.
  - `layouts/certifications/list.html`: Custom paginated layout for the certifications page.
  - `layouts/partials/sections/trainings.html`: Custom tabbed layout for Professional Training.
- `static/images/`: Local storage for profile pictures and manual certificate badges.

## 🎓 Certification Management
Certifications are managed as **individual files** to support pagination and manual ordering.
- **Location**: `content/certifications/`.
- **Ordering**: Controlled by the `weight` parameter in the front matter (Smaller = Higher on page).
- **Verification**: Each file should have a `url_credly` parameter for the verification link.
- **Images**: Prefer direct `images.credly.com` links for auto-preview. For manual certificates, use `/images/filename.png`.

## 💼 Experience & Training
Both sections use a **tabbed interface** (Bootstrap Pills).
- **Experience**: Configured in `hugo.yaml` under `params.experience.items`. Grouped by categories (Professional, Internships, Community Projects).
- **Training**: Configured in `hugo.yaml` under `params.trainings.items`. Uses the `experience-container` class for visual consistency.

## 🚀 Deployment Workflow
1. Make changes to content or configuration.
2. Run `hugo` locally (optional) to verify build.
3. Commit and push to `main` branch.
4. GitHub Actions will build and deploy automatically.
5. **CRITICAL**: Log into Cloudflare and "Purge Everything" to see changes live.

## 🤖 Rules for Agents
1. **Respect Manual Edits**: The user frequently makes manual tweaks to `hugo.yaml`. Always read the file before editing and avoid overwriting user-specific wording.
2. **Pathing**: Always use absolute paths for tools (e.g., `/home/ubuntu/personal-blog/...`).
3. **No Redundant Sections**: Do not re-enable the "About" or "Education" sections on the homepage without explicit permission; they were merged into the Hero and Training sections for UI/UX balance.
4. **Git Safety**: Always check `git status` before committing.
