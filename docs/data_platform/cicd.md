CI/CD process for deploying and managing a data platform is visible in multiple places. Starting from infrastructure, there should be no place for manually created server, network or any other resources, with tools like Terraform or Ansible managing all of it. This part is explained further in [Infrastructure as Code page](iac.md). Second CI/CD layer relates to application part, where all changes within containers running on a cluster should be tested and prepared in an automated way. Final place for CI/CD is within data flows that are going to be executed on orchestrator like Prefect, Apache Airflow or Dagster. This part shouldn't allow manual changes with everything being hosted on repository like GitHub or Gitlab. Here we will focus on typical CI/CD for application part, and also on Data Flow Automation

# Application Deployment Automation <a name="cicd"></a>
In this chapter, we’ll dive into the technical details of setting up a robust CI/CD pipeline using GitHub workflows, Dockerfiles, and Helm charts for deployments on a Kubernetes cluster. By combining these tools, we can streamline the process of building, testing, and deploying applications efficiently. We’ll walk through how GitHub workflows automate tasks, Dockerfiles enable consistent application environments, and Helm charts facilitate scalable Kubernetes deployments—ultimately creating a seamless pipeline from code commit to production-ready deployments. This setup is essential for maintaining agility, reliability, and scalability in modern software development practices.

1. Set up GitHub repository
We should start with creating or configuring a GitHub repository for a project. In our scenario we will use `main` branch that will be protected. Good start for `Branch protection rule` for `main` branch should look like this:
- `Require a pull request before merging` with `Require approvals: 1` should be turned on. With such configuration we are sure that review process is established for repository
- `Require status checks to pass before merging` with `Require branches to be up to date before merging` and CI job we will create in next steps should be added in `Status checks that are required`. This way we will be sure that our automation is passing on latest main code and there is no way to merge any pull request without it. By default GitHub is not forcing workflows to pass before merging.

There are more options worth considering, but they should be applied according to project's needs.

2. Create GitHub CI Workflow - code validation
All GitHub Actions workflows should be stored in `.github/workflows` folder, where we can define multiple workflows. For our CI/CD pipeline we can start with such `on` clause:
```
on:
  pull_request:
    branches: [ main ]
    paths:
      - "path_with_code_changes/**"
  workflow_dispatch:
```
This way workflow will be executed every time any change appears within pull request. There is also possibility to run workflow manually with `workflow_dispatch`, where we can also add additional properties like `debug` in case we need more information about failing workflow.

Inside, we can put basic CI checks, depending on technology we are using, below is simple example for parsing dbt:
```
  validate_dbt_project:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4.2.0
      - uses: actions/setup-python@v5.2.0
        with:
          python-version: "3.12"

      - name: Install dependencies
        env:
          DBT_ENV_SECRET_GITHUB_TOKEN: ${{ secrets.SECRET_GH_TOKEN }}
          PIP_ROOT_USER_ACTION: ignore
        run: pip install -qr path_with_code_changes/requirements.txt --no-build-isolation

      - name: Install dbt package depenedencies
        working-directory: path_with_code_changes
        env:
          DBT_ENV_SECRET_GITHUB_TOKEN: ${{ secrets.SECRET_GH_TOKEN }}
        run: dbt -q deps

      - name: Mount profiles.yml
        # Required by dbt parse.
        # The var is the contents of the profiles.yml, encoded in base64:
        # echo .github/workflows/act/profiles.yml | base64
        env:
          PROFILES_YML_BASE64: ${{ vars.PROFILES_YML_BASE64 }}
        run: echo $PROFILES_YML_BASE64 | base64 --decode > path_with_code_changes/profiles.yml

      - name: Validate the dbt project
        working-directory: path_with_code_changes
        env:
          DBT_ENV_SECRET_GITHUB_TOKEN: ${{ secrets.SECRET_GH_TOKEN }}
        run: dbt -q parse
```

Such workflow can be save within `ci.yml` file, and validate_dbt_project can be added to `Status checks that are required` explained in previous step. The thing that need to be watched out here is the fact, that `ci.yml` might not be executed for every change. To enable mandatory check, it's better to run such CI for every pull request.

