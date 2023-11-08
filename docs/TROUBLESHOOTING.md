# Troubleshooting

## Permission denied

When troubleshooting "permission denied" errors from `auth` for Workload
Identity, the first step is to ask the `auth` plugin to generate an OAuth access
token. Do this by adding `token_format: 'access_token'` to your YAML:

```yaml
- uses: 'google-github-actions/auth@v1'
  with:
    # ...
    token_format: 'access_token'
```

If your workflow _succeeds_ after adding the step to generate an access token,
it means Workload Identity Federation is configured correctly and the issue is
in subsequent actions. You can remove the `token_format` from your YAML. To
further debug:

1.  Enable [GitHub Actions debug logging][debug-logs] and re-run the workflow to
    see exactly which step is failing. Ensure you are using the latest version
    of that GitHub Action.

1.  Make sure you use `actions/checkout@v4` **before** the `auth` action in your
    workflow.

1.  If the failing action is from `google-github-action/*`, please file an issue
    in the corresponding repository.

1.  If the failing action is from an external action, please file an issue
    against that repository. The `auth` action exports Google Application
    Default Credentials (ADC). Ask the action author to ensure they are
    processing ADC correctly and using the latest versions of the Google client
    libraries. Please note that we do not have control over actions outside of
    `google-github-actions`.

If your workflow _fails_ after adding the step to generate an access token,
it likely means there is a misconfiguration with Workload Identity. Here are
some common sources of errors:

1.  Enable [GitHub Actions debug logging][debug-logs] and re-run the workflow to
    see exactly which step is failing. Ensure you are using the latest version
    of that GitHub Action.

1.  Ensure the value for `workload_identity_provider` is the full _Provider_
    name, **not** the _Pool_ name:

    ```diff
    - projects/NUMBER/locations/global/workloadIdentityPools/POOL
    + projects/NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER
    ```

1.  Ensure the `workload_identity_provider` uses the Google Cloud Project
    **number**. Workload Identity Federation does not accept Google Cloud
    Project IDs.

