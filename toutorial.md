# Eksploitacja binarna - 'buffer overflow'

## Potrzebne narzędzia
 - `gcc` (compiler)
 - `gdb` (debugger)
 - Python 2.7
   - `pip2` (do zaintslowania biblioteki)
   - biblioteka `pwntools`
 - `radare2` (nie jest wymagane)


## Plan tutoriala

1. Konfiguracja środowiska 
    1. Instalacja narzędzi
    2. Stowrzenie podatnej aplikacji
2. Analiza aplikacji za pomocą narzędzia `radare2`
    1. Stworzenie pliku z danymi wejściowymi
    2. Analiza podatnej funkcji
    3. Debuggowanie aplikacji
3. Wykorzystanie biblioteki `pwntools` i eksploitacja aplikacji za pomocą Pythona
    1. Uszkodznie (crash) aplikacji
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
jednak jeśli nie jest instalujemy pakiet _build-essentials_: `sudo apt install build-essential`.
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
    int v;
    printf("STACK IS HERE => %p\n", &v);
    ask_for_name();
    return 0;
}
```
Podatność spowodowana jest użyciem funckji `gets(char* str)`, która wczyta do bufora `char* name` dowolną ilość danych.
Następnie do sprawdzenia długości wprowadzonych danych wykorzystana jest funkcja `strlen (const char* str)`, 
którą bardzo łatwo oszukać, gdyż używa ona znaku _null_ (0x00 w ascii) do sprawdzenia długości stringa.
