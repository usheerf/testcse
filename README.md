# Postman API Onboarding: Enterprise Scaling Pilot

This repository demonstrates an automated onboarding pipeline for importing internal APIs into Postman, generating baseline tests, and establishing continuous API governance.

## 1. Build Decisions & Workflow Architecture
For the initial implementation, we selected the **Payment Refund API**, representing standard Lambda/API Gateway infrastructure. 

I utilized the `postman-api-onboarding-action` orchestrator because it cleanly chains the bootstrap and repo-sync processes, minimizing the YAML configuration required by the platform team. I mapped the `spec-url` dynamically to the raw GitHub content URL based on the commit SHA, ensuring that updates to the spec in the repo immediately trigger a sync to Postman Spec Hub.

## 2. Universal vs. Service-Specific Elements
When scaling this across 50 services, it is critical to separate the platform templates from service-level configurations.

*   **Universal (Platform-owned):** The GitHub Actions workflow structure, Postman authentication (API Key & Access Token secrets), and the generation logic for smoke/contract tests.
*   **Service-Specific (App Team-owned):** The OpenAPI spec itself, the target `workspace-name`, and environment variables (e.g., base URLs for the specific AWS Lambda/API Gateway endpoints).

## 3. Generated Checks vs. Business Logic
The automation generates baseline, smoke, and contract testing collections automatically from the OpenAPI spec.
*   **What the checks infer:** The automation validates API contracts (schema validation, data types, required fields) and basic availability (200 OK responses). It knows what a "refund" object should look like based on the YAML.
*   **What requires service-specific knowledge:** The automation *cannot* infer business rules. For example, it cannot test that a refund amount doesn't exceed the original transaction amount, or that a user has the correct IAM permissions. App teams must add these semantic assertions to the generated baseline collections.

## 4. Customer Ops Team Requirements
To replicate this, the customer's 4-person platform team must:
1. Provision a machine-user Postman API Key and Access Token.
2. Inject these tokens into their GitHub Organization as encrypted secrets.
3. Distribute the standard `.github/workflows/onboard-api.yml` template to application teams.
4. (For GitLab teams) Configure GitLab CI variables and inject the Postman CLI binary into their runner images.

## 5. Adaptation Analysis: Claims Processing API (GitLab CI & ECS)
The second provided spec, **Claims Processing API**, runs on mixed ECS/Lambda infrastructure and utilizes **GitLab CI** instead of GitHub Actions. 

**What stays the same:**
*   The OpenAPI spec structure.
*   The Postman workspace topology, Spec Hub upload, and test collection generation.

**What changes:**
*   We cannot use the pre-built `postman-api-onboarding-action` directly, as it is a GitHub-native action.
*   **Solution:** The platform team must translate the underlying Postman CLI commands into a `.gitlab-ci.yml` file. We will use the Postman CLI natively in GitLab to authenticate, push the spec to the Postman API, and utilize Newman to run the collections against the ECS/ALB staging environments.
*   The platform team must ensure their GitLab Runners have network line-of-sight to the ECS environments to execute the collections.

## Run Instructions & Validation
1. Ensure your Postman Enterprise trial is active.
2. Add `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN` as Repository Secrets.
3. Go to the **Actions** tab in this repo.
4. Select "Onboard Payment Refund API to Postman" and click **Run workflow**.
5. **Validation:** Check your Postman application. You should see a new "Payment Services" workspace containing the OpenAPI spec, linked environments, and generated collections.
