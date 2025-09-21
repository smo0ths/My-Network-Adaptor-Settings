# My-Network-Adaptor-Settings
Enforces IPv4 only, Cloudflare DoH, disables NetBIOS

##### This is my network adaptor settings, the script will find and set them for you. (run in PowerShell)
##### Network & internet > Ethernet/Wi-Fi will say (Unencrypted) its not, thats just UI. (change it if you want does not do anything but change the UI text)

```python
# Enforces IPv4 only, Cloudflare DoH, disables NetBIOS
$adapter = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Name -match "Ethernet" }
Get-NetAdapterBinding -Name $adapter.Name | Where-Object { $_.ComponentID -ne "ms_tcpip" } | ForEach-Object {
    Disable-NetAdapterBinding -Name $adapter.Name -ComponentID $_.ComponentID -Confirm:$false
}
Enable-NetAdapterBinding -Name $adapter.Name -ComponentID "ms_tcpip"
$adapters = Get-NetAdapter |
    Where-Object { $_.Status -eq "Up" -and $_.HardwareInterface -eq $true } |
    Where-Object { $_.InterfaceDescription -match "Wi-Fi|Ethernet" }
foreach ($adapter in $adapters) {
    Write-Host "Configuring adapter:" $adapter.Name -ForegroundColor Cyan
    Set-DnsClientServerAddress -InterfaceAlias $adapter.Name -ServerAddresses 1.1.1.1,1.0.0.1
    Add-DnsClientDohServerAddress -ServerAddress 1.1.1.1 `
        -DohTemplate "https://cloudflare-dns.com/dns-query" `
        -AllowFallbackToUdp:$false -ErrorAction SilentlyContinue
    Add-DnsClientDohServerAddress -ServerAddress 1.0.0.1 `
        -DohTemplate "https://cloudflare-dns.com/dns-query" `
        -AllowFallbackToUdp:$false -ErrorAction SilentlyContinue
    Set-DnsClientDohServerAddress -ServerAddress 1.1.1.1 -AutoUpgrade $true
    Set-DnsClientDohServerAddress -ServerAddress 1.0.0.1 -AutoUpgrade $true
    $nicConfig = Get-WmiObject -Class Win32_NetworkAdapterConfiguration -Filter "InterfaceIndex=$($adapter.ifIndex)"
    if ($nicConfig) {
        $nicConfig.SetTcpipNetbios(2) | Out-Null
    }
}
Write-Host "`nCurrent DoH configuration:" -ForegroundColor Green
Get-DnsClientDohServerAddress
Get-WmiObject -Class Win32_NetworkAdapterConfiguration -Filter "IPEnabled=TRUE" |
    Select-Object Description, TcpipNetbiosOptions
```
