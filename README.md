# Set Office Default Handlers (macOS 26.4+ / 26.7 / 27)

## Why the old method utilizing utiutil stopped working

Previous methods used `utiluti`, which calls the public
LaunchServices API. **Starting in macOS 26.4, macOS prompts the user for confirmation
on every default-app change** — not just the default browser as before. The user can
click "Keep", or ignore the dialog entirely, which stalls the script.

Apple provides **no** configuration profile or declaration to manage default apps on
macOS. Reference:
<https://scriptingosx.com/2026/03/macos-26-4-brings-more-default-app-confirmation-prompts/>

## What this does instead

`setDefaultHandlers.sh` writes the LaunchServices handler records directly into
`com.apple.launchservices.secure.plist`:

1. **`/Library/User Template/Non_localized/Library/Preferences/com.apple.LaunchServices/`**
   — every user account created *after* this runs is correct from first login, with
   zero prompts and zero user interaction. This is the fully reliable path.
2. **Each existing local user's home directory** — `lsd` is killed before the write so
   it cannot flush its cached copy over the change, then killed again afterwards so it
   re-reads the file. Confirmed on macOS 27.0 to take effect in the live session with
   no logout and no prompt.

No prompt is shown to the user in either case.

## Handlers configured

| Scope | Scheme / UTI | App |
|---|---|---|
| URL | `mailto` | Outlook |
| Type | `com.apple.mail.email` (.eml), `public.email-message` | Outlook |
| Type | `com.microsoft.outlook.msg` (.msg), `com.microsoft.outlook15.email-message` | Outlook |
| Type | `com.microsoft.outlook.oft` (.oft), `com.microsoft.outlook.template` (.emltpl) | Outlook |
| Type | `com.apple.ical.ics` (.ics), `com.apple.ical.vcs` (.vcs), `com.microsoft.outlook15.icalendar` | Outlook |
| Type | `.doc .docx .dotx .dot .rtf` UTIs | Word |
| Type | `.xls .xlsx .xlsm .xltx .csv` UTIs | Excel |
| Type | `.ppt .pptx .ppsx .potx` UTIs | PowerPoint |

Edit the `url_handlers` / `type_handlers` arrays at the top of the script to change it.

Deliberately **not** included:

- `message:` — Microsoft Outlook does not declare this URL scheme, only Apple Mail does.
  Setting it would create a dead handler record.
- `http` / `https` / `public.html` — these are tied to the default-browser flow and still
  require an interactive prompt regardless of method.

Apps that are not installed are skipped automatically (verified via `CFBundleIdentifier`
in `/Applications`, falling back to Spotlight).

## Intune deployment

### Why this ships as a LOB `.pkg`, not an Intune shell script

`setDefaultHandlers.sh` was originally deployed as an **Intune > Devices > macOS >
Shell scripts** remediation. That was moved to a signed, notarized **LOB app**
(`buildPkg.sh` builds it) so the deployment can carry an **Intune assignment filter**.
A plain shell script assignment has no reliable way to gate on device state, so it would
run — and log noise / no-op — on Macs that are still mid-enrolment or that don't have
Office yet. As a LOB app, assignment can be scoped with a filter that excludes:

- Devices that have **not yet completed the onboarding sequence** (e.g. Autopilot/ADE
  enrollment not yet reported complete).
- Devices that **do not yet have Microsoft Office installed** (no point seeding
  handlers for an app that isn't there yet, and it avoids extra log entries during the
  onboarding window).

The script itself is still a no-op/self-correcting if it *does* run early — the filter
is a targeting optimization, not a correctness requirement.

### LOB app

**Apps > macOS > Add > Line-of-business app** → upload the `.pkg` built by
`buildPkg.sh` (signed with `Developer ID Installer: Aaron Voges (HE8J54Z2AE)`,
notarized and stapled).

| Setting | Value |
|---|---|
| Bundle ID | `com.exxonmobil.pkg.SetOfficeDefaultHandlers` |
| Assignment | **Required**, scoped to a group **with** an assignment filter (see above) |
| Ignore app version | No — bump the pkg version in `buildPkg.sh` on every change, Intune reinstalls on version bump |

The postinstall script runs once at install time (root context) and is not persisted on
disk. No payload files are installed.

Log: `/Library/Logs/Microsoft/IntuneScripts/SetDefaultHandlers/setDefaultHandlers.log`

### Uninstalling / reverting

`uninstallDefaultHandlers.sh` (built into a LOB app by `buildUninstallPkg.sh`) removes
only the exact scheme/UTI → bundle id records this project manages, from the User
Template and every existing local user. It matches on both the handler key **and** the
currently-set bundle id, so it never touches a handler a user has since changed to
something else. After it runs the affected schemes/types are simply unset — it does not
restore whatever was set before `setDefaultHandlers.sh` first ran, since no prior state
is recorded.

**Apps > macOS > Add > Line-of-business app** → upload the `.pkg` built by
`buildUninstallPkg.sh` (`com.exxonmobil.pkg.SetOfficeDefaultHandlers.Uninstall`).
Assign **Required** to just the devices you want reverted, let it install once, then
unassign — like the install pkg it has no payload and only reruns its postinstall when
Intune reinstalls it (reassignment or version bump).

Log: `/Library/Logs/Microsoft/IntuneScripts/SetDefaultHandlers/uninstallDefaultHandlers.log`

### Compliance reporting (optional)

**Devices > macOS > Custom attributes > Add** → upload `checkDefaultHandlers.sh`,
data type **String**. Reports e.g.:

```
OK (jdoe): mailto=com.microsoft.Outlook, com.apple.mail.email=com.microsoft.Outlook, ...
NONCOMPLIANT 2 (jdoe): mailto=com.apple.mail, ...
```

## Caveats

- Verified on macOS 27.0 (26A428): for an **already logged-in** user the change is
  picked up by LaunchServices immediately, with no prompt and no logout, because
  `lsd` is killed before and after the write. A logout/login or restart is still the
  guaranteed fallback if `lsd` is busy at that moment; the daily run covers it.
- The user can still change the defaults back afterwards via Finder → Get Info or
  System Settings. The daily run will put them back.
- Directly editing `com.apple.launchservices.secure.plist` is not a documented Apple
  interface. It is the only option Apple currently leaves for silent management; file
  feedback via AppleSeed for IT asking for a real MDM payload.

## Testing on a single Mac

```sh
sudo ./setDefaultHandlers.sh
./checkDefaultHandlers.sh
```

Then log out and back in, and verify `mailto:` links open Outlook.
