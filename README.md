# # CircleCI : Setup, Configuration, and Deployment

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

```yaml
version: 2.1

# Define reusable commands
commands:
  install_dependencies:
    description: "Install project dependencies"
    steps:
      - run:
          name: Install dependencies
          command: npm install

# Define reusable jobs
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
      - run:
          name: Build project
          command: npm run build
      - store_artifacts:
          path: build
          destination: build

  deploy:
    docker:
      - image: cimg/node:16.10.0
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Deploy to production
          command: |
            if [ "$CIRCLE_BRANCH" == "main" ]; then
              npm run deploy
            fi

# Define workflows
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

<a name="pipeline-config"></a>
## 5. Pipeline and Branch Configuration

### Branch Filters
Control which branches trigger workflows:
```yaml
workflows:
  my_workflow:
    jobs:
      - build:
          filters:
            branches:
              only: main      # Only main branch
              ignore: develop # All except develop
```

### Path Filtering
Run jobs only when specific files change:
```yaml
workflows:
  my_workflow:
    jobs:
      - build:
          filters:
            branches:
              only: main
            tags:
              only: /^v.*/
```

### Scheduled Workflows
Run workflows on a schedule:
```yaml
workflows:
  nightly:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only:
                - main
    jobs:
      - build
```

<a name="build-deploy"></a>
## 6. Building and Deploying Your Project

### Building
The build job typically includes:
1. Checking out code
2. Installing dependencies
3. Running tests
4. Creating build artifacts

### Deploying
Common deployment strategies:
1. Direct deployment after successful build
2. Manual approval before deployment
3. Conditional deployment based on branch

Example with approval:
```yaml
workflows:
  build_deploy:
    jobs:
      - build
      - hold:
          type: approval
          requires:
            - build
      - deploy:
          requires:
            - hold
```

### Deployment to Common Platforms

**AWS S3:**
```yaml
- run:
    name: Deploy to S3
    command: |
      aws s3 sync build/ s3://my-bucket --delete
```

**Heroku:**
```yaml
- run:
    name: Deploy to Heroku
    command: |
      git push https://heroku:$HEROKU_API_KEY@git.heroku.com/my-app.git main
```

<a name="advanced-tips"></a>
## 7. Advanced Configuration Tips

### Caching
Speed up builds by caching dependencies:
```yaml
steps:
  - restore_cache:
      keys:
        - v1-dependencies-{{ checksum "package-lock.json" }}
        - v1-dependencies-
  - run: npm install
  - save_cache:
      paths:
        - node_modules
      key: v1-dependencies-{{ checksum "package-lock.json" }}
```

### Parallelism
Split tests across multiple containers:
```yaml
jobs:
  test:
    parallelism: 4
    steps:
      - run:
          command: |
            ./test.sh $(circleci tests glob "**/*.spec.js" | circleci tests split --split-by=timings)
```

### Orbs (Reusable Configs)
Use community orbs for common tasks:
```yaml
orbs:
  node: circleci/node@5.0.2
  aws-cli: circleci/aws-cli@2.0.2

jobs:
  build:
    executor: node/default
    steps:
      - node/install-packages
      - aws-cli/setup
```

### Conditional Steps
Run steps based on conditions:
```yaml
steps:
  - when:
      condition: << parameters.run_deploy >>
      steps:
        - run: npm run deploy
```

For more information, refer to the [official CircleCI documentation](https://circleci.com/docs/).
