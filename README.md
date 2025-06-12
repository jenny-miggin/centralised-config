# Centralised CircleCI Configuration System

This repository provides as centralied, reusable CircleCI configuration system that allows teams to share common CI/CD patterns while maintaining flexibility for team-specific customisations.

## 🏗️ Architecture Overview

This system uses **Dynamic Orb Loading** and **Job Override Patterns** to create a flexible, centralised CI/CD configuration that can be consumed by multiple projects and teams.

### Core Concept: Template + Override Pattern

1. **Central Template** (`central-config.yml`): Defines the standard workflow structure
2. **Team-Specific Configs**: Provide actual job implementations that override placeholder jobs
3. **Dynamic Loading**: Teams specify which configuration to use via URL parameters

## 📁 Repository Structure

```
centralised-config/
├── .circleci/
│   ├── central-config.yml          # Main template configuration
│   ├── frontend-team/
│   │   └── frontend-team.yml       # Frontend-specific jobs
│   ├── backend-team/
│   │   └── backend-team.yml        # Backend-specific jobs
│   ├── data-team/
│   │   └── data-team.yml           # Data team-specific jobs
│   ├── projects/
│   │   └── project.yml             # Project-specific overrides
│   ├── libs/
│   │   └── libs.yml                # Library build configurations
│   └── deploys/
│       └── deploys.yml             # Deployment-specific jobs
└── README.md
```

## 🎯 How It Works

### 1. Central Template (`central-config.yml`)

The core template defines:

**Parameters**:
- `orb`: URL to the team-specific configuration file
- `override-build-job`: Name of the build job to use
- `override-deploy-job`: Name of the deploy job to use

**Placeholder Jobs**:
- `build`: No-op placeholder (gets overridden)
- `test`: Standard test job (shared across teams)
- `deploy`: No-op placeholder (gets overridden)

**Standard Workflow**:
```yaml
workflows:
  build-and-test-and-deploy:
    jobs:
      - build:                                    # Overridden by team config
          name: << pipeline.parameters.override-build-job >>
          override-with: selected-orb/<< pipeline.parameters.override-build-job >>
      - test:                                     # Shared test job
          requires: [<< pipeline.parameters.override-build-job >>]
      - deploy:                                   # Overridden by team config
          name: << pipeline.parameters.override-deploy-job >>
          requires: [test]
          override-with: selected-orb/<< pipeline.parameters.override-deploy-job >>
```

### 2. Team-Specific Configurations

Each team directory contains job definitions that replace the no-op placeholders:

**Frontend Team** (`frontend-team/frontend-team.yml`):
```yaml
jobs:
  frontend-build:    # Replaces central build job
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run: echo "Build my frontend app"
      
  frontend-deploy:   # Replaces central deploy job
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run: echo "Deploy my frontend app" && sleep 300
```

**Backend Team** (`backend-team/backend-team.yml`):
```yaml
jobs:
  backend-build:     # Replaces central build job
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run: echo "Build my backend app"
```

## 🚀 Usage Examples

### For Consumer Projects

Projects can use this centralised configuration by specifying parameters. An example of this can be seen in [this example](https://github.com/jenny-miggin/my-demo-monorepo/blob/main/.circleci/continue_config.yml) 

### Via Pipeline Triggers

Projects can trigger this configuration dynamically using the [CircleCI API v2 Pipeline endpoint](https://circleci.com/docs/api/v2/index.html#tag/Pipeline/operation/triggerPipelineRun):

```bash
curl -X POST https://circleci.com/api/v2/project/gh/jenny-miggin/centralised-config/pipeline/run \
  --header "Circle-Token: $CIRCLECI_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "definition_id": "YOUR_PIPELINE_DEFINITION_ID",
    "parameters": {
      "orb": "https://raw.githubusercontent.com/.../frontend-team/frontend-team.yml",
      "override-build-job": "frontend-build",
      "override-deploy-job": "frontend-deploy"
    }
  }'
```

#### Setting up Pipeline Definitions

The Pipeline can be defined within Project Settings > Pipelines, but can also be programatically created:

**Step 1: Create a Pipeline Definition**

First, create a pipeline definition using the CircleCI API:

```bash
curl -X POST https://circleci.com/api/v2/project/gh/jenny-miggin/centralised-config/pipeline-definition \
  --header "Circle-Token: $CIRCLECI_API_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "name": "central-config-pipeline",
    "description": "Centralised configuration pipeline for team builds",
    "configuration_path": ".circleci/central-config.yml"
  }'
```

**Step 2: Get Your Definition ID**

List your pipeline definitions to get the definition ID:

```bash
curl -X GET https://circleci.com/api/v2/project/gh/jenny-miggin/centralised-config/pipeline-definition \
  --header "Circle-Token: $CIRCLECI_API_TOKEN"
```

Example response:
```json
{
  "items": [
    {
      "id": "abc123def-456-789-ghi-jklmnop",
      "name": "central-config-pipeline",
      "description": "Centralised configuration pipeline for team builds",
      "configuration_path": ".circleci/central-config.yml",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**Step 3: Use Definition ID in Triggers**

Copy the `id` field from the response above and use it as your `definition_id` in pipeline triggers.

## 🎛️ Available Configurations

### Team Configurations

| Team | Config URL | Build Job | Deploy Job |
|------|------------|-----------|------------|
| Frontend | `.../frontend-team/frontend-team.yml` | `frontend-build` | `frontend-deploy` |
| Backend | `.../backend-team/backend-team.yml` | `backend-build` | N/A |
| Data | `.../data-team/data-team.yml` | `data-build` | `data-deploy` |
| Libraries | `.../libs/libs.yml` | `libs-build` | `libs-deploy` |

### Specialized Configurations

- **Projects**: Project-specific overrides and customizations
- **Deploys**: Deployment-focused configurations
- **Libraries**: Shared library build patterns

## 🏆 Benefits

### ✅ **Centralised Management**
- **Single Source of Truth**: All CI/CD patterns managed in one place
- **Easy Updates**: Change once, apply everywhere
- **Consistency**: Enforced standards across all teams

### ✅ **Team Flexibility**
- **Customisable**: Teams can define their own build/deploy jobs
- **Override Pattern**: Keep common structure, customise specifics
- **Scalable**: Easy to add new team configurations

### ✅ **Maintenance Efficiency**
- **DRY Principle**: No duplicate configuration across projects
- **Version Control**: Centralized configuration versioning
- **Testing**: Test configuration changes in one place