3. Docker image build & push GitHub CI workflow

In separate workflow file CD part can be prepared, where basic setup is focused on building and pushing docker image. `on` clause can be set to be executed only on main branch, or we can have separate job that will build docker image on pull request, with `push` being enabled only on main. Better approach is to check build process already on pull_request, that's why below pipeline will be used both for `pull_request` and `push`:
```
name: Build and Push Docker Image

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout the code
        uses: actions/checkout@v4.2.0

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3.6.1

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3.3.0
        with:
          registry: ghcr.io
          username: ${{ secrets.GITHUB_ACTOR }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Docker Image
        id: docker_build
        uses: docker/build-push-action@v6.7.0
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64
          tags: ghcr.io/${{ github.repository }}/my-image:tag
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
```
This way on every change in our pull request, docker image will be build only, and once we are comfortable, after merging changed to `main`, push will happen to GitHub Container Registry (ghcr). In order to use image already on non-prod environment before merging changes, there is an option to modify tagging of the images, where we can use branch name as a tag on pull request, while using proper tagging for pushed changes to main.

4. Helm Chart Workflow

Similarly to docker images, we can handle lint, package and push process for helm charts. In pull request we can use chart-testing (ct) library that will validate correctness of our yml files, and once we are sure that it's working as expected, workflow executed on main branch should package and push helm chart.

In case of reusing Helm chart in multiple projects/repositories, it is good practise to have dedicated repository that will keep helm chart only, and in project repository there should be only helm values file located. It is not a convenient option in highly customized helm charts, where even small change requires upgrading chart. But in most cases only values should be enough to be modified, with bigger changes happening more because of security issues than feature requests.

