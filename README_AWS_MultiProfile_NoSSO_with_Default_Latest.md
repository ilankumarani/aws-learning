# AWS CLI Multi-Profile Setup on Windows (Without SSO)

This guide explains how to set up **multiple AWS CLI profiles on Windows without SSO**, including a **default profile**, and how to use them with **one-command switching** and **Terraform**.

---

## 0. Overview

Without SSO there is **no** `aws login` command.

Instead, you:

1. Create **IAM users** with **access keys**.
2. Configure **multiple profiles** in AWS CLI (including an optional **default**).
3. Use `--profile` or the `AWS_PROFILE` environment variable to switch.
4. (Optional) Create **PowerShell shortcuts** to simulate a "one-command login" feeling.

---

## 1. Create IAM Users and Access Keys

Do this once per environment (e.g. default, dev, stage, prod).

1. Go to **AWS Console → IAM → Users → Add user**.
2. Give a name, for example:
   - `default-user` (if you want a main/default account)
   - `dev-user`
   - `stage-user`
   - `prod-user`
3. Enable **programmatic access** (to get access keys).
4. Attach a permission policy:
   - For testing, you *can* use `AdministratorAccess`  
     (for real use, create a more restricted policy).
5. Finish the wizard and at the end **download the CSV** or copy:
   - `Access key ID`
   - `Secret access key`

You will end up with pairs such as:

- Default: `AKIADEFAULT...` / `DEFAULTSECRET...`
- Dev: `AKIADEV...` / `DEVSECRET...`
- Stage: `AKIASTAGE...` / `STAGESECRET...`
- Prod: `AKIAPROD...` / `PRODSECRET...`

Keep these **secret** and **safe**.

---

## 2. Install / Verify AWS CLI on Windows

Open **PowerShell** or **Command Prompt** and run:

```bash
aws --version
```

If it shows something like `aws-cli/2.x.x`, you’re good.

If not:

1. Download **AWS CLI v2** for Windows (MSI installer) from the AWS website.
2. Install it.
3. Run `aws --version` again to confirm.

---

## 3. Configure AWS CLI Profiles (Default + Multiple Profiles)

You configure one profile at a time.

### 3.1 Default profile (optional but common)

If you run `aws configure` **without** `--profile`, you are configuring the **default** profile:

```bash
aws configure
```

You’ll be asked:

- **AWS Access Key ID** → paste the **default** access key
- **AWS Secret Access Key** → paste the **default** secret key
- **Default region name** → e.g. `eu-west-2` (London) or your preferred region
- **Default output format** → `json` (or `text`/`table` if you prefer)

This will be used any time you **don’t** specify `--profile` and you **don’t** set `AWS_PROFILE`.

### 3.2 Dev profile

```bash
aws configure --profile dev
```

You’ll be asked:

- **AWS Access Key ID** → paste the **dev** access key
- **AWS Secret Access Key** → paste the **dev** secret key
- **Default region name** → e.g. `eu-west-2`
- **Default output format** → `json`

### 3.3 Stage profile

```bash
aws configure --profile stage
```

Fill in with the **stage** keys.

### 3.4 Prod profile

```bash
aws configure --profile prod
```

Fill in with the **prod** keys.

---

## 4. What Gets Written to the Config Files

AWS CLI stores settings in two files:

- Credentials:  
  `C:\Users\<your-username>\.aws\credentials`
- Config:  
  `C:\Users\<your-username>\.aws\config`

### 4.1 Example `credentials` file (with default + multiple profiles)

```ini
[default]
aws_access_key_id = AKIADEFAULTXXXXXXXX
aws_secret_access_key = DEFAULTSECRETXXXXXXXX

[dev]
aws_access_key_id = AKIADEVXXXXXXXX
aws_secret_access_key = DEVSECRETXXXXXXXX

[stage]
aws_access_key_id = AKIASTAGEXXXXXXXX
aws_secret_access_key = STAGESECRETXXXXXXXX

[prod]
aws_access_key_id = AKIAPRODXXXXXXXX
aws_secret_access_key = PRODSECRETXXXXXXXX
```

### 4.2 Example `config` file (with default + multiple profiles)

```ini
[default]
region = eu-west-2
output = json

[profile dev]
region = eu-west-2
output = json

[profile stage]
region = eu-west-2
output = json

[profile prod]
region = eu-west-2
output = json
```

> Notes:
> - In `credentials`, the sections are named just `default`, `dev`, `stage`, `prod`.
> - In `config`, the default section is `[default]`, and the others are `[profile dev]`, `[profile stage]`, `[profile prod]`.
> - When you don’t specify any profile, the **default** section is used.

---

## 5. Using the Profiles (Your “Login” Is the Keys)

There is no `aws login` here.  
Once the profiles are configured, you just **use them**.

