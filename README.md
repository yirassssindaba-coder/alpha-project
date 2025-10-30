# Add & Push Results — PowerShell Helper (UPDATED repo locations)

Skrip PowerShell ini menyalin folder hasil lokal ke repository Git, membuat commit bila perlu, lalu melakukan push ke remote. README ini sudah diperbarui karena lokasi repository remote telah berubah.

Baru: gunakan salah satu dari URL/command berikut untuk remote / cloning:
- HTTPS remote:
  ```
  https://github.com/yirassssindaba-coder/social-media-sentiment-analysis-results.git
  ```
- SSH remote:
  ```
  git@github.com:yirassssindaba-coder/social-media-sentiment-analysis-results.git
  ```
- `gh` (GitHub CLI) clone:
  ```
  gh repo clone yirassssindaba-coder/social-media-sentiment-analysis-results
  ```

Gunakan bentuk remote yang cocok untuk lingkungan Anda (HTTPS butuh PAT untuk push, SSH butuh SSH key terdaftar di GitHub).

---

## Ringkasan
- Tujuan: Salin folder hasil (mis. ringkasan, laporan) ke dalam repo git, commit hanya jika ada perubahan, dan push ke remote.
- Behavior defensif: clone jika perlu, buat initial commit bila repo kosong, cek remote branch existence sebelum pull, set upstream saat push.
- Tidak memodifikasi file remote selain yang Anda copy/commit.

---

## Persyaratan (Prerequisites)
- Git terinstal dan berada di PATH.
- (Opsional) Git LFS jika Anda ingin menyertakan file besar.
- Jika push via HTTPS: gunakan Personal Access Token (PAT) sebagai password.
- Jika push via SSH: pastikan SSH key Anda sudah ditambahkan ke GitHub dan remote menggunakan format `git@github.com:...`.

---

## Konfigurasi (contoh)
Edit variabel di bagian CONFIG sebelum menjalankan skrip.

Contoh default (sudah disesuaikan ke lokasi remote baru):
```powershell
# ===== CONFIG =====
$SourceFolder = "C:\Users\ASUS\Desktop\python-project-remote\results_summary-20251029-202723"
$RepoUrl     = "https://github.com/yirassssindaba-coder/social-media-sentiment-analysis-results.git"
# Alternatif (SSH)
# $RepoUrl   = "git@github.com:yirassssindaba-coder/social-media-sentiment-analysis-results.git"
$LocalClone  = "C:\Users\ASUS\Desktop\alpha-project"
$Branch      = "main"
$Subfolder   = ""      # empty -> use basename of SourceFolder
$UseGitLfs   = $false
# ==================
```

- `$SourceFolder`: folder yang akan dicopy (harus ada).
- `$RepoUrl`: remote target (ubah sesuai HTTPS atau SSH).
- `$LocalClone`: lokasi clone lokal (jika tidak ada, skrip akan meng-clone).
- `$Branch`: branch tujuan.
- `$Subfolder`: nama subfolder di repo tempat file akan dicopy (kosong => menggunakan nama folder sumber).
- `$UseGitLfs`: set ke `$true` jika ingin menjalankan `git lfs install`.

---

## Cara menjalankan
1. Buka PowerShell.
2. Edit bagian CONFIG di atas sesuai lingkungan Anda.
3. Salin seluruh skrip di bawah (blok PowerShell) lalu paste ke jendela PowerShell dan tekan Enter.
   - Atau simpan sebagai `add-and-push-results.ps1` dan jalankan:
     ```powershell
     Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
     .\add-and-push-results.ps1
     ```

Skrip akan menampilkan log langkah demi langkah dan memberi petunjuk jika terjadi masalah (git tidak ada, folder sumber tidak ditemukan, autentikasi gagal, dll).

---

