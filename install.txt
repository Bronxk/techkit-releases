# TechKit bootstrap for Windows PowerShell 5.1 and later:
#   irm <BaseUrl>/install.ps1 | iex
# Loads the newest techkit.exe for this machine into %LOCALAPPDATA%\TechKit\bin,
# checks size and SHA-256 against the release manifest and starts it.
# Keep this file ASCII only: PowerShell 5.1 reads scripts without BOM as ANSI.

& {
    # Public release location (GitHub Pages of Bronxk/techkit-releases).
    # TECHKIT_UPDATE_SOURCE overrides it.
    $BaseUrl = 'https://bronxk.github.io/techkit-releases'
    $Channel = 'stable'

    if ($env:TECHKIT_UPDATE_SOURCE) { $BaseUrl = $env:TECHKIT_UPDATE_SOURCE }
    if ($env:TECHKIT_UPDATE_CHANNEL) { $Channel = $env:TECHKIT_UPDATE_CHANNEL }

    $ErrorActionPreference = 'Stop'
    $ProgressPreference = 'SilentlyContinue'
    [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12

    function Fail([string]$msg) { throw "TechKit: $msg" }

    $cpu = $env:PROCESSOR_ARCHITEW6432
    if (-not $cpu) { $cpu = $env:PROCESSOR_ARCHITECTURE }
    switch ($cpu) {
        'AMD64' { $arch = 'amd64' }
        'ARM64' { $arch = 'arm64' }
        default { Fail "Prozessor $cpu wird nicht unterstuetzt (nur x64 und ARM64)." }
    }

    $manifestUrl = $BaseUrl.TrimEnd('/') + "/$Channel/latest.json"
    Write-Host "TechKit: lese $manifestUrl"
    try {
        $resp = Invoke-WebRequest -UseBasicParsing -Uri $manifestUrl -Headers @{ 'Cache-Control' = 'no-cache' }
    } catch {
        Fail "Update-Server nicht erreichbar: $($_.Exception.Message)"
    }
    $raw = $resp.Content
    if ($raw -is [byte[]]) { $raw = [Text.Encoding]::UTF8.GetString($raw) }
    $m = $raw | ConvertFrom-Json
    if ($m.schema -ne 1) { Fail "Unbekanntes Format der Update-Liste ($($m.schema))." }
    if ($m.channel -ne $Channel) { Fail "Update-Liste gehoert zum Kanal $($m.channel), erwartet $Channel." }
    $file = $m.files."windows/$arch"
    if (-not $file) { Fail "Version $($m.version) gibt es nicht fuer windows/$arch." }
    $want = ([string]$file.sha256).ToUpperInvariant()
    if ($want -notmatch '^[0-9A-F]{64}$') { Fail 'Update-Liste enthaelt keine gueltige SHA-256.' }
    $fileUrl = (New-Object System.Uri((New-Object System.Uri($manifestUrl)), [string]$file.url)).AbsoluteUri

    $bin = Join-Path $env:LOCALAPPDATA 'TechKit\bin'
    New-Item -ItemType Directory -Force -Path $bin | Out-Null
    $exe = Join-Path $bin 'techkit.exe'

    $current = $null
    if (Test-Path -LiteralPath $exe) { $current = (Get-FileHash -LiteralPath $exe -Algorithm SHA256).Hash }
    if ($current -eq $want) {
        Write-Host "TechKit $($m.version) ist schon da."
    } else {
        Write-Host "TechKit: lade $($m.version) fuer windows/$arch ($([math]::Round($file.size / 1MB, 1)) MB)"
        $tmp = Join-Path $bin ('techkit-' + [guid]::NewGuid().ToString('N') + '.tmp')
        try {
            Invoke-WebRequest -UseBasicParsing -Uri $fileUrl -OutFile $tmp
        } catch {
            Remove-Item -LiteralPath $tmp -Force -ErrorAction SilentlyContinue
            Fail "Download fehlgeschlagen: $($_.Exception.Message)"
        }
        $size = (Get-Item -LiteralPath $tmp).Length
        $got = (Get-FileHash -LiteralPath $tmp -Algorithm SHA256).Hash
        if ($size -ne [int64]$file.size -or $got -ne $want) {
            Remove-Item -LiteralPath $tmp -Force -ErrorAction SilentlyContinue
            Fail "Pruefsumme passt nicht (erwartet $want, erhalten $got). Datei verworfen."
        }
        if (Test-Path -LiteralPath $exe) {
            # a running techkit.exe can be renamed but not overwritten; TechKit removes .old on start
            Remove-Item -LiteralPath "$exe.old" -Force -ErrorAction SilentlyContinue
            if (Test-Path -LiteralPath "$exe.old") {
                Move-Item -LiteralPath $exe -Destination ("$exe." + [DateTime]::UtcNow.Ticks + '.old') -Force
            } else {
                Move-Item -LiteralPath $exe -Destination "$exe.old" -Force
            }
        }
        Move-Item -LiteralPath $tmp -Destination $exe -Force
        Write-Host "TechKit: Pruefsumme ok, gespeichert unter $exe"
    }

    Write-Host 'TechKit startet ...'
    Start-Process -FilePath $exe -WorkingDirectory $bin
}
