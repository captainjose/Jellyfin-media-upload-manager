# Jellyfin Media Upload Plugin

Adds a dashboard page for uploading files, an entire folder, or a zip archive directly
into one of your Jellyfin library folders. Targets Jellyfin **10.11.11**.

## What it does

- Lists your existing Jellyfin library folders as upload destinations (no manual path
  entry — pulled live from `ILibraryManager`, so it's always in sync and can't be pointed
  at an arbitrary directory).
- Three upload modes — Files, Folder, or Zip archive (see comparison below).
- Preserves folder structure on Folder and Zip uploads.
- Streams uploads to disk (doesn't buffer whole files in memory), so large video files are fine.
- Validates every destination path server-side, including zip-slip protection on archive
  extraction, and rejects path traversal attempts.
- Optionally queues a library scan automatically once the upload finishes.

## Files vs. Folder vs. Zip — which to use

| Mode | Best for | How it travels | Trade-off |
|---|---|---|---|
| **Files** | A single item or a few loose files (one movie, a couple of episodes) | Each file as its own part of one request | Simplest option, no prep work — but doesn't preserve any folder structure |
| **Folder** | A structured batch you want laid out as-is (a whole season, an album) | Every file inside uploads as its own separate HTTP request, with its relative path sent alongside it | Structure is preserved automatically, but many small requests means it's the least resilient to a dropped or flaky connection, and has the most per-file overhead |
| **Zip archive** | Large batches, many files, or an unreliable connection | The entire archive as **one** HTTP request | Most resilient and lowest overhead — but you have to zip it first, and the server briefly needs disk space for both the uploaded zip and its extracted contents |

Rule of thumb: reach for **Zip** once you're uploading more than a handful of files or
anything large enough that a mid-transfer hiccup would be annoying to redo. **Folder** is
fine for smaller structured batches on a solid connection. **Files** is for the simple case.

## Zip extraction details

- The plugin extracts each entry server-side using the same relative-path sanitizer as
  Folder uploads — an entry whose path resolves outside the destination folder (a classic
  "zip slip" attack, e.g. `../../etc/...`) is skipped and logged rather than extracted.
- Extraction is streamed entry-by-entry rather than using `ZipFile.ExtractToDirectory`
  directly, specifically so that path validation happens before any bytes are written.
- The uploaded zip itself is written to a temp file, extracted from there, then deleted —
  it is never kept in your library folder.
- **Not currently handled:** a cap on total uncompressed size ("zip bomb" protection).
  Since this plugin is admin-only (`RequiresElevation`), that's a lower-priority risk than
  it would be on a public upload form, but worth adding if you ever loosen that auth policy.

## Project layout

```
Jellyfin.Plugin.MediaUpload/
├── Jellyfin.Plugin.MediaUpload.csproj
├── Plugin.cs                      # Plugin registration, exposes the dashboard page
├── PluginConfiguration.cs         # Just the AutoScanAfterUpload toggle
├── Api/
│   └── UploadController.cs        # GET Destinations, POST Upload, POST UploadZip
├── Configuration/
│   └── configPage.html            # Dashboard UI (destination picker, file/folder inputs)
├── build.yaml                     # Plugin manifest (for a plugin repo / release packaging)
└── README.md
```

## Building

Requires the .NET 8 SDK.

```bash
cd Jellyfin.Plugin.MediaUpload
dotnet restore
dotnet build -c Release
```

This produces `bin/Release/net8.0/Jellyfin.Plugin.MediaUpload.dll`.

Package versions in the `.csproj` and `targetAbi` in `build.yaml` are already pinned to
**10.11.***/`10.11.0.0` to match your Jellyfin server. If you ever upgrade Jellyfin, bump
both to match, or Jellyfin will refuse to load the plugin on an ABI mismatch.

## Installing on benchermanlcr

Since Jellyfin runs natively via systemd there (not Docker), the plugin folder is typically:

```
/var/lib/jellyfin/plugins/Media Upload_1.0.0.0/
```

Steps:

1. `dotnet build -c Release` (above)
2. Copy the built `Jellyfin.Plugin.MediaUpload.dll` into a new folder there, e.g.:
   ```bash
   sudo mkdir -p "/var/lib/jellyfin/plugins/Media Upload_1.1.0.0"
   sudo cp bin/Release/net8.0/Jellyfin.Plugin.MediaUpload.dll "/var/lib/jellyfin/plugins/Media Upload_1.1.0.0/"
   ```
3. Restart Jellyfin: `sudo systemctl restart jellyfin`
4. In the dashboard: **Plugins → Media Upload** — the upload page should appear.

Confirm the `jellyfin` service user has write permission to your actual library paths
under `/mnt` — it already needs read access to serve media, but double-check write perms
since this plugin will be creating new files there.

## Known limitations / things to decide before production use

- **Folder uploads are per-file over HTTP.** The browser's folder picker gives you every
  file's relative path, but each still uploads as its own request payload — there's no
  single "stream a folder" primitive in a browser. For very large folders (hundreds of
  files or many GB), consider zipping client-side and unzipping server-side instead;
  that's a bigger change to the controller (extract logic + temp dir handling) — let me
  know if you want that version scaffolded too.
- **No upload resume/chunking.** A dropped connection on a huge 4K remux means starting
  that file over. Chunked upload (e.g. tus.io-style) is the fix if this becomes a problem —
  bigger scope than the initial version.
- **No duplicate-file handling policy.** Currently `FileMode.Create` silently overwrites
  anything already at that path. Decide if you want it to skip, rename, or prompt instead.
- **Auth is admin-only by design** (`RequiresElevation`). If you want non-admin techs to
  use this too, that policy needs loosening deliberately, not by accident.
