# Movement Improvement Proposals (MIPs)

Movement Improvement Proposals (MIP) describe standards for the Movement Network including the core blockchain protocol and the development platform (Move), smart contracts and systems for smart contract verification, standards for the deployment and operation of the Movement Network, APIs for accessing the Movement Network and processing information from the Movement Network.

MIPs are intended to cover changes that impact active services within the Movement ecosystem. In lieu of another medium for posting community proposals, the MIP issue tracker can be used to store exploratory proposals.

## How to submit a MIP

 1. Fork this repo into your own GitHub
 2. Copy [`TEMPLATE.md`](TEMPLATE.md) into your new MIP file in `mips/mip-xxx-<slug>.md`
    + Name your MIP file using the format `mip-<NNN>-<slug>.md`, where `<NNN>` is a zero-padded 3-digit number and `<slug>` is a short kebab-case summary of the title.
    + e.g., `mip-001-proposer-selection.md`.
    + When first drafting, use `mip-xxx-<slug>.md` as a placeholder — the MIP manager will assign the final number.
    + If your proposal needs diagrams or other assets, place them under `mips/diagrams/mip-<NNN>/`.
 3. Edit your MIP file
    - Fill in the YAML header (see instructions there)
    - Follow the template guidelines to the best of your ability
 4. Commit these changes to your repo
 5. Submit a pull request on GitHub to this repo.
 6. To start discussing your MIP, create a GH Issue for your MIP using the default Issue template

## Types of MIPs
* Standard: MIPs focusing on the changes to the Movement blockchain.
* Informational: MIPs for the purpose of providing additional information, context, supporting details about Movement blockchain.

## MIP Statuses
| Status | Description|
|:--|:--|
| `Draft` | Drafts are currently in process and not ready for review. No corresponding GH Issue will be created.|
| `In Review` | MIPs are ready for community review and feedback. See suggestions on providing feedback below. |
| `Ready for Approval` | MIPs are ready for Gatekeeper approval and feedback. |
| `Accepted `| MIPs has been accepted and will be implemented soon. |
| `Rejected` | A community decision has been made to not move forward with a MIP at this time.|
| `On Hold` | Some information is missing or prerequisites have not yet been completed. |

## Category
* Framework
* Core (Blockchain)
* Gas
* Cryptography
* Ecosystem

## Providing Feedback on a MIP
* Follow the discussion in the corresponding MIP issue
* If you were designing this change, what would you want to communicate? Is it being communicated in the MIP?
* As a community member, how are you impacted by this change? Does it provide enough information about the design and implementation details to assist with decision making?

## Notice regarding rejected or stale MIPs
* If a MIP Author is not actively engaging in their PR's and Issues, they will be closed after 14 days due to inactivity.
* If a MIP has been rejected, please review the feedback provided for further guidance.
