# Continuous Integration / Continuous Delivery (CI/CD)

Continuous Integration (CI) refers to merging developer changes to the main branch.

Continuous Delivery (CD) is often used when one talks about CI that also takes care of deployments.

## branch

A branch is a copy of the main branch with some changes that make it diverge from it. 

Once the feature or change in the branch is ready it can be merged back into the main branch.

## pull request

In GitHub merging a branch back to the main branch of software is quite often happening using a mechanism called pull request. 


## GitHub Actions

GitHub Actions work on a basis of workflows which is a series of jobs that are run when a certain triggering event happens.

For GitHub to recognize your workflows, they must be specified in .github/workflows folder.

The component of creating CI/CD pipelines with GitHub Actions is called a Workflow.

Each Workflow is its own separate file which needs to be configured using the YAML data-serialization language.

Each workflow must specify at least one Job, which contains a set of Steps to perform individual tasks.

A workflow starts when an event on GitHub occurs such as when someone pushes a commit to a repository or when an issue or pull request is created.

A workflow definition looks like the following script.

```

name: Deployment pipeline

on:
  push:
    branches:
      - master
  pull_request:
    branches: [master]
    types: [opened, synchronize]

jobs:
  simple_deployment_pipeline:
    runs-on: ubuntu-18.04
    steps:
      - uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow,job,took # selectable (default: repo,message)
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }} # required
        if: always()
      - uses: 8398a7/action-slack@v3
        with:
          status: custom
          fields: workflow,job,commit,repo,ref,author,took
          custom_payload: |
            {
              attachments: [{
              color: '${{ job.status }}' === 'success' ? 'good' : '${{ job.status }}' === 'failure' ? 'danger' : 'warning',
              text: `${process.env.AS_WORKFLOW}\n${process.env.AS_JOB} (${process.env.AS_COMMIT}) of ${process.env.AS_REPO}@${process.env.AS_REF} by ${process.env.AS_AUTHOR} ${{ job.status }} in ${process.env.AS_TOOK}`,
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v1
        with:
          node-version: '12.x'
      - name : GITHUB CONTEXT
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: |
          echo "NOT_SKIP=${{ true }}" >> $GITHUB_ENV
          echo "$GITHUB_CONTEXT"
          echo ${{ github.event_name }}
      - name: Get Commit Message
        run: echo "COMMIT_MSG=$(git log --format=%B -n 1 ${{ github.event.after }})" >> $GITHUB_ENV
      - name: Show Commit Message
        run : echo "$COMMIT_MSG"
      - name: Skipping commit for tagging and deployment
        if: contains(env.COMMIT_MSG, 'skip')
        run: echo "NOT_SKIP=${{ false }}" >> $GITHUB_ENV
      - name: npm install
        run: npm install
      - name: lint
        run: npm run eslint
      - name: Build
        run: npm run build
      - name: Test
        run: npm run test
      - name: e2e tests
        uses: cypress-io/github-action@v2
        with:
          command: npm run test:e2e --spec 'cypress/integration/11.9.js'
          start: npm run start-prod
          wait-on: http://localhost:5000
      - name: Contains skip true
        if: contains(env.NOT_SKIP, 'true')
        run : echo "$NOT_SKIP"
      - name: Contains skip false
        if: contains(env.NOT_SKIP, 'false')
        run : echo "$NOT_SKIP"
      - name: Bump version and push tag
        if: | 
          ( github.event_name == 'push' && contains(env.NOT_SKIP, 'true') )
        uses: anothrNick/github-tag-action@1.33.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BUMP: patch
          DRY_RUN: false
      - name: Deployment
        #if: |
        #  (github.event_name == 'push'  &&  env.COMMIT_MSG != 'skip')
        if: | 
          ( github.event_name == 'push' && contains(env.NOT_SKIP, 'true') )
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
            heroku_api_key: ${{secrets.HEROKU_API_KEY}}
            heroku_app_name: "app-part-example"
            heroku_email: "user@example.com"
            healthcheck:  https://app-part-example.herokuapp.com/health
            checkstring: "ok"
            rollbackonhealthcheckfailed: true
            
            

```

![alt text](https://github.com/jylhakos/Part11/blob/main/Part11.png?raw=true)

