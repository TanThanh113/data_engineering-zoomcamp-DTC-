# Terraform And GCP
---
## Prepare
### Introduction
Since there's a lot of information and definitions of Terraform available online, I'll leave the links here for your reference.

[Terraform](https://registry.terraform.io/browse/providers)
#### Download
You can download it here and follow the instructions [here](https://developer.hashicorp.com/terraform)

### GCP (Google Cloud Provider)
You can find more information [here](https://cloud.google.com/free?utm_source=pmax&utm_medium=display&utm_campaign=Cloud-SS-DR-GCP-1713664-GCP-DR-APAC-VN-en-PMAX-Display-PMAX-Prospecting-GenericCloud&utm_content=c--x--9222432-22046074022&utm_term&utm_source=PMAX&utm_medium=PMAX&utm_campaign=FY24-H2-apac-gcp-DR-campaign-VN&utm_content=vn-en&gclsrc=aw.ds&&https://ad.doubleclick.net/ddm/trackclk/N5295.276639.GOOGLEADWORDS/B26943865.344601469;dc_trk_aid=535898303;dc_trk_cid%3D163098484;dc_lat%3D;dc_rdid%3D;tag_for_child_directed_treatment%3D;tfua%3D;ltd%3D&gad_source=1&gad_campaignid=22046808689&gclid=CjwKCAjw-dfOBhAjEiwAq0RwI5an-F--cNZXFAaSS3aFiFXHDNYuVklBH4HqxIcx2qSsOKsKAVR1yRoCC0cQAvD_BwE).
 1. You need to log in to your account.
 2. Start using it for free.
 3. Here you will need a Visa card to register (if registration is successful, you will receive 300 USD to use on the Cloud).

## Operation
  1. Once you have installed everything, you need to create a bridge between Terraform and GCP (you can find instructions [here](https://docs.cloud.google.com/sdk/docs/install-sdk)).
  2. Open the uploaded file and import it
     ```
     gcloud auth application-default login
     ```
     This allows Terraform to link with GCP and use your account to create resources.
## Create a project from Project
Congratulations on reaching this stage!

```Now let's start creating a project.```
### GCP
To make it easier to understand, I'll use the example of a plot of land on Google Cloud.
1. Click New Project. (Create a Plot of Land)

   **Note**: Copy the Project ID immediately.
2. Enable API (Open Port):

   Go to APIs & Services > Library, find and enable these two:

    * Cloud Resource Manager API (For Terraform to manage projects).
    * IAM Service Account Credentials API (To borrow the name of a Service Account).
3. Create a Builder (Service Account):
   * Go to IAM & Admin > Service Accounts > Click Create Service Account.
     
     Name it: terraform-runner.
   * Assign permissions (Roles): Select permissions depending on the project (usually Storage Admin, BigQuery Admin).
     **```IMPORTANT: Copy this Service Account email.```**
### ON COMPUTER (TERMINAL/POWERSHELL)
The goal is for your computer to understand which project it's working on.
1. Identifying the Project
    * PowerShell
    * gcloud config set project [YOUR_PROJECT_ID]
2. Impersonation
    * This command allows your personal email to give instructions to the "builders" to do the work.
    * PowerShell
      ```
      gcloud iam service-accounts add-iam-policy-binding [EMAIL_SERVICE_ACCOUNT] ` 
      --member="user:[YOUR EMAIL]" ` 
      --role="roles/iam.serviceAccountTokenCreator"
      ```
3. Update ADC (Application Default)
   If you've logged in before, simply reset the payment project:

   PowerShell
   ```
    gcloud auth application-default set-quota-project [YOUR_PROJECT_ID]
   ```
### IDE
1. Preparing the Folder
    .Create a new folder -> Open it with VS Code -> Create two files: variables.tf and main.tf.
   
2. Code File Structure (Always 3 parts)

    Part 1: Original Provider (Using a personal email address).

    Part 2: Data Block (To obtain tokens from the Service Account).

    Part 3: Provider Alias ​​(Using borrowed tokens to build resources).
    ```
    Example
    terraform {
    required_providers {
        google = {
          source  = "hashicorp/google"
          version = "5.6.0"
        }
      }
    }

    # Provider 1: Use your personal email address to request permission to use your name.
    provider "google" {
      project = var.project
      region  = var.region
    }
    
    # Obtain a temporary token from your Service Account.
    data "google_service_account_access_token" "default" {
      provider               = google
      target_service_account = "EMAIL_SERVICE_ACCOUNTS"
      scopes                 = ["https://www.googleapis.com/auth/cloud-platform"]
      lifetime               = "3600s"
    }

    # Provider 2 (Alias: impersonated):
    provider "google" {
      alias        = "impersonated"
      access_token = data.google_service_account_access_token.default.access_token
      project      = var.project
      region       = var.region
    }
    ```
    The final steps involve creating the plan, which can be viewed [here](https://registry.terraform.io/browse/providers).

    **``Note``**: The parts that create services using impersonation require an additional value to be declared.
   ```
   provider      = google.impersonated
   ```

4. The Three Essential Commands
   
    Open the Terminal in VS Code and type:

    * terraform init (Run only once initially).

    * terraform plan (Check the blueprint before building).

    * terraform apply (Type yes to start actual construction).

```Principle```: When you are no longer learning/using it, delete it immediately.

---
### END
