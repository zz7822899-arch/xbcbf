name: RDP
on:
  workflow_dispatch:
jobs:
  secure-rdp:
    runs-on: windows-latest
    timeout-minutes: 3600
    steps:
      - name: Configure Core RDP Settings
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "SecurityLayer" -Value 0 -Force
          netsh advfirewall firewall delete rule name="RDP-Tailscale"
          netsh advfirewall firewall add rule name="RDP-Tailscale" dir=in action=allow protocol=TCP localport=3389
          Restart-Service -Name TermService -Force

      - name: Create RDP User with Static Password
        run: |
          $password = "admin@123"
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force

          if (-not (Get-LocalUser -Name "TOOLBOXLAP" -ErrorAction SilentlyContinue)) {
              New-LocalUser -Name "TOOLBOXLAP" -Password $securePass -AccountNeverExpires
          }

          Add-LocalGroupMember -Group "Administrators" -Member "TOOLBOXLAP"
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "TOOLBOXLAP"

          echo "RDP_CREDS=User: TOOLBOXLAP | Password: $password" >> $env:GITHUB_ENV

          if (-not (Get-LocalUser -Name "TOOLBOXLAP")) {
              Write-Error "User creation failed"
              exit 1
          }

      - name: Install Tailscale
        run: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $installerPath = "$env:TEMP\tailscale.msi"

          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
          Start-Process msiexec.exe -ArgumentList "/i", "`"$installerPath`"", "/quiet", "/norestart" -Wait
          Remove-Item $installerPath -Force

      - name: Establish Tailscale Connection
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-runner-$env:GITHUB_RUN_ID
          $tsIP = $null
          $retries = 0
          while (-not $tsIP -and $retries -lt 10) {
              $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
              Start-Sleep -Seconds 5
              $retries++
          }
          if (-not $tsIP) {
              Write-Error "Tailscale IP not assigned. Exiting."
              exit 1
          }
          echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV

      - name: Verify RDP Accessibility
        run: |
          Write-Host "Tailscale IP: $env:TAILSCALE_IP"
          $testResult = Test-NetConnection -ComputerName $env:TAILSCALE_IP -Port 3389
          if (-not $testResult.TcpTestSucceeded) {
              Write-Error "TCP connection to RDP port 3389 failed"
              exit 1
          }
          Write-Host "TCP connectivity successful!"

      - name: Maintain Connection
        run: |
          Write-Host "`n=== RDP ACCESS ==="
          Write-Host "Address: $env:TAILSCALE_IP"
          Write-Host "Username: TOOLBOXLAP"
          Write-Host "Password: admin@123"
          Write-Host "==================`n"
          while ($true) {
              Write-Host "[$(Get-Date)] RDP Active - Use Ctrl+C in workflow to terminate"
              Start-Sleep -Seconds 300
          }
