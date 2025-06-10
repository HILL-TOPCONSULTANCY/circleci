# CircleCI : Setup, Configuration, and Deployment

## Table of Contents
1. [Introduction to CircleCI](#introduction)
2. [Setting Up a Project with GitHub](#github-setup)
3. [Environment Variables Configuration](#env-vars)
4. [Writing the CircleCI Config File](#config-file)
5. [Pipeline and Branch Configuration](#pipeline-config)
6. [Building and Deploying Your Project](#build-deploy)
7. [Advanced Configuration Tips](#advanced-tips)

<a name="introduction"></a>
## 1. Introduction to CircleCI

CircleCI is a continuous integration and delivery platform that helps automate the build, test, and deployment processes. It integrates seamlessly with GitHub and other version control systems.

Key features:
- Automated testing on every commit
- Parallel job execution
- Customizable workflows
- Docker support
- Artifact caching

<a name="github-setup"></a>
## 2. Setting Up a Project with GitHub

### Step 1: Connect CircleCI to GitHub
1. Sign up at [CircleCI](https://circleci.com) using your GitHub account
2. Go to the CircleCI dashboard and click "Add Projects"
3. Find your GitHub repository in the list and click "Set Up Project"

### Step 2: Add the Project to CircleCI
1. Choose the operating system (Linux, macOS, Windows)
2. Select your language/framework (Node.js, Python, Java, etc.)
3. CircleCI will suggest a basic configuration file

### Step 3: Create a `.circleci` Directory
In your project root:
```bash
mkdir .circleci
```

<a name="env-vars"></a>
## 3. Environment Variables Configuration

### Adding Environment Variables
1. In the CircleCI dashboard, navigate to your project
2. Click "Project Settings" > "Environment Variables"
3. Add your variables (e.g., API keys, database URLs)

### Using Environment Variables
In your config file, reference them with `$VARIABLE_NAME`:
```yaml
- run:
    name: Deploy to production
    command: |
      echo "Deploying with API key $PRODUCTION_API_KEY"
```

### Contexts (For Organization-wide Variables)
1. Go to "Organization Settings" > "Contexts"
2. Create a new context (e.g., "production")
3. Add variables to the context
4. Reference the context in your config:
```yaml
jobs:
  deploy:
    context: production
```

<a name="config-file"></a>
## 4. Writing the CircleCI Config File

Create a `config.yml` file in your `.circleci` directory. Here's a comprehensive example:

# CircleCI Config File Explanation

## 1. Version Declaration
```yaml
version: 2.1
```
- This specifies we're using CircleCI configuration version 2.1
- Version 2.1 introduced features like orbs, pipelines, and enhanced workflow capabilities

## 2. Commands Section
```yaml
commands:
  install_dependencies:
    description: "Install project dependencies"
    steps:
      - run:
          name: Install dependencies
          command: npm install
```
- `commands`: Defines reusable command sequences
- `install_dependencies`: Custom command name
- `description`: Human-readable description
- `steps`: List of operations to execute
- `run`: Executes shell commands
- `name`: Label shown in CircleCI UI
- `command`: Actual shell command to run

## 3. Jobs Section
```yaml
jobs:
  build:
    docker:
      - image: cimg/node:16.10.0
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Run tests
          command: npm test
```
- `jobs`: Container for all job definitions
- `build`: Job name (can be anything)
- `docker`: Specifies execution environment
- `image`: Docker image to use (cimg/node:16.10.0 is CircleCI's Node image)
- `steps`: Ordered list of job operations
- `checkout`: Special CircleCI step that checks out your code
- `install_dependencies`: References our custom command
- `run`: Executes test command

## 4. Workflows Section
```yaml
workflows:
  version: 2
  build_and_deploy:
    jobs:
      - build:
          filters:
            branches:
              only:
                - main
                - develop
                - /feature-.*/
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: main
```
- `workflows`: Orchestrates job execution
- `version: 2`: Workflow syntax version
- `build_and_deploy`: Workflow name
- `jobs`: List of jobs in workflow
- `filters`: Controls when jobs run
- `branches: only`: Specifies which branches trigger
- `requires`: Sets job dependencies
- `/feature-.*/`: Regex for feature branches

## 5. Branch Filtering
```yaml
filters:
  branches:
    only: main      # Only main branch
    ignore: develop # All except develop
```
- `filters`: Conditionals for job execution
- `branches`: Branch-based filtering
- `only`: Whitelist branches
- `ignore`: Blacklist branches

## 6. Path Filtering
```yaml
filters:
  branches:
    only: main
  tags:
    only: /^v.*/
```
- `tags`: Filter based on git tags
- `/^v.*/`: Regex matching tags starting with 'v'

## 7. Scheduled Workflows
```yaml
triggers:
  - schedule:
      cron: "0 0 * * *"
      filters:
        branches:
          only:
            - main
```
- `triggers`: Defines automatic workflow triggers
- `schedule`: Time-based triggering
- `cron`: Standard cron syntax
- `"0 0 * * *"`: Runs at midnight UTC daily

## 8. Deployment Examples

**AWS S3 Deployment:**
```yaml
- run:
    name: Deploy to S3
    command: |
      aws s3 sync build/ s3://my-bucket --delete
```
- Uses AWS CLI to sync files
- `--delete` removes files in S3 not present locally

**Heroku Deployment:**
```yaml
- run:
    name: Deploy to Heroku
    command: |
      git push https://heroku:$HEROKU_API_KEY@git.heroku.com/my-app.git main
```
- Uses git push to Heroku
- `$HEROKU_API_KEY` is an env var from CircleCI settings

## 9. Advanced Features

**Caching:**
```yaml
- restore_cache:
    keys:
      - v1-dependencies-{{ checksum "package-lock.json" }}
- save_cache:
    paths:
      - node_modules
    key: v1-dependencies-{{ checksum "package-lock.json" }}
```
- `restore_cache`: Tries to find matching cache
- `save_cache`: Creates new cache
- `{{ checksum }}`: Generates key based on file content

**Parallelism:**
```yaml
jobs:
  test:
    parallelism: 4
```
- Runs job across 4 containers
- Splits test files automatically

**Orbs:**
```yaml
orbs:
  node: circleci/node@5.0.2
```
- `orbs`: Reusable configuration packages
- Import community-maintained configurations

## 10. Conditional Steps
```yaml
steps:
  - when:
      condition: << parameters.run_deploy >>
      steps:
        - run: npm run deploy
```
- `when`: Conditional execution
- `<< parameters >>`: References workflow parameters
- Only runs nested steps if condition is true
```

For more information, refer to the [official CircleCI documentation](https://circleci.com/docs/).
