# Add & Push Results — PowerShell Helper

Skrip PowerShell ini membantu menyalin sebuah folder lokal (mis. hasil export/summary) ke dalam sebuah repository Git lokal/remote, membuat commit yang diperlukan, lalu melakukan push ke remote (dengan penanganan aman untuk repositori kosong / branch remote tidak ada).

Gunakan skrip ini bila Anda ingin mengunggah snapshot hasil (mis. folder hasil eksperimen, ringkasan, laporan) ke repository GitHub/Git remote secara otomatis.

Catatan penting: baca bagian Prasyarat dan Konfigurasi sebelum menjalankan.

---

## Fitur utama
- Clone remote repository jika `LocalClone` belum ada.
- Jika folder lokal sudah ada tetapi bukan repo git, inisialisasi repo dan tambahkan `origin`.
- Menyalin seluruh isi `SourceFolder` ke subfolder di repo lokal (nama subfolder bisa diatur).
- Jika repo kosong (tidak ada commit), buat initial commit otomatis sehingga checkout branch aman.
- Membuat atau checkout branch target dengan aman.
- Cek keberadaan branch di remote (`git ls-remote`) sebelum melakukan `git pull --rebase` untuk menghindari error `couldn't find remote ref`.
- Stage, commit (hanya bila ada perubahan) dan push ke remote (dengan `-u` untuk membuat upstream bila belum ada).
- Opsi untuk menjalankan `git lfs install` bila Anda menggunakan Git LFS (flag `UseGitLfs`).

---

## Prasyarat
- Git terpasang dan dapat dijalankan dari PATH (`git` tersedia di terminal PowerShell).
- (Opsional) Jika menggunakan SSH remote, konfigurasi SSH keys dan gunakan URL remote `git@github.com:owner/repo.git`.
- Jika menggunakan HTTPS untuk push ke GitHub, siapkan credential (Git akan meminta username/password; gunakan Personal Access Token (PAT) sebagai password bila diperlukan).

---

## Konfigurasi
Edit bagian CONFIG di bagian atas skrip sebelum dijalankan:

- `$SourceFolder`  
  Path folder sumber yang ingin Anda salin ke repo (harus ada).

- `$RepoUrl`  
  URL remote Git (HTTPS atau SSH) — contoh `https://github.com/owner/repo.git` atau `git@github.com:owner/repo.git`.

- `$LocalClone`  
  Path folder lokal tempat repo akan di-clone atau digunakan.

- `$Branch`  
  Nama branch target (mis. `main` atau `master` atau branch lain).

- `$Subfolder`  
  Nama subfolder di dalam repo tempat isi `$SourceFolder` akan dicopy. Jika kosong, otomatis digunakan `basename` dari `$SourceFolder`.

- `$UseGitLfs`  
  `$true` jika Anda ingin menjalankan `git lfs install` (pastikan git-lfs sudah terinstal), `$false` jika tidak.

Contoh pengaturan awal (ubah sesuai lingkungan Anda):
```powershell
# ===== CONFIG =====
$SourceFolder = "C:\path\to\results"
$RepoUrl     = "https://github.com/yourusername/yourrepo.git"
$LocalClone  = "C:\path\to\local-repo"
$Branch      = "main"
$Subfolder   = ""
$UseGitLfs   = $false
# ====================
```

---

## Cara menjalankan
Ada dua cara:

1. Menjalankan interaktif (paste):
   - Buka PowerShell (bukan ISE; gunakan PowerShell 5+ atau PowerShell Core).
   - Edit variabel di bagian CONFIG.
   - Salin seluruh isi skrip (seluruh file) lalu paste ke jendela PowerShell, tekan Enter.

2. Menyimpan sebagai file `.ps1` lalu menjalankan:
   - Simpan skrip sebagai `add-and-push-results.ps1`.
   - Buka PowerShell, pindah ke folder skrip.
   - Jalankan:
     ```powershell
     Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
     .\add-and-push-results.ps1
     ```

Skrip akan mencetak log langkah demi langkah dan mengarahkan bila ada kesalahan.

---

## Isi skrip (salin & tempel seluruh blok ini ke PowerShell)
```powershell
# Robust PowerShell script to copy a local folder into a git repo and push,
# with safe remote-branch detection to avoid "couldn't find remote ref" errors.
# Copy & paste whole file into PowerShell (interactive) or save and run.
# Edit CONFIG variables below before running.

# ===== CONFIG =====
$SourceFolder = "C:\Users\ASUS\Desktop\python-project-remote\results_summary-20251029-202723"
$RepoUrl     = "https://github.com/yirassssindaba-coder/alpha-project.git"   # or SSH git@github.com:owner/repo.git
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
