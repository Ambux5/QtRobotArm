# Zadání projektu: Robotické rameno EEZYbotARM Mk2

## Arduino Mega 2560 + Raspberry Pi 4 (8 GB) + openSUSE + Qt 6

**Verze zadání:** 1.0\
**Jazyk projektu a dokumentace:** angličtina nebo čeština podle konvence
repozitáře; komentáře ke kritické matematice a bezpečnosti musí být
srozumitelné.\
**Cíl:** vytvořit skutečně funkční, modulární řídicí systém pro tříosou
konfiguraci robotického ramene inspirovaného EEZYbotARM Mk2.

------------------------------------------------------------------------

# 1. Cíl a rozsah projektu

Vytvoř profesionální řídicí systém pro robotické rameno založené na
konstrukci [EEZYbotARM
Mk2](https://www.instructables.com/EEZYbotARM-Mk2-3D-Printed-Robot/).

Systém se skládá ze dvou hlavních částí:

1.  **Arduino Mega 2560** -- nízkoúrovňové řízení tří digitálních
    polohových servomotorů, plánování pohybu a komunikace.
2.  **Raspberry Pi 4 Model B, 8 GB RAM** -- hlavní počítač s aplikací v
    C++/Qt 6, 3D vizualizací, kinematikou a plánováním trajektorií.

Raspberry Pi komunikuje s Arduinem přes USB sériovou linku.

Hlavní funkce systému:

-   ruční ovládání tří os;
-   ovládání koncového bodu pomocí souřadnic X, Y, Z v milimetrech;
-   realistická interaktivní 3D vizualizace robotického ramene;
-   výpočet dopředné a inverzní kinematiky;
-   plynulý a synchronizovaný pohyb všech tří serv;
-   plánování a náhled trajektorií;
-   kreslení kružnic v dosažitelném pracovním prostoru;
-   kalibrace, diagnostika, bezpečnostní kontroly a ukládání
    konfigurace.

Nejde o pouhou grafickou maketu. Ovládací prvky musí být propojené se
skutečnou komunikací s Arduinem. Simulační režim však musí být dostupný
i bez připojeného hardwaru.

## 1.1 Aktuální rozsah os

Aktuální sestava má tři servomotory:

  Osa   Funkce                               Označení
  ----- ------------------------------------ -----------
  1     Otáčení základny doleva/doprava      základna
  2     Zvedání a sklápění hlavního ramene   rameno
  3     Zvedání a sklápění předloktí         předloktí

Konstrukce EEZYbotARM Mk2 může mít ve své původní konfiguraci další
osu/chapadlo. V tomto projektu však **nepředpokládej připojené čtvrté
servo ani chapadlo**. Architekturu navrhni tak, aby bylo možné další osu
později doplnit, ale současná aplikace musí korektně fungovat se třemi
osami.

U tří os se řeší poloha koncového bodu, nikoli libovolná nezávislá
orientace nástroje. Aplikace nesmí slibovat plnou šestiosou robotickou
manipulaci.

------------------------------------------------------------------------

# 2. Pevně daná hardwarová konfigurace

## 2.1 Řídicí počítač

-   Raspberry Pi 4 Model B.
-   8 GB RAM.
-   ARM64 platforma.
-   Operační systém openSUSE Linux.
-   Připojení Arduina přes USB.

## 2.2 Operační systém

Cílovým systémem je openSUSE Linux. Přesnou variantu je nutné při prvním
spuštění zjistit -- může jít například o openSUSE Tumbleweed nebo Leap.

**Nepředpokládej Raspberry Pi OS, Ubuntu ani Debian.**

-   Používej Zypper pro instalaci systémových balíčků.
-   Ověř dostupnost Qt 6 a jednotlivých modulů v repozitářích konkrétní
    verze openSUSE.
-   Nepředpokládej, že každý Qt modul je dostupný jako balíček nebo že
    je dostupná stejná verze ve všech vydáních.
-   V instalační dokumentaci uveď případné rozdíly mezi Tumbleweed a
    Leap.
-   Pokud některá závislost není v repozitářích dostupná, popiš ověřenou
    alternativu. Nevymýšlej názvy balíčků.

## 2.3 Arduino

-   Arduino Mega 2560.
-   USB sériová komunikace s Raspberry Pi.
-   Firmware v C++ pro Arduino.
-   Výchozí rychlost sériové linky 115200 baud.

## 2.4 Servomotory

Používají se 3× **INJORA INJS025-270**.

Známé orientační parametry varianty:

-   digitální polohové servo;
-   rozsah varianty 270°;
-   řídicí pulzy přibližně 500--2500 µs;
-   neutrální pulz přibližně 1500 µs;
-   napájení 4,8--6 V;
-   kovové převody;
-   výrobcem uváděný moment přibližně 25 kg·cm.

Před finální implementací ověř parametry podle aktuální dokumentace
výrobce a konkrétní revize serva. Tyto hodnoty jsou výchozí technické
údaje, nikoli záruka, že mechanická sestava dovolí celý rozsah.

**Důležité:** 270° rozsah serva neznamená, že robotické rameno smí
využít celý rozsah. Limity každé osy musí být kalibrovány podle skutečné
mechaniky, uchycení a vůlí.

Používej řízení pulzní šířkou v mikrosekundách, například
`writeMicroseconds()`, pokud je kompatibilní s použitou knihovnou.
Nepředpokládej standardní mapování 0--180°.

## 2.5 Napájení

Serva nenapájej z 5V pinu Arduina.

-   Použij samostatný stabilizovaný zdroj odpovídající specifikaci serv.
-   Zdroj dimenzuj podle součtu proudových špiček všech tří serv a
    reálného zatížení.
-   Propoj GND napájecího zdroje serv s GND Arduina.
-   Zajisti vhodné vodiče, konektory a jištění.
-   Doporuč vhodné odrušení a dostatečnou kapacitu v napájecí větvi
    podle návrhu zapojení.
-   Nepředpokládej, že údaj o momentu serva určuje jeho proudový odběr.
    Proud ověř podle dokumentace nebo měřením.
-   Raspberry Pi a serva napájej vhodně oddělenými napájecími větvemi.

Připrav přehledné schéma zapojení a tabulku pinů.

------------------------------------------------------------------------

# 3. Arduino Mega 2560 -- firmware

Vytvoř modulární firmware v C++.

Doporučené výchozí přiřazení pinů (musí být konfigurovatelné):

  Servo             Pin Arduino Mega
  --------------- ------------------
  Základna                         3
  Hlavní rameno                    4
  Předloktí                        5

Ověř kompatibilitu vybraných pinů a knihovny. Nepředpokládej, že
libovolný PWM pin je nutný -- běžná Servo knihovna používá časovače
podle své implementace.

## 3.1 Funkční požadavky

Firmware musí podporovat:

1.  Inicializaci tří serv.
2.  Individuální nastavení pulzů a kalibrace.
3.  Individuální softwarové limity každé osy.
4.  Nastavení výchozí polohy.
5.  Obrácení směru osy.
6.  Plynulý pohyb všech tří os současně.
7.  Synchronizaci začátku a konce koordinovaného pohybu.
8.  Nastavení rychlosti z Qt aplikace.
9.  Omezení rychlosti a akcelerace.
10. Nezávislé zpracování sériové komunikace během pohybu.
11. Nouzové zastavení.
12. Stavové zprávy a potvrzování příkazů.
13. Detekci chybných, neúplných a mimo rozsah zadaných hodnot.

## 3.2 Synchronizovaný pohyb

Příklad požadované změny:

-   Servo 1: 90° → 150°.
-   Servo 2: 90° → 120°.
-   Servo 3: 130° → 70°.

Všechna serva mají zahájit pohyb koordinovaně. Plánovač má přizpůsobit
rychlost jednotlivých os tak, aby cíle dosáhly ve stejném čase, pokud to
dovolují jejich limity.

Použij neblokující plánovač. Nepoužívej dlouhé `delay()`, které by
zastavilo příjem příkazů nebo zpracování STOP.

Pro pohyb implementuj vhodný profil rychlosti, například trapezoidální
profil nebo profil s omezenou akcelerací a decelerací. Zohledni, že
běžné polohové servo nedostává příkaz skutečné rychlosti -- firmware
pouze plánuje změnu požadované polohy v čase. Servo může mít vlastní
vnitřní regulaci a fyzicky nemusí přesně sledovat požadovaný profil.

Rychlost vyjadřuj jednoznačně. Rozlišuj rychlost osy a rychlost
koncového bodu. Nepředstírej, že procentuální nastavení odpovídá přesně
fyzikální rychlosti bez kalibrace.

## 3.3 Pulzy a kalibrace

Pro každou osu ukládej minimálně:

-   minimální povolený pulz v µs;
-   maximální povolený pulz v µs;
-   neutrální/výchozí pulz;
-   směr;
-   offset;
-   softwarový minimální a maximální úhel;
-   výchozí požadovanou polohu;
-   rychlostní limit;
-   limit akcelerace.

Nikdy automaticky neposílej servo na 500 nebo 2500 µs bez ověření
mechanického rozsahu. Kalibrace musí začínat konzervativně a rozsah se
smí rozšiřovat pouze postupným testem.

## 3.4 Stav zařízení a odpojení

Firmware musí rozlišovat alespoň:

-   inicializace;
-   připraveno;
-   pohyb;
-   dokončeno;
-   zastaveno;
-   chyba.

Při ztrátě USB komunikace zastav přijímání nových plánovaných pohybů a
přejdi do předem definovaného stavu. Nepředpokládej, že odpojení USB
automaticky znamená bezpečné mechanické zastavení nebo odpojení napájení
serv.

Po opětovném připojení nesmí dojít k automatickému pohybu. Vyžaduj nové
navázání komunikace a explicitní povolení obsluhy.

------------------------------------------------------------------------

# 4. USB sériový protokol

Navrhni jednoduchý, dokumentovaný a rozšiřitelný protokol.

Výchozí parametry:

-   115200 baud;
-   8 datových bitů;
-   bez parity;
-   1 stop bit;
-   zprávy ukončené `\n`;
-   číselné hodnoty s jasně definovanými jednotkami;
-   parser odolný vůči neúplným a chybným zprávám.

Příklady zamýšlených příkazů:

``` text
HELLO
SET_ANGLES,90,120,70
MOVE_ANGLES,90,120,70,20
SET_SPEED,20
STOP
HOME
STATUS
```

XYZ příkaz může být součástí vyšší vrstvy, ale musí být jednoznačné, zda
IK probíhá v Raspberry Pi nebo v Arduinu. Doporučená architektura:
kinematika a plánování trajektorie probíhají na Raspberry Pi; Arduino
dostává bezpečně ověřené cílové úhly nebo malé segmenty pohybu.

Definuj odpovědi:

-   `ACK` -- příkaz přijat;
-   `ERROR` -- příkaz odmítnut, včetně kódu a popisu chyby;
-   `STATUS` -- stav zařízení a poslední požadované polohy;
-   `BUSY` -- probíhá pohyb;
-   `DONE` -- pohyb dokončen;
-   `STOPPED` -- pohyb zastaven.

Každý příkaz musí mít definovaný počet parametrů, rozsahy, jednotky a
chování při chybě.

Zvaž číslování příkazů nebo identifikátory sekvencí, aby bylo možné
spolehlivě přiřadit odpověď ke konkrétnímu příkazu.

Ošetři přetečení vstupního bufferu, příliš dlouhé zprávy, nečíselné
hodnoty a neplatné hodnoty. Nikdy nevykonávej částečně přijatý příkaz.

Protokol dokumentuj v `Documentation/SERIAL_PROTOCOL.md`.

------------------------------------------------------------------------

# 5. Raspberry Pi -- cílové softwarové prostředí

## 5.1 Technologie

Preferované technologie:

-   C++17 nebo novější;
-   Qt 6;
-   CMake;
-   GCC dostupný pro ARM64/openSUSE;
-   Qt Serial Port (`QSerialPort`);
-   Qt Quick 3D pro 3D vizualizaci, pokud je modul dostupný a výkonově
    vhodný;
-   Qt Quick / QML nebo Qt Widgets podle výsledného návrhu;
-   Eigen pro matematické výpočty, pokud přinese užitek;
-   Qt Test nebo GoogleTest pro jednotkové testy.

Zvol jednu konzistentní architekturu UI. Nemíchej bez důvodu Qt Widgets
a QML.

## 5.2 openSUSE a ARM64

Před implementací zjisti:

-   přesnou variantu openSUSE;
-   verzi systému;
-   architekturu;
-   verzi GCC;
-   dostupnou verzi Qt 6;
-   dostupnost Qt Quick 3D;
-   dostupnost Qt Serial Port;
-   dostupnost grafických ovladačů a OpenGL ES.

Nevytvářej instalační příkazy naslepo. Ověř názvy balíčků pro cílovou
verzi openSUSE. Rozliš Tumbleweed a Leap, pokud se balíčky liší.

Připrav:

-   `install_opensuse.sh`;
-   `build.sh`;
-   `run.sh`;
-   `diagnostics.sh`;
-   CMake konfiguraci;
-   README s ověřeným postupem instalace.

Instalační skript musí kontrolovat chyby a nesmí bez vysvětlení měnit
systémová nastavení.

## 5.3 Výkon Raspberry Pi 4

Optimalizuj aplikaci pro Raspberry Pi 4:

-   nepoužívej zbytečně složitou geometrii;
-   omez náročné stíny a efekty;
-   překresluj jen při změně dat nebo potřebném animačním kroku;
-   odděl GUI vlákno od sériové komunikace a výpočtů;
-   nepřetěžuj hlavní vlákno;
-   omez frekvenci stavových aktualizací na rozumnou hodnotu;
-   umožni snížení kvality 3D vykreslování;
-   ošetři případ, kdy hardwarová akcelerace nebo Qt Quick 3D není
    dostupná.

Aplikace musí mít také diagnostický režim, který zobrazí grafický
backend a případné problémy s akcelerací.

------------------------------------------------------------------------

# 6. Grafické rozhraní

Vytvoř moderní, profesionální a přehledné tmavé rozhraní inspirované
průmyslovými HMI, CNC a robotickými řídicími systémy.

Nepoužívej obyčejné okno s několika standardními tlačítky. Zároveň
upřednostni čitelnost, ergonomii a stabilitu před dekorativními efekty.

Hlavní navigace:

1.  Dashboard
2.  Manual Control
3.  XYZ Control
4.  Trajectory / Drawing
5.  Calibration
6.  Settings
7.  Diagnostics

## 6.1 Dashboard

Hlavní obrazovka má obsahovat:

**Dominantní 3D panel:** - realistický model robotického ramene; -
základnu, hlavní rameno a předloktí; - správně umístěné klouby; -
souřadnicový systém X/Y/Z; - pracovní prostor; - aktuální modelovanou
polohu; - cílový bod; - zobrazení plánované trajektorie; - orbitální
kameru, zoom a přednastavené pohledy.

**Ovládací a stavový panel:** - aktuální požadované X, Y, Z v mm; -
cílové X, Y, Z v mm; - požadované úhly jednotlivých os; - nastavenou
rychlost; - stav USB/Arduino; - stav pohybu; - poslední chybu; -
tlačítka HOME a STOP.

Jasně označuj, že úhly a poloha jsou vypočtené/požadované hodnoty. Bez
snímačů nelze tvrdit, že jde o změřenou skutečnou polohu serv.

## 6.2 Realistický 3D model

Model musí odpovídat mechanice EEZYbotARM Mk2, nikoli pouze připomínat
obecné robotické rameno.

Nejdříve analyzuj dostupnou konstrukční dokumentaci a geometrii. Pokud
jsou k dispozici vhodné CAD/STL modely a jejich licence dovoluje
použití, připrav import nebo odvození z nich. Pokud modely nejsou k
dispozici, vytvoř parametrickou geometrii a jasně označ, které části
jsou aproximace.

Každý článek musí mít správný transformační bod, lokální osu rotace a
hierarchii transformací.

Při změně úhlu se musí správně přepočítat všechny navazující články.

Umožni: - pohled zepředu; - pohled z boku; - pohled shora; - izometrický
pohled; - volné otáčení kamery; - zoom; - zobrazení souřadnicových os; -
zobrazení pracovní oblasti; - zobrazení trajektorie.

3D model musí používat stejnou kalibrační a geometrickou konfiguraci
jako kinematické výpočty.

------------------------------------------------------------------------

# 7. Geometrie a kinematika

Tato část je kritická. Nezačínej odhadnutými rozměry.

## 7.1 Parametrická geometrie

Vytvoř konfigurovatelný model obsahující alespoň:

-   výšku osy základny;
-   polohu osy ramene;
-   délku hlavního ramene;
-   polohu lokte;
-   délku předloktí;
-   polohu koncového bodu;
-   případné posuny mezi osami;
-   kalibrační offset každého serva;
-   směr otáčení každého serva;
-   mechanické limity.

Všechny délky zadávej v milimetrech.

Použij rozměry odpovídající skutečné verzi EEZYbotARM Mk2. Pokud je
nelze spolehlivě zjistit z dokumentace, připrav konfigurační položky a
požádej uživatele o změření. Nevymýšlej konkrétní rozměry.

## 7.2 Forward Kinematics (FK)

Implementuj výpočet polohy koncového bodu ze tří úhlů serv.

Ověř správné: - pořadí transformací; - znaménka os; - souřadnicový
systém; - offsety serv; - výšku základny; - rozměry článků.

## 7.3 Inverse Kinematics (IK)

Implementuj převod cílových souřadnic XYZ na tři požadované úhly.

Požadavky: - kontrola dosažitelnosti; - kontrola mechanických limitů; -
řešení více možných konfigurací; - výběr řešení s ohledem na nejmenší
změnu proti aktuálnímu stavu; - stabilní chování v blízkosti
singularit; - srozumitelná chyba při nedosažitelném bodu.

Pokud je to vhodné pro geometrii ramene, použij analytické řešení. Jinak
použij numerické řešení s omezením, konvergencí a testy. Výběr metody
zdůvodni.

U tříosého ramene neumožňuj zadání libovolné orientace koncového
nástroje jako nezávislého cíle. Primárním cílem je poloha koncového
bodu.

## 7.4 Testy kinematiky

Vytvoř jednotkové testy:

-   známé konfigurace úhlů → očekávané XYZ;
-   dosažitelné XYZ → platné úhly;
-   nedosažitelné XYZ → korektní chyba;
-   limity serv;
-   hraniční polohy;
-   opakovaný převod FK → IK → FK;
-   konzistence 3D modelu s FK.

Testy musí používat známé geometrické parametry a nesmí předstírat
přesnost skutečné konstrukce, dokud není změřena a kalibrována.

------------------------------------------------------------------------

# 8. XYZ ovládání

Vytvoř samostatnou obrazovku pro kartézské ovládání.

Příklad vstupu:

``` text
X = 150 mm
Y = 50 mm
Z = 120 mm
```

Aplikace musí:

1.  načíst cílové XYZ;
2.  vypočítat IK;
3.  ověřit dosažitelnost;
4.  ověřit limity všech serv;
5.  zobrazit cílový bod v 3D modelu;
6.  vytvořit plán pohybu;
7.  zobrazit odhad doby pohybu;
8.  vyžádat spuštění, pokud je vyžadováno potvrzení;
9.  odeslat příkaz Arduinu;
10. sledovat stav a dokončení.

Přidej jog ovládání: - X+ / X−; - Y+ / Y−; - Z+ / Z−.

Nastavitelný krok: - 1 mm; - 5 mm; - 10 mm; - vlastní hodnota.

Přidej přímé ovládání jednotlivých serv pro kalibraci a servisní účely.
Přímé ovládání úhlů musí stále respektovat limity.

------------------------------------------------------------------------

# 9. Plánování trajektorií a kreslení kružnice

Vytvoř samostatnou sekci `Trajectory / Drawing`.

Hlavním demonstračním úkolem je plynulé vykreslení kružnice koncovým
bodem ramene.

## 9.1 Parametry kružnice

Uživatel zadá: - střed kružnice X, Y, Z v mm; - poloměr v mm; - rovinu
nebo orientaci kružnice; - rychlost koncového bodu; - počet vzorkovacích
bodů nebo požadovanou přesnost; - směr pohybu; - počet opakování.

Podporované základní roviny: - XY; - XZ; - YZ.

Navíc umožni definovat libovolnou orientovanou rovinu pomocí dvou
kolmých jednotkových vektorů nebo normálového vektoru.

Parametrická rovnice:

\[ P(t) = C + R(`\cos`{=tex}(t)U + `\sin`{=tex}(t)V), `\quad `{=tex}t
`\in [0, 2\pi]`{=tex}\]

Kde: - `C` je střed kružnice; - `R` je poloměr; - `U` a `V` jsou kolmé
jednotkové vektory v rovině kružnice; - `t` je parametr.

## 9.2 Kontrola před spuštěním

Před pohybem: 1. vygeneruj celou trajektorii; 2. ověř každý vzorkovací
bod pomocí IK; 3. ověř limity všech serv; 4. ověř spojitost vybraného
řešení IK mezi sousedními body; 5. odhal skoky úhlů a singularity; 6.
ověř rychlostní a akcelerační limity; 7. zobraz neplatné úseky; 8.
odmítni spuštění celé trajektorie, pokud není bezpečně proveditelná.

Nesmí se stát, že aplikace spustí kružnici a až během pohybu zjistí, že
její část není dosažitelná.

## 9.3 Plynulost

Robot nesmí mezi jednotlivými vzorkovacími body zastavovat.

Neposílej body jako nezávislé příkazy s čekáním na úplné zastavení serva
po každém bodu.

Použij průběžné plánování nebo buffer segmentů. Zajisti: - plynulé
navazování segmentů; - omezení rychlosti a akcelerace; - přiměřenou
hustotu vzorkování; - konstantní nebo co nejvíce konstantní rychlost
koncového bodu podél trajektorie; - koordinaci všech tří os.

Uvědom si, že samotná synchronizace úhlů serv nezaručuje konstantní
rychlost koncového bodu. Rychlost koncového bodu plánuj v kartézském
prostoru a převáděj ji přes kinematiku na požadavky jednotlivých os.

Zohledni omezení polohových serv: nemají externí zpětnou vazbu skutečné
polohy ani přesné řízení rychlosti. V dokumentaci vysvětli, že přesnost
a plynulost závisí na zatížení, napájení, vůlích, vlastnostech serv a
kalibraci.

## 9.4 Náhled

Před spuštěním zobraz: - kružnici v 3D; - střed; - poloměr; - počáteční
bod; - směr pohybu; - případné nedosažitelné body; - očekávané úhly
serv; - odhad trvání.

Uživatel musí moci trajektorii spustit, pozastavit, pokračovat a
zastavit.

------------------------------------------------------------------------

# 10. Kalibrace a konfigurace

Vytvoř samostatnou obrazovku kalibrace.

Pro každé servo umožni nastavit: - název osy; - pin Arduina; - minimální
pulz v µs; - maximální pulz v µs; - neutrální pulz; - výchozí polohu; -
směr otáčení; - offset; - softwarové minimum a maximum; - maximální
rychlost; - maximální akceleraci.

Pro geometrii umožni nastavit všechny délky a posuny v mm.

Kalibrace musí být bezpečný řízený proces: 1. začít konzervativním
rozsahem; 2. pohybovat se po malých krocích; 3. kontrolovat mechanické
dorazy; 4. uložit ověřené limity; 5. nikdy automaticky neprojet celý
rozsah serva.

Konfiguraci ukládej do JSON souboru. Zajisti: - validaci při načtení; -
výchozí konfiguraci; - zálohu před přepsáním; - import a export; - verzi
schématu konfigurace; - srozumitelné chybové hlášení při neplatném
souboru.

------------------------------------------------------------------------

# 11. Bezpečnost a diagnostika

Implementuj: - softwarové limity všech os; - kontrolu dosažitelnosti; -
kontrolu celé trajektorie před spuštěním; - detekci odpojení Arduina; -
kontrolu komunikačních timeoutů; - ochranu proti chybným příkazům; -
ochranu proti přetížení komunikační fronty; - zobrazení chyb; - logování
příkazů a stavů; - potvrzení před automatickou trajektorií; - zákaz
automatického pohybu po připojení USB; - dostupné tlačítko STOP na všech
hlavních obrazovkách.

STOP musí být zpracován s nejvyšší prioritou v rámci softwarové
komunikace. Dokumentuj však, že USB a software nejsou bezpečnostně
certifikované nouzové zastavení.

Připrav možnost budoucího fyzického nouzového vypínače, který bezpečně
odpojí napájení serv. Nezaměňuj softwarové STOP s fyzickým odpojením
energie.

Bez enkodérů nebo jiné zpětné vazby zobrazuj pouze poslední
požadovanou/modelovanou polohu, nikoli změřenou skutečnou polohu.

------------------------------------------------------------------------

# 12. Architektura a struktura repozitáře

Navrhni modulární strukturu. Můžeš ji upravit, ale zachovej oddělení
odpovědností.

``` text
RobotArmProject/
├── README.md
├── CMakeLists.txt
├── Arduino/
│   ├── RobotArmController.ino
│   ├── ServoController.h
│   ├── ServoController.cpp
│   ├── MotionPlanner.h
│   ├── MotionPlanner.cpp
│   ├── SerialProtocol.h
│   └── SerialProtocol.cpp
├── RaspberryPi/
│   ├── CMakeLists.txt
│   ├── main.cpp
│   ├── Communication/
│   │   ├── SerialManager.h
│   │   └── SerialManager.cpp
│   ├── Kinematics/
│   │   ├── RobotGeometry.h
│   │   ├── ForwardKinematics.h
│   │   ├── ForwardKinematics.cpp
│   │   ├── InverseKinematics.h
│   │   └── InverseKinematics.cpp
│   ├── Motion/
│   │   ├── TrajectoryPlanner.h
│   │   └── TrajectoryPlanner.cpp
│   ├── Visualization/
│   │   ├── RobotModel/
│   │   └── Scene3D/
│   ├── Configuration/
│   ├── Diagnostics/
│   └── UI/
│       ├── Dashboard/
│       ├── ManualControl/
│       ├── XYZControl/
│       ├── TrajectoryControl/
│       ├── Calibration/
│       └── Settings/
├── Tests/
│   ├── Kinematics/
│   ├── Trajectory/
│   └── Protocol/
├── Scripts/
│   ├── install_opensuse.sh
│   ├── build.sh
│   ├── run.sh
│   └── diagnostics.sh
└── Documentation/
    ├── HARDWARE.md
    ├── SOFTWARE.md
    ├── SERIAL_PROTOCOL.md
    ├── CALIBRATION.md
    ├── KINEMATICS.md
    └── TROUBLESHOOTING.md
```

Každá třída a modul musí mít jasnou odpovědnost. Nevkládej veškerou
logiku do `MainWindow` nebo jediného `.cpp` souboru.

------------------------------------------------------------------------

# 13. Vývojové etapy

Neimplementuj celý projekt najednou. Postupuj po etapách a po každé
etapě vytvoř sestavitelný a otestovaný výsledek.

## Etapa 0 -- Analýza a technický návrh

-   Prohlédni existující repozitář.
-   Ověř cílovou verzi openSUSE a dostupné balíčky.
-   Analyzuj konstrukci EEZYbotARM Mk2.
-   Zjisti dostupné konstrukční rozměry a modely.
-   Sepiš předpoklady a neznámé parametry.
-   Navrhni architekturu, protokol a konfigurační schéma.
-   Nevymýšlej rozměry ani kalibraci.

## Etapa 1 -- Arduino

-   Implementuj ovládání tří serv.
-   Implementuj kalibraci pulzů.
-   Implementuj limity.
-   Implementuj synchronizovaný neblokující pohyb.
-   Implementuj STOP a STATUS.
-   Ověř pohyb nejprve s konzervativními limity.

## Etapa 2 -- USB komunikace

-   Implementuj dokumentovaný protokol.
-   Ověř příjem a odesílání.
-   Ověř chybové stavy.
-   Ověř odpojení a opětovné připojení.
-   Ověř, že po připojení nedojde k nevyžádanému pohybu.

## Etapa 3 -- Qt aplikace

-   Vytvoř CMake projekt.
-   Vytvoř hlavní okno a navigaci.
-   Implementuj QSerialPort.
-   Zobraz stav zařízení.
-   Přidej simulační režim bez Arduina.

## Etapa 4 -- Kinematika

-   Vytvoř parametrickou geometrii.
-   Implementuj FK.
-   Implementuj IK.
-   Přidej testy.
-   Ověř výpočty na známých konfiguracích.

## Etapa 5 -- 3D vizualizace

-   Vytvoř model odpovídající konstrukci.
-   Nastav správnou hierarchii kloubů.
-   Propoj model s kinematikou.
-   Přidej kameru, pohledy a souřadnicové osy.
-   Ověř shodu modelu s FK.

## Etapa 6 -- XYZ ovládání

-   Přidej zadávání XYZ.
-   Přidej jog pohyb.
-   Přidej kontrolu dosažitelnosti.
-   Ověř pohyb nejprve v simulaci, poté na skutečném rameni.

## Etapa 7 -- Trajektorie a kružnice

-   Implementuj generátor kružnice.
-   Implementuj kontrolu celé trajektorie.
-   Implementuj náhled v 3D.
-   Implementuj plynulé předávání segmentů.
-   Ověř nejprve v simulaci a následně při nízké rychlosti na hardwaru.

## Etapa 8 -- Dokončení

-   Kalibrace.
-   Diagnostika.
-   Bezpečnost.
-   Instalace na openSUSE.
-   Dokumentace.
-   Automatické spouštění po přihlášení, pokud si to uživatel zvolí.

------------------------------------------------------------------------

# 14. Pravidla pro GitHub Copilot

1.  Nejdříve analyzuj celý projekt a navrhni postup. Nezačínej generovat
    všechny soubory bez plánu.
2.  Před úpravou existujících souborů si přečti jejich aktuální obsah.
3.  Nevymýšlej rozměry ramene, parametry serv ani vlastnosti operačního
    systému.
4.  Pokud chybí důležitý parametr, vytvoř konfigurovatelnou položku a
    označ ji jako neověřenou. Pokud bez ní nelze bezpečně pokračovat,
    požádej uživatele o doplnění.
5.  Piš moderní, čitelný a modulární C++.
6.  Odděluj GUI, komunikaci, kinematiku, plánování a hardware.
7.  Používej jednotkové testy pro matematiku, protokol a plánování.
8.  Nevynechávej bezpečnostní kontroly.
9.  Nevytvářej nefunkční vizuální maketu -- ovládací prvky musí mít
    skutečné chování.
10. Po každé etapě projekt sestav a otestuj.
11. Všechny jednotky a souřadnicové konvence dokumentuj.
12. Při změně protokolu aktualizuj firmware, Qt aplikaci i dokumentaci.
13. Nepředpokládej dostupnost Qt Quick 3D bez ověření na cílovém
    openSUSE.
14. Nepoužívej API nebo knihovny, které nejsou dostupné pro cílovou
    ARM64 platformu, bez zdokumentované alternativy.
15. Každý bezpečnostně významný předpoklad výslovně zdokumentuj.
16. Nikdy netvrď, že fyzická poloha serv je změřena, pokud systém nemá
    odpovídající snímač.

------------------------------------------------------------------------

# 15. Kritéria dokončení projektu

Projekt je dokončen, když:

1.  Arduino Mega 2560 ovládá tři INJORA INJS025-270.
2.  Každé servo má samostatně nastavitelnou kalibraci a bezpečné limity.
3.  Všechna tři serva se pohybují koordinovaně.
4.  Rychlost pohybu lze měnit z aplikace.
5.  USB komunikace je spolehlivá a zdokumentovaná.
6.  Qt 6 aplikace se sestaví a běží na Raspberry Pi 4 s openSUSE.
7.  Aplikace zobrazuje realistický interaktivní 3D model EEZYbotARM Mk2.
8.  3D model používá stejnou geometrii a úhly jako kinematické výpočty.
9.  Funguje dopředná i inverzní kinematika.
10. Uživatel může zadávat cílové XYZ v milimetrech.
11. Nedosažitelné body jsou odmítnuty před pohybem.
12. Lze definovat kružnici v rovinách XY, XZ a YZ a také v libovolné
    orientované rovině.
13. Celá kružnice se před spuštěním ověří.
14. Robot dokáže v dosažitelném pracovním prostoru kružnici plynule
    vykreslit.
15. Funguje STOP, diagnostika a detekce odpojení.
16. Kalibrace a konfigurace zůstávají uloženy po restartu.
17. Existuje dokumentace zapojení, instalace, kalibrace, protokolu a
    kinematiky.
18. Projekt lze sestavit podle zdokumentovaného postupu na cílovém
    Raspberry Pi.

## Očekávaný konečný scénář

1.  Zapnu Arduino Mega 2560.
2.  Připojím Arduino přes USB k Raspberry Pi 4.
3.  Spustím aplikaci na openSUSE.
4.  Aplikace rozpozná Arduino, ale sama nespustí žádný pohyb.
5.  V 3D prostoru vidím model robotického ramene.
6.  Zadám souřadnice X, Y, Z.
7.  Aplikace vypočítá úhly serv a ověří dosažitelnost.
8.  Po potvrzení se rameno plynule přesune do cílové polohy.
9.  V sekci Drawing nastavím střed, poloměr a rychlost kružnice.
10. Aplikace zobrazí a ověří celou trajektorii.
11. Po potvrzení robot plynule vykreslí kružnici v dosažitelné části
    pracovního prostoru.

**Výsledkem má být rozšiřitelný základ profesionálního robotického
řídicího systému. Do budoucna musí být možné přidat chapadlo, další osu,
snímače polohy, kameru a automatizované úlohy.**
