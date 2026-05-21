---
layout: post
title:  "Building an Image Factory"
tags: automation aws packer infra
---

Sorry I've been quiet lately. My head has been down on my newest adventure. I'm so used to being the sole operator, platform engineer, SRE, or whatever that day brings that it's odd to take a step back and be tasked with providing enterprise cybersecurity for cloud environments that other teams are operating. I've had so many cool new projects that will make for some great technical blogs, so I figured I would start with this one. The idea is simple: how do you provide your organization with hardened operating systems that teams can actually deploy into the cloud? A lot of compliance terms and frameworks get tossed around, but the vision is this: how do you provide an image factory of CIS-hardened AMIs, bake in a custom baseline of tools, and share those images across numerous AWS accounts and organizations?

Here is my approach, the downfalls, the unknowns, and the fun parts. I apologize in advance that this is very GitLab centric CI/CD, but if you like the design, feel free to port it over to your source control system of choice.

## Scaffolding

The repository starts with the boring stuff first, because the boring stuff is what keeps the project usable after the first week. Besides `.gitignore` and `.gitattributes`, there is a short `README.md`, a `Brewfile` for local tooling, an `.editorconfig`, a `SECURITY.md`, and the usual `.gitlab/merge_request_templates` and `.gitlab/issue_templates` so reviews and issues don't turn into archaeology.

A small `Makefile` covers common local commands, but `.pre-commit-config.yaml` does most of the early heavy lifting. It gives the repo one place for file hygiene, formatting checks, and basic guardrails before anything gets near CI. Here's the first pass of hooks for the Image Factory.

```yaml
# See https://pre-commit.com for more information
# See https://pre-commit.com/hooks.html for more hooks
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: trailing-whitespace
        args: [--markdown-linebreak-ext=md]
      - id: check-shebang-scripts-are-executable

      # YAML
      - id: check-yaml

      # Cross platform
      - id: check-case-conflict
      - id: mixed-line-ending
        args: [--fix=lf]

  - repo: local
    hooks:
      - id: packer-fmt
        name: Packer format check
        entry: packer fmt -check -recursive packer
        language: system
        files: ^packer/.*\.pkr\.hcl$
        pass_filenames: false

      - id: shellcheck
        name: ShellCheck
        entry: shellcheck
        language: system
        types: [shell]
```

That runs on every commit and removes a lot of pointless review noise. If Packer formatting or shell linting is broken, I want the hook to catch it before a reviewer has to.

> Side note, I typically have a dedicated `pre-commit` CI job for making sure everything passes as a prerequisite to other pipelines.

The repo also has a `docs` directory for the usual odds and ends: architecture notes, decision records, and diagrams.

## AWS Prerequisites

Before the repo can build anything useful, AWS needs a few pieces in place. If the AMIs use encrypted EBS volumes, the KMS key policy has to let the build account use the key and let consumer accounts launch from the shared AMIs. You also need the normal network plumbing: VPC, subnets, routing, security groups, and outbound access so temporary build and test instances can pull updates, download packages, reach SSM, and install whatever baseline tooling your organization requires.

An optional AMI reaper Lambda is worth adding early. Failed builds, superseded images, and half-finished experiments shouldn't live forever. If the pipeline tags images during build, test, and publish, cleanup can be driven from those tags instead of guessing (thank you boto3).

The last prerequisite is identity. Packer, Terratest, and publishing should each have an IAM role, and CI should use OIDC to assume those roles. Long-lived AWS keys in CI variables are one of those things that feel convenient right up until they become an incident, and they make me feel like I need a shower if I have to use them.

## Codebase Structure

The layout is intentionally boring. Each image gets its own Packer root under `packer/images/<provider>/<image>`, and each root owns the same four files: `versions.pkr.hcl`, `variables.pkr.hcl`, `sources.pkr.hcl`, and `build.pkr.hcl`. When someone adds another operating system, the plugin versions, inputs, AMI lookup logic, and hardening steps all have a known place to live.

