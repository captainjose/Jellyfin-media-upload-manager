# Jellyfin Media Upload Plugin

A plugin for Jellyfin that lets you upload media straight from your browser — no FTP,
no mapped drives, no SSH — and a second page for tidying up your library afterward
(rename, move, or delete things without leaving the dashboard).

**Developer:** BCSDeveloping
**Repository:** [github.com/captainjose/Jellyfin-media-upload-manager](https://github.com/captainjose/Jellyfin-media-upload-manager)
**Discord:** [Join here for help or feedback](https://discord.gg/4xKNdKHbbR)

## Before you install

A few things worth knowing up front:

- **This only works with Jellyfin version 10.11.x.** It was built and tested against
  that exact version. If your server is on a different version, it probably won't load —
  see the Troubleshooting section below for what to do about that.
- **Only admins can use it.** It uses the same login as the rest of your Jellyfin
  dashboard, so nothing extra to set up — but regular (non-admin) accounts won't see it.
- **If your Jellyfin sits behind another web server** (nginx, Caddy, Traefik, Apache —
  common if you access it through a domain name rather than directly), that web server
  might block big files before they even reach Jellyfin. See Troubleshooting if uploads
  of larger files keep failing.
- **Jellyfin needs permission to write** to whatever folders you point this at, not just
  read from them. It already reads your media to play it back; this plugin also needs it
  to be able to save new files there.

## How to use it

Once it's installed (see below), you'll find two new items in the left-hand menu of
your Jellyfin dashboard: **Media Upload** and **Media Manager**.

### Uploading media

1. Open **Media Upload** from the sidebar.
2. Click the library folder you want to upload into (Movies, TV Shows, whatever you've
   set up) — it's already listed for you.
3. *(Optional)* Type a name into **Subfolder** if you want your upload to go into its
   own new folder — handy when adding a brand-new show or movie. Leave it blank to
   upload straight into the folder you picked.
4. Choose how you're uploading:
   - **Files** — pick one or more files by hand.
   - **Folder** — pick a whole folder at once; whatever's inside comes along with it,
     keeping the same layout.
   - **Zip archive** — pick a single `.zip` file, and the plugin unpacks it into place
     for you. This is the best choice for a big batch of files, since it travels as one
     upload instead of many, so it's far less likely to get interrupted partway through.
5. Click the upload button. You'll see every file listed with its own progress, so you
   can watch each one finish (or catch it early if something goes wrong).
6. When it's done, Jellyfin will automatically notice the new media (as long as "Scan
   library automatically after upload" stays checked, which it is by default).

### Managing what's already there

1. Open **Media Manager** from the sidebar.
2. Click a library folder to browse it — click into subfolders the same way you would
   in a normal file browser, and use the trail of links near the top to jump back up.
3. For anything in the list, you can:
   - **Rename** it — type the new name and you're done.
   - **Move** it — pick a destination library folder (can be the same one or a
     different one entirely), and optionally give it a new subfolder to move into.
     Great for fixing an upload that landed in the wrong place.
   - **Delete** it — you'll be asked to confirm first, since this can't be undone.
4. Use the **New folder** button to create an empty folder anywhere you're browsing.

## What it does

- Shows your actual Jellyfin library folders as upload destinations — you never have
  to type a file path by hand, and it can't be pointed anywhere outside your library.
- Three ways to upload: individual files, a whole folder, or a zip archive.
- Keeps folder structure intact when uploading a folder or a zip.
- Handles big files without choking — it doesn't try to hold the whole thing in memory.
- Checks every file's destination is safe before writing anything to disk.
- Can automatically tell Jellyfin to scan for new media once an upload finishes.
- Optional "subfolder" field so a new show or movie can get its own folder in one step.
- A separate **Media Manager** page to rename, move, or delete existing files and
  folders — including moving something into a brand-new subfolder along the way.

## Files vs. Folder vs. Zip — which should I use?

| Mode | Good for | Why |
|---|---|---|
| **Files** | One or two things — a single movie, a couple of episodes | Quickest option, nothing to prepare first |
| **Folder** | A neatly organized batch, like a full season | Keeps the folder layout, but sends each file separately, so a shaky connection can leave you redoing some files |
| **Zip archive** | Big batches, lots of files, or a connection you don't fully trust | Sends everything as one upload, so it's the most reliable option for anything large |

Simple rule: for just a file or two, use **Files**. For a well-organized batch on a
solid connection, **Folder** is fine. For anything big, or if your connection tends to
drop, zip it up first and use **Zip archive**.

## Project layout

```
Jellyfin.Plugin.MediaUpload/
├── Jellyfin.Plugin.MediaUpload.csproj
├── Plugin.cs                      # Registers the plugin and both dashboard pages
├── PluginConfiguration.cs         # The "auto-scan after upload" setting
├── Api/
│   ├── PathSecurity.cs            # Shared safety checks used by every endpoint below
│   ├── UploadController.cs        # Handles uploading files, folders, and zips
│   └── MediaManagerController.cs  # Handles browsing, renaming, moving, and deleting
├── Configuration/
│   ├── configPage.html            # Upload page layout
│   ├── configPage.js              # Upload page behavior
│   ├── mediaManagerPage.html      # Media Manager page layout
│   └── mediaManager.js            # Media Manager page behavior
├── build.yaml                     # Plugin info used when packaging a release
└── README.md
```

## Building it yourself

You'll need the **.NET 9 SDK** to build this (Jellyfin 10.11.x runs on .NET 9).

On Linux, Ubuntu's built-in package sources often only have older .NET versions, so the
most reliable way to get .NET 9 specifically is Microsoft's own install script:
```bash
wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 9.0
```
This puts it in `~/.dotnet`. Make it available in your terminal:
```bash
export DOTNET_ROOT=$HOME/.dotnet
export PATH=$DOTNET_ROOT:$PATH
dotnet --version   # should print a 9.0.x version
```
Add those same two `export` lines to your `~/.bashrc` too, or you'll have to repeat
this every time you open a new terminal.

Then build it:
```bash
cd Jellyfin.Plugin.MediaUpload
dotnet restore
dotnet build -c Release
```
This creates `bin/Release/net9.0/Jellyfin.Plugin.MediaUpload.dll` — that single file
is the whole plugin.

## Installing

### Option 1: Add the plugin repository (easiest)

Jellyfin can install and manage the plugin for you, and it sets up the folder permissions
correctly on its own, so you skip the trickiest step of the manual install.

1. In Jellyfin, go to **Dashboard → Plugins → Repositories** and click **+**.
2. Give it any name (for example, `Media Upload`) and paste this as the URL:
   ```
   https://github.com/captainjose/Jellyfin-media-upload-manager/releases/latest/download/manifest.json
   ```
   This link always points to the newest release, so you won't need to change it when
   updates come out.
3. Save. **This only tells Jellyfin where to look — it doesn't install anything yet.**
4. Open **Dashboard → Plugins → Catalog**, find **Media Upload**, and click **Install**.
5. Restart Jellyfin when it asks you to.

### Option 2: Manual install

Download the DLL directly:
[Jellyfin.Plugin.MediaUpload.dll](https://github.com/captainjose/Jellyfin-media-upload-manager/releases/latest/download/Jellyfin.Plugin.MediaUpload.dll)

Where Jellyfin looks for plugins depends on how you've set it up:

| Setup | Plugin folder is usually here |
|---|---|
| Installed directly on Linux | `/var/lib/jellyfin/plugins/` |
| Docker | Inside whichever folder you mounted as Jellyfin's config volume, then `plugins/` |
| Windows | `%ProgramData%\Jellyfin\Server\plugins\` |

Steps:

1. Download the `.dll` using the link above, or build it yourself (see "Building it yourself").
2. Make a folder for it and put the file inside:
   ```bash
   sudo mkdir -p "/var/lib/jellyfin/plugins/Media Upload_1.1.0.0"
   sudo cp bin/Release/net9.0/Jellyfin.Plugin.MediaUpload.dll "/var/lib/jellyfin/plugins/Media Upload_1.1.0.0/"
   ```
   (use whichever path matches your setup from the table above)
3. **Make sure Jellyfin actually owns that folder.** This trips people up more than
   anything else. Right now the folder probably belongs to whoever ran the `sudo`
   command — but Jellyfin runs as its own separate user account, and needs to be able
   to write into that folder, not just read the file inside it. Fix it like this:
   ```bash
   ls -la /var/lib/jellyfin/plugins/          # see what owner your other plugins use
   sudo chown -R jellyfin:jellyfin "/var/lib/jellyfin/plugins/Media Upload_1.1.0.0"
   ```
4. Restart Jellyfin (`sudo systemctl restart jellyfin`, or restart the container).
5. Open the dashboard — **Media Upload** and **Media Manager** should both show up in
   the sidebar.

## If something's not working

**Jellyfin won't start after I installed this**
This is almost always the folder-ownership thing from step 3 above, not a bug in the
plugin. Check your Jellyfin log — if you see a mention of "access denied" or
"permission denied" near a file called `meta.json`, that confirms it. Fix the folder's
owner as shown above and restart.

**The plugin doesn't show up at all**
Your Jellyfin server version probably doesn't match what this plugin was built for
(10.11.x). Check your server's version in the dashboard, then look in the
`Jellyfin.Plugin.MediaUpload.csproj` file — the package versions listed there need to
match your server's version, and `<TargetFramework>` needs to say `net8.0` instead of
`net9.0` if you're on an older Jellyfin (10.9.x or 10.10.x).

**I added the repository, but the plugin isn't installed**
Adding the repository only tells Jellyfin where to look — it doesn't install anything
by itself. Go to **Dashboard → Plugins → Catalog**, find **Media Upload**, click
**Install**, and restart Jellyfin once it finishes. (Restarting before you click Install
won't do anything, since there's nothing to load yet.)

**Media Upload doesn't show up in the Catalog**
Check that the repository is still listed under **Dashboard → Plugins → Repositories**
and that the URL was pasted exactly, with nothing cut off at either end. You can also
paste that URL into a browser — you should see a small block of text (or a file
download) rather than an error page. If you're on a Jellyfin version other than 10.11.x,
the Catalog hides it on purpose, since this build wouldn't work there.

**Big files fail to upload, but small ones work fine**
If Jellyfin sits behind another web server (nginx, Caddy, Traefik, Apache), that's
almost certainly the cause — it's blocking the file before it reaches Jellyfin at all.
Each one has its own setting to fix this:
- **nginx**: add `client_max_body_size 0;` (this means "no limit") to the settings for
  the site that proxies to Jellyfin.
- **Apache**: add `LimitRequestBody 0` in the same place.
- **Caddy**: doesn't limit this by default — but if you've set a `request_body` size
  limit yourself, remove or raise it.
- **Traefik**: also has no limit by default — check for a size-limiting rule you may
  have added, and raise or remove it.

**The list of folders is empty, or the page looks like it's showing an old version**
Your browser is probably holding onto an old cached copy of the page. In your browser's
developer tools (press F12), go to the **Application** tab, find **Service Workers**,
and click **Unregister**. Then, in the same tab, click **Clear site data**. Reload the
page after that.

**I uploaded something but it's not showing up in my library**
Make sure "Scan library automatically after upload" is turned on (it is by default). If
it's off, or you don't want to wait, you can trigger a scan yourself from
**Dashboard → Scheduled Tasks → Scan Media Library**.

## Things this doesn't do (yet)

- **Uploading a folder sends each file separately.** There's no way for a browser to
  send a whole folder as one upload, so with Folder mode, a connection hiccup partway
  through means redoing whatever didn't finish. Zip mode doesn't have this problem.
- **No pause/resume.** If an upload gets interrupted, you'll need to start that file
  over from the beginning.
- **Uploading a file with the same name as one that already exists will overwrite it**,
  with no warning. Worth being careful with names until this gets a proper fix.
- **This is admin-only on purpose.** If you'd want other, non-admin accounts to be able
  to use it too, that's a deliberate change someone would need to make, not something
  that happens by accident.
