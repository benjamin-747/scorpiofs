![ScorpioFS](docs/images/banner.png)

## Scorpio - FUSE Support for Mega/Monorepo Client

### What's the Fuse?

FUSE is the abbreviation for "FileSystem in Userspace".It's an interface for userspace programs to export a filesystem to the linux kernel.
The FUSE project consists of two components: the fuse kernel module (maintained in the regular kernel repositories) and the libfuse userspace library (maintained in this repository).
![FUSE](docs/images/FUSE_VFS.png)

When VFS receives a file access request from the user process and this file belongs to a certain fuse file system, it will forward the request to a kernel module named "fuse". Then, "fuse" converts the request into the protocol format agreed upon with the daemon and transmits it to the daemon process.

Currently, there have been many successful fuse based projects,

- [s3fs](https://github.com/s3fs-fuse/s3fs-fuse)
 makes you operate files and directories in S3 bucket like a local file system
 ![Github stars](https://img.shields.io/github/stars/s3fs-fuse/s3fs-fuse.svg)
- [sshfs](https://github.com/libfuse/sshfs) 
allows you to mount a remote filesystem using SFTP
![Github stars](https://img.shields.io/github/stars/libfuse/sshfs.svg)
- [google-drive-ocamlfuse](https://github.com/astrada/google-drive-ocamlfuse.git) lets you mount your Google Drive on Linux.
![Github stars](https://img.shields.io/github/stars/astrada/google-drive-ocamlfuse.svg)

### Why the Mega Need a FUSE?

Because the code organization requirements are different from the existing popular distributed version management software Git, clients targeting Monorepo need to implement various additional features to support code pull tasks for large repositories. These requirements include:

1. **Partial clone**: reduces the time required to obtain a working repository by not immediately downloading all Git objects.

2. **Background prefetch**: Download Git object data from all remote sources every hour, reducing the time required for front-end Git fetch calls.

3. **Sparse checkout**: Restrict the size of the working directory.

4. **File system monitor**: tracks recently modified files, eliminating the need for Git to scan the entire work tree.

5. **Submit graph**: Accelerate submission traversal and reachability calculations, and speed up commands such as git log.

6. **Multi pack index**: Implement fast object lookup in many package files.

7. **Incremental repackage**: Using multiple package indexes, repackage packaged Git data into fewer package files without interrupting parallel commands.

### Some Related

#### [VFS for Git](https://github.com/microsoft/VFSForGit) from Microsoft
VFS For Git is a preliminary attempt by Microsoft on the Monorepo client, which implemented the FUSE system based on Sqlite and Mutli pack index, achieving on-demand partial pull functionality. The client will perceive the user's "open directory" operation before pulling the code content under the corresponding directory.

#### [Sapling](https://sapling-scm.com/) from Meta 
The structure of Sapling is achieved through a multi-layered architecture, with each checkout corresponding to a mount point, followed by an Overlay layer. At the same time, it provides third-party interfaces for other programs to use, so that some heavy IO and computational parts do not need to be consumed by the performance of the virtual layer.

### Rust Crate

Scorpio is a Rust project, and the crate is named `scorpiofs`.

https://crates.io/crates/scorpiofs

### How to Use?

**Prerequisites:** Linux with FUSE enabled, `libfuse-dev`, and a running Mega/monorepo server. See [docs/develop.md](docs/develop.md) for system setup (may require `sudo` for FUSE).

1. Start the mono server (e.g. `http://localhost:8000`).
2. Edit **`scorpio.toml`** (not `config.toml`): set `base_url`, `workspace`, and `store_path`. The `config.toml` file is a **runtime state file** (tracks mounted workspaces), created automatically on first run.
3. Build and run the main binary:

```bash
cargo build --release
cargo run --release -- --config-path scorpio.toml --http-addr 0.0.0.0:2725
```

```bash
Usage: scorpio [OPTIONS]

Options:
  -c, --config-path <CONFIG_PATH>  Path to the configuration file [default: scorpio.toml]
      --http-addr <HTTP_ADDR>      HTTP bind address [default: 0.0.0.0:2725]
```

The `antares` binary provides overlay mount management (CLI and optional standalone HTTP server on port 2726):

```bash
cargo run --release --bin antares -- --config-path scorpio.toml list
cargo run --release --bin antares -- serve --bind 0.0.0.0:2726
```

### How to Interact?

**Recommended — Antares API** (nested under the main server at `/antares/*`):

```bash
curl http://localhost:2725/antares/health
curl -X POST http://localhost:2725/antares/mounts \
  -H "Content-Type: application/json" \
  -d '{"job_id":"job-1","path":"/third-party/mega"}'
curl http://localhost:2725/antares/mounts
```

See [docs/antares.md](docs/antares.md) for the full Antares API (including `GET /antares/mounts/{id}/ready`).

**Legacy API** (`/api/fs/*`, still supported):

```bash
curl -X POST http://localhost:2725/api/fs/mount \
  -H "Content-Type: application/json" \
  -d '{"path": "third-party/mega/scorpio"}'
curl http://localhost:2725/api/fs/mpoint
curl http://localhost:2725/api/fs/select/<request_id>
curl -X POST http://localhost:2725/api/fs/unmount \
  -H "Content-Type: application/json" \
  -d '{"path": "third-party/mega/scorpio"}'
```

See [docs/api.md](docs/api.md) for request/response details.

### How to Configure?

An example `scorpio.toml` is included in the repository root:

```toml
base_url = "http://localhost:8000"
lfs_url = "http://localhost:8000"
store_path = "/tmp/scorpio-megadir/store"
workspace = "/tmp/scorpio-megadir/mount"
config_file = "config.toml"
git_author = "MEGA"
git_email = "admin@mega.org"
dicfuse_readable = "true"
load_dir_depth = "3"
fetch_file_thread = "10"
```

### `scorpio.toml` Configuration Guide

- **`base_url`** — Mega/monorepo service base URL (e.g. `http://localhost:8000`).
- **`lfs_url`** — LFS endpoint URL (typically same host as `base_url`).
- **`workspace`** — FUSE mount point visible to users (not `mount_path`).
- **`store_path`** — Local directory for cached/stored files (must be writable).
- **`config_file`** — Runtime state file path (default `config.toml`; records `works=[]` mounted paths). This is **not** the main config file.
- **`git_author`** / **`git_email`** — Default Git author metadata.
- **`dicfuse_readable`** — Allow reading from read-only directories (`"true"` / `"false"`).
- **`load_dir_depth`** — Directory preload depth during initialization.
- **`fetch_file_thread`** — Concurrent download thread count.

Antares-specific keys use flat names in `scorpio.toml` (e.g. `antares_mount_root`, `antares_upper_root`). See [docs/antares.md](docs/antares.md#配置) for the full list.

### How to Contribute?

Contributions are welcome! Please follow these steps:
1. Fork the repository.
2. Create a new branch for your feature or bug fix.
3. Submit a pull request with a clear description of your changes.

### Reference
[1] Rachel Potvin and Josh Levenberg. 2016. Why Google stores billions of lines of code in a single repository. Commun. ACM 59, 7 (July 2016), 78–87. https://doi.org/10.1145/2854146
[2] Nicolas Brousse. 2019. The issue of monorepo and polyrepo in large enterprises. In Companion Proceedings of the 3rd International Conference on the Art, Science, and Engineering of Programming (Programming '19). Association for Computing Machinery, New York, NY, USA, Article 2, 1–4. https://doi.org/10.1145/3328433.3328435
[3] [libfuse](https://github.com/libfuse/libfuse.git) is the reference implementation of the Linux FUSE (Filesystem in Userspace) interface.
[4] [CS135 FUSE Documentation (hmc.edu)](https://www.cs.hmc.edu/~geoff/classes/hmc.cs135.201001/homework/fuse/fuse_doc.html#function-purposes)
[5] [sapling](https://github.com/facebook/sapling.git) : A cross-platform, highly scalable, Git-compatible source control system.
[6] [fuser](https://github.com/cberner/fuser.git) : A Rust library crate for easy implementation of FUSE filesystems in userspace.
[7] [Scalar](https://github.com/microsoft/git/blob/HEAD/contrib/scalar/docs/index.md) : Scalar is a tool that helps Git scale to some of the largest Git repositories. Initially, it was a single standalone git plugin based on Vfs for git, inheriting GVFS. No longer using FUSE. It implements aware partial directory management. Users need to manage and register the required workspace directory on their own. Ease of use can be improved through the fuse mechanism.

# Scorpio RoadMap

## **1. [libufse-fs] overlayFS + passthroughFS**
1. **Performance Optimization**  
   - Enhance performance by leveraging `mmap` and `eBPF`.

2. **Encryption Experimentation**  
   - Explore `rencfs` for file encryption capabilities.

3. **File Layer Management**  
   - Support file layer management for `Docker Build`.


## **2. Git Operation Functionality**
- Support More basic Git operations:  
  - `git log`  
  - `git status`  
  - `git add`  
  - Support `.gitignore` functionality.

## **3. Git LFS Support**

Integrate Git Large File Storage (LFS) for managing large files.

after mount: 
1. read the .libra_attribute in monorepo , store the patterns in the store path ..
2. get all maybe lfs point(blob);
3. if the file is lfs point, then download it.

before git push:

1. read the `.libra_attribute` in the store path ..
2. get all change lfs point(blob);
3. push changed blob to the lfs server;
4. get the lfs point(blob) from the lfs server;
5. build the commit with the lfs point(blob);


## **4. Directory Management**

1. **Local Directory Storage Recovery**  
   - Implement recovery functionality for local directory storage.

2. **Directory Change Monitoring**  
   - Monitor and address inconsistencies between local and remote storage directories.