```console
├── account-map.yaml
├── Brewfile
├── docs
├── Makefile
├── packer
│   ├── images
│   │   └── aws
│   │       ├── al2023
│   │       │   ├── build.pkr.hcl
│   │       │   ├── sources.pkr.hcl
│   │       │   ├── variables.pkr.hcl
│   │       │   └── versions.pkr.hcl
│   │       └── ubuntu24.04
│   │           ├── build.pkr.hcl
│   │           ├── sources.pkr.hcl
│   │           ├── variables.pkr.hcl
│   │           └── versions.pkr.hcl
├── README.md
├── scripts
│   ├── build.sh
│   └── gitlab
│       └── detect-packer-changes.sh
├── SECURITY.md
└── tests
    ├── terraform
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── providers.tf
    │   ├── README.md
    │   ├── variables.tf
    │   └── versions.tf
    └── terratest
        ├── build_test.go
        ├── checks
        │   ├── al2023-cis-level1.yaml
        │   └── ubuntu24.04-cis-level1.yaml
        ├── go.mod
        └── go.sum
```

The other important root-level file is `account-map.yaml`. It lists the consumer AWS accounts that should receive launch permissions after an AMI passes testing. I prefer keeping that as data in the repo instead of hiding it in CI variables. A merge request should show exactly who is being added or removed from the distribution list.

## Building the Image

The build wrapper stays small. It takes a Packer environment through `PKR_ENV`, initializes that image root, checks the template, and writes the final manifest into `artifacts/`. The local command and the CI command are the same thing, which makes build failures much easier to reproduce.

```sh
#!/usr/bin/env bash

set -euo pipefail

: "${PKR_ENV:? PKR_ENV is required}"
if ! command -v "packer" &>/dev/null; then
  echo "Error: Packer is not installed."
  exit 1
fi

echo "Initializing Packer environment in $PKR_ENV"
packer init "$PKR_ENV"

echo "Checking Packer configuration formatting..."
packer fmt -check "$PKR_ENV"

echo "Validating Packer configurations..."
packer validate "$PKR_ENV"

echo "Building Packer image..."
mkdir -p artifacts
packer build -on-error=cleanup "$PKR_ENV"
```

For AL2023, the Packer source starts by finding the base AMI, creating a timestamped name, and tagging the AMI and snapshots with enough metadata to make cleanup and audit work sane. Tags like `ImageFactoryManaged`, `ImageFactoryPublished`, `BaseImageProduct`, and `SourceAmi` give you a quick answer to what created the image, what it was based on, and whether it has been released.

```hcl
locals {
  build_timestamp = regex_replace(timestamp(), "[- TZ:]", "")
  cis_product     = "CIS Hardened Image Level 1 on Amazon Linux 2023"

  ami_name = var.ami_name != "" ? var.ami_name : format(
    "%s-%s-%s",
    var.ami_name_prefix,
    var.cis_marketplace_version,
    local.build_timestamp,
  )

  common_tags = merge(
    {
      Name                  = local.ami_name
      BaseImageProduct      = local.cis_product
      BaseImageVersion      = var.cis_marketplace_version
      CisBenchmarkLevel     = "1"
      ImageFactoryManaged   = "true"
      ImageFactoryPublished = "false"
      SourceAmi             = data.amazon-ami.this.id
    },
    var.tags,
  )
}

data "amazon-ami" "this" {
  region      = var.aws_region
  owners      = var.source_ami_owners
  most_recent = var.source_ami_most_recent

  filters = merge(
    {
      architecture        = var.source_ami_architecture
      name                = var.source_ami_name_filter
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    },
    var.source_ami_product_code != "" ? { product-code = var.source_ami_product_code } : {},
  )
}
```

The `amazon-ebs` source uses SSM Session Manager as the communicator. It sounds like a small choice, but it changes the operating model quite a bit. I don't need to punch SSH ingress into a build subnet, pass key pairs around, or explain why a temporary builder was reachable from the internet. The instance gets temporary SSM permissions, Packer connects through Session Manager, and the build network can stay private.

