# ip-check-powershell



$report = Join-Path ([Environment]::GetFolderPath("Desktop")) "Connection-Emergency.txt"

"Connection report — $(Get-Date -Format o)" |
    Set-Content $report

"`n=== NETWORK SETTINGS ===" | Add-Content $report
ipconfig /all | Add-Content $report
route print -4 | Add-Content $report

"`n=== PUBLIC IPv4 ===" | Add-Content $report
curl.exe -4 --connect-timeout 5 --max-time 10 https://api.ipify.org 2>&1 |
    Add-Content $report

"`n=== SSH SERVICE ===" | Add-Content $report
Get-Service sshd -ErrorAction SilentlyContinue |
    Format-List Name,Status |
    Out-String | Add-Content $report

"`n=== CONNECTION TESTS ===" | Add-Content $report

$targets = @(
    "fsn1-speed.hetzner.com"
    "hel1-speed.hetzner.com"
    "ash-speed.hetzner.com"
    "speedtest.ams1.nl.leaseweb.net"
    "speedtest.fra1.de.leaseweb.net"
    "speedtest.lon1.uk.leaseweb.net"
    "speedtest.mtl2.ca.leaseweb.net"
    "speedtest.sin1.sg.leaseweb.net"
)

foreach ($target in $targets) {
    Write-Host "Checking $target"
    try {
        $ips = @(
            [System.Net.Dns]::GetHostAddresses($target) |
            Where-Object {
                $_.AddressFamily -eq [System.Net.Sockets.AddressFamily]::InterNetwork
            } |
            ForEach-Object { $_.IPAddressToString } |
            Sort-Object -Unique
        )
    } catch {
        "$target : DNS lookup failed" | Add-Content $report
        continue
    }

    $ports = if ($target -eq "46.28.69.207") { @(22,443) } else { @(80,443) }

    foreach ($ip in $ips) {
        foreach ($port in $ports) {
            $client = [System.Net.Sockets.TcpClient]::new()
            $status = "No TCP connection"
            try {
                $task = $client.ConnectAsync($ip,$port)
                if ($task.Wait(3000) -and $client.Connected) {
                    $status = "TCP reachable"
                }
            } catch {
            } finally {
                $client.Dispose()
            }

            $line = "$target | ${ip}:$port | $status"
            Write-Host $line
            $line | Add-Content $report
        }
    }
}

Write-Host "`nSaved: $report"
