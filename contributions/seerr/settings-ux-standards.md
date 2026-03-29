# Seerr Settings UX Standards

Proposed visual and structural standards for all Settings pages in Seerr. Any new or modified settings tab should conform to these standards. Any existing tab that does not conform is a candidate for cleanup.

## Architecture

### Why Components, Not Just CSS

The CSS classes for settings pages (`.heading`, `.form-row`, `.text-label`, `.actions`, etc.) are already centralized in `globals.css`. The styling is consistent. The problem is structural: every settings tab copy-pastes the same JSX div nesting by hand. There is no shared React component that enforces the correct structure.

This means:
- New tabs are built by copying an existing tab and tweaking it, introducing subtle drift
- Layout fixes to one tab don't propagate to others
- Contributors have no single reference for "how to build a settings page"

The standard requires shared components that encode the correct structure. CSS handles the look; components handle the layout contract.

### Shared Components

#### `<SettingsSection>`

Every logical group of settings should be wrapped in a section with a header and optional description.

```tsx
<SettingsSection
  title="Proxy Settings"
  description="Configure an HTTP/HTTPS proxy for outbound API requests."
>
  {/* FormRow children */}
</SettingsSection>
```

Renders:
```html
<div class="mb-6">
  <h3 class="heading">Proxy Settings</h3>
  <p class="description">Configure an HTTP/HTTPS proxy...</p>
</div>
<div>{children}</div>
```

**Developer benefit:** Changing the heading hierarchy, spacing, or description style happens in one file instead of 15+.

#### `<FormRow>`

Every label + input pair should use a shared row component.

```tsx
<FormRow
  label="API Key"
  tip="Your unique API key for external access"
  required
  badges={['advanced']}
  error={touched.apiKey && errors.apiKey}
>
  <Field type="text" name="apiKey" />
</FormRow>
```

Renders the standard 3-column grid: label column (with tip, badges, required marker) and input column (with error display).

**Developer benefit:** Badge placement, label-tip ordering, error positioning, and accessibility attributes are consistent everywhere without each contributor having to remember the correct div nesting.

#### `<SettingsActions>`

The save/test button footer.

```tsx
<SettingsActions
  isSubmitting={isSubmitting}
  onTest={testConnection}
  isTesting={isTesting}
/>
```

**Developer benefit:** Consistent button text ("Save Changes" vs "Save"), icon usage, and Test button placement. Adding a new save behavior (e.g., confirmation dialog) only needs one change.

---

## Page Structure

### Page Header

Every settings tab has exactly **one** page-level header at the top:

```
h3.heading     "Network Settings"
p.description  "Configure network settings for your Seerr instance."
```

This is the tab's identity. It describes the entire page.

### Sections

When a tab contains logically distinct groups of fields, use `<SettingsSection>` to separate them.

**Use a section when:**
- 3 or more fields share a logical domain (e.g., "Proxy", "DNS", "Security")
- A group of related toggles (e.g., "Login Methods", "Auto-Approve")
- A distinct feature area on a multi-purpose tab

