Use **Windows → WSL2 Ubuntu → Docker Desktop** to run this ZIP. You do **not** need to install Go, Node.js or PostgreSQL directly on Windows for the container-based setup.

I checked the final ZIP’s checksum and setup scripts. **The package contains source code; its complete Windows/Docker runtime tests have not yet been demonstrated.** The steps below show how to run those checks and identify any failures accurately.

**1. Understand what you will test**

The ZIP still contains **SvelteKit + Go + PostgreSQL**.

| Feature | Available in the default local setup? |
|---|---|
| Public pages and responsive layout | Yes |
| Guest entry and account registration | Yes |
| Profiles, preferences and account recovery | Yes |
| Matchmaking and text chat | Yes |
| Reports and blocks | Yes |
| Mutual connections and friend invitations | Yes |
| Private moderation | Yes, after creating an operator |
| Actual audio calls | Requires separate TURN setup |
| Cashfree sandbox checkout | Requires separate payment configuration |
| Full privacy/operations workflows | Requires separate operations configuration |
| Proven 200-connected/25-call capacity | Requires dedicated load and real-media tests |

Start with the default local setup. A disabled payment or audio button is not, by itself, a defect.

**2. Prepare your Windows computer**

For a comfortable development setup, I suggest:

- A supported Windows 11 installation.
- Preferably **16 GB RAM**.
- At least **30 GB free disk space** for images, dependencies and browser tests.
- Internet access for downloading dependencies.
- Chrome or Edge; having both helps simulate two users.

These are practical development allowances, not measured minimum requirements.

Open **Task Manager → Performance → CPU** and check whether **Virtualization** is enabled. If it is disabled, WSL2 may require a BIOS/UEFI setting change.

You will use two terminals:

| Terminal | Used for |
|---|---|
| **PowerShell** | Windows setup and initial checksum check |
| **Ubuntu** | Every application command after WSL installation |

Do not paste Bash commands into PowerShell.

**3. Download and verify both files**

Create this folder using Windows File Explorer:

```text
C:\NoTitleDownloads
```

Place these two files inside it:

```text
notitle-final.zip
notitle-final.zip.sha256
```

Enable **View → Show → File name extensions** in Explorer. Ensure that the checksum file has not been renamed to `.sha256.txt`.

Open a normal **PowerShell** window and run:

```powershell
Set-Location C:\NoTitleDownloads

$expected = ((Get-Content .\notitle-final.zip.sha256 -Raw).Trim() -split '\s+')[0]
$actual = (Get-FileHash .\notitle-final.zip -Algorithm SHA256).Hash

if ($expected -notmatch '^[0-9a-fA-F]{64}$') {
    throw "The checksum file does not contain a valid SHA-256 value."
}

if ($actual -ine $expected) {
    throw "Checksum mismatch. Download the matching ZIP and checksum again."
}

Write-Host "Checksum OK: the ZIP matches the checksum file."
```

Expected result:

```text
Checksum OK: the ZIP matches the checksum file.
```

A mismatch means you should stop and obtain the matching files. The checksum verifies the downloaded bytes; it does not prove that the application passes its tests.

**4. Install Ubuntu through WSL2**

Open **PowerShell as Administrator**.

For a new installation, run:

```powershell
wsl --install -d Ubuntu-24.04
```

Restart Windows if requested.

Open **Ubuntu 24.04** from the Start menu and create your Linux username and password. The password will not appear while you type; that is normal.

In PowerShell, check:

```powershell
wsl --update
wsl --list --verbose
```

The Ubuntu distribution should show **VERSION 2**.

If that distribution shows version 1, run:

```powershell
wsl --set-version Ubuntu-24.04 2
```

If you already have a suitable Ubuntu WSL2 distribution, you can use it instead. Substitute its actual distribution name in the commands above. Microsoft documents the installation and version-selection commands here. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/wsl/install?utm_source=chatgpt.com)

**5. Install and connect Docker Desktop**

