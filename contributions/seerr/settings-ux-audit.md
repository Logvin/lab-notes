# Settings UX Audit

A tab-by-tab audit of Seerr's settings pages against the proposed [Settings UX Standards](settings-ux-standards.md). Screenshots captured from a live instance running the `develop` branch.

## How to Read This

Each tab gets:
- A full-page screenshot of the current state
- A focused screenshot highlighting the specific issue
- What the standard requires and what should change
- A reference to the relevant section of the standards doc

The goal is to get alignment on the [standards](settings-ux-standards.md) first, then apply changes tab-by-tab in separate PRs.

---

## 1. General Settings

![General Settings — Current](screenshots/settings-audit/01-general.png)

### Issues

**No section grouping** — 14 fields in a single flat list. Identity settings (API Key, Title, URL) run directly into discovery settings (Region, Language) and content rules (Blocklist, Hide Available Media) with no visual separation.

![No sections](screenshots/examples/no-sections-general.png)
*API Key, Application Title, Application URL flow directly into Enable Image Caching, Display Language, Discover Region... six different concerns in one unbroken list.*

**Standard:** [Page Structure > Sections](settings-ux-standards.md#sections) — Group related fields under `<SettingsSection>` blocks with H3 headings.

### Proposed Changes

Split into four `<SettingsSection>` groups:

| Section | Fields |
|---------|--------|
| **Application Identity** | API Key, Application Title, Application URL |
| **Discovery & Language** | Display Language, Discover Region, Discover Language, Streaming Region |
| **Content Visibility** | Enable Image Caching, Hide Available Media, Hide Blocklisted Items, Allow Partial Series Requests, Allow Special Episodes Requests, YouTube URL |
| **Blocklist** | Blocklist Content with Tags, Limit Content Blocklisted per Tag |

Each section gets an H3 heading and a one-line description, with `border-t border-gray-700` dividers between them. The page-level heading and description remain unchanged (they're already correct).

---

## 2. User Settings

![User Settings — Current](screenshots/settings-audit/02-users.png)

### Issues

**Mixed grouping patterns** — "Login Methods" uses a `fieldset`/`group` pattern that no other tab uses. "Default Permissions" lists 26 checkboxes flat with no sub-grouping. No visual separation between Login Methods, Request Limits, and Permissions.

**Standard:** [Page Structure > Sections](settings-ux-standards.md#sections) — Use `<SettingsSection>` for logical groups.

### Proposed Changes

Split into three `<SettingsSection>` groups:

| Section | Fields |
|---------|--------|
| **Login Methods** | Enable Local Sign-In, Enable Plex Sign-In, Enable New Plex Sign-In |
| **Request Limits** | Global Movie Request Limit, Global Series Request Limit |
| **Default Permissions** | All 26 permission checkboxes (consider sub-grouping: Request, Auto-Approve, Auto-Request, 4K, Issues, Blocklist) |

The Permissions section could further benefit from collapsible sub-groups (Request permissions, Auto-Approve permissions, 4K permissions, etc.) to reduce the wall-of-checkboxes effect.

---

## 3. Plex Settings

![Plex Settings — Current](screenshots/settings-audit/03-plex.png)

### Issues

**Two Save buttons on one page** — The Plex connection form and the Tautulli form each have their own independent Save Changes button. Users scrolling through can't tell which settings belong to which button.

![Two save buttons](screenshots/examples/two-save-buttons-plex.png)
*Both Save buttons visible on the same page. Which one saves what?*

**No Test button for Plex connection** — Plex connects to an external server, so a Test button is appropriate here.

**Standard:** [Save and Test Buttons](settings-ux-standards.md#save-and-test-buttons) — One Save per form. Test button for external service connections.

### What Already Works

The H3 sub-sections (Plex Settings, Plex Libraries, Manual Library Scan, Tautulli Settings) are well-structured with clear descriptions:

![Good sections](screenshots/examples/good-sections-plex.png)
*H3 headings with descriptions for each section. This is the pattern other tabs should follow.*

### Proposed Changes

Two options:
1. **Combine** into one form with one Save at the bottom (simpler)
2. **Separate** Tautulli into its own tab (cleaner separation of concerns)

Either way, add a Test button for the Plex connection section.

---

## 4. Services

![Services — Current](screenshots/settings-audit/04-services.png)

### What Already Works

This tab correctly uses the **Card + Modal** pattern for managing multiple Radarr/Sonarr server instances:

![Card layout](screenshots/examples/good-cards-services.png)
*"Default" badge on the active server, dashed-border "Add" tile for new instances. This is the documented exception for multi-instance management.*

### Issues

**Badge style mismatch** — Card badges ("Default", "SSL") use a different visual style than the form badges (Advanced, Experimental, Restart Required) on other tabs.

**Standard:** [Badges](settings-ux-standards.md#badges) — Standardize badge colors and styles across all contexts.

### Proposed Changes

Align card badge colors with the form badge palette. Keep the card layout; it's the right pattern for this use case.

---

## 5. Network Settings

![Network Settings — Current](screenshots/settings-audit/05-network.png)

### Issues

**Triple badge stacking** — DNS Cache and Force IPv4 Resolution each have three badges: Advanced + Restart Required + Experimental. CSRF Protection and HTTP(S) Proxy have two each. The visual noise drowns out the actual settings.

![Badge stacking](screenshots/examples/badge-stacking-network.png)
*Three badges on a single toggle. The field label is almost an afterthought.*

**No section grouping** — Flat list of toggles with no logical separation between core settings and experimental ones.

**Standard:** [Badges](settings-ux-standards.md#badges) — Maximum two badges per field. [Sections](settings-ux-standards.md#sections) — Group related fields.

### Proposed Changes

| Section | Fields | Section-level Badge |
|---------|--------|-------------------|
| **Core** | Enable Proxy Support, API Request Timeout | Restart Required (section-level) |
| **Security** | Enable CSRF Protection | Advanced, Restart Required |
| **Experimental** | Force IPv4 Resolution First, DNS Cache, HTTP(S) Proxy | Experimental (section-level), Restart Required (section-level) |

By placing shared badges at the section level, individual fields drop to zero or one badge each.

---

## 6. Metadata Providers

![Metadata Providers — Current](screenshots/settings-audit/06-metadata.png)

### Issues

**Terse page description** — "Settings for metadata provider" is an incomplete sentence fragment. Compare to General's "Configure global and default settings for Seerr."

![Terse description](screenshots/examples/terse-description-metadata.png)
*Fragment with no period. Doesn't tell the user what they can do here.*

**Inconsistent heading hierarchy** — H3 page title, then H4 for "Metadata Provider Status", then H2 for "Metadata Provider Selection". Peer sections at different heading levels.

![Mixed headings](screenshots/examples/mixed-headings-metadata.png)
*H4 "Metadata Provider Status" (small, in a card) next to H2 "Metadata Provider Selection" (much larger). The visual hierarchy is inverted.*

**Standard:** [Page Header](settings-ux-standards.md#page-header) — Complete sentence descriptions. [Heading Hierarchy](settings-ux-standards.md#heading-hierarchy) — H3 for page title, H3 for all sub-sections.

### What Already Works

Test + Save button placement is correct:

![Test and Save](screenshots/examples/good-test-save-metadata.png)
*Test (yellow) left of Save Changes (purple).*

### Proposed Changes

- Description: "Configure metadata providers for movie, series, and anime lookups."
- Fix heading hierarchy: both sub-sections should be H3
- Keep Test + Save pattern as-is

---

## 7. Notification Settings

![Notification Settings — Current](screenshots/settings-audit/07-notifications.png)

### What Already Works

This is one of the most consistent tabs:

![Sub-tabs](screenshots/examples/good-subtabs-notifications.png)
*Sub-tab bar for each notification agent. Each agent has its own Test + Save buttons.*

- Sub-tab pattern is appropriate for many distinct agents
- Each agent is a self-contained form with Test + Save
- Standard two-column form layout within each agent

### Issues

Minor: sub-tab bar wraps on smaller viewports. Consider horizontal scrolling.

### Proposed Changes

Minimal. Ensure each agent's form uses `<FormRow>` internally for consistency. Add horizontal scroll to the sub-tab bar.

---

## 8. Logs

![Logs — Current](screenshots/settings-audit/08-logs.png)

### What Already Works

- Data display tab correctly uses table pattern
- Filter and pagination controls are well-placed
- Inline `code` elements in description are appropriate

### Issues

Minor: description could be more consistent with other tabs' voice.

### Proposed Changes

Description: "View and filter application logs for your Seerr instance."

---

## 9. Jobs & Cache

![Jobs & Cache — Current](screenshots/settings-audit/09-jobs-cache.png)

### What Already Works

**This is the best-structured tab** and should be the reference implementation for data display tabs:

![Good data sections](screenshots/examples/good-sections-jobs.png)
*H3 section headings (Jobs, Cache, DNS Cache, Image Cache), each with its own description. Tables with inline action buttons.*

### Issues

Minor: page-level description could be slightly more concise.

### Proposed Changes

Use this tab as the reference for how data display tabs should be structured. No structural changes needed.

---

## 10. About

![About — Current](screenshots/settings-audit/10-about.png)

### Issues

**No page-level description** — The heading "About Seerr" jumps straight into the warning banner and stats.

![No description](screenshots/examples/no-description-about.png)
*Heading with no description. Every other tab has one.*

**Standard:** [Page Header](settings-ux-standards.md#page-header) — Every tab gets an H3 heading + one-line description.

### What Already Works

- Definition lists for key-value data (semantically correct)
- Clear H3 section headings (About Seerr, Getting Support, Support Seerr, Releases)
- Release history with changelog buttons

### Proposed Changes

Add description: "Information about your Seerr installation and available updates."

---

## Summary

| Issue | Tabs | Standard |
|-------|------|----------|
| No section grouping | General, Network | [Sections](settings-ux-standards.md#sections) |
| Multiple Save buttons | Plex | [Save Button](settings-ux-standards.md#save-button) |
| Badge stacking (3+) | Network | [Badges](settings-ux-standards.md#badges) |
| Missing/terse descriptions | Metadata, About | [Page Header](settings-ux-standards.md#page-header) |
| Mixed heading hierarchy | Metadata | [Heading Hierarchy](settings-ux-standards.md#heading-hierarchy) |
| No shared components | All form tabs | [Shared Components](settings-ux-standards.md#shared-components) |

## Implementation Plan

1. **Phase 1: Standards approval** — Get alignment on the [Settings UX Standards](settings-ux-standards.md) before writing any code
2. **Phase 2: Shared components** — Create `<SettingsSection>`, `<FormRow>`, `<SettingsActions>` as reusable components
3. **Phase 3: Reference implementation** — Apply to General as the first tab (most to gain from section grouping)
4. **Phase 4: Remaining tabs** — One PR per tab, each referencing the standard