```hcl
source "amazon-ebs" "al2023" {
  ami_description             = var.ami_description
  ami_name                    = local.ami_name
  ami_regions                 = var.ami_regions
  associate_public_ip_address = var.associate_public_ip_address
  encrypt_boot                = var.encrypt_boot
  instance_type               = var.instance_type
  kms_key_id                  = var.kms_key_id
  region                      = var.aws_region
  source_ami                  = data.amazon-ami.this.id

  communicator     = "ssh"
  pause_before_ssm = "30s"
  ssh_interface    = "session_manager"
  ssh_timeout      = var.ssh_timeout
  ssh_username     = var.ssh_username

  launch_block_device_mappings {
    delete_on_termination = true
    device_name           = "/dev/xvda"
    encrypted             = var.encrypt_boot
    kms_key_id            = var.kms_key_id
    volume_size           = var.root_volume_size
    volume_type           = var.root_volume_type
  }

  run_tags      = merge(local.common_tags, { ImageFactoryStage = "build" })
  snapshot_tags = local.common_tags
  tags          = local.common_tags
}
```

The hardening layer uses shell in this version because the first pass needed to stay readable and close to the AMI lifecycle. This is just an example baseline. In practice, you could start from an AWS Marketplace image that already comes hardened and layer your custom tooling on top. You could also move the hardening logic into Ansible if that fits your team better. The shape is the same either way: install the baseline agent, apply the deltas, enable the services, clean the machine, and seal it.

```hcl
build {
  sources = ["source.amazon-ebs.al2023"]

  provisioner "shell" {
    inline_shebang = "/bin/bash -e"

    environment_vars = [
      "UPDATE_PACKAGES=${var.update_packages}",
    ]

    inline = [
      "set -euo pipefail",
      "if [ \"$UPDATE_PACKAGES\" = \"true\" ]; then sudo dnf update -y; fi",
      "if ! rpm -q amazon-ssm-agent >/dev/null 2>&1; then sudo dnf install -y amazon-ssm-agent; fi",
      "if ! rpm -q rsyslog >/dev/null 2>&1; then sudo dnf install -y rsyslog; fi",
      "printf '%s\\n' 'install cramfs /bin/false' 'blacklist cramfs' | sudo tee /etc/modprobe.d/cramfs.conf >/dev/null",
      "printf '%s\\n' 'net.ipv4.conf.all.accept_redirects = 0' 'net.ipv4.conf.default.accept_redirects = 0' | sudo tee /etc/sysctl.d/99-imagefactory-hardening.conf >/dev/null",
      "sudo sysctl -p /etc/sysctl.d/99-imagefactory-hardening.conf",
      "sudo mkdir -p /etc/ssh/sshd_config.d",
      "printf '%s\\n' 'PermitRootLogin no' 'PermitEmptyPasswords no' | sudo tee /etc/ssh/sshd_config.d/10-imagefactory-hardening.conf >/dev/null",
      "sudo chmod 600 /etc/ssh/sshd_config.d/10-imagefactory-hardening.conf",
      "printf '%s\\n' 'umask 027' | sudo tee /etc/profile.d/99-imagefactory-umask.sh >/dev/null",
      "sudo chmod 644 /etc/profile.d/99-imagefactory-umask.sh",
      "sudo systemctl enable --now amazon-ssm-agent",
      "sudo systemctl enable --now rsyslog",
      "sudo dnf clean all",
      "sudo cloud-init clean --logs",
      "sudo rm -f /etc/ssh/ssh_host_*",
    ]
  }

  post-processor "manifest" {
    output     = "artifacts/${var.ami_name_prefix}-manifest.json"
    strip_path = true
  }
}
```

The Ubuntu image follows the same pattern with `apt`, `snap`, a different username, and a different root device. AL2023 and Ubuntu aren't identical, but the repo shape is close enough that a reviewer can find the operating-system-specific differences quickly.

## Build Pipelines

