# Member Zone — WordPress / MemberPress Implementation Notes

These notes accompany the front-end Member Zone prototype (in `index.html`). The prototype demonstrates the **planned experience and page structure**; this document describes how to implement it on the production WordPress site using MemberPress, and—most importantly—how to keep the member profile and the member directory in sync.

> Platform: WordPress · Membership/access: MemberPress · Directory: lives on the same WordPress site.

---

## 1. Pages to build

| Prototype page (`data-page-id`) | WordPress page / template | Protected? |
|---|---|---|
| `signin` (Member Login) | MemberPress login (page or `[mepr-login-form]`) | Public |
| `member-dashboard` (Member Zone) | Member dashboard page | Members only |
| `member-profile` (My Profile) | Account/profile edit | Members only |
| `member-resources` (Member Resources) | Resource library | Members only |
| `account-settings` (Account Settings) | MemberPress account (`[mepr-account-form]`) | Members only |
| `directory` (Member Directory) | Directory (CPT or directory plugin) | Public **or** members only — support both |

The Member Zone must use the site's standard header, footer, nav, colors, and typography (it already does in the prototype). It should not look like a separate third-party portal.

---

## 2. Access control (MemberPress rules)

Protect with MemberPress Rules → "Content matches" the Member Zone pages, or by URL path (e.g., `/member-zone/*`):

- **Protected:** Member Zone dashboard, My Profile, Member Resources, Board Meeting Minutes, member-only Education resources, restricted Self-Insurers Fund content, restricted event materials.
- **Membership/Rule logic:**
  - **Active members** → full access.
  - **Lapsed/inactive members** → blocked from protected content; show a renewal prompt (MemberPress "unauthorized" message or redirect to the account/renew page).
  - **Admin/staff** → full access (WordPress role-based; admins bypass rules).
  - **Optional access tiers by member type** (Contractor / Associate / Sustaining) → use separate MemberPress Memberships or Groups if certain content should differ by category.

### Front-end states the prototype represents
- **Logged out:** protected pages show a friendly restricted message + "Member Login" button (see `.mz-out` blocks). Resources show: *"This resource is available to Carolinas Roofing Association members. Please log in to access this content."*
- **Logged-in active member:** dashboard, profile, resources, and settings are visible (`.mz-in` blocks).
- **Lapsed/inactive:** treat like logged-out for protected content, but route to renewal. (Implement via MemberPress rule + unauthorized redirect.)
- **Admin/staff:** full access for content management.

> In the prototype these states are simulated with a `mz-auth` class on `<html>` (added on login, removed on logout). In production, MemberPress controls visibility server-side; the `.mz-in` / `.mz-out` pattern can be reproduced with `[mepr-active]` / `[mepr-unauthorized]` shortcodes or template conditionals (`current_user_can()`, `MeprUser`).

---

## 3. Profile ↔ Directory sync (the critical requirement)

**There must be ONE source of truth.** The member profile and the directory listing must read/write the same fields. Do **not** build two disconnected profile systems.

### Recommended architecture
1. **Source of truth = MemberPress custom user fields** (Options → Fields), stored as WordPress **user meta**.
2. The **directory listing displays the same user-meta fields** — it does not keep its own separate copy.

### Field mapping (MemberPress custom field → directory field)

| Profile field (My Profile) | MemberPress field slug (user meta) | Shown in public directory? |
|---|---|---|
| Company name | `mepr_company_name` | Yes |
| Primary contact name | `mepr_contact_name` | Optional |
| Member category/type | `mepr_member_category` | Yes |
| Email (public/directory) | `mepr_public_email` | Yes (if visibility allows) |
| Phone | `mepr_phone` | Yes, **if approved** |
| Website | `mepr_website` | Yes |
| Mailing address | `mepr_address` | No (private) |
| City | `mepr_city` | Yes |
| State | `mepr_state` | Yes |
| ZIP | `mepr_zip` | No (private) |
| Service areas | `mepr_service_areas` | Optional |
| Business type (res/comm/vendor/mfr/service) | `mepr_business_type` (multi) | Used for directory filters |
| Directory visibility | `mepr_directory_visibility` (public / members / hidden) | Controls listing |
| Logo | `mepr_logo` (attachment ID/URL) | Yes, if supported |

### Two directory build options — support either

