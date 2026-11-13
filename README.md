# LlamaFirewall + PyRIT — Layered LLM Security Pipeline

A proof-of-concept security gateway for LLM applications on Azure. It places a
layered scanning proxy in front of an OpenAI-compatible model endpoint, drives
continuous adversarial testing with automated red teaming, and ships every
decision to Log Analytics for dashboarding and SIEM correlation.

Built to answer one question in a bank-grade setting: **can a guardrail layer
stop prompt injection, jailbreaks and data exfiltration before a request ever
reaches the model — and can we prove it with repeatable red-team evidence?**

> This is a personal proof-of-concept / portfolio project. All endpoints,
> subscription IDs and host names in this repo are placeholders — replace them
> with your own before deploying.

---

## What it does

- **Guardrail proxy** — a FastAPI service exposing an OpenAI-compatible
  `/v1/chat/completions` endpoint on `:8080`. It sits between an application
  backend and the model endpoint (Azure AI Foundry, or local Ollama for lab
  profiles). Prompts run through a layered scanner stack; clean prompts are
  forwarded, blocked prompts never reach the model, and responses come back
  with scan metadata attached. Embeddings pass through unscanned.
- **Automated red teaming** — a PyRIT-driven harness replays an adversarial
  prompt dataset against the proxy and scores detection, so changes to the
  guardrail can be regression-tested.
- **Observability** — every scan decision is shipped to an Azure Log Analytics
  Workspace via a Data Collection Rule and managed identity, surfaced in an
  Azure Workbook dashboard, and available to Microsoft Sentinel.

**Result on the bundled dataset:** 98.85% detection across an 87-prompt
adversarial set — a +55 percentage-point improvement over a single-scanner
baseline.

---

## Architecture

```
          red-team harness (PyRIT)
                   │  adversarial prompts
                   ▼
 app backend ─► LlamaFirewall proxy (:8080) ─► model endpoint
                   │   layered scanner stack       (Azure AI Foundry
                   │                                 or local Ollama)
                   ▼
        Log Analytics Workspace ─► Azure Workbook ─► Microsoft Sentinel
```

The proxy is provisioned on an Azure VM via Bicep. Container images
(`firewall/Dockerfile`, `firewall/Dockerfile.gpu`) are provided for a
containerized deployment variant.

---

## Scanner stack

Up to 10 layers run in order on every prompt. Input layers gate the request;
output layers (enabled on the preprod/production profiles) gate the response.

**Input scanners**

| # | Scanner | Type | Detects |
|---|---|---|---|
| 1 | PromptGuard 2 | ML classifier | Injection syntax, jailbreaks |
| 1.5 | Perplexity filter *(optional)* | GPT-2 model | Adversarial suffixes (GCG, AutoDAN) |
| 2 | HiddenASCII | Rule-based | BiDi text, invisible characters, encoding tricks |
| 3 | Regex + custom patterns | Rule-based | XSS, SQLi, credential extraction, tool abuse |
| 4 | Llama Guard 3 (8B) | Semantic LLM | Social engineering, content safety, subtle jailbreaks |
| 5 | NOVA | YARA-style rules | Logic traps, tool injection, bio-weapon synthesis, political manipulation |
| 6 | Crescendo tracker | Stateful | Multi-turn escalation (Crescendo pattern) |

**Output scanners (preprod / production)**

| # | Scanner | Type | Detects |
|---|---|---|---|
| 7 | Llama Guard 3 (8B) output scan | Semantic LLM | Harmful content in model responses |
| 8 | CodeShield | Code analysis | Malicious code / dangerous syscalls in responses |
| 9 | Output sensitive-data regex | Rule-based | PII, API keys, connection strings in responses |

---

## Stack