1.  Ensure that you have the correct `permissions:` for the job in your workflow, per
    the [usage](../README.md#usage) docs, i.e.

    ```yaml
    permissions:
      contents: 'read'
      id-token: 'write'
    ```

1.  Ensure you have created an **Attribute Mapping** for any **Attribute
    Conditions** or **Service Account Impersonation** principals. You cannot
    create an Attribute Condition unless you map that value from the incoming
    GitHub OIDC token. You cannot grant permissions to impersonate a Service
    Account on an attribute unless you map that value from the incoming GitHub
    OIDC token.

    You can use the [GitHub Actions OIDC Debugger][oidc-debugger] to print the
    list of token claims and compare them to your Attribute Mappings and
    Attribute Conditions.

1.  Ensure you have the correct casing and capitalization. GitHub does not
    distinguish between "foobar" and "FooBar", but Google Cloud does. Ensure any
    **Attribute Conditions** use the correct capitalization.

1.  Check the specific error message that is returned.

    -   If the error message includes "failed to generate Google Cloud federated
        token", it means admission into the Workload Identity Pool failed. Check
        your [**Attribute Conditions**][attribute-conditions].

    -   If the error message inclues "failed to generate Google Cloud access
        token", it means Service Account Impersonation failed. Check your
        [**Service Account Impersonation**][sa-impersonation] settings and
        ensure the principalSet is correct.

1.  Enable `Admin Read`, `Data Read`, and `Data Write` [Audit Logging][cal] for
    Identity and Access Management (IAM) in your Google Cloud project.

    **Warning!** This will increase log volume which may increase costs. To keep
    costs low, you can disable this audit logging after you have debugged the
    issue.

    Try to authenticate again, and then explore the logs for your Workload
    Identity Provider and Workload Identity Pool. Sometimes these error messages
    are helpful in identifying the root cause.

1.  Ensure you have waited at least 5 minutes between making changes to the
    Workload Identity Pool and Workload Identity Provider. Changes to these
    resources are eventually consistent.


## Subject exceeds the 127 byte limit

If you get an error like:

```text
The size of mapped attribute exceeds the 127 bytes limit.
```

it means that the GitHub OIDC token had a claim that exceeded the maximum
allowed value of 127 bytes. In general, 1 byte = 1 character. This most common
reason this occurs is due to long repo names or long branch names.

**This is a limit imposed by Google Cloud IAM.** We have no control over
this value. It is documented [here][wif-byte-limit]. Please [file feedback
with the Google Cloud IAM team][iam-feedback]. The only mitigation is to use
shorter repo names or shorter branch names.


## Token lifetime cannot exceed 1 hour

If you get an error like:

```text
The access token lifetime cannot exceed 3600 seconds.
```

it means that there is likely clock skew between where you are running the
`auth` GitHub Action and Google's servers. You can either install and configure
ntp pointed at time.google.com, or adjust the `access_token_lifetime` value to
something less than `3600s` to allow for clock skew (`3300s` would allow for 5
minutes of clock skew).


## Dirty git or bundled credentials

By default, the `auth` action exports credentials to the current workspace so
that the credentials are automatically available to future steps and
Docker-based actions. The credentials file is automatically removed when the job
finishes.

This means, after the `auth` action runs, the workspace is dirty and contains a
credentials file. This means creating a pull request, compiling a binary, or
building a Docker container, will include said credential file. There are a few
ways to fix this issue:

-   Add and commit the following lines to your `.gitignore`:

    ```text
    # Ignore generated credentials from google-github-actions/auth
    gha-creds-*.json
    ```

    **This requires the `auth` action be v0.6.0 or later.**

-   Re-order your steps. In most cases, you can re-order your steps such
    that `auth` comes _after_ the "compilation" step:

    ```text
    1. Checkout
    2. Compile (e.g. "docker build", "go build", "git add")
    3. Auth
    4. Push
    ```

    This ensures that no authentication data is present during artifact
    creation.

-   In situations where `auth` must occur before compilation, you can use
    the output to exclude the credential:

    ```text
    1. Checkout
    2. Auth
    3. Inject "${{ steps.auth.outputs.credentials_file_path }}" into ignore file (e.g. .gitignore, .dockerignore)
    4. Compile (e.g. "docker build", "go build", "git add")
    5. Push
    ```

## Issuer in ID Token does not match the expected ones

If you get an error like:

```text
The issuer in ID Token https://github.<company>.net/_services/token does not match the expected ones: https://token.actions.githubusercontent.com/
```

it means that the OIDC token's issuer and the Attribute Mapping do not match.
There are a few common reasons why this happens:

1.  You made a typographical error. If you are using the public version of
    GitHub (https://github.com), the value for the `oidc.issuerUri` should be
    `https://token.actions.githubusercontent.com`.

1.  You are using a GitHub Enterprise _Cloud_ installation and your GitHub
    administrator has configured a [unique token
    URL](https://docs.github.com/en/enterprise-cloud@latest/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#switching-to-a-unique-token-url).
    Use that URL for `oidc.issuerUri` instead of the public value. You must
    contact your GitHub administrator for assistance - our team does not have
    visibility into how your GitHub Enterprise Cloud instance is configured.

1.  You are using a GitHub Enterprise _Server_ installation. In this case, you
    must contact your GitHub administrator to get the URL for OIDC token
    verification. This is usually `https://github.company.com/_services/token`,
    but it can be customized by the installation. Furthermore, your GitHub
    administrator may have disabled this functionality. You must contact your
    GitHub administrator for assistance  - our team does not have visibility
    into how your GitHub Enterprise Server instance is configured.


<a name="aggressive-replacement"></a>

## Aggressive *** replacement in logs

When you use a [GitHub Actions secret][github-secrets] inside a workflow, _each_
line of the secret is masked in log output. This is controlled by GitHub, not
the `auth` action. We cannot change this behavior.

This can be problematic if your secret is a multi-line JSON string, since it
means curly braces (`{}`) and brackets (`[]`) will likely be replaced as `***`
in the GitHub Actions log output. To avoid this, remove all unnecessary
whitespace from the JSON and save the secret as a single-line JSON string. You
can convert a multi-line JSON document to a single-line manually or by using a
tool like `jq`:

```sh
cat credentials.json | jq -r tostring
```


[attribute-conditions]: https://cloud.google.com/iam/docs/workload-identity-federation#conditions
[sa-impersonation]: https://cloud.google.com/iam/docs/workload-identity-federation#impersonation
[debug-logs]: https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging
[iam-feedback]: https://cloud.google.com/iam/docs/getting-support
[wif-byte-limit]: https://cloud.google.com/iam/docs/configuring-workload-identity-federation
[cal]: https://cloud.google.com/logging/docs/audit/configure-data-access
[github-secrets]: https://docs.github.com/en/actions/security-guides/encrypted-secrets
[oidc-debugger]: https://github.com/github/actions-oidc-debugger
