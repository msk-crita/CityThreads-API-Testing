# City Threads API Testing

API testing and validation repository for the **City Threads** application, using [Bruno](https://www.usebruno.com/).

This repository maintains API test collections, reusable test scripts, test scenarios, and defect-tracking documentation to support consistent API quality assurance.

## Project Information

| Item | Details |
|---|---|
| Project | City Threads |
| Testing Tool | Bruno |
| Testing Type | API Testing |
| API Environment | Test |
| Version Control | Git |
| Defect Tracking | GitHub Issues |
| Repository | [CityThreads-API-Testing](https://github.com/msk-crita/CityThreads-API-Testing) |

## Technology & Tools

- **Bruno** – API requests and test automation.
- **Git** – Version control.
- **GitHub** – Repository management and defect tracking.
- **Swagger / OpenAPI** – API documentation and contract reference.

## Repository Structure

```text
CityThreads-API-Testing/
│
├── auth-controller/
│   ├── AUTH-LOGIN-001.yml
│   ├── AUTH-LOGIN-002 - Invalid Password.yml
│   ├── AUTH-LOGIN-003 - Invalid Username.yml
│   ├── AUTH-LOGIN-004 - Empty Login.yml
│   ├── AUTH-LOGIN-005 - Empty Password.yml
│   ├── AUTH-LOGIN-006 - Missing Login.yml
│   ├── AUTH-LOGIN-007 - Missing Password.yml
│   ├── AUTH-LOGIN-008 - Missing Client Type.yml
│   ├── AUTH-LOGIN-009 - Missing Device Label.yml
│   └── folder.yml
│
├── environments/
│   └── CityThreads.yml  (Local only; excluded from Git)
│
├── .gitignore
├── opencollection.yml
└── README.md
```

Additional controller folders and test scenarios will be added as testing progresses.

## API Documentation

Swagger UI:

https://server.crita.in/city-threads-test/swagger-ui/index.html

OpenAPI JSON:

https://server.crita.in/city-threads-test/v3/api-docs

Use the API documentation as the reference for endpoint definitions, request parameters, schemas, and expected API behavior.

## Getting Started

### Prerequisites

1. Install [Bruno](https://www.usebruno.com/downloads).
2. Install [Git](https://git-scm.com/downloads).
3. Clone this repository.

```bash
git clone https://github.com/msk-crita/CityThreads-API-Testing.git
```

4. Open the cloned collection in Bruno.

### Environment Configuration

The local Bruno environment file is intentionally excluded from version control because it may contain credentials and authentication tokens.

Create a local environment in Bruno and configure the following variables:

| Variable | Description |
|---|---|
| `baseurl` | City Threads API base URL |
| `login` | Valid test account username |
| `password` | Valid test account password |

Example configuration:

```text
baseurl = https://server.crita.in/city-threads-test
login = <your-test-username>
password = <your-test-password>
```

Use the environment variables in requests instead of hard-coding credentials.

**Security:** Never commit actual passwords, access tokens, refresh tokens, API keys, or other sensitive information.

## Test Case Naming Convention

Test cases follow a consistent identifier format:

```text
<CONTROLLER>-<ENDPOINT>-<NUMBER>
```

Example:

```text
AUTH-LOGIN-001
AUTH-LOGIN-002
AUTH-LOGIN-003
```

Each test case should represent a specific scenario and have a clear, identifiable purpose.

### Test Coverage

API test scenarios should cover:

- Positive and successful requests.
- Invalid input and authentication failures.
- Missing and empty parameters.
- Null values and incorrect data types.
- Boundary and validation conditions.
- HTTP status codes.
- Response structure and schema.
- Authorization and access control.
- Security-related response validation.

Expected results should be based on API contracts, Swagger documentation, and functional requirements rather than simply copying the observed implementation response.

## Executing Tests

1. Open the City Threads collection in Bruno.
2. Select the appropriate environment.
3. Open the controller folder.
4. Select the required API test scenario.
5. Execute the request.
6. Review the HTTP response and test results.
7. Record any failed test cases for investigation.

Each test script should use Bruno's `test()` and `expect()` assertions, with descriptive test names containing the corresponding test case ID.

## Defect Tracking – GitHub Issues

All confirmed API defects should be documented using GitHub Issues.

[Create a GitHub Issue](https://github.com/msk-crita/CityThreads-API-Testing/issues/new/choose)

A defect report should include:

- Test Case ID.
- Controller and endpoint.
- Environment.
- Request method and endpoint URL.
- Steps to reproduce.
- Expected result.
- Actual result.
- HTTP status code.
- Sanitized request and response.
- Relevant screenshots or execution evidence.
- Severity and impact.

Never include real credentials, authentication tokens, or other sensitive information in GitHub Issues.

## Git Workflow

The `main` branch contains the shared API testing collection.

After making changes:

```bash
git status
git add .
git commit -m "Add or update API test scenarios"
git push origin main
```

Review staged files before committing to ensure no sensitive information is included.

## Security Guidelines

- Do not commit environment files containing credentials.
- Do not hard-code passwords or authentication tokens.
- Use Bruno environment variables for sensitive values.
- Redact tokens and personal information from screenshots and defect reports.
- Keep test credentials restricted to authorized team members.
- Review `.gitignore` before committing new files.

---

**Maintained by:** Crita Creative Testing Team

**Repository:** https://github.com/msk-crita/CityThreads-API-Testing