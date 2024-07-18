# AWS S3 Deploy GitHub Action

## Usage

```yaml
      - name: Deploy changes
        uses: goldrushcomputing/s3-deploy@v1
        with:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_BUCKET: ${{ vars.AWS_BUCKET }}
          AWS_REGION: ${{ vars.AWS_REGION }}
          SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
          SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}
```

## Arguments

S3 Deploy's Action supports inputs from the user listed in the table below:

| Input                   | Type   | Required | Description               |
| ----------------------- | ------ | -------- | ------------------------- |
| `AWS_ACCESS_KEY_ID`     | string | Yes      | The AWS_ACCESS_KEY_ID     |
| `AWS_SECRET_ACCESS_KEY` | string | Yes      | The AWS_SECRET_ACCESS_KEY |
| `AWS_BUCKET`            | string | Yes      | The AWS_BUCKET            |
| `AWS_REGION`            | string | Yes      | The AWS_REGION            |
| `SLACK_WEBHOOK_URL`     | string | Yes      | The SLACK_WEBHOOK_URL     |
| `SLACK_CHANNEL`         | string | Yes      | The SLACK_CHANNEL         |

### Example `deploy_staging.yml` with S3 Deploy Action

```yaml
---
# References:
# - https://github.com/marketplace/actions/checkout
# - https://github.com/marketplace/actions/create-env-file

name: Build and deploy

on:
  push:
    branches: master

jobs:
  deploy_staging:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch changes
        uses: actions/checkout@v4

      - name: Build env file
        uses: SpicyPizza/create-envfile@v2.0
        with:
          envkey_NODE_ENV: ${{ vars.STAGING_NODE_ENV }}
          directory: .
          file_name: .env

      - name: Deploy changes
        uses: goldrushcomputing/s3-deploy@v1
        with:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_BUCKET: ${{ vars.AWS_BUCKET }}
          AWS_REGION: ${{ vars.AWS_REGION }}
          SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
          SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}
```
---
### Example `Reuse workflow`

```yaml
---
# References:
# - https://docs.github.com/en/actions/using-workflows/reusing-workflows

jobs:
  build_env_file:
    runs-on: ubuntu-latest
    steps:
      - name: Start Notification
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}
          SLACK_TITLE: "Project: ${{ github.event.repository.name }}"
          SLACK_MESSAGE: ${{ github.workflow }}
          SLACK_COLOR: success
          SLACK_ICON: https://github.com/goldrushcomputing.png?size=48
          SLACK_FOOTER: ''
          SLACK_USERNAME: ${{ github.actor }}
          SLACK_WEBHOOK: ${{ vars.SLACK_WEBHOOK_URL }}

      - name: Make env file
        uses: SpicyPizza/create-envfile@v2.0
        with:
          envkey_NODE_ENV: ${{ vars.STAGING_NODE_ENV }}
          directory: .
          file_name: .env
          fail_on_empty: false
          sort_keys: false

      - name: Upload env_artifact
        uses: actions/upload-artifact@v4
        with:
          name: env_artifact
          path: .env
          if-no-files-found: error

  build:
    uses: goldrushcomputing/s3-deploy/.github/workflows/build.yml@v1
    needs: build_env_file
    with:
      SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
      SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}

  deploy:
    uses: goldrushcomputing/s3-deploy/.github/workflows/deploy.yml@v1
    needs: build
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    with:
      AWS_BUCKET: ${{ vars.AWS_BUCKET }}
      AWS_REGION: ${{ vars.AWS_REGION }}
      SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
      SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}

  notify-slack:
    uses: ./.github/workflows/slack_notification.yml
    needs: [build_env_file, build, deploy]
    if: ${{ always() }}
    with:
      SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
      SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}
      IS_FAILED: ${{ contains(needs.*.result, 'failure') }}
      IS_CANCELLED: ${{ !contains(needs.*.result, 'failure') && (contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')) }}
      IS_SUCCESSFUL: ${{ !(contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')) }}
```
