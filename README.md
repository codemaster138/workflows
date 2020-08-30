# workflows
In this repo I will document my git &amp; github actions workflows.
## How to read
In the following, every headline &amp; first paragraphwill describe one scenario. The content up to the next headline will explain what git workflow I tend to use in such a project and, if any, which github actions I use in that case.

# General git workflows
These are relatively non-specific workflows I use for almost every repo

## Anything with a compile/build process
If my project has a compile/build process then my git workflow will generally look something like this:
I write code on the master branch. Anything on the master branch of my repo is code that is currently WIP and _might_ not be in the latest release yet. When I feel that my code is ready to release, I will move to the `deploy` branch and pull in all changes using `git pull origin master`. When I push to github, a GitHub Actions workflow I have listening on the `depoly` branch will kick in and compile my code into the `<project root>/dist` directory. It will then use `git subtree split [...]` to push only the contents of the `dist` directory to the `production` branch. From here on, more project-specific workflows will take over and do things like **deploy to heroku** or **create a release**.

## Anything *without* a compile/build process
If my project does _**not**_ need to be compiled or built for production, I will adopt a slightly different workflow. I will still write code on the `master` and occasionally on `feature_*` branches, and still have a deploy branch to take care of deploying the code. My GitHub Actions will listen to pushes on that branch, and then create a new automated release. I will then go to my releases pannel and write a propper description of the release before publishing it.

# Projects using webpack
Whenever I have a project that uses webpack, I will adopt the workflow for [Anything with a compile/build process]. My GitHub actions will look like this:
```yaml
# This is a basic workflow to help you get started with Actions

name: deploy

# Controls when the action will run. Triggers the workflow on push or pull request
# events but only for the master branch
on:
  push:
    branches: [ deploy ]

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2
        name: Checkout repo
        with:
          ref: deploy
          token: ${{ secrets.GITHUB_TOKEN }}
      # Use the latest nodejs
      - name: Use Node.js 14.9.x
        uses: actions/setup-node@v1
        with:
          node-version: '14.9.x'
      # Install dependencies
      - name: Install dependencies
        run: npm install
      # Webpack is a devDependency
      - name: Install devDependencies
        run: npm install --only=dev
      # The dist script in package.json will compile the code for prod
      - name: Build Repo
        run: npm run dist
      # Commit the compiled code
      - uses: EndBug/add-and-commit@v4
        name: commit repo
        with:
          add: 'dist'
          author_name: Jake Sarjeant
          author_email: pygamer138@gmail.com
          message: 'Build for production'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      # Push to the production branch
      - name: Push to production
        run: git push origin `git subtree split --prefix dist deploy`:production --force
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
Most of the time, there will be additional steps to deploy the code to heroku/digitalOcean/my VPS/etc..

# Deploying to Heroku
if I need to deploy code to heroku, I will add the following steps to my config:
```yaml
       - name: Checkout Repo For Heroku
        uses: actions/checkout@v2
        with:
          ref: production
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: pull
        run: git pull origin production
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - uses: akhileshns/heroku-deploy@v3.4.6
        name: Deploy to Heroku
        with:
          heroku_api_key: ${{ secrets.HEROKU_TOKEN }}
          heroku_app_name: "<project-name>"
          heroku_email: "<owner-email>"
```
