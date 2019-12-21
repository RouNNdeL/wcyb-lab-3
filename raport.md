# Raport
## Krzysztof Zdulski
### 303731

# 1. Skan sieci
Wykonuję ogólny skan sieci w poszukiwaniu hostów. 

![](images/nmap_all.png)

Hosty `192.168.119.129` oraz `192.168.119.130` to maszyny _metasploitable_ i _vulnix_.
Natomiast `192.168.119.5` to nasza maszyna Kali Linux, a `192.168.119.254` to host VMWare z serwerem DHCP.

Dalej wykonujemy skan `SYN` oraz `OS` dla interesujących nas hostów.

![](images/nmap_host1.png)

![](images/nmap_host2.png)

# 2. Skan podatności 

- [Openvas](scans/openvas.html)
- [Nessus](scans/nessus.html)

Wybrałem podatność `CVE-2012-1823`

# 3. Ekploitacja podatności

Zaczynamy od wyszukania eksploitów powiązanych z wybraną podatnością. 
Po wyszukaniu wybieramu odpowiedni exploit oraz ustawiamy opcje `RHOSTS` na nasz cel.

![](images/msf_php1.png)

Następnie wybieramy payload `mterpreter_reverse_tcp`.

![](images/msf_php2.png)

Wyświetlamy opcje naszego payloadu i ustawiamy `LHOST` na IP Kali Linuxa.

![](images/msf_php3.png)

Gdy ustawiliśmy już wszytskie opcje wpisujemy `exploit` lub `run`. 
Widzimy, że udało nam się otworzyć sesję _meterpreter_.

![](images/msf_php4.png)

Możemy teraz wykonywać polecenia na maszynie, którą udało się skompromitować. 
Możemy np. utworzyć nowy process `bash` i sprawdzić w imieniu jakigo użytkowanika wykonujemy polcenia.
W tym przypadku jest to użytkownik `www-data`, domyślny użytkowanik serwerów HTTP.

![](images/msf_php5.png)

# 4. Listowanie użytkowników SMTP 

Podczas skanowania sieci zauważyliśmy, że host `192.168.119.130` (`vulnix`) ma otwarty port 25 - `SMTP`.
Korzystając z modułu _auxilary_ w metasploit możemy spróbować wylistować użytkowników `SMTP`.

Zaczynamy od znalezienia modułu korzystając z funckji `search` w metasploitable. 

![](images/msf_search_smtp.png)

Widzimy, że modułem, który chcemy wykorzytsać jest `smpt_enum`. 
Wybieramy ten moduł oraz ustawiamy jego parametr `RHOSTS.`
Jeśli chcemy użyć innej listy użytkowników możemy to zrobić zmieniając paramter `USER_FILE`.

![](images/msf_smtp_enum.png)

Po wpisaniu `run` czekamy chwilę, aż moduł znajdzie użytkowników.

![](images/msf_smtp_result.png)

# 5. Złamanie hasła SSH

Podczas skanowania sieci zauważyliśmy, że host `192.168.119.130` (`vulnix`) ma otwarty port 22 - `SSH`.
Korzystając z modułu _auxilary_ w metasploit możemy spróbować złamać hasło do usługi `SSH` dla użytkowanika user.

Korzystamy z modułu `ssh_login`.

![](images/msf_ssh_options.png)

Po ustawieniu paramterów uruchamiamy moduł i czekamy na rezultaty.

![](images/msf_ssh_result.png)

Dodatkowo _metasploit_ utworzył za nas sesje ssh do której jeśli chcemy możemy się podłączyć.