- **[LlamaFirewall](https://github.com/meta-llama/PurpleLlama)** — Meta's
  open-source LLM security framework, extended here with a custom scanner stack.
- **[PyRIT](https://github.com/Azure/PyRIT)** — Microsoft's open-source AI
  red-teaming toolkit.
- **NOVA** — YARA-style pattern-matching engine for prompts.
- **Ollama** — local OpenAI-compatible inference server, used in lab profiles in
  place of a cloud model endpoint.
- **Observability** — Azure Log Analytics → Azure Workbook → Microsoft Sentinel.

---

## Environment profiles

Four profiles set the VM size, model, scanner thresholds and topology.
`deploy.sh` and `setup_vm.sh` prompt for the profile interactively.

| Setting | home-lab | corp-lab | preprod | production |
|---|---|---|---|---|
| **Topology** | single VM | two VMs | single VM | single VM |
| **Proxy VM size** | B8ms | NC4as_T4_v3 (GPU) | D8s_v3 | D16s_v3 |
| **Red-team runner** | laptop | B2ms VM | laptop / CI | canary probe |
| **Model** | phi3:mini | phi3:mini | mistral:7b | llama3:8b |
| **Output scanning** | ✗ | ✗ | ✓ | ✓ |
| **Perplexity filter** | ✗ | ✗ | ✓ | ✓ |
| **Crescendo tracker** | ✓ | ✓ | ✓ | ✓ |
| **CodeShield** | ✗ | ✗ | ✓ | ✓ |
| **LAW retention** | 30 d | 30 d | 30 d | 90 d |
| **Access** | SSH tunnel | Azure Bastion | Azure Bastion | Azure Bastion |

---

## Repository layout

```
.
├── firewall/
│   ├── proxy.py              # FastAPI guardrail proxy — layered scanner stack
│   ├── Dockerfile            # Containerized build (CPU)
│   └── Dockerfile.gpu        # Containerized build (GPU)
├── infra/
│   ├── main.bicep            # Azure infra — profile-aware VM deployment
│   ├── main.bicepparam       # Parameters (SSH key, profile, region)
│   ├── deploy.sh             # Step 1: profile selector → deploy infra
│   ├── setup_vm.sh           # Step 2: VM bootstrap (--profile lab|preprod|production)
│   ├── teardown.sh           # Destroy all Azure resources + local cleanup
│   └── preprod/              # Preprod-specific Bicep + deploy script
├── redteam/
│   ├── pyrit_redteam.py      # Step 3: PyRIT red-team runner
│   ├── custom_attacks.yaml   # Adversarial dataset (87 prompts, 10 categories)
│   ├── gandalf_attacks.yaml  # Gandalf-derived attack set
│   ├── canary_probe.py       # Scheduled production canary
│   └── setup_pyrit.sh        # Local venv + PyRIT install
├── observability/
│   ├── deploy_workbook.py    # Deploy the Azure Workbook
│   ├── log_shipper.py        # Ship scan decisions to Log Analytics
│   └── workbook_content.json # Workbook definition
└── scripts/                  # Demo + toggle helpers (audit, bypass, no-LLM)
```

---

## Quickstart (home-lab profile)

The proxy runs **on the Azure VM**; PyRIT runs on your laptop and reaches the
proxy through an SSH tunnel. Set your SSH public key and (for preprod/prod) your
model endpoint in `infra/main.bicepparam` before deploying.

```bash
# 1. Deploy the Azure VM (from the repo root)
cd infra
./deploy.sh                        # choose the home-lab profile when prompted

# Get the VM's address from the deploy output, or query it:
az network public-ip list -g rg-llamapoc \
  --query "[].{ip:ipAddress, fqdn:dnsSettings.fqdn}" -o table
VM=azureuser@<fqdn-or-ip>

# 2. Bootstrap the proxy ON THE VM — copy the files up, then run the script there.
#    (setup_vm.sh installs Ollama, pulls the models and creates a service owned
#     by the VM's azureuser; do NOT run it on your laptop.)
scp setup_vm.sh ../firewall/proxy.py ../redteam/social_engineering_pt.nov "$VM":~/
ssh "$VM" 'bash ~/setup_vm.sh --profile lab 2>&1 | tee ~/setup.log'

# 3. Open an SSH tunnel to the proxy (leave running in its own terminal)
ssh -N -L 8080:localhost:8080 "$VM"

# 4. In a second terminal, run the red-team suite from your laptop
cd ../redteam
./setup_pyrit.sh
python pyrit_redteam.py --endpoint http://localhost:8080 --prompts-file custom_attacks.yaml

# 5. Tear down when finished — the VM bills while it runs
cd ../infra && ./teardown.sh
```

---

## Observability

The proxy writes one structured record per scan decision. On preprod/production
profiles these ship to Log Analytics through a Data Collection Rule using the
VM's system-assigned managed identity — no static workspace key on the host. The
Azure Workbook (`observability/deploy_workbook.py`) renders detection rates,
blocked categories and latency, and the same table is queryable from Sentinel
for alerting.

---

## Security notes

- No credentials are stored in this repo. VMs use SSH key auth with password
  auth disabled; log ingestion uses managed identity; local inference (Ollama)
  needs no key.
- The adversarial dataset under `redteam/` contains deliberately malicious
  prompts and synthetic PII for testing only. It is not real data.
- Replace every placeholder (endpoints, subscription ID, SSH key, VM FQDN) with
  your own values before deploying.

---

## License

MIT. LlamaFirewall, PyRIT and the other named tools are the property of their
respective owners.