### 5.1 Using the default profile

If you don’t specify anything, AWS CLI uses the **default** profile:

```bash
aws sts get-caller-identity
```

This uses the `[default]` configuration.

### 5.2 Using named profiles explicitly

To test each named profile:

```bash
aws sts get-caller-identity --profile dev
aws sts get-caller-identity --profile stage
aws sts get-caller-identity --profile prod
```

You should see output with:

- `Account` – the AWS account ID
- `Arn` – the ARN of the IAM user for that profile

If you see an error, the keys might be wrong, expired, or not allowed.

---

## 6. One-Command Profile Switching in PowerShell

To make it feel like a single-command “login” per environment, you can add helper functions that set the `AWS_PROFILE` environment variable.

### 6.1 Open your PowerShell profile

In PowerShell, run:

```powershell
notepad $PROFILE
```

- If the file doesn’t exist, PowerShell will ask to create it – accept.

### 6.2 Add helper functions

Paste this into the file:

```powershell
function aws-default {
    Remove-Item Env:AWS_PROFILE -ErrorAction SilentlyContinue
    Write-Host "Switched AWS profile to 'default' (no AWS_PROFILE set)"
}

function aws-dev {
    $env:AWS_PROFILE = "dev"
    Write-Host "Switched AWS profile to 'dev'"
}

function aws-stage {
    $env:AWS_PROFILE = "stage"
    Write-Host "Switched AWS profile to 'stage'"
}

function aws-prod {
    $env:AWS_PROFILE = "prod"
    Write-Host "Switched AWS profile to 'prod'"
}
```

Save and close Notepad, then **restart PowerShell**.

### 6.3 Use the shortcuts

Now you can do:

```powershell
aws-default
aws sts get-caller-identity   # uses the default profile

aws-dev
aws sts get-caller-identity   # uses dev profile automatically

aws-stage
aws sts get-caller-identity   # now uses stage profile automatically

aws-prod
aws sts get-caller-identity   # now uses prod profile automatically
```

- `aws-default` removes `AWS_PROFILE`, so CLI falls back to the **default** profile.
- The others set `AWS_PROFILE` so the corresponding profile is used automatically.

You no longer need `--profile` on every command while your PowerShell session is open.

---

## 7. Using Profiles with Terraform

In your Terraform `provider "aws"` configuration, you can specify which profile to use.

### 7.1 Using the default profile

If you want Terraform to use the default profile, you can either:

- Omit the `profile` field:

  ```hcl
  provider "aws" {
    region = "eu-west-2"
  }
  ```

  (Terraform will use the default AWS CLI credentials resolution order; if `AWS_PROFILE` is not set, it will use `default`.)

- Or explicitly set `profile = "default"`:

  ```hcl
  provider "aws" {
    region  = "eu-west-2"
    profile = "default"
  }
  ```

### 7.2 Single environment example (dev)

```hcl
provider "aws" {
  region  = "eu-west-2"
  profile = "dev"
}
```

Then run:

```bash
terraform init
terraform plan
terraform apply
```

Terraform will use the `dev` profile’s keys.

### 7.3 Multiple environments (dev + stage)

```hcl
provider "aws" {
  region  = "eu-west-2"
  profile = "dev"
}

provider "aws" {
  alias   = "stage"
  region  = "eu-west-2"
  profile = "stage"
}
```

You can then reference the second provider as `aws.stage` in your resources if needed.

---

## 8. How AWS CLI Chooses a Profile (Technical Notes)

The AWS CLI decides which profile to use in this order:

1. **`--profile` flag** on the command:
   ```bash
   aws s3 ls --profile dev
   ```
2. **`AWS_PROFILE` environment variable**:
   ```powershell
   $env:AWS_PROFILE = "dev"
   aws s3 ls  # uses dev
   ```
3. **`default` profile** in config/credentials (if nothing else is set).

So our PowerShell functions in section 6 are simply:

- Setting `AWS_PROFILE` to `dev`, `stage`, or `prod`, or
- Removing `AWS_PROFILE` to fall back to the **default** profile.

---

## 9. Summary

- There is **no `aws login`** command without SSO.
- You **create IAM users and access keys** and configure them as:
  - A **default** profile (via `aws configure`), and/or
  - **Named** profiles (via `aws configure --profile <name>`).
- Profiles are stored in:
  - `~/.aws/credentials`
  - `~/.aws/config`
- You select the profile using:
  - `--profile` on commands, or
  - `AWS_PROFILE` environment variable (set via PowerShell helper functions).
- Terraform can use:
  - The **default** profile (no `profile` or `profile = "default"`), or
  - A named profile (`profile = "dev"`, `profile = "stage"`, etc.).

This setup gives you clean, separate **default**, **dev**, **stage**, and **prod** profiles on Windows without using SSO.