The parent pipeline has three stages: detect, dispatch, and secret detection. The detect job figures out which image roots changed, writes a small matrix artifact, and generates a child pipeline. The dispatch job starts that generated child pipeline. This keeps the parent pipeline fast without using a giant static matrix that rebuilds every image because one line changed in one Packer directory.

```yaml
stages:
  - detect
  - dispatch
  - secret-detection

variables:
  SECRET_DETECTION_ENABLED: "true"
  PACKER_VERSION: "1.15.1"
  PACKER_BUILD_IMAGE: "amazonlinux:2023"
  AWS_REGION: "us-east-2"

include:
  - local: .gitlab/ci/detect/*.gitlab-ci.yml
  - local: .gitlab/ci/dispatch/*.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
```

The change detector compares the base and head SHAs, walks changed files under `packer/images/**`, finds the nearest directory containing `versions.pkr.hcl`, and emits a child pipeline job for each changed image. The generated job forwards variables like `PKR_ENV`, `PKR_IMAGE`, and `PKR_CHECKS_FILE`, so one provider pipeline can handle many image directories without copy and paste.

```sh
find_packer_root() {
  path="$1"
  dir="${path%/*}"

  while [ "$dir" != "." ] && [ "$dir" != "packer/images" ]; do
    if [ -f "$dir/versions.pkr.hcl" ]; then
      printf '%s\n' "$dir"
      return 0
    fi
    dir="${dir%/*}"
  done

  return 1
}

checks_file() {
  case "$1" in
    al2023)
      printf 'checks/al2023-cis-level1.yaml'
      ;;
    ubuntu24.04)
      printf 'checks/ubuntu24.04-cis-level1.yaml'
      ;;
    *)
      printf 'checks/al2023-cis-level1.yaml'
      ;;
  esac
}
```

The AWS child pipeline is where the real lifecycle happens. It builds the AMI, extracts the Packer manifest, launches a test instance, runs hardening checks over SSM, and publishes only after those checks pass.

```yaml
stages:
  - build
  - test
  - publish

packer:build:
  extends: .packer
  stage: build
  script:
    - ./scripts/build.sh
    - |
      manifest="$(find artifacts -type f -name '*-manifest.json' | sort | tail -n 1)"
      artifact_ids="$(jq -r '[.builds[] | select(.artifact_id != null and .artifact_id != "") | .artifact_id] | last // ""' "$manifest")"
      primary_artifact="${artifact_ids%%,*}"
      ami_region="${primary_artifact%%:*}"
      ami_id="${primary_artifact#*:}"
      ami_name="$(aws ec2 describe-images --region "$ami_region" --image-ids "$ami_id" --query 'Images[0].Name' --output text)"

      {
        printf 'AMI_ARTIFACT_IDS=%s\n' "$artifact_ids"
        printf 'AMI_REGION=%s\n' "$ami_region"
        printf 'AMI_ID=%s\n' "$ami_id"
        printf 'AMI_NAME=%s\n' "$ami_name"
      } > artifacts/packer.env
  artifacts:
    reports:
      dotenv: artifacts/packer.env
    paths:
      - artifacts/

terratest:ami:
  extends: .terratest
  stage: test
  needs:
    - job: packer:build
      artifacts: true
  script:
    - cd tests/terratest
    - go test -v -timeout 45m . -args -ami_name "$AMI_NAME" -checks_file "${PKR_CHECKS_FILE:-checks/al2023-cis-level1.yaml}"
```

The `artifacts/packer.env` file is the handoff between build and test. GitLab loads it as a dotenv report, so the test stage doesn't have to parse the Packer manifest again. It gets the AMI name from the previous job and uses that as the input for the Terratest fixture.

The other part I care about is authentication. The build and test jobs use GitLab OIDC to assume an AWS role. No long-lived AWS access keys in CI, no local credentials pasted into variables, and no mystery user showing up in CloudTrail. The job writes the GitLab OIDC token to a file, exports `AWS_ROLE_ARN`, and lets the AWS SDK credential chain handle the rest.