## Skrip (salin & paste seluruh blok ini ke PowerShell)
```powershell
# Robust PowerShell script to copy a local folder into a git repo and push,
# with safe remote-branch detection to avoid "couldn't find remote ref" errors.
# Copy & paste whole file into PowerShell (interactive) or save and run.
# Edit CONFIG variables below before running.

# ===== CONFIG =====
$SourceFolder = "C:\Users\ASUS\Desktop\python-project-remote\results_summary-20251029-202723"
$RepoUrl     = "https://github.com/yirassssindaba-coder/social-media-sentiment-analysis-results.git"   # or SSH: git@github.com:yirassssindaba-coder/social-media-sentiment-analysis-results.git
$LocalClone  = "C:\Users\ASUS\Desktop\alpha-project"
$Branch      = "main"
$Subfolder   = ""      # empty -> use basename of SourceFolder
$UseGitLfs   = $false
# ==================

function Abort($msg) { Write-Host "ERROR: $msg" -ForegroundColor Red; throw $msg }

# Ensure git exists
try { Get-Command git -ErrorAction Stop | Out-Null } catch { Abort "git not found in PATH." }

# Resolve source
try { $SourceFolder = (Resolve-Path -LiteralPath $SourceFolder -ErrorAction Stop).ProviderPath } catch { Abort "Source folder not found: $SourceFolder" }
if (-not $Subfolder) { $Subfolder = Split-Path -Leaf $SourceFolder }

Write-Host "Source: $SourceFolder"
Write-Host "Remote: $RepoUrl"
Write-Host "Local clone: $LocalClone"
Write-Host "Branch: $Branch"
Write-Host "Subfolder: $Subfolder"
Write-Host ""

# Prepare local clone
if (-not (Test-Path $LocalClone)) {
    New-Item -ItemType Directory -Path $LocalClone -Force | Out-Null
    Write-Host "Cloning $RepoUrl -> $LocalClone ..."
    $cloneExit = & git clone $RepoUrl $LocalClone
    if ($LASTEXITCODE -ne 0) { Abort "git clone failed. Check remote URL and credentials." }
} else {
    Push-Location $LocalClone
    try {
        if (-not (Test-Path ".git")) {
            Write-Host "Initializing repo at $LocalClone and adding remote..."
            git init
            git remote remove origin 2>$null
            git remote add origin $RepoUrl
        } else {
            Write-Host "Using existing repo at $LocalClone"
            # fetch remote refs but don't error if origin missing
            try { git remote show origin >/dev/null 2>&1 } catch {}
        }
    } finally { Pop-Location }
}

# Optional git lfs
if ($UseGitLfs) { Write-Host "Running git lfs install..."; git lfs install 2>$null }

# Copy files
$ResolvedLocalClone = (Resolve-Path -LiteralPath $LocalClone).ProviderPath
$DestFolder = Join-Path -Path $ResolvedLocalClone -ChildPath $Subfolder
Write-Host "Copying $SourceFolder -> $DestFolder"
try {
    New-Item -ItemType Directory -Path $DestFolder -Force | Out-Null
    Copy-Item -Path (Join-Path $SourceFolder '*') -Destination $DestFolder -Recurse -Force
} catch { Abort "Copy failed: $($_.Exception.Message)" }

# Work in repo
Push-Location $ResolvedLocalClone
try {
    # Ensure at least one commit exists
    $hasCommits = $true
    try { git rev-parse --verify HEAD >/dev/null 2>&1 } catch { $hasCommits = $false }
    if (-not $hasCommits) {
        Write-Host "Repository has no commits. Creating initial commit..."
        if (-not (Test-Path "README.md")) { "## repo" | Out-File -FilePath README.md -Encoding UTF8 }
        git add --all
        git commit -m "Initial commit (created by script)" 2>$null
        if ($LASTEXITCODE -ne 0) {
            git commit --allow-empty -m "Initial empty commit (fallback)" 2>$null
            if ($LASTEXITCODE -ne 0) { Abort "Failed to create initial commit." }
        }
        Write-Host "Initial commit done."
    }

    # Ensure branch exists locally or create it
    $branchExistsLocal = $false
    try {
        $localBranches = (& git for-each-ref --format='%(refname:short)' refs/heads/) -split "`n" | ForEach-Object { $_.Trim() } | Where-Object { $_ -ne "" }
        if ($localBranches -contains $Branch) { $branchExistsLocal = $true }
    } catch {}

    if ($branchExistsLocal) {
        Write-Host "Checking out existing branch $Branch..."
        & git checkout $Branch
        if ($LASTEXITCODE -ne 0) { Abort "git checkout $Branch failed." }
    } else {
        Write-Host "Creating and checking out branch $Branch..."
        & git checkout -b $Branch
        if ($LASTEXITCODE -ne 0) { Abort "git checkout -b $Branch failed." }
    }

    # Stage changes
    Write-Host "Staging files..."
    & git add --all

    # Commit if changes exist
    $status = (& git status --porcelain) -join "`n"
    if (-not [string]::IsNullOrWhiteSpace($status)) {
        $msg = "Add results snapshot: $Subfolder - $(Get-Date -Format yyyyMMdd-HHmmss)"
        & git commit -m $msg
        if ($LASTEXITCODE -ne 0) { Abort "git commit failed." }
        Write-Host "Committed changes."
    } else {
        Write-Host "No changes to commit."
    }

    # Ensure remote origin exists, add if missing
    $hasOrigin = $false
    try { $remotes = (& git remote) -split "`n"; if ($remotes -contains "origin") { $hasOrigin = $true } } catch {}
    if (-not $hasOrigin) {
        Write-Host "Adding remote origin -> $RepoUrl"
        & git remote add origin $RepoUrl
    }

    # Check if remote branch exists before attempting pull
    $remoteHasBranch = $false
    try {
        $ls = (& git ls-remote --heads origin $Branch) -join "`n"
        if (-not [string]::IsNullOrWhiteSpace($ls)) { $remoteHasBranch = $true }
    } catch {
        # ls-remote may fail if origin not accessible; we treat as branch absent
        $remoteHasBranch = $false
    }

    if ($remoteHasBranch) {
        Write-Host "Remote branch origin/$Branch exists. Pulling --rebase..."
        & git pull --rebase origin $Branch
        if ($LASTEXITCODE -ne 0) { Write-Host "git pull returned non-zero exit code (proceeding to push)." -ForegroundColor Yellow }
    } else {
        Write-Host "Remote branch origin/$Branch does not exist. Skipping pull; pushing will create upstream branch."
    }

    # Push and set upstream
    Write-Host "Pushing to origin/$Branch..."
    & git push -u origin $Branch
    if ($LASTEXITCODE -ne 0) {
        Write-Host "git push finished with non-zero exit code. Check credentials or remote permissions." -ForegroundColor Yellow
    } else {
        Write-Host "Push succeeded." -ForegroundColor Green
    }

} finally {
    Pop-Location
}