Download Docker Desktop from the [official Windows installation page](https://docs.docker.com/desktop/setup/install/windows-install/).

After installation:

1. Start **Docker Desktop**.
2. Open **Settings → General**.
3. Enable **Use the WSL 2 based engine**, if that option is shown.
4. Open **Settings → Resources → WSL Integration**.
5. Enable integration for your Ubuntu distribution.
6. Apply the changes and wait for Docker to start.

Use **Linux containers**. If WSL Integration is missing, check whether Docker is currently using Windows containers. Docker documents this integration in its WSL guide. [Docker Docs](https://docs.docker.com/desktop/features/wsl/?utm_source=chatgpt.com)

Open the **Ubuntu terminal** and run:

```bash
docker version
docker compose version
docker info
```

Verify:

- `docker version` displays both client and server information.
- `docker compose version` reports **2.30 or newer**.
- `docker info` completes without a connection error.

You do not need to install another Docker Engine inside Ubuntu when using Docker Desktop’s WSL integration.

**6. Install the small Ubuntu prerequisites**

Run in **Ubuntu**:

```bash
sudo apt update
sudo apt install -y python3 unzip curl make ca-certificates
```

Verify Python:

```bash
python3 --version
```

The package requires **Python 3.11 or newer** for some test scripts.

Go, Node.js and PostgreSQL will be supplied by Docker images.

**7. Copy and extract the ZIP into Ubuntu**

Keep the working project in Ubuntu’s Linux filesystem. Docker recommends this for better bind-mount performance. [docs.docker.com](https://docs.docker.com/desktop/features/wsl/best-practices/?utm_source=chatgpt.com)

Run in **Ubuntu**:

```bash
mkdir "$HOME/notitle-windows-test"
cd "$HOME/notitle-windows-test"

cp /mnt/c/NoTitleDownloads/notitle-final.zip .
cp /mnt/c/NoTitleDownloads/notitle-final.zip.sha256 .

sha256sum -c notitle-final.zip.sha256
```

Expected:

```text
notitle-final.zip: OK
```

If `mkdir` reports that the folder already exists, choose a new folder name rather than extracting over previous work.

Extract:

```bash
unzip notitle-final.zip
cd notitle
```

Verify your location:

```bash
pwd
ls
```

Your path should end with:

```text
/notitle-windows-test/notitle
```

You should see files and folders such as:

```text
START_HERE.md
README.md
Makefile
server
web
scripts
deploy
docs
```

Restore executable permissions on the supplied shell scripts:

```bash
find scripts deploy -type f -name '*.sh' -exec chmod u+x {} +
```

**All remaining application commands run from this `notitle` directory unless stated otherwise.**

**8. Prepare dependencies and development secrets**

Run:

```bash
umask 077
set -o pipefail
./scripts/bootstrap.sh prepare 2>&1 | tee ../notitle-bootstrap.log
```

This command:

- Downloads the pinned build images.
- Generates genuine Go and npm dependency-lock files.
- Downloads dependencies.
- Formats source files.
- Generates development secrets.
- Creates the local environment configuration.

It can take some time on the first run.

A successful run ends with:

```text
Prepared local secrets and real dependency locks. Review changed source/lock files; no application containers started.
```

Check that the required files exist without displaying their contents:

```bash
test -s server/go.sum && echo "Go dependency file exists"
test -s web/package-lock.json && echo "npm lockfile exists"
test -f deploy/dev/.env && echo "Local environment file exists"
test -d .local/secrets && echo "Local secret directory exists"
```

**Do not continue if preparation failed.** Missing image tags, unavailable package versions, registry access errors or compilation problems need to be resolved at the reported step. Do not substitute arbitrary dependency versions or create empty lockfiles.

You do not need to enter database passwords manually. Preparation generates them.

**9. Start the application**

Run:

```bash
./scripts/start.sh dev
```

The script starts PostgreSQL, applies database migrations, initializes private keys, and starts the backend and frontend.

Expected final message:

```text
Development is ready at http://localhost:5173 ...
```

Check the containers:

```bash
./scripts/compose.sh dev ps
```

The main running services should include:

- `postgres`
- `api`
- `web`

Migration and key-initialization services perform one-time work; they do not need to remain running.

Check health:

```bash
curl --fail --show-error -i http://localhost:5173/health/live
curl --fail --show-error -i http://localhost:5173/health/ready
```

Both should return **HTTP 200** once local startup completes.

Check the database migration status:

```bash
./scripts/compose.sh dev run --rm migrate status
```

The cumulative application should be at **schema version 9**.

If startup fails, inspect:

```bash
./scripts/compose.sh dev logs --tail=100 api web postgres
```

**10. Open the website in Windows**

Open Chrome or Edge and visit:

```text
http://localhost:5173
```

Use **`localhost` consistently**. Do not alternate between `localhost`, `127.0.0.1` and your computer’s IP address: the application checks the browser origin.

Useful local URLs:

| URL | Purpose |
|---|---|
| `http://localhost:5173/` | Homepage |
| `http://localhost:5173/chat` | Guest entry and real chat |
| `http://localhost:5173/register` | Account registration |
| `http://localhost:5173/login` | Account login |
| `http://localhost:5173/recovery` | Account recovery |
| `http://localhost:5173/profile` | Profile |
| `http://localhost:5173/preferences` | Matching preferences |
| `http://localhost:5173/friends` | Mutual connections |
| `http://localhost:5173/settings` | Account controls |
| `http://localhost:5173/account/premium` | Premium account screen |
| `http://localhost:5173/design-preview` | Simulated design states only |

**Use `/chat` for functional testing.** `/design-preview` does not prove that matchmaking, payments or audio work.

**11. Test two independent users**

Use:

- **Chrome** as User A.
- **Edge** as User B.

Alternatively, use one normal window and one private/incognito window.

Two ordinary tabs in the same browser profile usually share the same identity. They are suitable for takeover testing, not for simulating two people.

In both browsers:

1. Open `/chat`.
2. Complete the required gender selection.
3. Confirm the adult requirement and policy acceptance.
4. Enter as a guest.
5. Keep matching preferences at **Everyone** for the initial test.
6. Start searching in both browsers.

Expected:

- Both users enter the same conversation.
- Each sees the permitted public information about the other.
- Private account identifiers and usernames are not displayed to the other user.

Test these actions:

| Action | Expected result |
|---|---|
| A sends `Hello from User A` | B receives the message |
| B replies `Hello from User B` | A receives the reply |
| A sends `<b>local test</b>` | It appears as literal text, not rendered HTML |
| A selects **End match** | Both users return to idle |
| Both search again | They can match again when eligible |
| A selects **Next match** | A searches again; B returns to idle |

The matching rules discourage immediate repeat pairs. With only two users, allow several seconds and ensure **both are searching** before diagnosing a failure.

**12. Test account creation, profiles and recovery**

Use disposable test accounts.

For User A:

1. Open `/register`.
2. Choose a unique private username.
3. Enter a password satisfying the displayed rules.
4. Complete the profile and confirmations.
5. If registering from a guest session, confirm the existing gender when requested.
6. Submit registration.
7. Save the recovery code privately when shown.
8. Confirm that you saved it.

Repeat for User B.

Then test:

| Test | Procedure | Expected result |
|---|---|---|
| Profile edit | Change display name or bio in `/profile` | Saved values remain after reopening the page |
| Preferences | Update available choices in `/preferences` | Valid choices save |
| Login | Log out, then log in | Correct credentials work |
| Invalid password | Make one deliberate incorrect attempt | Login is rejected |
| Recovery | Log out, open `/recovery`, use the saved code | Password reset succeeds and replacement recovery information is handled as shown |
| Old password | Try the password replaced during recovery | It no longer works |

A Free account should not gain Premium-only matching preferences merely because you selected a frontend control.

Do not repeatedly submit wrong credentials: rate limiting is expected behavior.

**13. Test mutual connections and invitations**

With both users in an active conversation:

1. User A chooses **Stay Connected**.
2. Check that one user’s choice alone does not create a completed friendship.
3. User B also chooses **Stay Connected**.
4. Open `/friends` and verify the accepted connection.
5. End the current conversation.
6. Keep both users online and available.
7. Send a friend invitation from A.
8. Accept it from B.

Expected: a conversation starts between those users.

Repeat to test:

- Declining an invitation.
- Leaving an invitation unanswered until its approximately 30-second expiry.
- Attempting to invite a busy user.
- Removing a connection.

An unanswered invitation should not remain usable indefinitely, and an invitation should not interrupt someone who is already busy.

**14. Test reporting, blocking and account deletion**

Run these after the friendship tests because they intentionally change relationships.

For report/block testing:

1. Start a conversation.
2. Send a clearly artificial message, such as:

   ```text
   LOCAL TEST MESSAGE — synthetic report evidence
   ```

3. Open the report action.
4. Select the available server-provided evidence.
5. State clearly that this is a local synthetic test.
6. Submit.
7. Block the other test identity.

Expected:

- The active interaction ends when blocking applies.
- The blocked pair cannot match while the block remains effective.
- Unblocking does not automatically restore friendship.

For deletion, create a **third disposable account**:

1. Open `/settings`.
2. Complete recent password confirmation if requested.
3. Read and confirm the deletion action.
4. Submit.
5. Try signing in again.

Expected: account access is revoked.

**Access revocation and completed data erasure are different outcomes.** The default local configuration does not establish the full approved retention/purge workflow, so do not mark complete erasure as passed based only on a successful deletion request.

**15. Create a local private-admin account**

This is optional for basic chat testing, but necessary to review submitted reports.

There is **no default administrator username/password**.

Keep the local application running. In the same Ubuntu terminal, from the project root, build the provisioning image:

```bash
docker build -f server/Dockerfile -t notitle-admin-provision:local .
```

Create a private temporary directory outside the project:

```bash
NOTITLE_ADMIN_DIR=$(mktemp -d "$HOME/notitle-admin-local.XXXXXX")
chmod 700 "$NOTITLE_ADMIN_DIR"
```

Create the operator input interactively. The password is entered without being echoed or placed in shell history:

```bash
python3 - "$NOTITLE_ADMIN_DIR/operator-input.json" <<'PY'
import getpass
import json
import os
import sys

password = getpass.getpass("Choose a local admin password (12–128 characters): ")
confirmation = getpass.getpass("Confirm the password: ")

if password != confirmation:
    raise SystemExit("Passwords do not match.")
if not 12 <= len(password) <= 128:
    raise SystemExit("Password must contain 12–128 characters.")

fd = os.open(sys.argv[1], os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, "w") as stream:
    json.dump({
        "username": "localowner",
        "password": password,
        "roles": ["owner"]
    }, stream)

print("Private operator input created.")
PY
```

If that command reports an error, resolve it before continuing.

Copy the local migration connection file and set the ownership required by the provisioning container:

```bash
cp .local/secrets/migration_database_url \
  "$NOTITLE_ADMIN_DIR/migration-database-url"

chmod 600 \
  "$NOTITLE_ADMIN_DIR/operator-input.json" \
  "$NOTITLE_ADMIN_DIR/migration-database-url"

sudo chown -R 65532:65532 "$NOTITLE_ADMIN_DIR"
```

Run the one-time provisioning command:

```bash
docker run --rm --read-only --user 65532:65532 \
  --network notitle-local_default \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --ulimit core=0 \
  --mount "type=bind,src=$NOTITLE_ADMIN_DIR,dst=/private" \
  --mount type=volume,src=notitle-local_safety_keys,dst=/keys,readonly \
  --entrypoint /adminctl \
  notitle-admin-provision:local \
  --input-file /private/operator-input.json \
  --database-url-file /private/migration-database-url \
  --mfa-key-file /keys/admin_mfa_key \
  --enrollment-file /private/new-operator-enrollment.txt
```

These network/volume names assume you have used the default Compose project name throughout this guide.

Read the enrollment file privately:

```bash
sudo less "$NOTITLE_ADMIN_DIR/new-operator-enrollment.txt"
```

Use the supplied secret to add a **time-based code** to your authenticator app. Press `q` to exit `less`. Do not share or screenshot this enrollment material.

Open:

```text
http://localhost:9090
```

Sign in with:

- Username: `localowner`
- The password you chose.
- The current authenticator code.

After verifying login and preserving your credentials appropriately, remove the temporary provisioning files:

```bash
sudo rm -- \
  "$NOTITLE_ADMIN_DIR/operator-input.json" \
  "$NOTITLE_ADMIN_DIR/migration-database-url" \
  "$NOTITLE_ADMIN_DIR/new-operator-enrollment.txt"

sudo rmdir -- "$NOTITLE_ADMIN_DIR"
unset NOTITLE_ADMIN_DIR
```

Ordinary deletion does not guarantee forensic erasure; use this procedure only for disposable local credentials.

In the moderation interface, verify that you can:

1. Find your synthetic report.
2. Open it with the requested work purpose.
3. View submitted evidence.
4. Add a test note.
5. Resolve the case.

If provisioning reports an ambiguous database failure, do not repeatedly recreate the same operator. Inspect the error and existing state first.

**16. Test mobile and desktop layouts**

In Chrome or Edge:

1. Open the application.
2. Press **F12**.
3. Press **Ctrl + Shift + M** to toggle device emulation.
4. Choose **Responsive**.
5. Test these viewport widths:

| Width | Purpose |
|---:|---|
| 320 px | Small mobile layout |
| 375 px | Common mobile layout |
| 390 px | Common mobile layout |
| 430 px | Larger mobile layout |
| 768 px | Tablet layout |
| 1366 px | Laptop layout |
| 1920 px | Desktop layout |

At each relevant size, test:

- Homepage navigation.
- Guest entry and registration.
- Chat input and send button.
- Long messages.
- Friends and report dialogs.
- Profile/settings forms.
- Vertical scrolling.
- Unexpected horizontal scrolling.
- Keyboard navigation with **Tab**.
- Browser zoom at **200%**.

**Device emulation does not test a real phone’s keyboard, microphone or browser behavior.**

The supplied local setup binds to your laptop’s loopback address. A phone cannot reach it by entering `localhost`, because that refers to the phone itself. Real-phone testing needs a separately configured reachable origin and, for microphone testing, an appropriate secure context. Avoid changing network exposure and origins casually just to make the URL open.

**17. Test duplicate tabs and reconnect behavior**

For duplicate tabs:

1. In User A’s browser profile, open a second `/chat` tab.
2. Observe the takeover flow.
3. Confirm that taking over does not create two independent users.
4. Verify that the older tab cannot continue controlling the replacement interaction.

For reconnects, the automated smoke tests below are the more reproducible local check.

You can also try the browser’s **Network → Offline** setting briefly and return it to **Online**, but first verify that the WebSocket actually disconnects. Browser network emulation does not always affect existing sockets the same way.

The intended reconnect window is 25 seconds after the server recognizes a matched disconnection. Refreshing the page can discard in-memory resume information, so it is not equivalent to a brief transport interruption.

Turning off Wi-Fi does not reliably test a `localhost` connection: the browser and server are on the same computer.

**18. Run the full automated verification**

Save your manual observations first. Stop the development stack to free resources:

```bash
./scripts/compose.sh dev down --remove-orphans
```

This preserves your normal local database volumes.

Run:

```bash
set -o pipefail
./scripts/verify.sh 2>&1 | tee ../notitle-verification.log
```

Immediately after completion, check:

```bash
echo $?
```

Expected exit status:

```text
0
```

The suite is designed to run:

- Source and release checks.
- Python operations fixtures.
- Go checks and tests.
- Real PostgreSQL integration.
- Database permissions checks.
- Migration up/down/up tests.
- Frontend type and lint checks.
- Browser fixtures.
- Dependency-security checks.
- Static-site/API integration smoke tests.

**Its migration-reversal tests use separately named disposable databases.** Do not change those scripts to point at your retained local database.

A successful run prints a completion message beginning:

```text
Executed build, backend, database, migration, frontend/browser-fixture...
```

If it fails:

1. Record the first failing stage.
2. Read the error immediately before the failure.
3. Keep the log.
4. Do not treat earlier passing stages as a complete pass.
5. Resolve the failure before repeating the relevant checks.

Browser fixtures and simulated payment screens do not prove real-device audio or real Cashfree integration.

**19. Test the compiled static version**

After the automated tests, start:

```bash
./scripts/start.sh static
```

Open:

```text
http://localhost:8080
```

Check:

```bash
./scripts/compose.sh static ps

curl --fail --show-error -i http://localhost:8080/health/live
curl --fail --show-error -i http://localhost:8080/health/ready
```

Repeat the important two-browser journeys at port **8080**:

- Guest matching.
- Text messages.
- Account login.
- Friends.
- Report/block.
- Mobile layout.

Additional checks:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  http://localhost:8080/design-preview
```

Expected:

```text
404
```

Check an unsupported URL:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  http://localhost:8080/this-page-does-not-exist
```

Expected:

```text
404
```

This mode tests the compiled frontend served by Caddy. It still uses local preview settings and does not constitute a production launch.

**20. Understand the remaining audio and payment tests**

For actual audio, the package requires:

- A configured coturn relay.
- Its private supervisor/watchdog.
- Valid relay credentials and control certificates.
- Suitable network reachability.
- Explicit browser microphone permission.
- The application’s audio configuration.

Follow these files inside the extracted project:

```text
deploy/turn/README.md
deploy/audio/README.md
docs/phase7-relay-runbook.md
docs/phase7-acceptance.md
```

Two browser windows on one laptop are useful for initial call controls, but use headphones to avoid feedback. They do not replace real-phone and different-network testing.

For Cashfree sandbox, follow:

```text
docs/FINAL_PAYMENT_GUIDE.md
docs/FINAL_ENVIRONMENT_REFERENCE.md
deploy/payments/README.md
docs/phase8-acceptance.md
```

The sequence is:

1. Obtain **sandbox** Payment Gateway credentials.
2. Store them in protected backend files.
3. Configure matching sandbox backend/frontend environments.
4. Prepare the test catalog.
5. Rebuild the frontend with sandbox checkout enabled.
6. Provide an appropriate reachable webhook endpoint.
7. Test successful, failed, cancelled and repeated payment events.
8. Check server-verified Premium grants.
9. Test refunds and reconciliation.

Cashfree cannot deliver webhooks directly to your laptop’s `localhost`. A deliberately configured sandbox endpoint is needed.

**Changing `PAYMENTS_ENABLED` or `AUDIO_ENABLED` alone is insufficient.** Also, passing laptop tests does not establish the 200-connected/25-call capacity requirement; that requires the workloads in `docs/phase10-capacity.md`.

**21. Stop and restart safely**

If running development mode:

```bash
./scripts/compose.sh dev down --remove-orphans
```

If running static mode:

```bash
./scripts/compose.sh static down --remove-orphans
```

**Do not add `-v` or `--volumes` to these normal shutdown commands.** That can delete retained data.

To start again later:

1. Start Docker Desktop.
2. Open Ubuntu.
3. Run:

```bash
cd "$HOME/notitle-windows-test/notitle"
./scripts/start.sh dev
```

Or, for the compiled version:

```bash
cd "$HOME/notitle-windows-test/notitle"
./scripts/start.sh static
```

Accounts should remain in PostgreSQL. Active conversations and temporary guest state are expected to end when the backend stops.

**22. Troubleshoot common problems**

| Problem | What to check |
|---|---|
| `docker: command not found` | Docker Desktop’s integration with the correct Ubuntu distribution |
| Cannot connect to Docker daemon | Start Docker Desktop and wait until its engine is ready |
| Compose version rejected | Update Docker Desktop; this project requires Compose 2.30+ |
| `Permission denied` for a script | Run the shell-script `chmod` command from step 7 |
| Missing `go.sum` or `package-lock.json` | Preparation did not finish successfully |
| `manifest unknown` or package version unavailable | Record the exact pinned image/package; it needs verification, not a guessed replacement |
| Registry timeout or access denied | Check authorized network/proxy access |
| Container exits with code 137 | Check Docker’s memory use and whether the process was killed |
| Browser cannot open the site | Check `compose ... ps` and the correct port |
| HTTP 403 / Origin rejection | Use the exact configured `http://localhost:5173` or `:8080` origin |
| Users do not match | Check separate identities, both searching, compatible preferences and no block |
| One tab disconnects another | Expected duplicate-identity protection; use two browser profiles |
| Admin login fails | Check the operator exists, password, current TOTP and clock synchronization |
| Call or checkout unavailable | These features are disabled in the default local configuration |

For a development startup failure, collect:

```bash
./scripts/compose.sh dev ps
./scripts/compose.sh dev logs --tail=100 api web postgres
```

For a static startup failure, replace `dev` with `static`.

When sharing an error for troubleshooting, provide **the command, first error and relevant short log excerpt**. Keep passwords, recovery codes, authenticator enrollment, database connection files and gateway keys private.

Your initial local-testing milestone is: **both health endpoints pass, two independent users can chat, accounts and friendships work, a synthetic report can be reviewed, the compiled site works, and `verify.sh` exits successfully.**
