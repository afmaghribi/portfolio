# afmaghribi's Portfolio

This is a personal portfolio and blog built with [Hugo](https://gohugo.io/) and the [hugo-profile](https://github.com/gurusabarish/hugo-profile) theme.

## 🚀 Quick Start

1. **Connect to GitHub**:
   ```bash
   git remote add origin https://github.com/afmaghribi/personal-blog.git
   git branch -M main
   git push -u origin main
   ```

2. **Enable Deployment**:
   - Go to your repository settings on GitHub.
   - Select **Pages** on the left sidebar.
   - Under **Build and deployment > Source**, select **GitHub Actions**.

## 📝 How to Manage Your Site

### Add a New Blog Post
```bash
hugo new blogs/my-first-post.md
```
Edit the file in `content/blogs/my-first-post.md`.

### Update Profile Info
All your personal details (name, bio, social links, experience) are in `hugo.yaml`.

### Preview Locally
If you have Hugo installed:
```bash
hugo server -D
```
Your site will be at `http://localhost:1313`.

## 🛠 Deployment
This site uses **GitHub Actions** for automated deployment. Every time you `git push`, the site is automatically built and published to `https://afmaghribi.github.io/personal-blog/`.