Write-Host "`nFinished. Local repo at: $ResolvedLocalClone"
Write-Host "If authentication fails, use a Personal Access Token for HTTPS or configure SSH keys for SSH remotes."
```

---

## Troubleshooting & tips
- Jika Anda melihat `fatal: couldn't find remote ref main` — ini berarti branch `main` belum ada di remote. Skrip sudah memeriksa keberadaan branch di remote; jika tidak ada, skrip melewati `git pull` dan akan membuat upstream saat `git push -u origin <branch>`.
- Autentikasi:
  - HTTPS: saat diminta username gunakan username GitHub Anda; untuk password gunakan PAT (Personal Access Token) dengan scope `repo`.
  - SSH: pastikan private key tersedia di `~/.ssh` dan public key telah ditambahkan ke GitHub.
- Jika push gagal karena credential manager: pasang Git Credential Manager (GCM) atau gunakan SSH.

---

Jika Anda mau, saya dapat:
- Memperbarui skrip agar default `$RepoUrl` langsung mengarah ke `social-media-sentiment-analysis-results` (sudah saya lakukan di contoh di atas).
- Membuat contoh one-liner non-interactive (SSH) yang otomatis commit tanpa prompt (butuh SSH key).
- Membuat versi yang hanya menyalin hasil ke folder lokal (tanpa git) — beri tahu jika Anda ingin itu.