Workflow for Helm Lint, Package and Push, where lint will be executed on both pull request and push, while helm package and push only on main branch workflows. [CT (chart testing)](https://github.com/helm/chart-testing) is used to validate syntax of helm charts. It can optionally also verify additional things, like incrementing of the helm chart version every time some change was introduced:
```
name: Helm Lint, Package and Push
on:
  push:
    branches:
      - main
    paths:
      - 'charts/**'
  pull_request:
    branches:
      - main
    paths:
      - 'charts/**'

jobs:
  lint-test:
    name: Lint and Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4.2.0
        with:
          fetch-depth: 0

      - name: Set up Helm
        uses: azure/setup-helm@v5
        with:
          version: v3.16.1

      - uses: actions/setup-python@v5.2.0
        with:
          python-version: '3.12'
          check-latest: true

      - name: Add necessary dependencies
        run: |
            #!/bin/bash
            helm repo add ${{ vars.CHART_REGISTRY_NAME }} https://{{ vars.CHART_REGISTRY_URL }} --username '${{ secrets.CHART_REGISTRY_USER }}' --password '${{ secrets.CHART_REGISTRY_PASSWORD }}'

      - name: Set up chart-testing
        uses: helm/chart-testing-action@v2.6.1

      - name: Run chart-testing (lint)
        run: |
          ct lint --target-branch ${{ github.event.repository.default_branch }} --check-version-increment=${{ vars.version_increment }}
```

For helm push we can introduce mechanism that will push only helm charts that are modified. Usage of [GitHub action dorny/paths-filter](https://github.com/dorny/paths-filter) can help with that. It's not mandatory step though, so below code is just assuming that helm package needs to be prepared and pushed. In addition, it is assumed that Chart Museum is used to store all helm charts and [GitHub action helm-push-action](https://github.com/Goodsmileduck/helm-push-action) is used for that:

```
  helm_push:
      name: "$Helm Push"
      runs-on: ubuntu-latest
      steps:
        - name: Checkout
          uses: actions/checkout@v4

        - uses: dyvenia/helm-push-action@master
          env:
            HELM_EXPERIMENTAL_OCI: 1
            SOURCE_DIR: './charts'
            CHARTMUSEUM_REPO_NAME: ${{ vars.REPOSITORY_NAME }}
            CHART_FOLDER: ${{ matrix.directory }}
            CHARTMUSEUM_URL: '${{ vars.CHART_REGISTRY_URL }}'
            CHARTMUSEUM_USER: '${{ secrets.CHART_REGISTRY_USER }}'
            CHARTMUSEUM_PASSWORD: ${{ secrets.CHART_REGISTRY_PASSWORD }}
```

5. CD preparation

Once there is docker image and helm chart with values prepared, it's good to prepare CD workflow that will deploy application into kubernetes cluster. We are assuming that whole environment is not accessible from the internet in any other way than via VPN, so our GitHub runner should be located on a virtual machine with access to it, ideally within same VPC or connected through VPN to environment. Additionally, such virtual machine should have configured access to Kubernetes cluster through proper RBAC policy or any other option that will limit access to the cluster only to deployments.

Once such GitHub runner is configured, it is possible to prepare additional workflow that will handle deployment of the application to DEV and PROD environments. In below example additionally [SOPS](https://github.com/getsops/sops) was configured that is enabling keeping secrets on repository encrypted. To decrypt secrets.yaml files, it is necessary to use eg. AWS KMS, that's why access key and token are provided in last step:
```
name: Deploy
on:
  push:
    branches:
      - main
    paths:
      - 'etc/helm_values/{{ application_name }}/**'
  workflow_dispatch:

jobs:
  dev_deploy:
    runs-on: dev-runner

    steps:
    - name: Checkout code
      uses: actions/checkout@v4.1.7

    - name: Add necessary dependencies
      run: |
        helm repo add ${{ vars.CHART_REGISTRY_NAME }} ${{ vars.CHART_REGISTRY_URL }} --username '${{ vars.CHART_REGISTRY_USER }}' --password '${{ vars.CHART_REGISTRY_PASSWORD }}'
        helm repo update ${{ vars.CHART_REGISTRY_NAME }}

    - name: Run Helm upgrade commands
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        helm secrets upgrade {{ application_name }} --install ${{ vars.CHART_REGISTRY_NAME }}/{{ chart_name }} \
          -n {{ namespace }} \
          -f etc/helm_values/{{ application_name }}/secrets-dev.yaml \
          -f etc/helm_values/{{ application_name }}/values-dev.yaml
```

Once dev deployment will be successful, there can be manual approval process implemented. Below example requires GitHub Enterprise licence, but there are free alternatives on the market.

```
jobs:
  prod_deploy_approval:
    runs-on: ubuntu-latest
    needs: [ dev_deploy ]
    environment: prod_approve
    steps:
      - name: "Prod deployment approval"
        run: |
          echo "Approving deployment to PROD"
```

In addition, environment needs to be prepared in repository settings. `Settings > Environments > New environment`. After providing environment name (`prod_approve` in above case), please make sure that `Required reviewers` checkbox is enabled and there are people/teams provided. Once it is configured, only manual approval from listed people/teams will enable prod deployment. One downside of such approach is the situation when we are not releasing particular version to production. In such situation workflow will stay in `Waiting` state for 30 days, and after that people from the list will receive notification that this workflow has timed-out.

Code for production deployment should look almost identical to dev_deploy job above. The only difference should be with secrets and values files, and also dedicated production GitHub runner should be used.

With these configuration we managed to prepare basic CI/CD workflow for our application. Improvements can we introduce:
- introduce GitOps approach using ArgoCD or similar tool
- extend testing phase during CI process
- introduce advanced alerting in case of an issue during deployment. Currently only person involved in the process will receive notification from GitHub
- prepare workflow for rollback in case of issues with new version
- add more CI checks, like verification of hardcoded secrets in repository (it can be handled with GitHub's Code scanning feature) or additional lint script for aligning with company's coding standard
- release job that will show on GitHub recent version prepared
- further use of GitHub's Environment feature. By using it, there is possibility to check history of deployments directly from GitHub

# Data Flow Automation <a name="flow"></a>
TBD