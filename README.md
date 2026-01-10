# ECS Task Run

[![Test](https://github.com/ikhrustalev/ecs-task-run/actions/workflows/test_action.yml/badge.svg)](https://github.com/ikhrustalev/ecs-task-run/actions/workflows/test_action.yml)
[![Release](https://img.shields.io/github/v/release/ikhrustalev/ecs-task-run)](https://github.com/ikhrustalev/ecs-task-run/releases)
[![License](https://img.shields.io/github/license/ikhrustalev/ecs-task-run)](LICENSE)

A GitHub Action that runs one-off ECS tasks using configuration from an existing service. Perfect for migrations, scripts, health checks, and ad-hoc commands.

## Why This Action?

Running an ECS task normally requires specifying task definition, network configuration, launch type, and more. This action copies all that from an existing service — just provide the command.

| Use Case            | Example                                   |
| ------------------- | ----------------------------------------- |
| Database migrations | `command: "python manage.py migrate"`     |
| One-time scripts    | `command: "python cleanup.py"`            |
| Smoke tests in prod | `command: "curl http://localhost/health"` |
| Debug commands      | `command: "printenv"`                     |

## Features

- **Zero config** - inherits task definition, networking, and launch type from service
- **Log streaming** - real-time CloudWatch logs in GitHub Actions output
- **Timeout support** - automatically stops runaway tasks
- **Exit code handling** - fails the workflow if task fails

## Inputs

| Input          | Description                                            | Required | Default |
| -------------- | ------------------------------------------------------ | -------- | ------- |
| `cluster_arn`  | ARN of the ECS cluster                                 | Yes      | —       |
| `service_name` | Name of the ECS service to copy configuration from     | Yes      | —       |
| `command`      | Command to run in the container                        | Yes      | —       |
| `wait`         | Wait for task completion                               | No       | `true`  |
| `stream_logs`  | Stream CloudWatch logs while task runs                 | No       | `true`  |
| `shell`        | Wrap command in shell (`true`, `false`, or shell path) | No       | `true`  |
| `timeout`      | Timeout in seconds (0 = no timeout)                    | No       | `0`     |

## Outputs

| Output      | Description                                    |
| ----------- | ---------------------------------------------- |
| `task_arn`  | ARN of the started task                        |
| `task_id`   | ID of the started task                         |
| `exit_code` | Exit code of the task (only when `wait: true`) |

## Prerequisites

Configure AWS credentials before using this action:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/my-role
    aws-region: us-east-1
```

Required IAM permissions:

- `ecs:DescribeServices`
- `ecs:DescribeTaskDefinition`
- `ecs:RunTask`
- `ecs:DescribeTasks` (if `wait: true`)
- `ecs:StopTask` (if `timeout` > 0)
- `logs:DescribeLogStreams` (if `stream_logs: true`)
- `logs:GetLogEvents` (if `stream_logs: true`)

## Examples

### Database Migration

```yaml
- name: Run migration
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: api-service
    command: python manage.py migrate
```

### Python Script

```yaml
- name: Run cleanup script
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: worker-service
    command: python scripts/cleanup.py --dry-run
```

### Multi-line Command

```yaml
- name: Run setup tasks
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: api-service
    command: |
      echo "Starting migration..."
      rails db:migrate
      echo "Seeding data..."
      rails db:seed
```

### With Timeout

```yaml
- name: Run with 5 minute timeout
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: api-service
    command: python long_running_task.py
    timeout: 300
```

### Fire and Forget

```yaml
- name: Trigger async job
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: worker-service
    command: python process_queue.py
    wait: false
```

### Without Shell Wrapper

By default, commands are wrapped in `/bin/sh -c "..."`. Disable this for direct execution:

```yaml
- name: Run binary directly
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: api-service
    command: /app/bin/migrate
    shell: false
```

### Custom Shell

```yaml
- name: Run with bash
  uses: ikhrustalev/ecs-task-run@v1
  with:
    cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
    service_name: api-service
    command: echo $BASH_VERSION
    shell: /bin/bash
```

### Full Pipeline Example

```yaml
name: Deploy with Migration

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: us-east-1

      - name: Run database migration
        uses: ikhrustalev/ecs-task-run@v1
        with:
          cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
          service_name: api-service
          command: python manage.py migrate
          timeout: 300

      - name: Deploy new version
        uses: ikhrustalev/ecs-service-deploy@v1
        with:
          cluster_arn: arn:aws:ecs:us-east-1:123456789012:cluster/prod
          service_name: api-service
```

## How It Works

1. Reads configuration from the specified ECS service (task definition, network config, launch type)
2. Runs a new task with your command override
3. Streams CloudWatch logs to GitHub Actions output
4. Waits for completion and returns exit code

## Related

- [ecs-service-deploy](https://github.com/ikhrustalev/ecs-service-deploy) — Lightweight ECS service redeployment

## License

MIT
