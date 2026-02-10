## Initialize Repository

```bash
git init
git remote -v
git config --list
git config user.email
git config --global user.email
```

## Global email (applies to all repos)
```bash 
git config --global user.email "sanduns@mitesp.com"
```
## Project-specific email (recommended)
```bash
git config user.email "hakssiwantha@gmail.com"
git config user.email "sanduns@mitesp.com"
```
## Add GitHub remotes
```bash
git remote add origin git@github.com:your-username/my-new-app.git
git remote add personal git@github.com:your-username/your-repo-name.git
git remote add work git@github.com:your-username/your-repo-name.git
```
## Add GitLab remote
```bash
git remote add work git@gitlab.com:work_group/my-new-app.git
```
Use work email only for this repository
```bash
git config user.email "hakssiwantha@gmail.com"
git config user.email "sanduns@mitesp.com"
```
## Commit & Push
```bash
git add .
git commit -m "Added new feature"
```
Push to all remotes
```bash
git push origin main
git push personal main
git push work main
```