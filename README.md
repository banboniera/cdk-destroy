# Destroy CDK Stack Action

A GitHub composite action that handles AWS CDK stack destruction with validation and detailed reporting.

## Features

- 🗑️ Automated CDK stack destruction
- ⏱️ Destruction timing and duration tracking
- 📊 Detailed destruction summary with JSON outputs
- 🔄 Status reporting and error handling
- ⏲️ Configurable destruction timeout
- 📁 Supports custom working directory

## Usage

```yaml
steps:
  - name: Checkout Code
    uses: actions/checkout@v4

  - name: Setup CDK Environment
    uses: banboniera/setup-cdk@v1
    with:
      aws-region: ${{ vars.AWS_REGION }}
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
      skip-dependencies: "false"

  - name: Synthesize CDK App
    run: npm run synth:staging

  - name: Prepare Artifacts for Destruction
    run: cp package*.json ./cdk.out/

  - name: Upload Synthesized Template
    uses: actions/upload-artifact@v4
    with:
      name: cdk-synth
      path: ./cdk.out
      retention-days: 1

  - name: Destroy CDK Stack
    uses: banboniera/destroy-cdk-stack@v1
    with:
      stack-name: 'my-stack-name'
      aws-region: ${{ vars.AWS_REGION }}
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `stack-name` | The name of the stack to deploy | true | |
| `aws-region` | The AWS region to deploy the stack to | true | |
| `aws-access-key-id` | The AWS access key ID | true | |
| `aws-secret-access-key` | The AWS secret access key | true | |
| `node-version` | The version of Node.js to use | false | `20` |
| `cdk-version` | The version of AWS CDK to install | false | `latest` |
| `working-directory` | Working directory for npm commands | false | `.` |
| `timeout-seconds` | Timeout duration for stack deployment in seconds | false | `1800` |

## Outputs

| Output | Description |
|--------|-------------|
| `destruction-status` | The status of the destruction (success/failure) |

## Destruction Summary

The action provides a detailed destruction summary including:

- Stack name and region
- Destruction status with visual indicators
- Destruction duration in human-readable format

## Error Handling

The action includes comprehensive error handling:

- Timeout monitoring and reporting
- Detailed error messages with exit codes
- Proper status propagation to GitHub Actions

## Example

```yaml
name: CDK Destroy Staging

on:
  workflow_dispatch: # Allows manual triggering of the workflow

env:
  APPLICATION: ${{ vars.APP_NAME }}-Staging

jobs:
  build-synth:
    name: Build and Synthesize
    runs-on: ubuntu-latest
    environment: staging
    concurrency:
      group: build-synth-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    # Global Environment Variables:
    env:
      APP_NAME: ${{ vars.APP_NAME }}
      ENVIRONMENT: staging
      DOMAIN_NAME: ${{ vars.DOMAIN_NAME }}
      VPS_IP: ${{ secrets.VPS_IP }}

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup CDK Environment
        uses: banboniera/setup-cdk@v1
        with:
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          skip-dependencies: "false"

      - name: Synthesize CDK App
        run: npm run synth:staging

      - name: Prepare Destroy Artifacts
        run: cp package*.json ./cdk.out/

      - name: Upload Synthesized Template
        uses: actions/upload-artifact@v4
        with:
          name: cdk-synth
          path: ./cdk.out
          retention-days: 1

  static-site:
    name: Destroy Static Site Stack
    needs: [build-synth]
    runs-on: ubuntu-latest
    concurrency:
      group: static-site-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Destroy Stack
        uses: banboniera/destroy-cdk-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-Static-Site-Stack
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}

  ecr-repositories:
    name: Destroy ECR Repositories Stack
    needs: [static-site]
    runs-on: ubuntu-latest
    concurrency:
      group: ecr-repositories-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Destroy Stack
        uses: banboniera/destroy-cdk-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-ECRRepositories-Stack
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}

  certificate-waf:
    name: Destroy Certificate WAF Stack
    needs: [static-site]
    runs-on: ubuntu-latest
    concurrency:
      group: certificate-waf-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Destroy Stack
        uses: banboniera/destroy-cdk-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-Certificate-Waf-Stack
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}

  zone-public:
    name: Destroy Public Hosted Zone Stack
    needs: [certificate-waf]
    runs-on: ubuntu-latest
    concurrency:
      group: zone-public-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Destroy Stack
        uses: banboniera/destroy-cdk-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-PublicHostedZone-Stack
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
```
