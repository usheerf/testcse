# Postman API Onboarding: Enterprise Scaling Pilot

This repository demonstrates an automated onboarding pipeline for importing internal APIs into Postman, generating baseline tests, and establishing continuous API governance.

## 1. Build Decisions & Workflow Architecture
For the initial implementation, I selected the **Payment Refund API**, representing the customer's standard standard AWS Lambda/API Gateway infrastructure. 

I utilized the custom `postman-api-onboarding-action` orchestrator because it cleanly chains the bootstrap and repo-sync processes, minimizing the YAML configuration required by the platform team. I mapped the `spec-url` dynamically to the raw GitHub content URL based on the commit SHA, ensuring that updates to the spec immediately trigger a sync to Postman Spec Hub. 

*Note on Permissions:* The automation's default behavior attempts to write a `.github/workflows/ci.yml` file to schedule continuous test runs. Because GitHub's default `GITHUB_TOKEN` restricts workflow modification to prevent infinite loops, I explicitly set `generate-ci-workflow: 'false'` to securely bypass this limitation without requiring the customer to mint a broadly scoped Personal Access Token (PAT).

## 2. Universal vs. Service-Specific Elements
When scaling this across 50 services, it is critical to separate the platform templates from service-level configurations.

*   **Universal (Platform-owned):** The GitHub Actions workflow structure, Postman authentication (API Key & Access Token secrets), and the generation logic for smoke/contract tests.
*   **Service-Specific (App Team-owned):** The target `project-name`, the OpenAPI spec, and environment variables. 
*   **The `baseUrl` Pattern:** By design, the automation generates the Postman Environment with an empty `baseUrl`. This is a feature, not a bug, it respects developer state, allowing engineers to set `baseUrl` to `localhost` without CI/CD overwriting it on the next run. For production deployments, the platform CI/CD pipeline dynamically injects the real AWS API Gateway URL using the `env-runtime-urls-json` input.

## 3. Generated Checks vs. Business Logic
The automation generates baseline, smoke, and contract testing collections automatically.
*   **What the checks infer:** The automation validates API contracts (schema validation, data types, required fields) and basic availability (200 OK responses). It guarantees the API shape matches the OpenAPI specification.
*   **What requires service-specific knowledge:** The automation *cannot* infer business rules. For example, looking at the Payment API, it cannot automatically test that a partial refund doesn't exceed the original amount, that a transaction is under 180 days old, or that a transaction is strictly in the 'captured' state. Application teams must add these semantic assertions to the generated baseline collections.

## 4. Customer Ops Team Requirements
To replicate this org-wide, the customer's 4-person platform team must:
1. Provision a machine-user Postman API Key and Access Token.
2. Inject these tokens into their GitHub Organization as encrypted secrets.
3. Distribute the standard `.github/workflows/onboard-payment-api.yml` template to application teams.
4. (For GitLab teams) Configure GitLab CI variables and inject the Postman CLI binary into their runner images.

## 5. Adaptation Analysis: Claims Processing API (GitLab CI & ECS)
The second provided spec, **Claims Processing API**, runs on mixed ECS/Lambda infrastructure and utilizes **GitLab CI** instead of GitHub Actions. 

**What stays the same:**
*   The OpenAPI spec structure.
*   The Postman workspace topology, Spec Hub upload, Mock Server generation, and test collection generation.

**What changes:**
*   We cannot use the pre-built `postman-api-onboarding-action` directly, as it is a GitHub-native action.
*   **Solution:** The platform team must translate the underlying Postman CLI commands into a `.gitlab-ci.yml` file. We will use the Postman CLI natively in GitLab to authenticate, push the spec to the Postman API, and utilize Newman to run the collections against the ECS/ALB staging environments.
*   The platform team must ensure their GitLab Runners have network line-of-sight to the internal ECS environments to execute the collections.

## Run Instructions & Validation
1. Ensure your Postman Enterprise trial is active.
2. Add `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN` as Repository Secrets.
3. Go to the **Actions** tab in this repo.
4. Select "Onboard Payment Refund API to Postman" and click **Run workflow**.
5. **Validation:** Check your Postman application. You should see a new "Payment Services" workspace containing the OpenAPI spec, linked environments, and generated collections. The repository will also be updated with `.postman/` and `postman/` directories establishing Docs-as-Code.