**Do not use a section when:**
- Fewer than 3 fields; spacing between form rows is sufficient
- The entire tab is one logical group (e.g., a single notification agent's settings)

### Section Dividers

Sections should have visual separation. Use the existing `border-t border-gray-700` pattern with top padding between sections. This is the same treatment currently used above the `.actions` save bar.

---

## Form Patterns

### Pattern 1: Inline Form (default)

A single Formik form with `<FormRow>` fields and a `<SettingsActions>` footer.

Used for: General, Users, Network, Plex, Metadata, Notification agents.

**Rules:**
- One form per logical scope
- One save button per form
- Fields use `<FormRow>` for consistent layout
- Validation errors shown inline below each field

### Pattern 2: Card + Modal Form

A grid of cards representing instances, with Add/Edit/Delete actions that open modal dialogs.

Used for: Services (Radarr/Sonarr servers), Override Rules.

**When to use:** Managing multiple instances of a configurable thing. The card shows a summary; the modal shows the full edit form.

**Rules:**
- Grid layout: `grid-cols-1 lg:grid-cols-2 xl:grid-cols-3`
- Add button as dashed-border tile at end of grid
- Edit/Delete buttons in card footer
- Modal forms follow the same `<FormRow>` standard internally

### Mixing Patterns

Do not mix inline form and card+modal patterns within a single logical section. If a tab needs both (e.g., Plex connection settings + library toggles), use separate `<SettingsSection>` blocks with clear headers.

---

## Save and Test Buttons

### Save Button

- **One** per form, bottom-right in `<SettingsActions>`
- `buttonType="primary"`
- Text: "Save Changes" (idle), "Saving..." (submitting)
- Icon: `ArrowDownOnSquareIcon`
- Disabled when: submitting, or form is invalid/pristine

If a tab currently has multiple independent save buttons, the forms should be restructured. Options:
1. Combine into one form with one save
2. Separate into distinct sections, each with a clearly scoped save
3. Move one form to a different tab entirely

### Test Button

Appropriate when the settings connect to an external service and the test validates connectivity without saving.

- Position: Left of Save button
- `buttonType="warning"`
- Text: "Test" (idle), "Testing..." (testing)
- Disabled when: testing, or required connection fields are empty

**Tabs that should have Test:** Metadata, all Notification agents, Tautulli, Plex (server connection)
**Tabs that should not:** General, Users, Network, Jobs, Logs, About

---

## Badges

Three badge types from `SettingsBadge`:

| Badge | Color | Meaning | Tooltip |
|-------|-------|---------|---------|
| `restartRequired` | Blue | Seerr must restart for changes to take effect | "Seerr must be restarted..." |
| `advanced` | Red | Misconfiguration may break functionality | "Incorrectly configuring..." |
| `experimental` | Yellow | May cause unexpected behavior | "Enabling this setting..." |

### Rules

- **Maximum two badges per field.** If a field genuinely needs all three, it likely belongs behind an "Advanced Settings" section toggle rather than in the main field list.
- **Badge order** (when stacked): `restartRequired`, then `advanced` or `experimental`
- **Placement:** Inline after the label text, before any `label-tip`
- **Section-level badges:** When all fields in a section share the same badge (e.g., all proxy fields require restart), consider placing the badge on the section header instead of repeating it on every field.

---

## Field Labels

### Label Classes

| Class | Usage |
|-------|-------|
| `.text-label` | Standard text input, select, or complex field |
| `.checkbox-label` | Checkbox or toggle field |
| `.label-tip` | Description text below the label (gray, smaller font) |
| `.label-required` | Red asterisk after label text for required fields |
| `.error` | Validation error message below the input |

### Guidelines

- Every non-obvious field should have a `label-tip` explaining what it does or when to change it
- Obvious fields (Port, SSL, Hostname) do not need a tip
- Tips should be concise: one sentence, no period at the end
- Required fields must show `.label-required`
- Error messages appear only after the field has been touched

---

## Conditional Fields

When a toggle enables a group of sub-settings (e.g., "Enable Proxy" reveals proxy host/port/auth):

- **Indent** sub-fields with `ml-4 mr-2`
- **Show/hide** with conditional rendering, not disabled state
- The parent toggle's `label-tip` should mention that enabling it reveals additional options
- Sub-fields animate in (no jarring layout shift)

---

## Data Display Tabs

Logs, Jobs & Cache, and About are read-only data views, not forms.

- **Table component** for tabular data (Logs, Cache stats, Job list)
- **List component** for key-value display (About)
- Action buttons inline with rows (Run Now, Flush Cache, Copy)
- **No save buttons.** These tabs do not modify settings.

These tabs are out of scope for the form standards above. They follow their own layout patterns appropriate to data display.

---

## Accessibility

- Form groups with related fields should use `role="group"` with `aria-labelledby`
- Checkbox labels should be clickable (wrapping `<label>` around both checkbox and text)
- Tooltip-only information should also be available via `label-tip` for users who can't hover
- Badge tooltips should use `aria-label` so screen readers convey the meaning
- Form errors should be associated with their field via `aria-describedby`

---

## i18n

- All user-facing text (labels, tips, badges, button text, error messages) must use i18n message keys
- No hardcoded English strings in component JSX
- New keys should follow the existing namespace pattern: `components.Settings.{TabName}.{fieldName}`
- After adding keys, run `pnpm i18n:extract` to update locale files
