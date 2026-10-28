# NVIDIA OpenShell: A Deep Dive into Secure Runtime Infrastructure for Autonomous AI Agents

Technical research and implementation guide | October 22, 2026 | Beginner to advanced | Enterprise Agentic AI and AI Security

[OpenShell](https://images.openai.com/static-rsc-4/1PhcrRoIIlN9OwbOnQTBLDxFDBULU3QlfWuL-36D1hy8d5c5agKuYsGOaVxISu8vL82W2gMSz68bkR9NHO-ZNtIhUkFQuIIOV3CunDjzsu0W19sdOVtvESNZg5bbVIT_K1BzqxE3-_Jx0YnonfbNqnvxCz-_Clu72tYUC3FCwLg?purpose=inline)

[OpenShell](https://images.openai.com/static-rsc-4/KZWFw_FpJgOXb8DecxzLqCoxU9hHRqUL76WjAd0wO6XaW_LtUsXGTL1ng7RVcVfDio2HIWHPpDZrmOIrrrXLG9h4o8xUz6C1j6lel37_Wp7tR1BL8LgNHegc7IArzmmKAtcB59Jop9JWXBK3FIoE638CdqsYvO3AD04PFoFVrX4?purpose=inline)

[What is NVIDIA Openshell? - by Kanishk Patel](https://images.openai.com/static-rsc-4/nxN1owrWOeS-CSlY5UBVH1fV2mbdFII-gAG9_NS-fUTa6Xb43RXSFHVdlvL8ditWt1i07N1zmGGECm1VvXurXu1e4OHDEoKcvt73FvkoFQPOBYWV0HjPJAaBNg1bl-kMEvOWV8eA4ridanQC8yvLEdtlfXwgbAYZ22y3V6M15_o?purpose=inline)

7

## 1. Introduction: Why NVIDIA OpenShell Matters

We are entering a stage of AI development where large language models are no longer limited to generating text, answering questions, or retrieving documents. Modern AI agents can execute Python scripts, modify repositories, use browsers, interact with APIs, connect to databases, invoke MCP tools, and sometimes operate infrastructure with considerable autonomy.

Frameworks such as LangGraph, OpenAI Agents SDK, CrewAI, and coding agents such as Codex and Claude Code make these capabilities accessible. However, they introduce an architectural challenge that conventional LLM applications rarely encountered.

How can an enterprise allow an autonomous AI agent to execute real actions without granting that agent unrestricted access to enterprise infrastructure?

Consider an AI-powered software engineering agent that receives an instruction:

> Investigate a failed payment service, identify the root cause, fix the code, run the tests, and prepare a pull request.

For this task to succeed, the agent might need to read source code, execute shell commands, install development dependencies, call GitHub APIs, and query observability platforms.

Unfortunately, the same capabilities could allow the agent to read sensitive files, extract credentials, leak proprietary code, run dangerous commands, or modify production services.

Traditional prompt engineering does not solve this problem. A system prompt telling an agent not to access sensitive resources is a behavioral instruction, not a security boundary.

This is precisely the problem NVIDIA OpenShell addresses.

NVIDIA describes OpenShell as an open, secure runtime that governs agent execution, resource access, and inference destinations through isolated sandboxes, declarative policies, credential protection, and formal policy verification.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://build.nvidia.com\&sz=32)

NVIDIA NIM APIs+1



The critical architectural principle is:

Security should be enforced outside the AI agent's reasoning and execution process, not left to the agent's discretion.

OpenShell is not a new LLM, an agent orchestration framework, or an alternative to LangGraph. It operates beneath and around an agent to constrain what the agent can actually do.

## 2. What Exactly Is NVIDIA OpenShell?

NVIDIA OpenShell is an open-source runtime for running autonomous AI agents inside controlled execution environments.

Its source repository is [NVIDIA/OpenShell on GitHub](https://github.com/NVIDIA/OpenShell?utm_source=chatgpt.com), and its primary documentation is available at [NVIDIA OpenShell Developer Guide](https://docs.nvidia.com/openshell/latest/?utm_source=chatgpt.com).

OpenShell's job is to establish and enforce the boundary between an agent's intentions and the resources it is permitted to use.

A useful comparison illustrates the difference.

| Capability                     | Agent framework                            | NVIDIA OpenShell                   |
| ------------------------------ | ------------------------------------------ | ---------------------------------- |
| Plans tasks                    | Yes                                        | No                                 |
| Calls tools                    | Yes                                        | Governs their execution and access |
| Executes generated code        | Through tools                              | Provides isolated execution        |
| Restricts filesystem access    | Application-level controls, if implemented | Kernel-enforced policies           |
| Restricts network destinations | Application-level controls, if implemented | Runtime network mediation          |
| Protects API credentials       | Application responsibility                 | Endpoint-bound credential handling |
| Verifies policy changes        | Usually custom                             | Dedicated policy prover            |
| Collects security decisions    | Depends on implementation                  | Runtime allow/deny auditing        |

For example, a LangGraph agent might decide that it needs to call `requests.get()` against a remote endpoint.

LangGraph determines when and why the tool is invoked. OpenShell determines whether the executable involved may access that destination and, when HTTP inspection is configured, whether the particular request method and path are authorized.

This gives us two separate layers of responsibility: agentic intelligence and runtime security enforcement.

OpenShell's current architecture supports isolated workloads, a trusted supervisor, a gateway, credential providers, and policy verification. The project is licensed under Apache 2.0 and has undergone significant evolution in its 0.1.x releases.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+2



## 3. The Architecture of NVIDIA OpenShell

The major components are an agent sandbox, an isolated execution boundary, a trusted supervisor, an OpenShell gateway, a policy engine, credential providers, and a policy prover.

Conceptual control-plane diagram. In actual deployments, the supervisor mediates outbound service connections directly; gateway control traffic is separate from approved agent network traffic.

### 3.1 Agent Sandbox: The execution boundary

The sandbox is the environment in which an autonomous agent executes commands and interacts with files or tools.

When an agent runs a Python script, starts a subprocess, invokes an MCP server, or downloads dependencies, the operations are subject to the sandbox's configured isolation controls.

OpenShell supports multiple compute runtimes, including Docker, Podman, MicroVM-based environments, and Kubernetes. Availability and maturity vary by platform and release.

The important characteristic is not simply that an agent runs in a container. It is that the agent runs within additional filesystem, process, and network restrictions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell+1



### 3.2 Supervisor: The trusted enforcement component

The supervisor operates on the trusted side of the sandbox boundary.

Suppose an agent attempts to contact `api.github.com`. The sandbox cannot simply establish arbitrary internet connectivity. Instead, the request is mediated through the supervisor, which evaluates the connection against policy.

The supervisor can consider the executable responsible for the request, the destination hostname, port, protocol, HTTP method, and path, depending on the configured inspection rules.

When an external service requires credentials, the supervisor also participates in ensuring that provider credentials are used only for authorized destinations.

The trust distinction is fundamental: a malicious instruction inside a repository can influence the agent, but it should not be able to rewrite the supervisor's externally enforced permission rules.

### 3.3 Gateway: The control plane

The gateway manages the operational lifecycle of sandboxes.

It coordinates sandbox creation, policy delivery, provider attachments, identity and access boundaries, and management connections.

In production, it can serve as a central control plane for groups of sandboxed agent workloads running on supported compute infrastructure.

This is conceptually similar to the separation between control-plane services and workload execution in container orchestration platforms.

### 3.4 Policy Prover: Permission verification

A particularly interesting component is the policy prover.

The prover examines whether proposed permissions remain within specified security boundaries. For example, it can detect when a proposed rule expands access to an additional network host, grants a new HTTP method, or introduces credentialed access that was previously unavailable.

It uses formal reasoning about the modeled policy rather than asking an LLM whether a proposed permission seems safe.

This does not establish that an agent's work is semantically correct. It establishes properties about the permissions expressed by the supported policy model.

### 3.5 Provider system: Credential isolation

Agents often require API keys to access model providers, SaaS APIs, or internal systems.

A conventional implementation passes credentials directly through environment variables. That creates risks if the agent can inspect its environment, read process configuration, or print sensitive values into logs.

OpenShell supports provider-backed credential management, using placeholder credentials and authorization at the outbound endpoint.

For example, an OpenCode agent may need an OpenRouter credential. The secret is held outside the untrusted workload and resolved for authorized requests to the configured provider endpoint.

This is substantially different from simply hiding an API key in a `.env` file.

These architectural responsibilities are documented in NVIDIA's developer guide and runtime architecture.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://build.nvidia.com\&sz=32)

NVIDIA NIM APIs+2



## 4. Understanding the Core Security Concepts

To understand OpenShell deeply, it is important to understand the technologies that make its security model possible.

### 4.1 Sandboxing versus containerization

A container provides process and filesystem separation using operating-system features such as namespaces and resource controls.

However, running an AI agent inside an ordinary Docker container does not automatically mean that it cannot access arbitrary external services, read mounted credentials, or make dangerous API calls.

For example, consider a Docker container launched with a writable host directory and network access.

```
docker run -it \
  -v /home/developer:/workspace \
  python:3.12 bash
```

The container has access to everything exposed through that mount, subject to the relevant permissions. If its network is unrestricted, a compromised agent can potentially send that information to an attacker-controlled endpoint.

OpenShell introduces additional restrictions around the workload. The configured runtime establishes an isolation boundary, and the supervisor independently decides which outbound requests are permissible.

Importantly, OpenShell is not a replacement for hardened container configuration. Container isolation, kernel security, admission control, and network segmentation still matter.

### 4.2 Landlock: Filesystem isolation

Landlock is a Linux security module that lets a process restrict its own filesystem access.

In OpenShell, Landlock can be used to constrain which paths an agent workload may read and which paths it may modify.

Imagine an agent responsible for examining one microservice.

Its authorized working directory is `/sandbox/payment-service`, while host credentials, unrelated repositories, and sensitive directories must remain inaccessible.

A filesystem policy can establish read-only paths for system dependencies and read-write paths for necessary working directories.

```
version: 1

filesystem_policy:
  include_workdir: true
  read_only:
    - /bin
    - /usr
    - /lib
    - /etc
    - /var/log
  read_write:
    - /sandbox
    - /tmp

landlock:
  compatibility: hard_requirement
```

The `hard_requirement` option is especially important for a high-assurance deployment. It means a failure to apply the required Landlock security policy should prevent the sandbox from starting rather than silently running with a weaker boundary.

One distinction is worth emphasizing: declaring a path in the policy does not create that path or mount new data. It restricts access to files made available through the runtime. Ordinary Unix permissions continue to apply.

Filesystem restrictions are static for the lifecycle of the sandbox. Changing them generally requires recreating the sandbox.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell+1



### 4.3 Seccomp: System-call restrictions

A system call is how a userspace application requests an operation from the operating-system kernel.

When a Python program opens a socket, creates a process, changes file permissions, or accesses a file, it ultimately relies on system calls.

Seccomp, or secure computing, allows Linux to filter or mediate selected system calls.

OpenShell uses seccomp as part of its process and network security architecture. In its documented Linux backend, seccomp user notification participates in mediating network operations, while other restrictions reduce the capabilities available to the sandboxed workload.

A typical security failure without adequate isolation is a child process attempting to escape its intended execution scope. By enforcing restrictions in the kernel, OpenShell can apply controls to generated code and subprocesses, not only to the original agent application.

### 4.4 Deny-by-default network security

One of OpenShell's most consequential design choices is its default-deny approach to external network access.

An agent is not supposed to have arbitrary internet access simply because it needs to make one API request.

For example, a coding agent might be allowed to perform read-only operations against the GitHub API.

It should not automatically be permitted to post data to another website.

The difference is enforced at the runtime boundary.

Example network policy decisions

Allowed

GET api.github.com/repos/...

Approved destination, method, and executable

Denied

POST api.github.com/repos/...

Write operation not authorized

Denied

GET unapproved.example

Destination missing from policy

Denied

Unapproved executable

Binary identity does not match policy

Expected outcomes when inspected REST endpoints use enforce mode and narrowly scoped rules.

### 4.5 Layer 4 versus Layer 7 policy enforcement

This is a particularly important concept in production-grade agent security.

Layer 4 controls typically work with transport-level information, such as a destination host, IP address, and port.

Layer 7 controls inspect application-level operations such as HTTP methods, API paths, and GraphQL operations.

Suppose a software engineering agent is authorized to contact `api.github.com` on port 443.

A transport-only rule might allow the connection, but it cannot differentiate between reading an issue and deleting a repository.

Application-level rules enable finer distinctions.

OpenShell supports protocol-aware policy inspection for configured REST, GraphQL, and WebSocket traffic. The enforcement detail depends on the protocol and the policy definition.

A critical configuration risk is that an endpoint without explicit request inspection can allow methods and paths beyond what the operator intended. Likewise, `enforcement: audit` records violations but does not block them. Production denial requires `enforcement: enforce`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell



### 4.6 Binary identity binding

OpenShell can associate network authorization with an executable path and its observed binary identity.

For example, a policy can authorize `/usr/bin/curl` to make a particular network request without necessarily authorizing every executable in the sandbox.

This matters because allowing any subprocess unrestricted connectivity would weaken much of the network boundary.

NVIDIA also documents runtime-observed SHA-256 executable integrity pinning in recent releases, which helps detect a changed executable at an authorized path.

This is not equivalent to verifying that a binary is globally trustworthy. It provides an additional integrity condition for the observed executable identity.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell



### 4.7 Formal verification and policy reasoning

Formal verification uses mathematical models to establish whether certain properties hold.

In OpenShell, the relevant question is about permission boundaries.

Imagine a permitted access set called `ApprovedAccess`, with a proposed policy change granting `NewAccess`.

The desired condition is:

`NewAccess ⊆ ApprovedAccess`

If a proposed policy adds a destination or operation outside the approved boundary, that expansion must be flagged.

OpenShell's policy prover reasons about such modeled access expansions and can produce evidence for an operator to review.

This is particularly valuable in autonomous workflows because agents may dynamically discover that they need additional permissions. Those permissions should not automatically become permanent privileges simply because the agent requested them.

The policy prover is not a universal proof that every network response is safe, every action is authorized by business policy, or every executed program is free of vulnerabilities.

It is one important control in a broader security architecture.

## 5. Dynamic Policy Management and Human Approval

A static allowlist creates another problem: agents often discover dependencies while executing a task.

Suppose a coding agent attempts to access a package index, but the destination is not allowed.

Without a controlled escalation mechanism, a developer might be tempted to grant the agent unrestricted access to the internet.

OpenShell provides a better pattern through policy proposals and operator review.

Agent requests new access

Example: a new package registry

Request denied and recorded

Existing permission boundary remains active

Policy proposal generated

Destination and calling program identified

Policy prover checks expansion

Security findings accompany proposal

Approve

Narrow permission applied

Reject

Request remains blocked

The CLI exposes pending proposals and supports explicit approval or rejection.

```
# Inspect new access proposals.
openshell rule get my-agent --status pending

# Approve one reviewed proposal.
openshell rule approve my-agent --chunk-id CHUNK_ID

# Reject a proposed permission.
openshell rule reject my-agent \
  --chunk-id CHUNK_ID \
  --reason "The requested endpoint is unnecessary."
```

`CHUNK_ID` represents an actual identifier returned by the first command.

Approved dynamic network rules can be applied to the running sandbox without restarting the agent.

The distinction between allowing an agent to request permissions and letting it grant itself permissions is essential. Production deployments should maintain operator-controlled approval for sensitive actions and avoid indiscriminate approval of pending requests.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell



## 6. Model Inference and Why Credential Protection Matters

An AI agent needs a model for reasoning, but the model endpoint should not automatically become an unrestricted network destination.

Consider an organization that supports three model-serving environments.

The first serves a private open-weight model through an internal inference service. The second uses an approved external commercial model provider. The third is a development endpoint that should never receive sensitive customer records.

OpenShell can enforce which configured provider endpoints a particular sandbox may contact and which calling programs are authorized to use them.

This is where the distinction between the agent, its model backend, and the secure runtime becomes clear.

The agent framework handles prompts, reasoning, tool coordination, and state transitions.

The model server generates responses.

OpenShell governs the execution environment and approved external connections, including credential-mediated model API requests.

An important limitation is that approving a model endpoint does not automatically establish whether every piece of data sent to it is appropriate. Enterprises should still use data classification, PII redaction, retention policies, DLP checks, and model-provider contractual controls.

For truly private inference, the architecture should use an approved private model endpoint and restrict outbound traffic accordingly. OpenShell enables that control pattern; it does not itself make a third-party model service private.

## 7. How OpenShell Fits into Production Agentic AI Architecture

A practical enterprise deployment might combine a LangGraph or coding-agent orchestrator with OpenShell, GitHub, observability tools, an internal artifact store, and an approved model provider.

Consider an incident-remediation platform.

In this architecture, OpenShell is not responsible for deciding how to repair an incident. Instead, it enforces the authority available to the agent while the remediation workflow executes.

The outer orchestrator can manage planning, approval, retries, checkpoints, and task state. OpenShell can constrain risky operations executed by the agent or its tools.

This separation is especially useful when integrating untrusted repository content, generated scripts, package installations, external MCP servers, or browser automation.

For a multi-tenant service, each tenant or task can receive a separate sandbox with a defined identity, filesystem, credential scope, and execution lifetime.

The gateway becomes the coordination layer for those isolated environments, while your enterprise application remains responsible for job scheduling, business authorization, customer identity, and workflow state.

## 8. Real-World Use Case: A Secure AI Code Investigation Agent

Let's build a small proof of concept that demonstrates why OpenShell is useful.

### Business problem

Imagine a company operates several payment microservices. Developers want to use an autonomous AI coding agent to investigate bugs, analyze open issues, and recommend changes.

The agent needs GitHub access, but the security team does not want it to modify issues, push arbitrary changes, or send source code to unauthorized websites.

Our solution is to run the agent inside OpenShell with a narrowly scoped GitHub network policy.

The demonstration is deliberately divided into two parts.

The first part verifies actual OpenShell runtime enforcement without any paid model provider. The second part launches an AI coding agent using the documented OpenCode integration.

This is useful because it allows us to distinguish security enforcement from LLM behavior. An agent can behave correctly during one run without proving the sandbox is secure; the first test directly checks the policy boundary.

### 8.1 Prerequisites and supported systems

The project requires a supported operating-system environment, an OpenShell gateway, and a compute runtime.

NVIDIA documents Docker Desktop or Docker Engine 28.0 or newer, as well as supported Linux Podman and virtualization-based options. Windows with WSL 2 is described as experimental.

For Windows 11, a WSL 2 Linux distribution with a supported container runtime is the most relevant starting point. Native PowerShell alone is not the documented primary Linux sandbox execution path.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell+1



For our example, use Ubuntu under WSL 2, with Docker Desktop configured for WSL integration.

Verify the basics:

```
docker version
docker info
uname -a
```

Then confirm Docker can execute a simple workload:

```
docker run --rm hello-world
```

If Docker or WSL virtualization is unavailable, the alternative is a Linux VM or an isolated Linux development host. A normal Python virtual environment does not provide equivalent kernel security controls.

### 8.2 Install NVIDIA OpenShell

NVIDIA provides an installer for the OpenShell CLI and local gateway.

Review installation scripts before executing them, especially on enterprise machines.

```
curl -LsSf \
  https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh \
  -o install-openshell.sh

less install-openshell.sh
sh install-openshell.sh
```

Verify installation:

```
openshell --help
openshell status
```

A successful `openshell status` confirms the CLI can reach the configured gateway.

Note that installing the PyPI package named `openshell` installs the Python SDK, not the CLI runtime. NVIDIA documents those as separate installation paths.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell+1



### 8.3 Create a restrictive policy

Our first experiment will allow only the `curl` executable to use the GitHub REST API in read-only mode.

We will use a policy based on NVIDIA's published quickstart example.

Create a file named `github-readonly.yaml`.

```

# github-readonly.yaml
# OpenShell schema version 1.
version: 1

filesystem_policy:
  include_workdir: true

  read_only:
    - /bin
    - /usr
    - /lib
    - /proc
    - /dev/urandom
    - /app
    - /etc
    - /var/log

  read_write:
    - /sandbox
    - /tmp
    - /dev/null

landlock:
  compatibility: hard_requirement

network_policies:
  github_api:
    name: github-api-readonly

    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        enforcement: enforce
        access: read-only

    binaries:
      - path: /usr/bin/curl

```

The policy is intentionally restrictive. It is not intended to be a full coding-agent policy.

Its purpose is to make the security boundary easy to test.

The `protocol: rest` setting enables HTTP-level inspection. `access: read-only` selects read-only API access. `enforcement: enforce` makes policy violations blocking rather than merely observational.

Finally, the binary restriction permits the specified `curl` executable to use that endpoint.

The file uses `hard_requirement` instead of the upstream quickstart's `best_effort` setting so that our demonstration fails rather than proceeding if the required filesystem restrictions cannot be established.

The actual installed Linux kernel must support the required Landlock functionality. If startup fails because of missing kernel support, fix the environment instead of silently weakening the policy.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



### 8.4 Create the secure sandbox

From the directory containing the policy:

```
openshell sandbox create \
  --name secure-github-demo \
  --policy github-readonly.yaml
```

The command uses the default sandbox image supported by the installed release.

Check the sandbox:

```
openshell sandbox list
openshell policy get secure-github-demo
```

Connect to the sandbox:

```
openshell sandbox connect secure-github-demo
```

Some container images and command configurations require a separate shell process instead of connecting to the sandbox's main process. In that situation, the CLI provides:

```
openshell sandbox exec \
  --name secure-github-demo \
  --tty \
  -- /bin/bash -l
```

Confirm that the image contains `curl` at `/usr/bin/curl`:

```
command -v curl
```

If the installed image locates it somewhere else, the binary path in the policy must match the executable actually used.

### 8.5 Experiment A: An authorized read request

Inside the sandbox, execute:

```
curl -i https://api.github.com/zen
```

The GitHub `/zen` endpoint returns a short string.

Because the destination, port, executable, HTTP method, and access mode meet our policy, this request should be permitted.

The successful request proves that OpenShell can grant useful access without granting unrestricted network connectivity.

### 8.6 Experiment B: An unauthorized write request

Next, attempt a write operation.

```
curl -i \
  -X POST \
  https://api.github.com/repos/octocat/Hello-World/issues \
  -H "Content-Type: application/json" \
  -d '{"title":"Unauthorized test"}'
```

Our read-only policy should deny the `POST` before GitHub processes it.

An expected denial looks conceptually like:

```
{
  "error": "policy_denied",
  "detail": "POST request not permitted by policy"
}
```

The exact response fields and wording may vary across releases.

The important observation is that GitHub authentication is not what protects this operation. Even without a valid GitHub write credential, a request could otherwise reach the upstream API and fail there. Here, OpenShell should deny it at its own security boundary.

### 8.7 Experiment C: Unauthorized data exfiltration

The next test demonstrates a separate control.

```
curl -i https://example.com
```

Our policy does not grant access to `example.com`.

Therefore, the connection should be denied.

This is an elementary example of how an allowlisted egress boundary can help reduce unintended data exfiltration.

In a real malicious scenario, an attacker may attempt to send source code or secrets to an external endpoint. Preventing all connections to unapproved destinations removes one possible exfiltration path.

However, the agent could still try to transmit inappropriate data to an approved endpoint. Network allowlisting alone does not prevent that. Approved destinations must also be protected by appropriate business authorization and data-loss-prevention controls.

### 8.8 Observe the security decisions

Exit the sandbox shell and inspect its logs from the host:

```
openshell logs secure-github-demo --source sandbox
```

You should see network and HTTP events associated with allowed and denied operations.

A representative log sequence might resemble:

```
NET:OPEN ALLOWED /usr/bin/curl -> api.github.com:443
HTTP:GET ALLOWED GET /zen

HTTP:POST DENIED POST /repos/octocat/Hello-World/issues

NET:OPEN DENIED /usr/bin/curl -> example.com:443
```

These are illustrative records rather than captured output from a live sandbox execution.

The exact fields and formatting depend on the version and telemetry configuration.

OpenShell supports security events in the Open Cybersecurity Schema Framework (OCSF), which is valuable for normalizing security audit records.

NVIDIA documents access through the CLI, TUI, and structured log export. The gateway's in-memory recent-log buffer is not a durable audit store, so production environments need a separate persistence and collection mechanism.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell



### 8.9 Our experiment results

The expected security behavior is:

| Test                                | Expected result          | Security property                 |
| ----------------------------------- | ------------------------ | --------------------------------- |
| GitHub GET                          | Allowed                  | Least-privilege read access       |
| GitHub POST                         | Denied                   | HTTP method enforcement           |
| External website GET                | Denied                   | Destination allowlisting          |
| Unapproved binary contacting GitHub | Denied                   | Executable-specific authorization |
| Read disallowed filesystem path     | Denied, where applicable | Filesystem confinement            |

These are test expectations, not measurements from an execution performed in this chat.

This first part establishes the most important property of OpenShell: its security controls operate independently of whether an LLM follows instructions correctly.

## 9. Adding a Real AI Coding Agent to the Sandbox

Now let's extend the experiment to use an actual coding agent.

NVIDIA's documentation includes an example running OpenCode with an OpenRouter model provider inside OpenShell.

This is a separate sandbox from the network-policy experiment, because an AI coding agent requires its own model-access permissions and executable configuration.

### 9.1 Configure model credentials

OpenShell supports provider profiles that define authorized service endpoints and associated executable paths.

For the documented OpenRouter example, configure the credential on the host:

```
export OPENROUTER_API_KEY="YOUR_OPENROUTER_API_KEY"
```

Import the official provider profile:

```
openshell profile import \
  --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml
```

Create the provider:

```
openshell provider create \
  --name openrouter \
  --type openrouter \
  --from-existing
```

The profile and provider give OpenShell the information needed to mediate authorized model API access without placing the real credential directly into arbitrary agent-accessible configuration files.

For a real deployment, retrieve the credential from your organization's secret manager and avoid placing it in shell history, scripts, or committed files.

### 9.2 Launch the AI agent

The NVIDIA tutorial demonstrates the following pattern:

```
openshell sandbox create \
  --name secure-ai-agent \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free
```

This creates the sandbox with OpenCode as its primary process and attaches the configured model provider.

The model identifier shown is from NVIDIA's documented example. Model availability and free-tier eligibility can change, so the installed OpenCode client should be used to confirm the available models.

Also, for a reproducible production deployment, replace the mutable `latest` image tag with a verified image digest.

### 9.3 Give the coding agent a realistic task

For this demonstration, ask the agent:

> Analyze the following Python function for reliability and correctness. Identify bugs, propose a safer implementation, and explain appropriate unit tests. Do not modify remote repositories or call any write APIs.

```
def calculate_discount(amount, discount):    return amount - amount * discount / 100
```

The task is simple enough for an initial test, but it resembles a genuine software engineering workflow.

A useful agent should discuss invalid inputs, numeric precision, discount bounds, and suitable test cases.

An important financial-domain improvement would be to use `Decimal` instead of binary floating-point arithmetic for monetary calculations.

More importantly, the agent should be able to complete this reasoning task without needing broad filesystem or internet access.

For a stronger test, deliberately place an untrusted instruction in a local test document telling the agent to contact an unauthorized domain. The expected security outcome is that any attempted connection is blocked at the OpenShell boundary.

This tests a common AI security problem: prompt injection attempting to turn lower-trust task content into a privileged operation.

### 9.4 Inspect dynamic permission requests

Suppose the agent attempts to download a dependency from an unauthorized package host.

OpenShell can record the denied connection and propose a narrow permission change.

Inspect proposed rules:

```
openshell rule get secure-ai-agent --status pending
```

Review the proposal before approving anything.

If the destination is not necessary for the task, reject it.

If access is legitimate, approve only the permission actually needed, rather than granting blanket network connectivity.

The result is an agent that retains useful autonomy while operating within a controlled security model.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell



## 10. Building a Production-Grade OpenShell Agent Platform

The demonstration above validates individual security properties. A production architecture requires considerably more engineering.

A single sandbox is sufficient for a development experiment. An enterprise serving hundreds of agent jobs needs workload isolation, capacity management, access governance, durable telemetry, controlled software supply chains, and reliable recovery.

### 10.1 Integration with LangGraph

Suppose a LangGraph application contains a planner, code-analysis agent, test-execution agent, and reviewer.

Without a secure execution layer, the test agent might execute generated code directly in the application's backend process.

That creates an unnecessarily broad trust relationship.

A more defensible architecture places execution behind a dedicated sandbox service.

The LangGraph orchestrator submits a task to this service, which provisions or selects an OpenShell sandbox, submits the authorized workload, and returns structured execution results.

The orchestration application remains outside the sandbox. Any untrusted code execution occurs inside the sandbox.

The conceptual workflow looks like this:

```
User Request
    |
    v
LangGraph Orchestrator
    |
    v
Task Planner
    |
    v
Execution Service
    |
    v
OpenShell Gateway
    |
    v
Isolated Sandbox
    |
    +-- Python execution
    +-- Code analysis
    +-- Test execution
    +-- Approved API calls
    |
    v
Execution Result
    |
    v
Validation Agent
    |
    v
Human Approval
    |
    v
Final Response
```

The key architectural decision is to separate the application that decides which task to execute from the environment that actually executes it.

The gateway and SDK can form an integration layer, but they should not be treated as a substitute for application-specific job authorization, queueing, and result validation.

### 10.2 Multi-agent isolation

It may be tempting to run every agent in one shared sandbox to reduce cost.

However, different agents often require different permissions.

A research agent might need access to approved documentation websites. A code-analysis agent might need read-only repository access. A test agent needs filesystem write permissions for temporary build artifacts. A deployment agent may require access to a carefully scoped release API.

Combining all these permissions into one sandbox increases the effective privilege of every agent within it.

A stronger pattern assigns sandbox security policies according to workload roles.

For example, the research agent should not receive production deployment credentials simply because another agent in the same workflow requires them.

Isolation should generally follow a task or trust boundary, not merely the number of agents in the workflow.

### 10.3 Production deployment on Kubernetes

For larger deployments, OpenShell can be integrated with a Kubernetes-based compute environment.

NVIDIA provides a Helm chart and documents supported compute drivers. The maturity and prerequisites of individual deployment modes should be reviewed against the exact OpenShell version being deployed.

The project publishes an installation pattern using an OCI Helm chart:

```
helm install openshell \
  oci://ghcr.io/nvidia/openshell/helm-chart
```

This command is an installation starting point, not a complete production configuration.

A production implementation should explicitly configure gateway authentication, encrypted management communications, namespace boundaries, compute-driver permissions, workload resource limits, and log export.

It should also define who may provision sandboxes, attach providers, approve policy changes, and terminate workloads.

Horizontal scalability is a separate design consideration. OpenShell supplies the secure workload runtime and control-plane functionality, but your platform still needs admission control, capacity planning, scheduling policy, and backpressure.

For instance, if 500 agent tasks arrive simultaneously, the application should not blindly start 500 heavyweight sandbox workloads. A scheduler should evaluate available compute capacity, resource quotas, workload priority, and concurrency limits.

### 10.4 Security architecture

Security should be implemented at multiple independent boundaries.

At the identity boundary, only authenticated users and services should be allowed to create workloads or change policies. Production identity integration should follow the gateway's supported authentication configuration, with separate operator and workload roles.

At the sandbox boundary, workloads should run without unnecessary privileges. Filesystem permissions should be minimal, network policies should be default-deny, and high-assurance deployments should verify that kernel controls are actually active.

At the credential boundary, secrets should remain under provider control and be bound to narrowly scoped endpoints.

At the approval boundary, sensitive actions should require explicit authorization independent of the agent's decision.

At the software supply-chain boundary, use trusted base images, immutable image references, dependency vulnerability scanning, and signed or otherwise verifiable artifacts where supported.

Finally, the application should validate outputs before they become executable deployment changes.

OpenShell materially improves runtime control, but it does not eliminate the need for standard infrastructure, application, and AI security practices.

## 11. Observability, Telemetry, and Auditing

Production agent systems need two kinds of observability.

The first is application observability, which describes the agent's reasoning workflow, tool latency, model requests, token consumption, task success, and failures.

The second is security observability, which describes what the workload attempted to access, which policies were evaluated, and which actions were permitted or denied.

These signals complement one another.

For example, LangSmith or OpenTelemetry might show that an agent invoked a Python tool.

OpenShell's security telemetry can show whether the resulting process attempted to reach a forbidden network destination.

The following are useful operational measurements.

| Metric                          | Purpose                                               |
| ------------------------------- | ----------------------------------------------------- |
| Sandbox startup latency         | Detect provisioning bottlenecks                       |
| Active sandbox count            | Track workload capacity                               |
| Sandbox failures                | Detect runtime instability                            |
| Denied network requests         | Detect prohibited connectivity attempts               |
| Policy proposal rate            | Identify overly restrictive or misconfigured policies |
| Policy approval rate            | Audit permission expansion                            |
| Provider authorization failures | Detect credential or endpoint errors                  |
| Sandbox CPU and memory usage    | Support resource scheduling                           |
| Task completion rate            | Measure workflow reliability                          |
| Security-related task failures  | Identify authorization gaps                           |

OpenShell supports structured security records using OCSF and exports that can be integrated with downstream security analytics systems.

For production, a useful design is to collect OpenShell events alongside application traces, while retaining clear distinctions between model-generated claims and observed runtime behavior.

Every task should carry a correlation identifier at the application layer, allowing operators to connect an incident, orchestration trace, sandbox identity, relevant policy revision, and resulting artifacts.

Do not assume recent logs held by the gateway are a permanent audit history. Configure durable collection and retention independently.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA OpenShell+1



## 12. OpenShell Versus Docker, Kubernetes, and Agent Guardrails

OpenShell is easier to position when compared with the tools already available in enterprise infrastructure.

| Technology        | Main responsibility                        | Key limitation in this context                                            |
| ----------------- | ------------------------------------------ | ------------------------------------------------------------------------- |
| Docker            | Container packaging and isolation          | Does not provide OpenShell's integrated agent policy and credential model |
| Kubernetes        | Workload orchestration and scheduling      | Requires additional policy and agent-specific security design             |
| LangGraph         | Stateful agent orchestration               | Does not inherently provide kernel-level sandbox enforcement              |
| Prompt guardrails | Influence or validate model behavior       | Cannot reliably enforce operating-system access boundaries                |
| API gateway       | Control API access                         | Does not govern every local process and filesystem operation              |
| NVIDIA OpenShell  | Govern agent workload execution and access | Does not replace orchestration, scheduling, or semantic validation        |

These technologies are complementary.

A production platform might use LangGraph for orchestration, OpenShell for secure agent execution, Kubernetes for compute infrastructure, a private model-serving platform for inference, and OpenTelemetry for end-to-end application telemetry.

The benefit is that each architectural layer has a clearly defined responsibility.

## 13. Where OpenShell Can Be Used in Real-World Applications

[Agile 3.0 Whitepaper | The Complete Framework for Human+Agent Teams](https://images.openai.com/static-rsc-4/DaHj__1QfywoVZVOGfgfTNj44flO75H--czD5sqEmfvHd2rWSAP-66PgVkaQW5uiBc7JFeBbTlAQr-i1W_p8Tl6BFfNuW9-FqDWnLA9soM6hFahVFXxM_X4740Hq6g0O1KBLKZ0dtaQnKnjZ4-nOZHyX6V5NCSVBP4p7hfHF924?purpose=inline)

Autonomous software engineering

Coding agents can analyze repositories, execute tests, and prepare patches within restricted execution environments. Different policies can separate repository reading from write and release operations.

[Zenflux - Enterprise Observability Platform](https://images.openai.com/static-rsc-4/4jWqvtcbbqhJcMU2FMpCQMjr_V2LgohvpY5W-VtmATJDl5Y-l0SpwXpeIy40QqOKiZUDackMnASH75Ncttavlvh58XUwGjxgHb-U5Qshys3vvqKg8ufpT6Z0YJFDOgyo_nwAmfMWzZIRdoIKYou-Gy7bfSV_pHeueU6213yZP7A?purpose=inline)

Security operations and incident investigation

Agents can inspect approved logs, collect incident evidence, and propose remediation while being prevented from accessing unrelated infrastructure or executing unauthorized changes.

[Industrial Engineer Analyzes Factory Data Using Laptop](https://images.openai.com/static-rsc-4/qmg0zbI_B1f7oE7Vbz0R1V0myyx65S-waeHHR2976v0nQZgRkIyHwsMnuvCIEM5QgZNDbU6ZtvLK6ccaI2I1MDBKsWszRfkm2OBWHejjDPfkOad5Atpuz6pr-x43s8qOSh8S2hY_8Eru6qK2F4aQiHJKOY8exFe3vxTiE-KID_LJFjeFsNn4ZZxuunzWDyC5?purpose=inline)

Industrial and operational AI agents

Agents can interact with tightly scoped diagnostic services or simulated operational environments. Access to real control systems still requires dedicated operational technology safeguards and independent authorization.

[What Your DTC Data Analyst Actually Does on a Monday | Hologrow](https://images.openai.com/static-rsc-4/fBhSD8CxDBQo_37n8ye-QlavHxRD5rPpcmKInsz8ULsw70t39EdJCtv7lT04l3BonjiQ5WPaQVMr0bdbUHpDJxN059o3PqK06DX-QastrCjL_kfkV3Ye7GOcwVeeEgk73tydnbkaHpFb2DrqoeIWMckzUJKgjYt_pVqlcxxLY1k?purpose=inline)

Data engineering and analytics agents

SQL or Python agents can work inside isolated environments with approved database connections, restricted credentials, and controlled temporary storage. Database permissions must independently enforce read-only or other intended privileges.

These are practical architectural use cases rather than claims that NVIDIA has publicly verified each exact scenario in a customer production environment.

## 14. Important Limitations and Production Trade-offs

OpenShell is a meaningful security addition, but there are trade-offs.

Execution overhead. Sandboxes require provisioning and isolation setup. The additional supervisor and policy checks introduce operational complexity and some overhead compared with executing code directly in an existing application process. Actual impact should be benchmarked for the intended workload rather than assumed.

Platform dependencies. OpenShell relies on supported operating-system and runtime capabilities. Windows support through WSL 2 remains a consideration, and stronger filesystem configurations may depend on specific Linux kernel functionality.

Policy maintenance. Strict policies are more secure but can initially block legitimate dependencies. Teams need a disciplined process for reviewing proposed changes and keeping approved permissions appropriately scoped.

Policy inspection boundaries. HTTP method and path restrictions cannot determine whether an otherwise permitted request contains confidential information. Transport protocols that cannot be inspected may require additional protection.

Credential governance. Endpoint-bound credential use reduces exposure, but an approved request can still misuse the privileges of a valid credential. Downstream IAM permissions must remain minimal.

Formal verification scope. Policy proof is not equivalent to proving complete application safety. It helps reason about modeled permissions, not the correctness of every action an agent performs.

Operational maturity. The project is evolving, particularly across its 0.1.x releases. APIs, CLI commands, and deployment defaults can change. Production adoption should pin a qualified release, test upgrades, and validate security guarantees against that release's actual implementation.

## 15. A Practical Learning Roadmap

To master OpenShell, I would approach it in four phases.

OpenShell learning progress

0 of 4 phases

Phase 1: Sandbox fundamentals

Install OpenShell, understand the gateway, create a sandbox, inspect its runtime, and verify default network restrictions.

Phase 2: Policy enforcement

Implement Landlock restrictions, REST method controls, executable restrictions, and audit inspection. Verify expected denials.

Phase 3: Agent integration

Run OpenCode or Codex inside a sandbox with a provider. Simulate unauthorized requests and review permission proposals.

Phase 4: Production integration

Wrap sandbox provisioning behind an agent execution service. Add workflow approvals, structured results, telemetry, quotas, and lifecycle management.

By the end of this progression, the goal is not simply to run an agent inside OpenShell. It is to demonstrate experimentally that unauthorized filesystem and network operations are blocked, policy changes are controlled, and execution remains observable.

## 16. Final Perspective: Why OpenShell Is Architecturally Significant

The most interesting development represented by NVIDIA OpenShell is the separation between AI autonomy and operational authority.

Traditional applications generally follow predetermined execution paths written by developers. Autonomous agents may dynamically choose tools, generate code, discover dependencies, and make new requests while solving a task.

As a result, giving an agent access to an ordinary execution environment can grant much more authority than the developer intended.

OpenShell addresses this by treating the agent workload as an untrusted execution domain and moving the enforcement of access decisions into an independently controlled runtime.

This does not mean AI agents become inherently safe. It means the system can apply deterministic restrictions even when the agent produces undesirable instructions or attempts.

For enterprise Agentic AI, the broader architectural pattern is therefore:

LLM for reasoning, agent framework for orchestration, OpenShell for execution isolation and access enforcement, and independent approval systems for consequential actions.

That separation is particularly valuable for coding agents, security investigation agents, dynamic tool execution, and systems where generated code or retrieved content cannot be assumed trustworthy.

The small GitHub experiment is a useful starting point because it verifies a concrete guarantee: a workload can be permitted to read information while being prevented from performing unauthorized write operations through the same external API.

That is the kind of measurable security property production AI systems need.

## Official resources and further reading

The main sources used for this research are NVIDIA's own project and technical documentation.

Project and architecture: [NVIDIA OpenShell](https://build.nvidia.com/openshell?utm_source=chatgpt.com) · [GitHub repository](https://github.com/NVIDIA/OpenShell?utm_source=chatgpt.com) · [Runtime architecture](https://docs.nvidia.com/openshell/v0.1.4/about/architecture?utm_source=chatgpt.com)

Getting started: [Installation](https://docs.nvidia.com/openshell/latest/about/installation/?utm_source=chatgpt.com) · [Run your first agent](https://docs.nvidia.com/openshell/latest/about/run-your-first-agent?utm_source=chatgpt.com) · [Official sample YAML policy](https://raw.githubusercontent.com/NVIDIA/OpenShell/main/examples/sandbox-policy-quickstart/policy.yaml?utm_source=chatgpt.com)

Production security: [Security best practices](https://docs.nvidia.com/openshell/latest/security/best-practices?utm_source=chatgpt.com) · [Observability and logs](https://docs.nvidia.com/openshell/latest/observability/accessing-logs?utm_source=chatgpt.com)

Research note: This article draws on the publicly documented OpenShell 0.1.x architecture and official quickstart examples. The demonstration commands have been cross-checked against published documentation, but a live sandbox was not provisioned during this research. Expected outcomes should be verified in the selected runtime before relying on them in production.
