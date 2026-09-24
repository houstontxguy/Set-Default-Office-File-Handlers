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

```sh
sudo ./setDefaultHandlers.sh
./checkDefaultHandlers.sh
```

Then log out and back in, and verify `mailto:` links open Outlook.
