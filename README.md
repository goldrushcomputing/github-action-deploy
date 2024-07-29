# Deploy use GitHub Action

### Example `Reuse workflow to Deploy into AWS S3`

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
    uses: goldrushcomputing/github_action_deploy/.github/workflows/s3_build.yml@v1.0.1
    needs: build_env_file

  deploy:
    uses: goldrushcomputing/github_action_deploy/.github/workflows/s3_deploy.yml@v1.0.1
    needs: build
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    with:
      AWS_BUCKET: ${{ vars.AWS_BUCKET }}
      AWS_REGION: ${{ vars.AWS_REGION }}

  notify-slack:
    uses: goldrushcomputing/github_action_deploy/.github/workflows/slack_notification.yml@v1.0.1
    needs: [build_env_file, build, deploy]
    if: ${{ always() }}
    with:
      SLACK_WEBHOOK_URL: ${{ vars.SLACK_WEBHOOK_URL }}
      SLACK_CHANNEL: ${{ vars.SLACK_CHANNEL }}
      IS_FAILED: ${{ contains(needs.*.result, 'failure') }}
      IS_CANCELLED: ${{ !contains(needs.*.result, 'failure') && (contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')) }}
      IS_SUCCESSFUL: ${{ !(contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')) }}
```

---
### Example `Reuse workflow to Deploy into AWS EC2`

```
jobs:
  build_env_file:
    runs-on: ubuntu-latest
    outputs:
      WORKSPACE: ${{ vars.WORKSPACE }}
      CORE_DIRECTORY: ${{ vars.CORE_DIRECTORY }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Copy ${{ vars.ENV_DEPLOY }} file to ${{ vars.ENVIRONMENT }}_a
      uses: appleboy/scp-action@v0.1.7
      with:
        host: ${{ secrets.BACKEND_HOST }}
        username: ${{ secrets.BACKEND_USERNAME }}
        key: ${{ secrets.BACKEND_PRIVATE_KEY }}
        source: ${{ vars.ENV_DEPLOY }}
        target: ${{ vars.WORKSPACE }}/${{ vars.CORE_DIRECTORY }}

  build_and_deploy_staging_a:
    uses: goldrushcomputing/github_action_deploy/.github/workflows/ec2_build_and_run_backend_server.yml@v1.0.1
    needs: build_env_file
    secrets:
      BACKEND_HOST: ${{ secrets.BACKEND_HOST }}
      BACKEND_USERNAME: ${{ secrets.BACKEND_USERNAME }}
      BACKEND_PRIVATE_KEY: ${{ secrets.BACKEND_PRIVATE_KEY }}
    with:
      BRANCH_NAME: ${{ vars.BRANCH_NAME }}
      IMAGE_NAME: ${{ vars.IMAGE_NAME }}
      CONTAINER_NAME: ${{ vars.CONTAINER_NAME }}
      DOCKER_NETWORK_NAME: ${{ vars.DOCKER_NETWORK_NAME }}
      WORKSPACE: ${{ vars.WORKSPACE }}
      CORE_DIRECTORY: ${{ vars.CORE_DIRECTORY }}
      NODE_USER_ID: ${{ vars.NODE_USER_ID }}
      NODE_GROUP_ID: ${{ vars.NODE_GROUP_ID }}
      NODE_PORT: ${{ vars.NODE_PORT }}
```