```yaml
.packer:
  image:
    name: "$PACKER_BUILD_IMAGE"
    entrypoint: [""]
  id_tokens:
    AWS_OIDC_TOKEN:
      aud: "https://gitlab.com"
  variables:
    AWS_WEB_IDENTITY_TOKEN_FILE: "$CI_PROJECT_DIR/.aws/gitlab-oidc-token"
  before_script:
    - |
      : "${AWS_OIDC_TOKEN:?Missing GitLab AWS OIDC token.}"
      : "${AWS_PACKER_BUILD_OIDC_ROLE_ARN:?Missing AWS Packer build OIDC role ARN.}"

      mkdir -p "$CI_PROJECT_DIR/.aws"
      printf '%s' "$AWS_OIDC_TOKEN" > "$AWS_WEB_IDENTITY_TOKEN_FILE"
      export AWS_ROLE_ARN="$AWS_PACKER_BUILD_OIDC_ROLE_ARN"
      export AWS_ROLE_SESSION_NAME="gitlab-$CI_PROJECT_ID-$CI_PIPELINE_ID-$CI_JOB_ID"
```

## Testing and Scanning AMIs

Building an AMI is not enough. The pipeline needs to boot the image and prove that the expected hardening controls are present on a running instance.

The test fixture launches one EC2 instance from the AMI name produced by Packer. It discovers the build VPC and subnets by tag, attaches a temporary SSM-capable instance profile when one is not provided, and avoids SSH entirely.

```hcl
data "aws_ami" "test" {
  count = var.tests_enabled ? 1 : 0

  most_recent = true
  owners      = var.ami_owners

  filter {
    name   = "name"
    values = [var.ami_name]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "test" {
  count = var.tests_enabled ? 1 : 0

  ami                         = one(data.aws_ami.test[*].id)
  instance_type               = var.instance_type
  subnet_id                   = sort(one(data.aws_subnets.test[*].ids))[0]
  vpc_security_group_ids      = var.security_group_ids != null ? var.security_group_ids : [one(aws_security_group.test[*].id)]
  iam_instance_profile        = var.iam_instance_profile != null ? var.iam_instance_profile : one(aws_iam_instance_profile.test[*].name)
  associate_public_ip_address = var.associate_public_ip_address

  tags = merge(
    var.tags,
    {
      Name = format("%s-test", var.ami_name)
    },
  )
}
```

The Go test loads a YAML file of hardening checks, waits for SSM to report that the instance is connected, and then runs each command through `AWS-RunShellScript`. A check passes when the command exits successfully and, when needed, stdout contains the expected value.

```go
type hardeningCheck struct {
	ID                   string `yaml:"id"`
	Description          string `yaml:"description"`
	Command              string `yaml:"command"`
	ExpectStdoutContains string `yaml:"expect_stdout_contains,omitempty"`
}

func TestAMIHardeningChecks(t *testing.T) {
	t.Parallel()
	logger.Default = logger.Discard

	if *amiName == "" {
		t.Skip("ami_name flag must be set")
	}

	const tfDir = "../terraform"

	defer ts.RunTestStage(t, "destroy", func() {
		destroyTerraform(t, tfDir)
	})

	ts.RunTestStage(t, "deploy", func() {
		applyTerraform(t, tfDir)
	})

	ssmClient := aws.NewSsmClient(t, awsRegion)

	ts.RunTestStage(t, "validate", func() {
		validate(t, tfDir, ssmClient)
	})
}
```

The checks use plain YAML so security engineers can review them without having to read Go. Adding another assertion means updating the relevant checks file and letting the same test harness run it.

```yaml
checks:
  - id: 1.1.1.1-cramfs-disabled
    description: cramfs filesystem module is disabled
    command: "! lsmod | grep -q cramfs && modprobe -n -v cramfs 2>&1 | grep -qE 'install /bin/(true|false)'"

  - id: 1.5.1-aslr-enabled
    description: kernel.randomize_va_space is set to 2 (full ASLR)
    command: "sysctl -n kernel.randomize_va_space"
    expect_stdout_contains: "2"

  - id: 5.2.6-ssh-root-login-disabled
    description: SSH PermitRootLogin is set to no
    command: "sshd -T 2>/dev/null | grep -i '^permitrootlogin' | awk '{print $2}'"
    expect_stdout_contains: "no"
```