**Option A — Directory reads user meta directly (simplest, truly single source).**
- Directory is a WP template/loop or shortcode that queries users (by active MemberPress membership) and prints the mapped user-meta fields.
- Editing the profile updates user meta → directory reflects it immediately. No sync job needed.
- Plugins that work this way: directory add-ons that render from WP users / MemberPress fields.

**Option B — Directory uses a Custom Post Type (CPT) or directory plugin with its own records.**
- Each member has a directory CPT entry (e.g., `cra_member`) with ACF/custom fields.
- You **must** sync: on `personal_options_update` / MemberPress profile-save hooks (`mepr-account-is-updated`, `profile_update`), copy mapped user meta → the CPT fields (and create the CPT post on new membership).
- Keep the mapping table above as the single mapping definition. Example hook:
  ```php
  add_action('mepr-account-is-updated', function($user_id){
      $post_id = cra_get_member_post($user_id); // find/create CPT
      foreach (cra_field_map() as $userMetaKey => $cptField) {
          update_post_meta($post_id, $cptField, get_user_meta($user_id, $userMetaKey, true));
      }
  });
  ```
- Reverse-protect: the directory CPT fields should be **read-only in admin** (or hidden) so members only edit via the profile, preventing drift.

> **Recommendation:** Option A (directory reads MemberPress user meta) avoids sync bugs entirely. Choose Option B only if a directory plugin you want requires its own CPT — and then wire the save hook above.

### Email handling
- Support **two** email fields:
  - **Account login email** (`user_email`) — used to sign in.
  - **Public directory email** (`mepr_public_email`) — shown in the directory.
- Decide with the association whether changing the **login email** should also update the **public directory email**. Default recommendation: keep them independent; if the member leaves the public email blank, fall back to the login email for the directory.

### Save confirmation & moderation
- After save, show: *"Your profile has been updated."* (prototype shows this).
- If the association wants moderation, gate directory-visible changes behind admin approval: write edits to a `*_pending` meta, show *"Some changes may require association approval before they appear publicly,"* and publish on approval. Otherwise changes go live immediately.

---

## 4. Directory access model (support both)
- The prototype directory (`#directory`) is currently public and searchable/filterable (Company, Category, City/State, Website, Email; Phone only if approved).
- If the final directory is **public**: leave it open; the Member Zone simply links to it.
- If the final directory is **members-only**: protect the directory page with a MemberPress rule. The Member Zone link still works for logged-in members; logged-out users get the restricted message.
- Either way, **directory fields pull from the synced member profile data** (section 3).

---

## 5. Member Resources
- Protect the Resources page (and individual files) with MemberPress.
- Document categories used in the prototype filter: Forms, Programs, Governance (incl. Board Meeting Minutes & Bylaws), Safety, Publications.
- Board Meeting Minutes / governance docs should be members-only (optionally a sub-tier if only certain members may view).
- Store files in the Media Library or a protected uploads location; for true file protection use MemberPress-protected URLs (not just hidden links).
- Logged-out message: *"This resource is available to Carolinas Roofing Association members. Please log in to access this content."*

---

## 6. Branding / copy rules (carry into WordPress)
- Use **Carolinas Roofing Association** (never "Carolina Roofing Association").
- Use **CRSMCA** only as historical/transitional context, e.g. *"Formerly CRSMCA,"* or *"Your existing CRSMCA login is being modernized as part of the new Carolinas Roofing Association website."*
- Keep the Member Zone visually identical to the main site (header, footer, buttons, colors, type, spacing).
- Accessibility: labeled form fields, clear button text, visible focus states, good contrast, keyboard-friendly nav (the prototype follows these).

---

## 7. Prototype → production checklist
- [ ] Recreate the six pages on WordPress using the site theme.
- [ ] Define MemberPress Memberships (and Groups/tiers if needed).
- [ ] Add MemberPress custom fields per the mapping table (section 3).
- [ ] Choose directory Option A or B; if B, wire the profile-save sync hook.
- [ ] Add MemberPress Rules to protect the pages in section 2.
- [ ] Configure login, account, and password-reset via MemberPress.
- [ ] Decide login-email vs public-email behavior and moderation policy.
- [ ] Protect resource files (not just links).
- [ ] QA the four states: logged out, active, lapsed, admin.
