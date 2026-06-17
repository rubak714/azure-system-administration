# ☁️ Azure AZ-104 Hands-On Labs

![Exam](https://img.shields.io/badge/AZ--104-Hands--On%20Lab-0078D4?logo=microsoftazure&logoColor=white)
![Built in](https://img.shields.io/badge/Built%20in-Azure%20Portal-0078D4?logo=microsoftazure&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Learning%20in%20public-orange)

Seven hands-on Azure labs. I built all of them in the Azure portal while studying for the AZ-104 (Azure Administrator) exam.

Five labs each cover one exam domain. One lab covers Key Vault, a supporting service the exam relies on. The last lab is a capstone. It ties the other labs together into one small end-to-end system.

Everything here is built in the portal on purpose. That is how the exam works. It is also where the small settings are easiest to learn.

These are study labs. I do not claim they are more than that. Each one is written to show that I understand the work. That means the reason behind each choice, and the mistakes that are easy to make. They are not copies of a tutorial. If you are a recruiter reading this, the value is in the reasoning inside each lab's README.

---

## 📑 Table of Contents
1. [🧪 The labs](#-the-labs)
2. [📂 How each lab folder is laid out](#-how-each-lab-folder-is-laid-out)
3. [🚀 Running these yourself](#-running-these-yourself)
4. [📤 Pushing this to GitHub](#-pushing-this-to-github)
5. [🧭 Where this goes next](#-where-this-goes-next)
6. [🙏 Credit](#-credit)
7. [📄 License](#-license)

---

## 🧪 The labs

| # | Folder | Exam domain | What it covers |
|---|--------|-------------|----------------|
| 1 | [`01-identity-governance`](./01-identity-governance) | Identity & Governance (20-25%) | Entra users/groups, RBAC vs Entra roles vs Policy, locks, tags, budgets |
| 2 | [`02-secure-storage`](./02-secure-storage) | Storage (15-20%) | Redundancy, SAS vs stored access policy vs RBAC, blob tiers, private endpoint |
| 3 | [`03-hub-spoke-network`](./03-hub-spoke-network) | Networking (15-20%) | NSG evaluation, non-transitive peering, Bastion, Load Balancer |
| 4 | [`04-compute-autoscale`](./04-compute-autoscale) | Compute (20-25%) | Availability set vs zone vs VMSS, autoscale, App Service slots, ARM export |
| 5 | [`05-monitor-backup`](./05-monitor-backup) | Monitor & Maintain (10-15%) | Metrics vs logs, alerts + action groups, Backup vs Site Recovery |
| 6 | [`06-keyvault-secrets`](./06-keyvault-secrets) | Supporting service | Secrets, keys, certificates, the RBAC vs access-policy model, managed-identity access |
| 7 | [`07-capstone-end-to-end`](./07-capstone-end-to-end) | All domains | A small web app stood up end to end: secure, scaling, monitored, recoverable |

The suggested order is 1 to 7. The Key Vault lab (6) builds on the identity and storage labs. You can do it any time after labs 1 and 2. The capstone (7) reuses parts from the earlier labs, so do it last.

---

## 📂 How each lab folder is laid out

- `README.md`: the walkthrough. It has the scenario, a Mermaid diagram, the portal steps that matter, the details that are easy to miss, a break-it-and-fix-it exercise, and the key exam points the lab reinforces.
- `screenshots/`: my portal screenshots. It also has a short list of what to capture, with a reminder to blur any secrets.
- `cli-reference/commands.md`: the same actions written as Azure CLI commands, for cross-checking. The labs themselves are done in the portal.

---

## 🚀 Running these yourself

You need an Azure subscription. A free account works. Each lab notes roughly how long it takes. Each lab also reminds you to delete the resources afterwards, so you do not use up credit. Start with `01-identity-governance`.

---

## 📤 Pushing this to GitHub

I push this repository one lab at a time. I do not upload it all at once. This way the commit history shows clear, ordered progress.

I add the scaffold first:

```bash
git init
git add README.md LICENSE .gitignore
git commit -m "chore: repo scaffold and landing page"
git branch -M main
git remote add origin https://github.com/<your-username>/azure-az104-labs.git
git push -u origin main
```

Then I finish one lab, add its screenshots, and commit that lab on its own:

```bash
git add 01-identity-governance
git commit -m "docs: add identity and governance lab"
git push
```

I repeat that three-command pattern for each lab. The history then reads as scaffold, then lab 1, then lab 2, and so on. It does not read as one large commit.

---

## 🧭 Where this goes next

Some of these labs may grow beyond study work. A lab could turn into a real application, an infrastructure-as-code project, or a deployment pipeline. When that happens, I will move the folder into its own repository. I will name that repository for what it does, not for the exam it came from. I will also link it back here. The networking lab and the capstone are the most likely candidates.

---

## 📄 License

Released under the MIT License. See [LICENSE](./LICENSE).