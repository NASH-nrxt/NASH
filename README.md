# NASH on CyberGym

*An AI Harness powered by self-optimizing security Skill Tree*

**English** · [中文](README_zh.md)

---

## Abstract

NASH generated 1,501 PoCs across the 1,507 real-world vulnerability tasks in [CyberGym](https://github.com/sunblaze-ucb/cybergym). Of these, 1,377 were fixed-clean PoCs, yielding a strictly verified success rate of 91.4%.

CyberGym Level 1 requires an agent to generate one final PoC using only the vulnerability description and the pre-patch source repository. The PoC must trigger the vulnerability on the vulnerable target without triggering it on the fixed target.

## Background

CyberGym evaluates whether an agent can reproduce real-world vulnerabilities from a vulnerability description and a pre-patch source tree. It contains 1,507 real vulnerabilities from the ARVO and OSS-Fuzz families and is one of the strongest public benchmarks for evaluating agent behavior through reproducible outcomes.

## Method Overview

NASH is a unified security-task Harness for AI-enabled offense and defense. It integrates specialized Skills, task orchestration, evidence management, trusted verification, and experience feedback into an auditable execution loop. Its core principle is “verifiable self-evolution”: transferable experience is distilled from controlled task trajectories and used to continuously improve Skills and execution strategies only after leakage checks and validation gating, creating a closed loop that strengthens security tasks from execution and judgment to verification and experience accumulation.

### Skill Tree Routing

NASH compresses transferable vulnerability-reproduction experience from cross-task trajectories into a hierarchical Skill Tree. The root node stores general reproduction discipline, while lower-level nodes are organized by input format, vulnerability mechanism, and reproduction workflow. Based only on the public vulnerability description and task type, the Skill router selects at most two applicable nodes and combines them with the root node to form the effective skill for the current task.

### Building the Base NASH Skill Tree

The base NASH Skill Tree is built before the formal evaluation. The system trains on and analyzes execution trajectories generated from security-task datasets, distilling transferable reproduction strategies, routing features, and risk-avoidance rules from successful and failed examples to progressively build and optimize the Skill Tree. Candidate updates contain no specific answers and must pass leakage checks and validation-set gating; updates that do not produce a strict improvement are rolled back. During the CyberGym evaluation, the system does not retrieve historical PoCs, patches, or raw cross-task trajectories. This mechanism draws on the trajectory-driven skill-optimization approach of [Microsoft SkillOpt](https://github.com/microsoft/skillopt), extending a single-document skill into a routable, hierarchical skill package.

### Isolated Agent Execution

For each task, the solving agent runs in an isolated task-specific container. It reads the vulnerability description, pre-patch source code, and selected Skill, then performs source localization, input-format reconstruction, candidate PoC construction, vulnerable-side validation, and final submission. Fixed-side verification is performed independently by the host evaluator.

Successful experience and failure patterns produced during a task are not exposed directly to later agents as searchable cross-task sample memory. Instead, they enter the offline Skill Tree update process. The system periodically aggregates recurring reproduction obstacles, misclassification patterns, and effective operations from multiple task trajectories, abstracts them into task-independent candidate Skills, and updates the Skill Tree. In this way, cross-task experience is incorporated offline as general-purpose skills.

The goal is to turn transferable security-research experience into stable process priors, helping the agent identify input boundaries, length/count/offset relationships, format constraints, sanitizer trigger conditions, and both-crash risks more effectively.

## Benchmark Results

| Metric | Value |
|---|---:|
| Tasks | 1,507 |
| Verified successes | 1,377 |
| Unsuccessful tasks | 130 |
| Final-submission success rate | 91.37% |

Following review by the CyberGym team, exit code `71` was confirmed to indicate an out-of-memory (OOM) condition and therefore does not count as a successful crash. Four tasks previously counted as successful are affected: `verified_success` has been adjusted from 1,381 to 1,377, and the final-submission success rate from 91.64% to 91.37%.

| Outcome | Count | Description |
|---|---:|---|
| `verified_success` | 1,377 | The final PoC triggers a qualifying crash in the vulnerable image but not in the fixed image. |
| `unsuccessful` | 106 | Verification completed, but the final PoC did not meet the strict success criterion: the vulnerable image did not produce a qualifying crash, or the fixed image produced a non-clean result. |
| `incomplete_verification` | 18 | The selected final PoC returned exit code 0 or 300 on the vulnerable image, so fixed-side verification was not performed. |
| `no_final_submission` | 6 | The agent did not produce a valid final PoC. |
| **Total** | **1,507** | |

Task IDs were inadvertently exposed through agent-visible target image names and sometimes influenced the agent's reasoning, but our trajectory audit found no evidence that they directly supplied a solution or enabled bypassing source-code analysis and dynamic verification; all agent-visible task identifiers will be anonymized in future submissions.

## Evaluation Setup

The experiment follows the official CyberGym Level 1 constraints and the requirements described in the [CyberGym FAQ](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md).

| Item | Setting |
|---|---|
| Model | DeepSeek-V4-flash-0731 |
| Trials | One valid run per task |
| Per-task wall-clock limit | 240 minutes |
| Final PoC policy | Only one `final_poc` is retained as the final submission for each task |
| Scoring | Final-submission metric: the final PoC must trigger the vulnerable target and must not trigger the fixed target |

### Task Inputs and Dynamic Environment

For each task, the agent can access:

- **Level 1 inputs:** a textual vulnerability description and the pre-patch source code, together with task-relevant Skills when available: `description.txt` and `repo-vul.tar.gz`.
- **Dynamic execution environment:** a cleaned vulnerable image for local execution, debugging, and vulnerability reproduction. The image is derived from the vulnerable image in the official dataset, with leakage sources such as `/src/**/.git` and `/tmp/poc` removed.
- A task-specific workspace and submission interface.

The agent cannot access:

- `repo-fix.tar.gz`;
- patch diffs;
- the reference PoC;
- the fixed image or fixed-side verification feedback;
- original Git history, upstream PRs, commits, issues, changelogs, or release notes, except for project files like CHANGELOG/NEWS that originally come with the task source code;
- another task's workspace, trajectories, PoCs, or verification results.

Some evaluation logs contain attempts to access Git or patch artifacts, but no information was obtained.

The dynamic environment follows the CyberGym FAQ guidance: when a vulnerable image is provided to the agent for dynamic analysis, this setting must be disclosed in the public materials, and leakage sources such as `/src/**/.git` and `/tmp/poc` must be removed.

### Isolation and Network Policy

- Each task runs in a dedicated Docker container.
- Each task uses an isolated workspace. The container is removed after the task finishes, while the workspace is retained for auditing.
- The container root filesystem is read-only, and the Docker socket is not mounted.
- Container capabilities are minimized, with only those required for debugging and dynamic analysis retained.
- The Agent container runs on an internal Docker network.
- The CyberGym submission service is bound only to a controlled local Docker network gateway and is not exposed to the public Internet.
- The PoC verification container runs without external network access.
- The Agent container has no direct route to the public Internet. All outbound HTTP/HTTPS requests must pass through a domain-allowlisted proxy.
- The external allowlist contains exactly one hostname: `api.deepseek.com`, which is used exclusively for model inference API requests over HTTPS. No wildcard domains, other subdomains, or public IP addresses are allowlisted.
- The proxy applies a default-deny policy. Except for `api.deepseek.com:443`, GitHub, search engines, project websites, issue and pull-request pages, code-hosting services, software repositories, and all other public domains and IP addresses are blocked. Direct outbound connections that attempt to bypass the proxy are also unreachable.
- `localhost`, `127.0.0.1`, and the internal Docker network gateway bypass the proxy only for communication within the container and access to the local CyberGym submission service. They do not provide an Internet egress path.
- The separate container used to execute the vulnerable target runs with `network=none`. It cannot access either the public Internet or the model API. The Agent can request local command execution in that container only through a restricted per-task Unix socket broker.
- Network probes confirmed that direct access to GitHub failed and that access to GitHub through the proxy returned `403 Forbidden`. Only the allowlisted model API endpoint could establish a proxy connection. The evaluation trajectories contain zero web-search or web-fetch requests, and no external data-retrieval activity involving `curl`, `wget`, `git clone`, SSH, or SCP was identified.

This policy is intended to follow CyberGym's isolation and disclosure principles: the CyberGym service is deployed in a controlled local environment and is not exposed to the public Internet. Where limited network access is enabled, the reachable scope is explicitly disclosed, and the trajectories are audited for shortcut-solving behavior involving externally retrieved patches, issues, changelogs, commits, or existing PoCs.

## Usage and Cost Estimates

| Metric | Per-task average | Aggregate total |
|---|---:|---:|
| Non-cached input tokens | 116,594.24 | 175,707,519 |
| Cache-read tokens | 41,080,023.04 | 61,907,594,721 |
| Output tokens | 152,764.98 | 230,216,824 |
| LLM requests | 201.04 | 302,967 |
| Wall-clock time | 2,501.58 seconds | 3,769,882 seconds / 1,047.2 hours |
| Estimated model cost | CNY 5.84 | CNY 8789.8 |

The evaluation spanned a DeepSeek pricing adjustment. For consistency, the reported cost was normalized using the post-increase rate.
