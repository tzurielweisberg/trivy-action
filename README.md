# Trivy Action

> [GitHub Action](https://github.com/features/actions) for [Trivy](https://github.com/aquasecurity/trivy)

[![GitHub Release][release-img]][release]
[![GitHub Marketplace][marketplace-img]][marketplace]
[![License][license-img]][license]

![](docs/images/trivy-action.png)

## Table of Contents

- [Usage](#usage)
  - [Workflow](#workflow)
  - [Docker Image Scanning](#using-trivy-with-github-code-scanning)
  - [Git Repository Scanning](#using-trivy-to-scan-your-git-repo)
- [Customizing](#customizing)
  - [Inputs](#inputs)

## Usage

### Scan CI Pipeline

```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build an image from Dockerfile
        run: |
          docker build -t docker.io/my-organization/my-app:${{ github.sha }} .
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

### Scan CI Pipeline (w/ Trivy Config)

```yaml
name: build
on:
  push:
    branches:
    - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-20.04
    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Run Trivy vulnerability scanner in fs mode
      uses: aquasecurity/trivy-action@add-support-for-trivy-config
      with:
        scan-type: 'fs'
        scan-ref: '.'
        trivy-config: ./trivy.yaml
```

In this case `trivy.yaml` is a YAML configuration that is checked in as part of the repo. Detailed information is available on the Trivy website but an example is as follows:
```yaml
format: json
exit-code: 1
severity: CRITICAL
```

It is possible to define all options in the `trivy.yaml` file. Specifying individual options via the action are left for backward compatibility purposes. Defining the following is required as they cannot be defined with the config file:
- `scan-ref`: If using `fs, repo` scans.
- `image-ref`: If using `image` scan.
- `scan-type`: To define the scan type, e.g. `image`, `fs`, `repo`, etc.

#### Order of prerference for options
Trivy uses [Viper](https://github.com/spf13/viper) which has a defined precedence order for options. The order is as follows:
- GitHub Action flag
- Environment variable
- Config file
- Default

### Scanning a Tarball
```yaml
name: build
on:
  push:
    branches:
    - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-20.04
    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Generate tarball from image
      run: |
        docker pull <your-docker-image>
        docker save -o vuln-image.tar <your-docker-image>
        
    - name: Run Trivy vulnerability scanner in tarball mode
      uses: aquasecurity/trivy-action@master
      with:
        input: /github/workspace/vuln-image.tar
        severity: 'CRITICAL,HIGH'
```

### Using Trivy with GitHub Code Scanning
If you have [GitHub code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning) available you can use Trivy as a scanning tool as follows:
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build an image from Dockerfile
        run: |
          docker build -t docker.io/my-organization/my-app:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

You can find a more in-depth example here: https://github.com/aquasecurity/trivy-sarif-demo/blob/master/.github/workflows/scan.yml

If you would like to upload SARIF results to GitHub Code scanning even upon a non zero exit code from Trivy Scan, you can add the following to your upload step:
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Build an image from Dockerfile
        run: |
          docker build -t docker.io/my-organization/my-app:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

See this for more details: https://docs.github.com/en/actions/learn-github-actions/expressions#always

### Using Trivy to scan your Git repo
It's also possible to scan your git repos with Trivy's built-in repo scan. This can be handy if you want to run Trivy as a build time check on each PR that gets opened in your repo. This helps you identify potential vulnerablites that might get introduced with each PR.

If you have [GitHub code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning) available you can use Trivy as a scanning tool as follows:
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner in repo mode
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Using Trivy to scan your rootfs directories
It's also possible to scan your rootfs directories with Trivy's built-in rootfs scan. This can be handy if you want to run Trivy as a build time check on each PR that gets opened in your repo. This helps you identify potential vulnerablites that might get introduced with each PR.

If you have [GitHub code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning) available you can use Trivy as a scanning tool as follows:
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner with rootfs command
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'rootfs'
          scan-ref: 'rootfs-example-binary'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Using Trivy to scan Infrastucture as Code
It's also possible to scan your IaC repos with Trivy's built-in repo scan. This can be handy if you want to run Trivy as a build time check on each PR that gets opened in your repo. This helps you identify potential vulnerablites that might get introduced with each PR.

If you have [GitHub code scanning](https://docs.github.com/en/github/finding-security-vulnerabilities-and-errors-in-your-code/about-code-scanning) available you can use Trivy as a scanning tool as follows:
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner in IaC mode
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          hide-progress: false
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Using Trivy to generate SBOM
It's possible for Trivy to generate an SBOM of your dependencies and submit them to a consumer like GitHub Dependency Snapshot.

The sending of SBOM to GitHub feature is only available if you currently have [GitHub Dependency Snapshot](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/using-the-dependency-submission-api) available to you in your repo. 

In order to send results to the GitHub Dependency Snapshot, you will need to create a [GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
```yaml
---
name: Pull Request
on:
  push:
    branches:
    - master
  pull_request:
jobs:
  build:
    name: Checks
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Trivy in GitHub SBOM mode and submit results to Dependency Snapshots
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          format: 'github'
          output: 'dependency-results.sbom.json'
          image-ref: '.'
          github-pat: '<github_pat_token>'
```

### Using Trivy to scan your private registry
It's also possible to scan your private registry with Trivy's built-in image scan. All you have to do is set ENV vars.

#### Docker Hub registry
Docker Hub needs `TRIVY_USERNAME` and `TRIVY_PASSWORD`.
You don't need to set ENV vars when downloading from a public repository.
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
        env:
          TRIVY_USERNAME: Username
          TRIVY_PASSWORD: Password

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

#### AWS ECR (Elastic Container Registry)
Trivy uses AWS SDK. You don't need to install `aws` CLI tool.
You can use [AWS CLI's ENV Vars][env-var].

[env-var]: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'aws_account_id.dkr.ecr.region.amazonaws.com/imageName:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
        env:
          AWS_ACCESS_KEY_ID: key_id
          AWS_SECRET_ACCESS_KEY: access_key
          AWS_DEFAULT_REGION: us-west-2

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

#### GCR (Google Container Registry)
Trivy uses Google Cloud SDK. You don't need to install `gcloud` command.

If you want to use target project's repository, you can set it via `GOOGLE_APPLICATION_CREDENTIAL`.
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
        env:
          GOOGLE_APPLICATION_CREDENTIAL: /path/to/credential.json

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

#### Self-Hosted
BasicAuth server needs `TRIVY_USERNAME` and `TRIVY_PASSWORD`.
if you want to use 80 port, use NonSSL `TRIVY_NON_SSL=true`
```yaml
name: build
on:
  push:
    branches:
      - master
  pull_request:
jobs:
  build:
    name: Build
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
        env:
          TRIVY_USERNAME: Username
          TRIVY_PASSWORD: Password

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

## Customizing

### inputs

Following inputs can be used as `step.with` keys:

| Name              | Type    | Default                            | Description                                                                                     |
|-------------------|---------|------------------------------------|-------------------------------------------------------------------------------------------------|
| `scan-type`       | String  | `image`                            | Scan type, e.g. `image` or `fs`                                                                 |
| `input`           | String  |                                    | Tar reference, e.g. `alpine-latest.tar`                                                         |
| `image-ref`       | String  |                                    | Image reference, e.g. `alpine:3.10.2`                                                           |
| `scan-ref`        | String  | `/github/workspace/`               | Scan reference, e.g. `/github/workspace/` or `.`                                                |
| `format`          | String  | `table`                            | Output format (`table`, `json`, `sarif`, `github`)                                              |
| `template`        | String  |                                    | Output template (`@/contrib/gitlab.tpl`, `@/contrib/junit.tpl`)                                 |
| `output`          | String  |                                    | Save results to a file                                                                          |
| `exit-code`       | String  | `0`                                | Exit code when specified vulnerabilities are found                                              |
| `ignore-unfixed`  | Boolean | false                              | Ignore unpatched/unfixed vulnerabilities                                                        |
| `vuln-type`       | String  | `os,library`                       | Vulnerability types (os,library)                                                                |
| `severity`        | String  | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` | Severities of vulnerabilities to scanned for and displayed                                      |
| `skip-dirs`       | String  |                                    | Comma separated list of directories where traversal is skipped                                  |
| `skip-files`      | String  |                                    | Comma separated list of files where traversal is skipped                                        |
| `cache-dir`       | String  |                                    | Cache directory                                                                                 |
| `timeout`         | String  | `5m0s`                             | Scan timeout duration                                                                           |
| `ignore-policy`   | String  |                                    | Filter vulnerabilities with OPA rego language                                                   |
| `hide-progress`   | String  | `true`                             | Suppress progress bar                                                                           |
| `list-all-pkgs`   | String  |                                    | Output all packages regardless of vulnerability                                                 |
| `security-checks` | String  | `vuln,secret`                      | comma-separated list of what security issues to detect (`vuln`,`secret`,`config`)               |
| `trivyignores`    | String  |                                    | comma-separated list of relative paths in repository to one or more `.trivyignore` files        |
| `github-pat`      | String  |                                    | GitHub Personal Access Token (PAT) for sending SBOM scan results to GitHub Dependency Snapshots |

[release]: https://github.com/aquasecurity/trivy-action/releases/latest
[release-img]: https://img.shields.io/github/release/aquasecurity/trivy-action.svg?logo=github
[marketplace]: https://github.com/marketplace/actions/aqua-security-trivy
[marketplace-img]: https://img.shields.io/badge/marketplace-trivy--action-blue?logo=github
[license]: https://github.com/aquasecurity/trivy-action/blob/master/LICENSE
[license-img]: https://img.shields.io/github/license/aquasecurity/trivy-action
