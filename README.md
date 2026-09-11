# Zuplo Self Hosted Doctor

`zuplo-self-hosted-doctor` checks a self-hosted Zuplo installation from a machine that can
reach its Kubernetes API and installation endpoints. It uses your kubeconfig and
does not require kubectl, Helm, OpenSSL, Node.js, or Go to be installed. If your
kubeconfig uses an external authentication helper, that helper must be installed
and you must be signed in.

## Download

Download the `.tgz` for your machine from the
[releases page](https://github.com/zuplo/zuplo-self-hosted-doctor/releases). Choose the
architecture of the machine running Doctor, not the Kubernetes nodes.

| Machine                   | Filename suffix     |
| ------------------------- | ------------------- |
| Linux, Intel/AMD 64-bit   | `linux_amd64.tgz`   |
| Linux, ARM 64-bit         | `linux_arm64.tgz`   |
| macOS, Intel              | `darwin_amd64.tgz`  |
| macOS, Apple silicon      | `darwin_arm64.tgz`  |
| Windows, Intel/AMD 64-bit | `windows_amd64.tgz` |
| Windows, ARM 64-bit       | `windows_arm64.tgz` |

Choose a `zuplo-self-hosted-doctor_...tgz` asset. GitHub's automatic source-code archives do
not contain the executable. Replace `1.2.3` in the examples with your downloaded
version.

### Verify the download

Download `checksums.txt` from the same release. Before extracting, calculate
your archive's SHA256 hash using the command for your operating system:

```sh
# Linux
sha256sum zuplo-self-hosted-doctor_1.2.3_linux_amd64.tgz
# macOS
shasum -a 256 zuplo-self-hosted-doctor_1.2.3_darwin_arm64.tgz
```

```powershell
# Windows (PowerShell)
Get-FileHash .\zuplo-self-hosted-doctor_1.2.3_windows_amd64.tgz -Algorithm SHA256
```

Substitute your downloaded filename. The hash must match its entry in
`checksums.txt` (ignoring letter case). If it differs, do not run the binary;
download the archive again.

## Extract and run

### Linux and macOS

Use `tar` to extract the archive for your machine, then run the binary directly:

```sh
# Linux Intel/AMD example; substitute your downloaded filename.
tar -xzf zuplo-self-hosted-doctor_1.2.3_linux_amd64.tgz
./zuplo-self-hosted-doctor version
./zuplo-self-hosted-doctor verify
```

If the executable bit was lost while copying the file, restore it with
`chmod +x ./zuplo-self-hosted-doctor`. Optionally move the binary to a directory already on
your PATH to run `zuplo-self-hosted-doctor` without the `./` prefix.

### Windows (PowerShell)

Use the built-in `tar` command on Windows 10/11 to extract the archive:

```powershell
tar -xzf .\zuplo-self-hosted-doctor_1.2.3_windows_amd64.tgz
.\zuplo-self-hosted-doctor.exe version
.\zuplo-self-hosted-doctor.exe verify
```

If `tar` is unavailable, use your organization's approved archive utility to
extract both the gzip and tar layers. Run the resulting `zuplo-self-hosted-doctor.exe`, not
the archive. Optionally add the directory containing it to your user PATH.

These downloads are not publisher-signed or notarized. If macOS, Windows, or
endpoint protection blocks execution, ask your IT team to approve the binary
using your organization's normal software approval process.

## Connect to your installation

Doctor uses the current kubeconfig context. To select a different cluster:

```sh
./zuplo-self-hosted-doctor verify --kubeconfig /path/to/kubeconfig --context my-cluster
```

On Windows, use `.\zuplo-self-hosted-doctor.exe` and your Windows kubeconfig path instead.
Connect to any VPN required to reach the cluster and installation hostnames.
Your Kubernetes credentials need permission to read the installation resources,
including Helm release Secrets and Zuplo Configuration resources. `verify` reads
resources and performs DNS, TLS, and HTTP probes; it does not change the
installation.

For non-default installation names, use `--namespace-system`, `--namespace`, and
`--release`. For an internal certificate authority, use
`--ca-file /path/to/company-ca.pem` to add its CA certificates to system trust.

## Commands

| Command                     | Purpose                                                                                  |
| --------------------------- | ---------------------------------------------------------------------------------------- |
| `verify`                    | Run all verification checks against an installed release.                                |
| `verify --list`             | List check IDs, dependencies, and troubleshooting links without connecting to a cluster. |
| `version`                   | Print the Doctor version and source commit.                                              |
| `--help` or `verify --help` | Show commands or verification flags.                                                     |

`preflight` and `smoke` appear in help but are not implemented. Both exit with
code 3; they do not check prerequisites or deploy a sample project.

## Available verification checks

Run the complete suite with `./zuplo-self-hosted-doctor verify`. To run a particular check,
use `./zuplo-self-hosted-doctor verify --only CHECK_ID`, replacing `CHECK_ID` with an ID
below. Doctor also runs that check's dependencies automatically.

| Check ID                | What it checks                                                                                                                                                                                                                                                                                  |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `helm-release`          | Finds the Helm release and requires its status to be `deployed`. Warns if the image registry password is empty, or the builder password is empty for a non-ECR provider.                                                                                                                        |
| `configuration-present` | Loads `Configuration/default` and requires an account name.                                                                                                                                                                                                                                     |
| `deployments-ready`     | Requires all Deployments in the system and release namespaces to have `Available=True`. Reports pod image-pull, crash-loop, and container configuration errors for unavailable Deployments.                                                                                                     |
| `ingress-reachable`     | Waits for the ingress LoadBalancer address, then probes port 80 with an unknown hostname. Requires the expected Zuplo HTTP 404 default-backend response.                                                                                                                                        |
| `configuration-matches` | Compares the live Configuration with Helm values for the account, deployment subdomain, management hostname and dedicated ingress setting, builder settings, certificate settings, and ingress controllers. Also compares a local values file when `--values` is supplied.                      |
| `management-ingress`    | Finds an Ingress for the management API hostname in the system namespace with the expected shared or dedicated ingress class and a TLS block referencing a Secret.                                                                                                                              |
| `clusterissuer-ready`   | Requires the cert-manager ClusterIssuer `zuplo-cluster-issuer` to be Ready. Issues affecting only deployment certificates produce warnings. Skips when neither management nor deployment certificates use cert-manager.                                                                         |
| `certificates-ready`    | Requires management cert-manager Certificates to exist and be Ready; warns about unreadable or unready deployment Certificates. Includes ACME challenge diagnostics for problems. Skips when neither certificate configuration uses cert-manager.                                               |
| `dns-records`           | Resolves the management hostname and a probe hostname under the deployment wildcard domain, then compares them with the ingress LoadBalancer. Accepts proxy routes verified by expected HTTPS responses; warns about unverified proxies or unexpected targets and fails on missing DNS records. |
| `management-api-tls`    | Checks the management API on port 443 for a trusted certificate valid for its hostname, rejects the HAProxy fallback certificate, and optionally checks the issuer with `--expect-issuer`.                                                                                                      |
| `management-api-auth`   | Requires an unauthenticated `GET /v1/deployments` to return HTTP 401 with `No Authorization Header`. If an API key is available, also requires an authenticated request to return HTTP 200 with a deployment list.                                                                              |
| `builder-config`        | For Docker, checks the registry, Secret name, and nonempty `.dockerconfigjson` in that Secret. For ECR, checks the repository and looks for an IRSA role annotation on `builder-service-account`; warns when workload identity cannot be detected. Does not build or push an image.             |

### Select or skip checks

```sh
# Check deployment health and its Configuration dependency.
./zuplo-self-hosted-doctor verify --only deployments-ready

# Select several checks with comma-separated IDs or repeated flags.
./zuplo-self-hosted-doctor verify --only helm-release,configuration-matches,builder-config
./zuplo-self-hosted-doctor verify --only management-ingress --only dns-records

# Run the suite without the management API authentication check.
./zuplo-self-hosted-doctor verify --skip management-api-auth

# Show the exact dependency list for this version.
./zuplo-self-hosted-doctor verify --list
```

`--only` includes dependencies recursively. `--skip` accepts the same
comma-separated or repeated syntax and takes precedence over `--only`. Unknown
IDs are rejected before connecting to Kubernetes.

A failed check causes dependent checks to be skipped. Warnings, checks that are
not applicable, and explicit `--skip` requests do not automatically block
dependent checks. A dependent check can still need data from a skipped check.
Even a filtered verification run requires Kubernetes API access.

### Compare a local values file

```sh
./zuplo-self-hosted-doctor verify --only configuration-matches --values /path/to/values.yaml
```

The file is an additional comparison against the live Configuration; it does not
replace the installed Helm values. Supply the intended complete values for the
fields listed above: omitted fields use Doctor's chart defaults, so a partial
override file can report differences. This checks selected Configuration fields,
not every Helm value or Secret value.

### Check private certificates and the expected issuer

```sh
./zuplo-self-hosted-doctor verify --only management-api-tls --ca-file /path/to/company-ca.pem
./zuplo-self-hosted-doctor verify --only management-api-tls --expect-issuer '^Company Issuing CA$'
```

`--ca-file` adds PEM CA certificates to system trust for TLS and HTTPS probes,
including proxy-route and authentication checks. `--expect-issuer` is a regular
expression matched against the management certificate issuer's common name. The
two flags can be combined.

### Verify an API key

Without an API key, `management-api-auth` checks only that unauthenticated
requests are rejected. Set `ZUPLO_API_KEY` in the process environment to also
check authenticated access:

```sh
# After supplying ZUPLO_API_KEY through your shell or secret manager:
./zuplo-self-hosted-doctor verify --only management-api-auth

# Read the key from a different environment variable instead:
./zuplo-self-hosted-doctor verify --only management-api-auth --api-key-env MY_ZUPLO_API_KEY
```

The same commands work in PowerShell with `.\zuplo-self-hosted-doctor.exe`; the key must be
available as `$env:ZUPLO_API_KEY` (or the variable selected by `--api-key-env`).
Pass the environment variable's name to `--api-key-env`, not the key itself.
Doctor verifies TLS again before sending the key as a Bearer token. An unset or
empty variable leaves the authenticated request untested, even if the check
passes. Unset the variable to run only the unauthenticated probe.

## Options and defaults

| Flag                           | Default                     | Purpose                                                                                                            |
| ------------------------------ | --------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `--kubeconfig PATH`            | Standard kubeconfig loading | Select the kubeconfig file.                                                                                        |
| `--context NAME`               | Current kubeconfig context  | Select the cluster context.                                                                                        |
| `--namespace-system NAME`      | `zuplo-system`              | System namespace.                                                                                                  |
| `--namespace NAME`             | `zuplo`                     | Release/runtime namespace.                                                                                         |
| `--release NAME`               | `zuplo`                     | Helm release name; also determines the ingress LoadBalancer Service name.                                          |
| `--only IDS`                   | All checks                  | Run selected check IDs and their dependencies.                                                                     |
| `--skip IDS`                   | None                        | Skip selected check IDs.                                                                                           |
| `--wait`                       | `true`                      | Poll checks that support waiting; use `--wait=false` for one attempt.                                              |
| `--no-wait`                    | `false`                     | Run polling checks once; takes precedence over `--wait`.                                                           |
| `--timeout DURATION`           | `0` (per-check defaults)    | Set a positive polling timeout such as `30s` or `5m` for each polling phase. This is not a total command deadline. |
| `--output FORMAT`, `-o FORMAT` | `text`                      | Choose `text` or `json`.                                                                                           |
| `--strict`                     | `false`                     | Exit with code 2 if there are warnings and no failures.                                                            |
| `--verbose`, `-v`              | `false`                     | Write probe diagnostics to stderr.                                                                                 |
| `--values PATH`                | None                        | Also compare a local Helm values YAML file.                                                                        |
| `--expect-issuer REGEX`        | None                        | Require the management certificate issuer common name to match.                                                    |
| `--api-key-env NAME`           | `ZUPLO_API_KEY`             | Environment variable containing the optional API key.                                                              |
| `--ca-file PATH`               | System trust only           | Add private CA certificates from a PEM file.                                                                       |
| `--list`                       | `false`                     | Print the check catalog as text and exit without cluster access.                                                   |
| `--help`, `-h`                 |                             | Show help.                                                                                                         |

By default, `deployments-ready` polls every 5 seconds for up to 5 minutes;
`ingress-reachable` polls every 5 seconds for up to 2 minutes to obtain a
LoadBalancer address and then up to another 2 minutes for the HTTP probe;
`certificates-ready` polls every 20 seconds for up to 10 minutes. `--no-wait`
still performs network requests, which have their own timeouts.

```sh
# Take a single snapshot without polling.
./zuplo-self-hosted-doctor verify --no-wait

# Allow each polling phase up to 30 seconds.
./zuplo-self-hosted-doctor verify --timeout 30s

# Save a machine-readable report and treat warnings as unsuccessful.
./zuplo-self-hosted-doctor verify --output json --strict > doctor-report.json

# Include probe diagnostics, with JSON and diagnostics in separate files.
./zuplo-self-hosted-doctor verify --output json --verbose > doctor-report.json 2> doctor-debug.log
```

## Results and exit codes

Results are `pass`, `warn`, `fail`, or `skip`. A skipped check has not verified
that part of the installation. Text output streams results as checks complete;
JSON output contains the completed report. Warnings and failures include a
diagnostic step and troubleshooting link. Share the version output with support;
review diagnostic output for sensitive details before sharing it.

| Exit code | Meaning                                                                                                |
| --------- | ------------------------------------------------------------------------------------------------------ |
| 0         | No checks failed; warnings are allowed unless `--strict` is used. Skipped checks can still be present. |
| 1         | At least one check failed, unless the Configuration failure below applies.                             |
| 2         | Warnings with `--strict`, with no failures.                                                            |
| 3         | Doctor could not run verification, or `configuration-present` failed. Check the reported error.        |
