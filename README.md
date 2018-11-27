# Pobieranie danych z rejestru TERYT

GUS udostępnia usługę sieciową [TERYT ws1](http://eteryt.stat.gov.pl/eTeryt/rejestr_teryt/udostepnianie_danych/baza_teryt/usluga_sieciowa_interfejsy_api/opis_uslugi_sieciowej.aspx?contrast=default).<br>
Dane przesyłane są w formacie XML protokołem [SOAP](https://pl.wikipedia.org/wiki/SOAP).<br>
Potrzebujemy więc klienta, który obsłuży ten protokół, np. [zeep](https://python-zeep.readthedocs.io/en/master/):
```bash
pip install zeep
```

Aby aplikacja mogła pobierać dane z rejestru musimy przejść uwierzytelnienie.<br>
Dane do logowania zapiszemy w słowniku:
```python
CREDENTIALS = {
    'wsdl': 'https://uslugaterytws1test.stat.gov.pl/wsdl/terytws1.wsdl',
    'username': 'TestPubliczny',
    'password': '1234abcd'
}
```

Są to dane do środowiska testowego.<br>
Aby korzystać z usługi na produkcji,<br>
należy wysłać maila do GUSu z prośbą o założenie prywatnego konta.

Tworzymy nową instancję klienta:
```python
from zeep import Client
from zeep.wsse.username import UsernameToken

token = UsernameToken(
    username=CREDENTIALS['username'],
    password=CREDENTIALS['password']
)
client = Client(wsdl=CREDENTIALS['wsdl'], wsse=token)
```

Sprawdzamy czy uda się nawiązać połączenie z usługą:
```python
print(client.service.CzyZalogowany())
```

Jeżeli wszystko jest ok, powinno wypisać ```True```.

Kiedy mamy już do dyspozycji obiekt klienta, <br>
możemy na nim wywoływać metody dostępne w TERYT ws1<br>
(pełna lista metod w [instrukcji](http://eteryt.stat.gov.pl/eteryt/files/instrukcja_techniczna_uslugi_teryt_ws1.zip)).

Wiele z tych metod wymaga podania daty jako argumentu. Zatem:
```python
from datetime import datetime

STATE_DATE = datetime.now()
```

Spróbujmy pobrać listę województw:
```python
client.service.PobierzListeWojewodztw(STATE_DATE)
```

Wywołanie zwróci listę obiektów słownikopodobnego typu <i>JednostkaTerytorialna</i>.<br>
Oto jeden z nich:
```python
{
    'GMI': None,
    'NAZWA': 'DOLNOŚLĄSKIE',
    'NAZWA_DOD': 'województwo',
    'POW': None,
    'RODZ': None,
    'STAN_NA': '1/2/2018 12:00:00 AM',
    'WOJ': '02'
}
```

Możemy więc, powołując się na klucz ```'NAZWA'```, stworzyć listę złożoną z nazw:
```python
[e['NAZWA'] for e in client.service.PobierzListeWojewodztw(STATE_DATE)]
```

I voilà:
```python
['DOLNOŚLĄSKIE', 'KUJAWSKO-POMORSKIE', 'LUBELSKIE', 'LUBUSKIE', 'ŁÓDZKIE', 
'MAŁOPOLSKIE', 'MAZOWIECKIE', 'OPOLSKIE', 'PODKARPACKIE', 'PODLASKIE', 
'POMORSKIE', 'ŚLĄSKIE', 'ŚWIĘTOKRZYSKIE', 'WARMIŃSKO-MAZURSKIE', 
'WIELKOPOLSKIE', 'ZACHODNIOPOMORSKIE']
```

## Wyszukiwanie jednostek

Jednostki terytorialne możemy wyszukiwać m.in. po nazwie:
```python
client.service.WyszukajJPT(nazwa='Warszawa')
```

```python
[{
    'GmiNazwa': None,
    'GmiNazwaDodatkowa': 'miasto stołeczne, na prawach powiatu',
    'GmiRodzaj': None,
    'GmiSymbol': None,
    'PowSymbol': '65',
    'Powiat': 'Warszawa',
    'WojSymbol': '14',
    'Wojewodztwo': 'MAZOWIECKIE'
}, {
    'GmiNazwa': 'Warszawa',
    'GmiNazwaDodatkowa': 'gmina miejska, miasto stołeczne',
    'GmiRodzaj': '1',
    'GmiSymbol': '01',
    'PowSymbol': '65',
    'Powiat': 'Warszawa',
    'WojSymbol': '14',
    'Wojewodztwo': 'MAZOWIECKIE'
}]
```

Wyszukiwanie z użyciem identyfikatora TERC<br>
wymaga stworzenia specjalnego obiektu klasy <i>identyfikatory</i>.<br>
Posłuży nam do tego fabryka, którą dostarcza klient:
```python
factory = client.type_factory('ns2')
```

Prefix ```'ns2'``` wskazuje na określoną przestrzeń nazw,<br>
w której zadeklarowane zostały klasy obiektów.<br>
Ale skąd wiemy jakiego namespace użyć? Oto ściągawka:
```python
print(client.wsdl.dump())
```

Powyższe polecenie wyświetli wszystkie dostępne prefixy, klasy, metody,<br> 
jednym słowem cały schemat usługi.

Tworzymy nowy identyfikator podając string z numerem TERC:
```python
identyfikator = factory.identyfikatory(terc='1465011')
```

Ale to nie wszystko. Metoda ```WyszukajJednostkeWRejestrze()```<br> 
wymaga podania listy identyfikatorów, a nie pojedynczego numeru. 

Uruchamiamy fabrykę ponownie:
```python
array = factory.ArrayOfidentyfikatory(identyfikator)
```

Do pełni szczęścia pozostaje nam wskazać kategorię szukanej jednostki.<br>
Zgodnie z instrukcją "0" oznacza wyszukiwanie wśród wszystkich rodzajów.
```python
client.service.WyszukajJednostkeWRejestrze(identyfiks=array, kategoria=0, DataStanu=STATE_DATE)
```

```python
[{
    'GmiNazwa': 'Warszawa',
    'GmiNazwaDodatkowa': 'gmina miejska, miasto stołeczne',
    'GmiRodzaj': '1',
    'GmiSymbol': '01',
    'PowSymbol': '65',
    'Powiat': 'Warszawa',
    'WojSymbol': '14',
    'Wojewodztwo': 'MAZOWIECKIE'
}]
```

Prościej wygląda sprawa z miejscowościami. Można od razu użyć numeru SIMC:
```python
client.service.WyszukajMiejscowosc(identyfikatorMiejscowosci='0329898')
```

```python
[{
    'GmiRodzaj': '2',
    'GmiSymbol': '04',
    'Gmina': 'Pcim',
    'Nazwa': 'Pcim',
    'PowSymbol': '1209',
    'Powiat': 'myślenicki',
    'Symbol': '0329898',
    'WojSymbol': '12',
    'Wojewodztwo': 'MAŁOPOLSKIE'
}]
```

## Pobieranie katalogów

Usługa umożliwia również pobieranie całych zbiorów danych.<br>
W odpowiedzi na żądanie otrzymujemy obiekt klasy <i>PlikKatalog</i><br>
posiadającej właściwości: 
* <i>nazwa_pliku</i> – string z sugerowaną nazwą pliku
* <i>plik_zawartosc</i> – string z zakodowaną w [Base64](https://pl.wikipedia.org/wiki/Base64) treścią pliku zip
* <i>opis</i> – string z opisem pliku.

A zatem przesłany zostaje plik binarny,<br>
podobnie jak załączniki w poczcie elektronicznej. 

Pobierzemy teraz katalog miejscowości:
```python
catalog = client.service.PobierzKatalogSIMC(STATE_DATE)
```

Jego właściwości zapiszemy do zmiennych:
```python
filename = catalog['nazwa_pliku']
content = catalog['plik_zawartosc']
```

Można oczywiście użyć własnej nazwy pliku, np.: ```filename='katalog.zip'```,<br> 
a także zmienić ścieżkę: ```filename=os.path.expanduser('~/Desktop/katalog.zip')```.

Zawartość pliku odkodujemy używając funkcji ```b64decode()```<br>
z modułu base64 z biblioteki standardowej:
```python
from base64 import b64decode

decoded = b64decode(content)
```

A tak wygląda treść zakodowana i odkodowana (fragment):
```python
CONTENT: c0bANbUuAEcAAAAU0lNQ19VcnplZG9
DECODED: b'\x01\x1c\x00\x00\x00SIMC_Urzedowy_2018-11-23.'
```

Odkodowaną treść zapiszemy jako plik na dysku pod wskazaną nazwą:
```python
with open(filename, 'wb') as file:
    file.write(decoded)
    file.close()
```

Plik zip został "zmaterializowany" i można go otworzyć.<br> 
Ale nie będziemy przecież otwierać tego ręcznie.<br>
Biblioteka standardowa oferuje odpowiednie narzędzia:
```python
from zipfile import ZipFile

zf = ZipFile(filename, 'r')
```

Otrzymaliśmy pythonową reprezentację zapisanego wcześniej pliku.<br>
Sprawdźmy co znajduje się wewnątrz:
```python
print(zf.namelist())
```

Widzimy, że znajdują się tam dwa pliki, XML i CSV:
```python
['SIMC_Urzedowy_2018-11-23.xml', 'SIMC_Urzedowy_2018-11-23.csv']
```

Przeczytajmy teraz pierwszy z nich (ale nie wszystko, kilobajt tylko, stąd n=1024):
```python
with zf.open(zf.namelist()[0]) as xml_file:
    print(xml_file.read(n=1024))
```

Oczywiście parsowanie XML to temat na osobny artykuł,<br>
ale wylistujmy sobie chociaż nazwy miejscowości jakimś prostym narzędziem:
```python
from xml.dom import minidom

with zf.open(zf.namelist()[0]) as xml_file:
    DOMTree = minidom.parse(xml_file)

    children = DOMTree.childNodes
    for row in children[0].getElementsByTagName('row'):
        print(row.getElementsByTagName('NAZWA')[0].childNodes[0].toxml())
```

Konsola zaczyna wyrzucać kolejne nazwy miejscowości:
```python
Zagórze
Zacisze
Dobrzyków
Dzwonów Dolny
Dzwonów Górny
...
```

Zwróćmy uwagę, że metoda ```ZipFile.open()``` zwraca objekt klasy <i>ZipExtFile</i>,<br> 
która dziedziczy po <i>io.BufferedIOBase</i>.<br>
O ile parser XML nie miał problemu z obsługą tego typu danych,<br>
to w przypadku CSV sprawa się komplikuje:
```python
import csv

with zf.open(zf.namelist()[1]) as csv_file:
    csv_reader = csv.reader(csv_file, delimiter=";")
    for row in csv_reader:
        print(row)
```

Przy wykonywaniu pętli rzuciło wyjątkiem:
```
_csv.Error: iterator should return strings, not bytes (did you open the file in text mode?)
```

Dzieje się tak, ponieważ obiekty klasy <i>io.BufferedIOBase</i>,<br>
a co za tym idzie również <i>ZipExtFile</i>,<br>
reprezentują strumienie binarne, a nie tekstowe.

W komunikacie jest podpowiedź, aby plik otworzyć w trybie tekstowym.<br> 
Jak czytamy w opisie metody ```ZipFile.open()``` w [dokumentacji Pythona](https://docs.python.org/3.5/library/zipfile.html?highlight=#zipfile.ZipFile.open):
><i>Use io.TextIOWrapper for reading compressed text files in universal newlines mode.</i>

Nic, tylko zastosować:
```python
import csv
import io

with zf.open(zf.namelist()[1]) as csv_file:
    text_file = io.TextIOWrapper(csv_file)
    csv_reader = csv.reader(text_file, delimiter=";")
    for row in csv_reader:
        print(row)
```

I już możemy cieszyć się widokiem kolejnych rekordów:
```python
['\ufeffWOJ', 'POW', 'GMI', 'RODZ_GMI', 'RM', 'MZ', 'NAZWA', 'SYM', 'SYMPOD', 'STAN_NA']
['02', '16', '01', '5', '03', '1', 'Zagórze', '0363122', '0363100', '2018-01-02']
['02', '16', '01', '5', '03', '1', 'Zacisze', '0363168', '0363145', '2018-01-02']
['02', '16', '01', '5', '03', '1', 'Dobrzyków', '0363263', '0363257', '2018-01-02']
['02', '09', '02', '2', '03', '1', 'Dzwonów Dolny', '0363330', '0363323', '2018-01-02']
['02', '09', '02', '2', '03', '1', 'Dzwonów Górny', '0363346', '0363323', '2018-01-02']
```

Po skończonej pracy nie zapomnijmy zamknąć archiwum:
```python
zf.close()
```
Niby taka błaha sprawa jak odpytanie API,<br>
a ile się można nowych rzeczy nauczyć :slightly_smiling_face:.
