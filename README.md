# My-Network-Adaptor-Settings
Enforces IPv4 + IPv6 only, Cloudflare DoH only, disables NetBIOS

##### This is my network adaptor settings, the script will find and set them for you. (run in PowerShell)
##### Network & internet > Ethernet/Wi-Fi will say (Unencrypted) its not, thats just UI. (change it if you want does not do anything but change the UI text)
##### powershell needs internet connectivity

```python
# Enforces IPv4 + IPv6, Cloudflare DoH, disables NetBIOS
Remove-DnsClientDohServerAddress -ServerAddress 8.8.8.8 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 8.8.4.4 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 2001:4860:4860::8888 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 2001:4860:4860::8844 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 9.9.9.9 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 149.112.112.112 -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 2620:fe::fe -ErrorAction SilentlyContinue
Remove-DnsClientDohServerAddress -ServerAddress 2620:fe::9 -ErrorAction SilentlyContinue

$adapters = Get-NetAdapter |
    Where-Object { $_.Status -eq "Up" -and $_.HardwareInterface -eq $true } |
    Where-Object { $_.InterfaceDescription -match "Wi-Fi|Ethernet" }
foreach ($adapter in $adapters) {
    Write-Host "Configuring adapter:" $adapter.Name -ForegroundColor Cyan

    # Ensure TCP/IP binding is enabled
    Enable-NetAdapterBinding -Name $adapter.Name -ComponentID "ms_tcpip" -ErrorAction SilentlyContinue
    # Ensure IPv6 binding is enabled
    Enable-NetAdapterBinding -Name $adapter.Name -ComponentID "ms_tcpip6" -ErrorAction SilentlyContinue
    # Set both IPv4 and IPv6 DNS servers
    Set-DnsClientServerAddress -InterfaceAlias $adapter.Name -ServerAddresses `
        1.1.1.1,1.0.0.1,2606:4700:4700::1111,2606:4700:4700::1001

    # Add DoH servers (IPv4 + IPv6)
    foreach ($dns in @("1.1.1.1","1.0.0.1","2606:4700:4700::1111","2606:4700:4700::1001")) {
        Add-DnsClientDohServerAddress -ServerAddress $dns `
            -DohTemplate "https://cloudflare-dns.com/dns-query" `
            -AllowFallbackToUdp:$false -ErrorAction SilentlyContinue
        Set-DnsClientDohServerAddress -ServerAddress $dns -AutoUpgrade $true -ErrorAction SilentlyContinue
    }
    # Disable NetBIOS (modern CIM version)
    $nicConfig = Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration -Filter "InterfaceIndex=$($adapter.ifIndex)"
    if ($nicConfig) {
        $null = Invoke-CimMethod -InputObject $nicConfig -MethodName SetTcpipNetbios -Arguments @{TcpipNetbiosOptions=2}
    }
}

Write-Host "`nCurrent DoH configuration:" -ForegroundColor Green
Get-DnsClientDohServerAddress

Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration -Filter "IPEnabled=TRUE" |
    Select-Object Description, TcpipNetbiosOptions

# Restart all active Wi-Fi/Ethernet adapters and show ipconfig /all
$adapters = Get-NetAdapter |
    Where-Object { $_.Status -eq "Up" -and $_.HardwareInterface -eq $true } |
    Where-Object { $_.InterfaceDescription -match "Wi-Fi|Ethernet" }
foreach ($adapter in $adapters) {
    Write-Host "Restarting adapter:" $adapter.Name -ForegroundColor Yellow
    Restart-NetAdapter -Name $adapter.Name -Confirm:$false
}

# Give adapters time to reinitialize
Start-Sleep -Seconds 20

# IPv4 tests
Resolve-DnsName example.com -Server 1.1.1.1 -ErrorAction Continue
Invoke-RestMethod -Uri "https://api64.ipify.org?format=json" -ErrorAction Continue
Test-NetConnection 1.1.1.1 -Port 53 -ErrorAction Continue

# IPv6 tests wrapped
$hasIPv6 = Get-NetIPAddress -AddressFamily IPv6 |
    Where-Object { $_.IPAddress -notlike "fe80::*" -and $_.PrefixOrigin -ne "WellKnown" }
if ($hasIPv6) {
    Write-Host "`nIPv6 connectivity detected — running IPv6 tests..." -ForegroundColor Green
    Resolve-DnsName example.com -Server 2606:4700:4700::1111 -ErrorAction Continue
    Test-NetConnection 2606:4700:4700::1111 -Port 53 -ErrorAction Continue
    ping -6 ipv6.google.com
}
else {
    Write-Host "`nNo global IPv6 address detected — skipping IPv6 tests." -ForegroundColor Yellow
}

# Dump the full adapter configuration
Write-Host "`nCurrent network configuration:" -ForegroundColor Green
ipconfig /all
```
