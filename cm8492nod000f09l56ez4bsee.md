---
title: "How to Deploy LibreChat (Your Own GPT) on Azure with Terraform and GitHub Actions"
seoTitle: "Deploy LibreChat on Azure Using Terraform"
seoDescription: "Deploy LibreChat on Azure with Terraform and GitHub Actions; explore open-source and cloud-native technologies"
datePublished: Tue Mar 11 2025 08:49:34 GMT+0000 (Coordinated Universal Time)
cuid: cm8492nod000f09l56ez4bsee
slug: how-to-deploy-librechat-your-own-gpt-on-azure-with-terraform-and-github-actions
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/67l-QujB14w/upload/e2d5b84805e0671eb94d24343d5f0375.jpeg
tags: github, azure, terraform, openai, azure-container-apps, azure-openai, librechat

---

At the start of this year, I came across the LibreChat project, which had nearly 20,000 stars on [GitHub](https://github.com/danny-avila/LibreChat) at the time. As a long-time ChatGPT Plus user, I was intrigued by this open-source solution, which labeled itself as an “enhanced ChatGPT” clone. Being a staunch advocate for open-source technologies, I was curious to explore its capabilities and wondered if I could seamlessly deploy LibreChat using cloud-native technologies on the Azure platform. Join me as I walk you through my journey.

## **What is LibreChat?**

LibreChat is a free and open-source AI chat platform created by [Danny Avila](https://www.librechat.ai/authors/danny). With contributions from over 200 developers, the project continues to gain momentum and has even outlined an [roadmap for 2025](https://www.librechat.ai/blog/2025-02-20_2025_roadmap). Its key design principles focus on delivering a **user-friendly interface**, supporting a **wide range of AI providers**, offering **extensibility**, and prioritizing **security**. You can find more detailed information about the project [here](https://www.librechat.ai/). For the basic setup the following components are needed:

* **API/UI**: The core application that powers LibreChat's interface and backend.
    
* **MongoDB**: A database used to store and handle dynamic, flexible data.
    

Enhanced features:

* **Search**: Provides a fast and efficient way to explore and navigate past conversations.
    
* **RAG API**: Offers context-aware responses by leveraging user-uploaded files to enhance interactions.
    

What sets LibreChat apart is its flexibility. The platform can be customized to suit your specific requirements, making it adaptable for a range of use cases and deployments.

For my installation, I decided to proceed with the core application, MongoDB, and optionally enable the search integration, as I currently don’t have a need for the RAG API.

## Choosing the right Azure Service(s)

With this decision in mind, I knew I needed to deploy 1-2 applications along with a MongoDB instance. Azure provides a wide range of solutions for hosting web applications, but given my strong passion for cloud-native technologies, containerization was the obvious (and only, let’s be honest) choice for me. However, using Kubernetes felt like overkill for such a lightweight setup, so I choose Azure Container Apps as the home for LibreChat.

[Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) is a serverless platform specifically designed for scenarios like mine - just provide a container image, configure a few settings, and let the platform handle the rest. It’s a perfect match for this use case!

And then it hit me like a bolt of lightning striking the Hill Valley Courthouse. Azure Container Apps has another feature that’s a perfect fit for this scenario **K**ubernetes **E**vent-**D**riven **A**utoscaling **(KEDA)**. With [KEDA](https://keda.sh/), Azure Container Apps can automatically scale HTTP-based applications down to zero when they’re not in use. While this does introduce a brief wait time as the app spins up, it’s completely acceptable for my use case as i dont need the application to be running 24/7. An added bonus? This approach not only saves costs but also reduces environmental impact by lowering CO2 emissions.

The natural choice for deploying MongoDB on Azure is [CosmosDB with the MongoAPI](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/introduction) (at least for me). While I could have opted to run MongoDB in an Azure Container App, I’m not a big fan of running databases in containers. Instead, I selected the serverless option for the CosmosDB account, allowing me to pay only for what I use—aligning perfectly with the pay-as-you-go model of the container app setup.

> CosmosDB provides a lifetime free tier for one CosmosDB account per Azure subscription, making it an appealing option for this setup. However, it's important to note that the free tier does not support the serverless capability.

With the Azure services selected, my go-to tools for deploying this setup are Terraform and GitHub Actions.

## Let’s Get Practical: Getting Started

When working with Terraform, one crucial consideration is the management of the state file. The standard practice when using Terraform with Azure is to store the state file in an Azure Storage Account.

The next important aspect is authentication. Historically, Terraform authentication was handled using User Principals or Machine Accounts (Service Principals). However, this approach required storing sensitive credentials and secrets, introducing potential security risks.

In the modern cloud-native world, the de facto standard has shifted to OIDC (OpenID Connect). By leveraging OIDC and Workload Identity Federation, we can utilize managed identities to authenticate and deploy resources with Terraform. This eliminates the need to manage secrets manually and avoids the hassle of creating additional Entra ID (formerly Azure AD) permissions typically required for service principals. It’s a much cleaner, more secure, and practical approach for modern infrastructure-as-code workflows. Check out [this blog](https://thomasthornton.cloud/2025/02/27/deploying-to-azure-secure-your-github-workflow-with-oidc/) for a more in-depth exploration of the topic.

To begin the setup, start by cloning the repository and installing the required tools: **Azure CLI**, **GitHub CLI**, and **jq**.

For a faster setup, I’ve created a small script that takes the following inputs. These inputs can be provided either through a `.env` file or directly within the script:

* **name -** the project name
    
* **scope -** the scope, example: tf for the terraform
    
* **location -** the Azure Region to deploy to
    
* **tag -** tags for the resources created by the script
    
* **ghRepo -** the GitHub repo from where the Terraform code is deployed
    

Before Azure Resources are created, the script will attempt to retrieve the current Subscription ID and Tenant ID using the Azure CLI. You can customize this behavior to suit your specific requirements.

The script will create the following resources in Azure:

* **Resource Group**
    
* **Storage Account with Blob Container**
    
* **Managed Identity with Owner Role Assignment**
    
* **Federated Credentials**
    
* **Several GitHub Secrets** - for the ease of use in the GitHub Actions
    

> For simplicity, I assigned the managed identity the **Owner** role on my Azure subscription. However, this approach is not suitable for production environments. In a production-ready setup, you should follow the principle of least privilege and assign only the minimum permissions necessary for your deployment to function correctly.

Let’s take a closer look at the federated credentials.

* The first credential grants the Managed Identity permission to deploy Terraform code, but only from the **main branch**. This ensures that code deployment from other branches, such as the **dev branch**, is blocked.
    
* The second credential provides permission to execute Terraform validation tasks, such as running **CI workflows** with `terraform plan` on pull requests.
    

This setup adds an extra layer of control and security to the deployment process.

```bash
# create federated credential for main branch
az identity federated-credential create \
    --resource-group $rg \
    --identity-name "$miName" \
    --name "fc-github-$name-$scope-branch" \
    --issuer "https://token.actions.githubusercontent.com" \
    --subject "repo:$ghRepo::ref:refs/heads/main"\
    --audiences "api://AzureADTokenExchange"

# create federated credential for pull requests
az identity federated-credential create \
    --resource-group $rg \
    --identity-name "$miName" \
    --name "fc-github-$name-$scope-pr" \
    --issuer "https://token.actions.githubusercontent.com" \
    --subject "repo:$ghRepo:pull_request"\
    --audiences "api://AzureADTokenExchange"
```

Now we are set and done. You can find the complete code for the script in my GitHub repository [here](https://github.com/philwelz/chat/blob/main/scripts/up.sh).

## Up & Running: Final Steps for Your Deployment

Before getting LibreChat up and running, you need to configure an authentication method. I opted to use GitHub as the authentication backend. You can follow the workflow [here](https://www.librechat.ai/docs/configuration/authentication/OAuth2-OIDC/github) to set up LibreChat authentication using a GitHub App.

Once the GitHub App is created, you'll need to create the following GitHub secrets:

* `LIBRECHAT_APP_ID`
    
* `LIBRECHAT_APP_SECRET`
    

> Once the desired users have registered, it's important to disable further registrations by setting `ALLOW_REGISTRATION = false`, as permission management is not implemented. Alternatively, you can choose a different authentication method that better fits your requirements.

Now it's time to configure the AI endpoints. LibreChat supports a wide range of AI providers. The Terraform code available in the repository enables **Azure OpenAI** and **OpenAI** by default. If you wish to disable these endpoints and adopt the config with your AI endpoints, you can do so by setting the following variables:

* `openai_enabled = false`
    
* `azure_openai_enabled = false`
    

The default models for both AI providers are:

* gpt-4o
    
* gpt-4o-mini
    
* o3-mini
    

> If you keep the default settings with OpenAI enabled, you'll need to create a GitHub secret named `OPENAI_API_KEY` and provide your OpenAI Platform API key.

***Optional:*** *To enable the Search integration with MeiliSearch, simply set the variable* `search_enabled` *to* `true`*.*

***Optional:*** *Custom domain integration.* *I’m using* *Cloudflare* *for managing the custom domain setup. However, you’re free to use your own domain provider or skip this configuration altogether and stick with Azure's default settings.*

You can find the configuration options for LibreChat [here](https://github.com/philwelz/chat/blob/main/src/_locals.tf). Refer to the official [documentation](https://www.librechat.ai/docs/configuration/dotenv) for a detailed overview of the available settings.

Once you’ve made the necessary adjustments to the Terraform code and LibreChat config, you’re ready to push it and trigger the deployment using GitHub Actions.

## Expanding the Setup: CI and Extras

By this point, all the heavy lifting is complete, and your own GPT should be up and running. Now it’s time to ensure that your Terraform setup and providers are fully up-to-date and to add some Terraform documentation. But don’t worry - I’ve got you covered!

In the repository, you’ll find a [workflow](https://github.com/philwelz/chat/blob/main/.github/workflows/tf-docs.yml) for [terraform-docs](https://github.com/terraform-docs/gh-actions), an action that ensures the Terraform documentation for your code is always up-to-date by automatically committing changes to an open pull request.

But that’s not all. The repository also includes a [Renovate workflow](https://github.com/philwelz/chat/blob/main/.github/workflows/renovate.yml), which ensures that your Terraform setup and providers stay up-to-date. [Renovate-Bot](https://docs.renovatebot.com/) monitors GitHub releases for updates to these components. Additionally, it keeps MeiliSearch and LibreChat current by tracking new container image releases. The config for Renovate can be found [here](https://github.com/philwelz/chat/blob/main/.github/renovate.js).

To enable Renovate to monitor GitHub releases, GitHub authentication needs to be configured. This can be achieved using either a Personal Access Token (PAT) or a GitHub App. I chose to use a GitHub App. Once the GitHub App is set up, the following GitHub secrets must be created for use with GitHub Actions:

* `RENOVATE_APP_ID`
    
* `RENOVATE_PRIVATE_KEY`
    

Last but no least, Labeler. I like labeler because the automation helps streamline workflow organization and makes it easier to quickly identify the purpose of a pull request. As you can see in the example, based on certain changes to files, labels are attached to the pull request.

```yaml
cicd:
- changed-files:
  - any-glob-to-any-file:
    - .github/**

terraform:
- changed-files:
  - any-glob-to-any-file:
    - src/**

documentation:
- changed-files:
  - any-glob-to-any-file:
    - '**/*.md'
```

## How to get started

You can access the complete Terraform code and GitHub Actions workflows in this [GitHub repository](https://github.com/philwelz/chat). The state of the repository reflects the exact LibreChat GPT version I’m currently using. In other words, this code is live, actively maintained, and subject to regular improvements and changes. It’s a great starting point for setting up your own GPT instance and experimenting with Generative AI. Feel free to dive in, explore, and customize it to suit your needs!

## Conclusion

Deploying LibreChat on Azure with Terraform and GitHub Actions is an exciting and practical journey, combining the best of open-source innovation with the flexibility and scalability of cloud-native technologies. By utilizing Azure Container Apps and CosmosDB, you can build a cost-efficient and sustainable setup tailored to your needs. The adoption of OIDC authentication and GitHub Actions simplifies deployment, ensuring a secure and streamlined workflow. Additionally, tools like Renovate and terraform-docs help keep your infrastructure up-to-date and well-documented. This approach not only enables you to operate your own GPT but also embraces modern cloud-deployment best practices, providing a powerful and flexible platform for AI-driven applications.