The custom check format isn't mandatory. If a team doesn't want to maintain a separate validation harness, the same checks can move into Ansible validation playbooks. Ansible can use SSM to run checks on the temporary instance without opening SSH, which keeps the network model mostly the same while moving the assertions into a tool more operators already know. That is probably where this project goes over time.

This isn't a full substitute for every scanner or every benchmark. The wider program should still include vulnerability scanning, package inventory, and AWS Inspector coverage. These tests catch direct build regressions immediately: the service didn't start, the kernel setting didn't stick, the SSH drop-in didn't get read, or the baseline package never landed.

## Tagging and Sharing Images

Publishing is gated to the default branch. Merge requests can build and test, but they don't share AMIs to the organization. Once a default-branch build passes, the publish job tags the image as published and grants launch permissions to each account in `account-map.yaml`.

```yaml
publish:ami:
  extends: .terratest
  stage: publish
  needs:
    - job: packer:build
      artifacts: true
    - job: terratest:ami
  script:
    - |
      published_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
      account_ids="$(sed -n "s/.*: '\([0-9]\{12\}\)'$/\1/p" account-map.yaml)"

      if [ -z "$account_ids" ]; then
        printf 'No AWS account IDs found in account-map.yaml.\n' >&2
        exit 1
      fi

      for artifact in $(printf '%s' "$AMI_ARTIFACT_IDS" | tr ',' ' '); do
        ami_region="${artifact%%:*}"
        ami_id="${artifact#*:}"

        aws ec2 create-tags \
          --region "$ami_region" \
          --resources "$ami_id" \
          --tags \
            Key=ImageFactoryPublished,Value=true \
            Key=ImageFactoryPublishedAt,Value="$published_at"

        for account_id in $account_ids; do
          aws ec2 modify-image-attribute \
            --region "$ami_region" \
            --image-id "$ami_id" \
            --launch-permission "Add=[{UserId=$account_id}]"
        done
      done
  rules:
    - if: '$CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH'
```

The operational details matter here. The job reads artifact IDs from the Packer manifest instead of assuming one region. It tags the AMI after tests pass, not before. It fails closed if the account map is empty. Those small choices make the release process repeatable without someone babysitting every run.

## Tradeoffs and Unknowns

The main tradeoff is time. Building an AMI, booting it, waiting for SSM, running checks, and cleaning everything up is slower than normal application CI. The change-detection pipeline keeps the cost down by rebuilding only affected images. The boot test is still worth it when the alternative is distributing a broken base image to dozens of accounts.

Benchmark drift needs active ownership. CIS guidance, Marketplace images, distro defaults, and security agents all change. The validation checks should live beside the image definition so every hardening change can be reviewed with the matching test change.

Marketplace image handling has its own operational messiness. Product codes, owner IDs, naming patterns, and regional availability are all things you have to test in a real AWS account. Parameterizing the inputs helps, but it doesn't remove the fact that AWS Marketplace images aren't always as smooth as a vanilla owner-and-name AMI lookup.

Finally, AMI sharing is only part of distribution. Consumers still need a sane way to discover the latest approved AMI, whether that is through tags, SSM parameters, Service Catalog, Terraform data sources, or an internal platform workflow. Sharing the image makes it available. It doesn't automatically make every team use it correctly.

## Wrapping Up

An image factory isn't a compliance shortcut. It is a controlled path for choosing a trusted base, applying organization-specific deltas, testing the running result, and sharing the AMI only after the pipeline proves it is ready.

That's the real goal: turn "please use the hardened image" from a slide deck request into something teams can actually use.

---

If you liked (or hated) this blog, feel free to check out my [GitHub](https://github.com/RoseSecurity)!
