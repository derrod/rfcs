# Disclaimer

**This RFC is not done, it is written with some assumptions about the current updater that may not be correct since it has been some time since I looked at it in-depth. Before submitting I will re-read the OBS updater source code to ensure that my assumptions are right, or fix the RFC where they are not.**

# Summary

This RFC proposes a number of changes to the OBS updater and update process to add new features (e.g. branches) as well as a number of potential improvements to efficiency.

# Motivation

The current OBS updater model does not support many features that have been desired (branches/opt-in betas, repairing, deleting/renaming files) and is also split between macOS and Windows. On top of that several improvements to its performance may be possible through reworking it to use newer technologies or different workflows entirely.

Additionally we have seen issues with the current way of how updates are delivered to users. The updater binary is not served from a CDN and each update request will query the server for the list of delta patches to be downloaded. These can be enough to saturate the uplink and compute capacity of the origin server(s) as they are currently set up.

This RFC suggests major changes to the existing updater to address these shortcomings and introduce new features and distribution model that may be used for adjacent projects (plugin manager) as well.

Another objective of this RFC is to overhaul the patch generation process, ideally making it transparent (open-source) as well as runnable on CI for nightly builds.

# Detailed changes

## Changes to OBS

OBS' settings menu will need to accomodate the new options for update branches and opt-in betas. Additionally it will need to accomodate changes to the process of downloading the updater executable and have new options for the "repair" feature added somewhere in the UI (e.g. under the current "Check for Updates" option).

Primarily this would consist of a new section in the "Advanced" settings, giving users the option to select from a dropdown of public branches (e.g. `stable`, `release-candidate`, `nightly`) and potentially also including an option for private branches that can be enabled using a "password" for targeted testing of specific fixes.

The update check process would also need support for handling cases where branches are removed (e.g. one-off test branches) or superseeded by the release branch (e.g. release candidate) and suggest the user switch their branch and update. 

## Changes to the updater and patch generation process

### Delta patches

Deltas shall be created using zstd, which improves generation speed of patches and in some cases can improve compression efficiency versus the current method. The update from the latest (or most commonly used) version may also be generated with a higher compression setting to further reduce the download size for users.

These newer patches may omit the signing that is currently present, as they are implicitly signed through the hash in the signed manifest.

### Manifests

The new manifests could use a binary format to reduce size and potentially improve parsing speed, examples include flatbuffers or protobuf but a custom format may also be possible.

These new manifests would include a number of new features:
- List of files to be deleted
- Additional metadata (e.g. version, app name, branch/beta)
- zstd compression (smaller size)
- Cryptographic signature

File deletion can be used to remove old files that are no longer needed (e.g. older libx264 libraries) or clean up when names have been changed (e.g. decklink-**ou**put). In extreme cases they may be used to remove specific versions of plugins that cause issues with a new version, for that purpose a file hash may be supplied and only a matching file will be deleted. A warning should be shown to the user in that case.

Additional metadata may be used in the updater to show additional progress and status information.

Compression and singing shall be done so that the manifest body is compressed and then signed, the final manifest shall then consist of a simple header (magic, body size, signature size), the compressed manifest body, and the signature.

Manifests may be unsigned or use a different signing certificate for non-release branches.

### Retrieving patches/files

The current patcher contacts the obsproject server to receive a list of delta patches (if available) before updating. This can cause a significant load on the server and this new updater is intended to work without relying on an external service to provide patch information.

There are two possible approaches to fixing this issue:

1. When downloading a file, first do a request to the CDN to check if a delta patch is available, if this 404s continue with downloading the entire file
2. Include a list of available patches in the file entry in the manifest, this list shall consist of a pair of (old hash, patch hash) pairs that the client will iterate over to find one that matches the existing file's hash (again with fallback to downloading the entire file in case of a 404)

Either option has potential downsides over the current approach. With 1. the number of 404s could increase significantly, even if cached by the CDN with a moderate TTL (e.g. 10 minutes). For the latter the manifest size would be increased, and adding more patches later on would require a regeneration of the manifest. However in either case the load on the server should still be reduced significantly, as only static files are required for the updater to work.

In either case the strucutre of how files are stored on the storage backend shall be changed so that no contact with the server is necessary to determine the patch file name.

Example:
- Delta patch: `data/obs-studio/<hash of file path>/<new version hash>/<old version hash>.patch`
- Full file: `data/obs-studio/<hash of file path>/<new version hash>/file.bin`

### Support for branches

Support for branches essentially consists of changing how the manifest is fetched from the obsproject backend. This could be as simple as introducing a new URL parameter indicating the target branch. The OBS application could indicate the desired branch to the updater via a command line parameter.

### UI updates

The UI may be updated with additional information when updating, for instance the current UI only shows the file and a progress bar. A slightly overhauled UI may show information such as
* Origin/Target version and branch
* Current file + \<current index\>/\<total files\>
* Total and per-file progress indicators
* Speed indicators (network/disk)

### (possibly) macOS compatability

Due to the complexity of the proposed changes large refactoring and rewriting may be required. This may open up an opportunity to make the updater cross-platform by moving platform dependant code to separte files so the "core" of the updater can be reused on both Windows and macOS.

## Changes to the obsproject backend

### CDN redirection for updater/manifest binaries

In order to reduce load on the obsproject servers the updater and manifest shall be downloaded from the CDN. The concern has been that relying on a CDN may cause issues when it is unavailable or otherwise has to be bypassed. For that reason the process of retrieving the updater binary and manifest should consist of the updater contacting the server and then being redirected to the CDN.

This redirection may take one of two forms:
1. 302/303/307 redirect to the CDN URL of the current updater or manifest
2. JSON response with several pieces of information such as updater location, CDN hostname, and updater version

For the updater we may reuse the existing method of checking the ETag with the CDN after the redirect happens to ensure we have the latest version. Alternatively with #2 we may simply check the version of the existing binary (if it exists).

The application's response to a given query may be cached for a short period (e.g. 1 minute) to further reduce stress on the server.

- For the updater the request may look something like `GET /updater?platform=win32`
- For manifests the request may look something like `GET /manifest?app=obs-studio&branch=stable`.

In order to avoid CDN caching causing issues the file name for the updater or manifest that the client is redirected to shall be unique to each version. For instance `updater.<commit hash>.exe` and `<branch>_<obs version>_<revision>.manifest`.

### Support for branches

Support for branches will be required not just for the manifest, but also for other elements such as the MOTD and update checks.

### New file structure

The server would need to support the new file strucutre as proposed in the client section, a cleanup script may be used to periodically remove old patches or versions. This could be done by simply removing directories that do not correspond to target hashes currently present in the latest manifest or a collection of existing manifests. Since the client will fall back to downloading the entire file deleting patches is easily possible, though regenerating manifests may be recommneded depending on the approach taken to fetching patches.

# Drawbacks

The transition from old updater to new might end up being very messy. And without the underlying system being reused for plugins or macOS the amount of work may prove to be not worth it.

# Future Work

The underlying distribution and patching methodology could be used in the future for a planned plugin manager.

Assuming plugins are not going to be loaded at runtime, choosing to install or update a plugin could close OBS and invoke the updater in a "plugin updater mode" in which it downloads and installs a list of supplied plugins by fetching their manifests and files from the OBS CDN. OBS would then be restarted like with the existing updater.

## Links

- Example manifest protobuf definition: https://gist.github.com/derrod/f0c20f098b595f0181e98fd7f6a0d7ea
