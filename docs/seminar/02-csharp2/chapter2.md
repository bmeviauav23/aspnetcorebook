---
authors: kszicsillag,tibitoth
---

# C# alapok II.

## Előkészítés

Első lépésként hozzunk létre egy .NET C# konzolalkalmazást: a projektsablon szűrőben válasszuk a C# nyelv - Windows platform - *Console* projekttípust.
A szűrt listában válasszuk a *Console App* sablont (**ne** a .NET Framework-ös legyen). A neve legyen *HelloCSharp2*.
A solutiont ne tegyük külön mappába (*Place solution and project in the same directory* legyen bekapcsolva).
A megcélzott framework verzió legyen .NET 8.

### Legfelső szintű utasítások, implicit globális névtér-hivatkozások

Csodálkozzunk rá, hogy a generált projekt mindössze egyetlen érdemi sort tartalmaz.

``` csharp
Console.WriteLine("Hello, World!");
```

C# 10 óta a program belépési pontját adó forrásfájlt jelentősen lerövidíthetjük:

* a fájl tetején lévő using-okat elhagyhatjuk, ha azok implicit hivatkozva vannak. Az implicit hivatkozott using-ok projekttípustól függenek és a [dokumentációból](https://docs.microsoft.com/en-us/dotnet/core/project-sdk/overview#implicit-using-directives) olvashatjuk ki
* a `Main` függvényt tartalmazó osztály deklarációját (`namespace` blokk, `class` blokk) elhagyhatjuk, ezt a fordító generálja nekünk
* a `Main` függvény deklarációját szintén generálja a fordító. A metódus neve nem definiált, nem (biztos, hogy) `Main`. A metódus szignatúrája attól függ, milyen utasításokat adunk meg a forrásfájlban. Például, ha nincs return, akkor `void` visszatérési értékű. A paramétere viszont mindig `string[] args`.
* a függvény blokkba nem foglalt kód a generált belépési pont függvény belsejébe kerül. Függvényt is írhatunk, az a belépési pontot tartalmazó generált osztály tagfüggvénye lesz.
* típusokat, osztályokat is definiálhatunk, de csak a legfelső szintű kódot követően

!!! warning
    Fontos észrevétel a fentiekből: ezen képesség nem változtatja meg a C# semmilyen alapvető jellemzőjét, például ugyanúgy minden függvénynek osztályon belül kell lennie. A fordítás során a legfelső szintű utasítások kódja úgy egészül ki, ami már minden szabálynak megfelel.

!!! warning "Láthatóság"
    A legfelső szintű kód olyan, amit a program más részéről nem tudunk hívni, hiszen nem is ismerjük a burkoló osztály nevét. Emiatt nincs értelme legfelső szintű kódban láthatósági beállításnak (`private`, `protected` stb.) vagy propertynek.

Akadályozzuk meg a program azonnali lefutását egy blokkoló hívással.

``` csharp hl_lines="2"
Console.WriteLine("Hello, World!");
Console.ReadLine();
```

Próbáljuk ki a generált projektet mindenféle egyéb változtatás nélkül, fordítás (menu:projekten jobbklikk\[Build\]) után. Nézzünk bele a kimeneti könyvtárba (menu:projekten jobbklikk\[Open Folder in File Explorer\], majd menu:bin\[Debug \> net8.0\]): látható, hogy az alkalmazásunkból a fordítás során egy cross-platform bináris (\<projektnév\>.dll) és .NET Core v3 óta egy platform specifikus futtatható állomány (Windows esetén \<projektnév\>.exe) is generálódik. Kipróbálhatjuk, hogy az exe a szokott módon indítható (pl. duplaklikkel), míg a dll a `dotnet` paranccsal.

``` cmd
dotnet <projektnév.dll>
```

!!! tip "Parancssor aktuális mappája"
    A dotnet parancshoz a dll könyvtárában kell lennünk. Ehhez a legegyszerűbb, ha a Windows fájlkezelőben a megfelelő könyvtárban állva az elérési útvonal mezőt átírjuk a `cmd` szövegre, majd ++enter++-t nyomunk.

Adjunk a létrejövő projekthez egy `Dog` osztályt *Dog.cs* néven, ez lesz az adatmodellünk:

``` csharp
public class Dog
{
    public string Name { get; set; }
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime DateOfBirth { get; set; }
    private int AgeInDays => DateTime.Now.Subtract(DateOfBirth).Days;
    public int Age => AgeInDays / 365;
    public int AgeInDogYears => AgeInDays * 7 / 365;
    public override string ToString() =>
            $"{Name} ({Age} | {AgeInDogYears}) [ID: {Id}]";
}
```

Az adatmodell az előző órán létrehozotthoz nagyon hasonlít, ennek viszont nincsen explicit konstruktora és a `Name` és `DateOfBirth` tulajdonságok publikusan is állíthatók.

Hozzunk létre egy `Dog` példányt objektum inicializációs szintaxissal, majd írjuk ki ezt a példányt a kezdeti köszöntő szöveg helyett:

``` csharp
Dog banan = new Dog
{
    Name = "Banán",
    DateOfBirth = new DateTime(2014, 06, 10)
};
Console.WriteLine(banan);
```

Ezzel kész a kiinduló projektünk.

## Implicit típusdeklaráció

A `var` kulcsszó jelentősége: ha a fordító ki tudja találni a kontextusból az értékadás jobb oldalán álló érték típusát, nem szükséges a típus nevét explicit megadnunk, az implicit következik a kódból.
Ebben az esetben a típus egyértelműen `Dog`.
Ha csak deklarálni szeretnénk egy változót (nem adunk értékül a változónak semmit), akkor nem használhatjuk a `var` kulcsszót, ugyanis nem következik a kódból a változó típusa.
Ekkor explicit meg kell adnunk a típust.

``` csharp hl_lines="6-11"
Dog banan = new Dog
{
   Name = "Banán",
   DateOfBirth = new DateTime(2014, 06, 10)
};
var watson = new Dog { Name = "Watson" };

var unnamed = new Dog { DateOfBirth = new DateTime(2017, 02, 10) };
var unknown = new Dog { };
//watson = 3; // (1)
//var error;  // (2)

Console.WriteLine(banan);
Console.ReadLine();
```

1. Fordítási hiba: a `watson` deklarációjakor eldőlt, hogy ő `Dog` típus, utólag nem lehet megváltoztatni és például számértéket értékül adni. Ez nem JavaScript.
2. Fordítási hiba: implicit típust csak úgy lehet deklarálni, ha egyúttal inicializáljuk is. Az inicializációs kifejezés alapján dől el (implicit) a példány típusa.

Próbáljuk ki a nem forduló sorokat, nézzük meg a fordító hibaüzeneteit!

!!! warning "Erőss típusoság"
    A `var` nem a gyenge típusosság jele a C#-ban, nem úgy, mint pl. JavaScript-ben. Az inicializációs sor után a típus egyértelműen eldől, utána már csak ennek a típusnak megfelelő műveletek végezhetők, például egy értékadással nem változtathatjuk meg a típust.

A `var`-t tipikusan akkor alkalmazzuk, ha:

* hosszú típusneveket nem akarunk kiírni
* feleslegesnek tartjuk az inicializáció mindkét oldalán kiírni ugyanazt a típust
* anonim típusokat használunk (később)

## Init-only setter

Az objektum inicializáció működéséhez szükséges a megfelelő láthatóságú setter. Viszont egy ilyen settert nem csak objektum inicializációkor lehet használni, hanem bármikor átállíthatjuk egy példány adatát (mutáció).

Az alábbi példa egy ilyen utólagos módosításra / mutációra.

``` csharp hl_lines="2"
var watson = new Dog { Name = "Watson" };
watson.Name = "Sherlock";
```

Ez így hiba nélkül lefordul.

Kizárólag az inicializációra korlátozhatjuk a setter meghívását az init-only setterrel (`init` kulcsszó).

``` csharp hl_lines="3"
public class Dog
{
    public string Name { get; init; }
    //...
}
```

Ezután az inicializációs sor továbbra is lefordul, de a névátírásos már nem. Ez utóbbi sort kommentezzük ki.

!!! tip "Init-only setter konstruktorból"
    Init-only settert az osztály konstruktorából is meg lehet hívni - hiszen az is inicializáció.

!!! tip "Használata"
    Init-only settert több okból kifolyólag is használhatunk, például a típus példányainak immutábilis kezelését akarjuk kikényszeríteni, vagy csak inicializációra akarjuk korlátozni a propertyk beállítását, de nem akarunk ehhez konstruktort írni.

Jelen formájában az init-only setter nem tudja helyettesíteni a kötelező konstruktor paramétert, mert nem kötelező kitölteni ezt a propertyt.
Erre a megoldás a C# 11-ben bevezetett `required` kulcsszó a property előtt.

``` csharp hl_lines="3"
public class Dog
{
    public required string Name { get; init;  }
    //...
}
```

Ezzel kötelezővé válik a `Name` kitöltése, ha a `Dog` példányt inicializáljuk.

## Indexer operátor, nameof operátor, index inicializáló

A collection initializer analógiájára jött létre az *index initializer* nyelvi elem, ami a korábbihoz hasonlóan sorban hív meg egy operátort, hogy már inicializált objektumot kapjunk vissza. A különbség egyrészt a szintaxis, másrészt az ilyenkor meghívott metódus, ami az index operátor.

!!! tip "operátor felüldefiniálás"
    Saját típusainkban lehetőségünk van definiálni és felüldefiniálni operátorokat, mint pl. +, -, indexelés, implicit cast, explicit cast, stb.

Tegyük fel, hogy egy kutyához bármilyen, üzleti logikában nem felhasznált információ kerülhet, amire általános struktúrát szeretnénk.
Vegyünk fel a `Dog` osztályba egy `string-object` szótárat, amiben bármilyen további információt tárolhatunk!
Ezen felül állítsuk be a `Dog` indexerét, hogy az a `Metadata` indexelését végezze:

``` csharp hl_lines="4-10"
public class Dog
{
    //...
    public Dictionary<string, object>  Metadata { get; } = new();
    public object this[string key]
    {
        get => Metadata[key];
        set => Metadata[key] = value;
    }
}
```

!!! tip "Konstruktor típus nélkül"
    A `new` operátor utáni konstruktorhívás sok esetben elhagyható, ha a bal oldal alapján amúgy is tudható a típus.

!!! tip "névtér hivatkozások"
    Az újabb projektsablonok sokkal kevesebb névtérdeklarációt (`using`) generálnak alapból. Ha kell, vegyük fel a szükségeseket a fel nem oldott néven állva a gyorsművelet (villanykörte) eszközzel (++ctrl+period++)

Az objektum inicializáló és az index inicializáló vegyíthető, így az alábbi módon tudunk felvenni további tulajdonságokat a kutyákhoz a legfelső szintű kódba:

``` csharp
var pimpedli = new Dog
{
    Name = "Pimpedli",
    DateOfBirth = new DateTime(2006, 06, 10),
    ["Chip azonosító"] = "123125AJ"
};
```

Mivel indexelni általában kollekciókat szokás (tömb, lista, szótár), ezért ezekben az esetekben igen jó eszköz lehet az index inicializáló.
Vegyünk fel egy új kutyaszótárt a kutyák kitenyésztése után:

``` csharp
var dogs = new Dictionary<string, Dog>
{
    ["banan"] = banan,
    ["watson"] = watson,
    ["unnamed"] = unnamed,
    ["unknown"] = unknown,
    ["pimpedli"] = pimpedli
};

foreach (var dog in dogs)
{
    Console.WriteLine($"{dog.Key} - {dog.Value}");
}
```

Próbáljuk ki - minden név-kutya párt ki kell írnia a szótárból.

Elsőre jó ötletnek tűnhet kiváltani a szövegliterálokat a `Name` property használatával.

``` csharp
var dogs = new Dictionary<string, Dog>
{
    [banan.Name] = banan,
    [watson.Name] = watson,
    [unnamed.Name] = unnamed,
    [unknown.Name] = unknown,
    [pimpedli.Name] = pimpedli
};
//ArgumentNullException!
```

Ez azonban kivételt okoz, amikor a kutya neve nincs kitöltve, azaz `null` értékű.
Esetünkben elég lenne az adott változó neve szövegként.
Erre jó a `nameof` operátor.

``` csharp
var dogs = new Dictionary<string, Dog>
{
    [nameof(banan)] = banan,
    [nameof(watson)] = watson,
    [nameof(unnamed)] = unnamed,
    [nameof(unknown)] = unknown,
    [nameof(pimpedli)] = pimpedli
};
```

Ez a változat már nem fog kivételt okozni.

A `nameof` operátor sokfajta nyelvi elemet támogat, vissza tudja adni egy változó, egy típus, egy property vagy egy függvény nevét is.

A szótár feltöltését megírhatjuk kollekció inicializációval is.
Ehhez kihasználjuk, hogy a szótár típus rendelkezik egy `Add` metódussal, amelyik egyszerűen egy kulcsot és egy hozzátartozó értéket vár:

``` csharp
var dogs = new Dictionary<string, Dog>
{
    { nameof(banan), banan },
    { nameof(watson), watson },
    { nameof(unnamed), unnamed },
    { nameof(unknown), unknown },
    { nameof(pimpedli), pimpedli }
};
```

## Using static

Ha egy osztály statikus tagjait vagy egy statikus osztályt szeretnénk használni, lehetőségünk van a `using static` kulcsszavakkal az osztályt bevonni a névfeloldási logikába.
Ha a `Console` osztályt referáljuk ilyen módon, lehetőségünk van a rajta levő metódusok meghívására az aktuális kontextusunkban anélkül, hogy az osztály nevét kiírnánk:

``` csharp hl_lines="2 5-7"
using System;
using static System.Console;
//..
foreach (var dog in dogs)
    /*Console.*/WriteLine($"{dog.Key} - {dog.Value}");
/*Console.*/WriteLine(banan);
/*Console.*/ReadLine();
```

!!! note "névfeloldás"
    Az általános névfeloldási szabály továbbra is él: ha egyértelműen feloldható a hivatkozás, akkor nem szükséges kitenni a megkülönböztető előtagot (itt: osztály), különben igen.

## Nullozható típusok

Természetesen a referenciatípusok mind olyan típusok, melyek vehetnek fel `null` értéket, viszont esetenként jó volna, ha a `null` értéket egyébként felvenni nem képes típusok is lehetének ilyen értékűek, ezzel pl. jelezvén, hogy egy érték be van-e állítva vagy sem.
Pl. egy szám esetén a 0 egy konkrét, helyes érték lehet a domain modellünkben, a `null` viszont azt jelenthetné, hogy nem vett fel értéket.

Vizsgáljuk meg, hogy a konzolra történő kiíráskor miért lesz az aktuális év **Watson** kutya életkora!
Valamelyik `Console.WriteLine` sorhoz vegyünk fel egy töréspontot (++f9++), majd debuggolás közben a **Locals** ablakban (debuggolás közben menu:Debug\[Windows \> Locals\]) figyeljük meg az egyes példányok adatait.
Watsont kinyitva láthatjuk, hogy a turpisság abból fakad, hogy a `DateOfBirth` adat típusa, a `DateTime` nem referenciatípus, és alapértelmezés szerinti értéket veszi fel, ami **0001. 01. 01. 00:00:00** - hiszen nem állítottunk be mást.

Ismeretlen születési dátumú, korú egyedek helyes tárolásához az `Age` tulajdonság típusát változtassuk `int?`-re!
Az `int?` szintaktikai édesítőszere a `Nullable<int>`-nek, egy olyan struktúrának, ami egy `int` értéket tárol, és tárolja, hogy az be van-e állítva vagy sem.
A `Nullable<int>` szignatúráit megmutathatjuk, hogyha a kurzort a típusra helyezve ++f12++-t nyomunk.

Módosítsuk a `Dog` `Age` és `DateOfBirth` tulajdonságait is, hogy tudjuk, be vannak-e állítva az értékeik:

``` csharp hl_lines="5-8"
public class Dog
{
    //...

    public DateTime? DateOfBirth { get; set; }
    private int? AgeInDays => (-DateOfBirth?.Subtract(DateTime.Now))?.Days;
    public int? Age => AgeInDays / 365;
    public int? AgeInDogYears => AgeInDays * 7 / 365;

    //...
}
```

!!! tip "Aritmentikai operátorok"
    Örvendezzünk, hogy az alap aritmetikai operátorok pont úgy működnek, ahogy szeretnénk (`null` bemenetre `null` eredmény), nem kellett semmilyen trükk.

Az `AgeInDays` akkor ad vissza `null` értéket, ha a `DateOfBirth` maga is `null` volt.
Tehát ha nincs megadva születési dátumunk, nem tudunk életkort sem számítani.
Ennek kifejezésére használhatjuk a `?.` (Elvis, magyarban **Kozsó** - `null` conditional operator) operátort: a kiértékelendő érték jobb oldalát adja vissza, ha a bal oldal nem `null`, különben `null`-t.
A kifejezést meg kellett változtatnunk, hogy a `DateOfBirth`-ből vonjuk ki a jelenlegi dátumot és ezt negáljuk, ugyanis a `null` vizsgálandó érték a bináris operátor bal oldalán kell, hogy elhelyezkedjen.

??? note "Elvis operátor"
    Az Elvis operátor nevének eredetére több magyarázatot is lehet találni, a források annyiban nagyrészt megegyeznek, hogy a kérdőjel tekeredő része az énekes jellegzetes bodorodó hajviseletére emlékeztet, a pontok pedig a szemeket jelölik, így végülis a ?. egy Elvis emotikonként fogható fel. Ezen logika mentén adódik a magyar megfelelő, a Kozsó operátor, hiszen a szem körül tekergőző legikonikusabb hajtincs a magyar zenei kultúrában [Kozsó](https://hu.wikipedia.org/wiki/Kozso) nevéhez köthető.

Ha így futtatjuk az alkalmazást, az `AgeInDays` és a származtatott tulajdonságok értéke `null` (vagy kiírva üres) lesz, ha a születési dátum nincs megadva.

## Rekord típus

A rekord típusok speciális típusok, melyek:

* egyenlőségvizsgálat során érték típusokra jellemző logikát követnek, azaz két példány akkor egyenlő, ha adataik egyenlőek
* könnyen immutábilissá tehetők, könnyen kezelhetők immutábilis típusként

A `Dog` típus ezzel szemben jelenleg:

* nem immutábilis, hiszen a születési dátum bármikor módosítható (sima setter)
* egyenlőségvizsgálat során a normál referencia szerinti összehasonlítást követ

Az automatikusan generálódó egyedi azonosítót iktassuk ki a `Dog` osztályból, hogy az adat alapú összehasonlítást könnyebben tesztelhessük.

``` csharp
public Guid Id { get; } = Guid.Empty;
```

Vegyünk fel egy logikailag megegyező példányt.

``` csharp hl_lines="2"
var watson = new Dog { Name = "Watson" };
var watson2 = new Dog { Name = watson.Name };
```

Ismét álljunk meg debug során valamelyik `WriteLine` soron.
A **Locals** ablakban nézzük meg, hogy a két példány minden adata megegyezik.
A **Watch** ablakban (debuggolás közben menu:Debug\[Windows \> Watch \> Watch 1\]) értékeljük ki a `watson == watson2` kifejezést.
Láthatjuk, hogy ez az egyenlőségvizsgálat hamist ad, ami technikailag helyes, mert két különböző memóriaterületről van szó, a referenciák nem ugyanoda mutatnak a memóriában.
Sok esetben azonban nem ezt szeretnénk, hanem például a dupla rögzítés elkerülésére az adatok alapján történő összehasonlítást, ami érték típusoknál van.
Referencia típusoknál klasszikusan ezt a `GetHashCode`, `Equals` függvények felüldefiniálásával értük el (vagy az `IComparable<T>`, `IComparer<T>` interfészre épülő logikákkal).
Egy újabb lehetőség a rekord típus használata.

### Pozíció alapú megadás

Vegyünk fel a `Dog` típus adatainak megfelelő rekord típust, mindössze egy kifejezésként. A `Dog` típus alá:

``` csharp
public record class DogRec(
    Guid Id,
    string Name,
    DateTime? DateOfBirth=null,
    Dictionary<string, object> Metadata=null
);
```

!!! note
    A `record class` jelölőből a `class` elhagyható.

Ez az ún. pozíció alapú megadási forma, ami a leginkább rövidített megadási formája a rekord típusnak.
Ebből a rövid formából, mindenfajta extra kód írása nélkül a fordító számos dolgot generál:

* a zárójelen belüli felsorolásból konstruktort és dekonstruktort
* a zárójelen belüli felsorolás alapján propertyket `get` és `init` tagfüggvényekkel
* alapértelmezett logikát az érték szerinti összehasonlításhoz
* klónozó és másoló konstruktor logikákat
* alapértelmezett formázott kiírást, szöveges reprezentációt (`ToString` implementációt)

Így egy könnyen kezelhető, immutábilis, az összehasonlításokban érték típusként viselkedő adatosztályunk lesz.

!!! warning
    Az `Id`-nek nem tudjuk beállítani ebben a formában az alapértelmezett `Guid.Empty` értéket vagy a `Metadata`-nak az új példányt, mert az egyenlőségjeles kifejezésekből alapértelmezett konstruktorparaméter-értékek lesznek, amik csak statikus, fordítási időben kiértékelhető kifejezések lehetnek.

Vegyünk fel a többi Watson példány mellé két újabbat, de itt már az új rekord típusunkat használjuk.

``` csharp
var watson3 = new DogRec(Guid.Empty, "Watson");
var watson4 = new DogRec(Guid.Empty, "Watson");
```

A fentebbi **Watch** ablakos módszerrel ellenőrizzük a `watson3 == watson4` kifejezés értékét.
Ez már igaz érték lesz az adatmező alapú összehasonlítási logika miatt.

Próbáljuk ki ugyanezt a kiértékelést az alábbi változattal:

``` csharp hl_lines="2"
var watson3 = new DogRec(Guid.Empty, "Watson");
var watson4 = new DogRec(Guid.Empty, "Watson", DateTime.Now.AddYears(-1));
```

Ez hamis értéket ad, az egyenlőségnek minden mezőre teljesülnie kell, nem csak a mindkettőben kitöltöttekre.

A `DogRec` típus alapvetően immutábilis, a példányainak alapadatai inicializálás után nem módosíthatók. Próbáljuk felülírni a nevet.

``` csharp hl_lines="3"
var watson3 = new DogRec(Guid.Empty, "Watson");
var watson4 = new DogRec(Guid.Empty, "Watson", DateTime.Now.AddYears(-1));
watson4.Name = watson3.Name + "_2"; //<= nem fordul
```

Nem fog lefordulni, mert minden property init-only típusú. A sor jobboldala egyébként lefordulna, tehát a lekérdezés (getter hívás) működne.

Ha immutábilis típusokkal dolgozunk, akkor mutáció helyett új példányt hozunk létre megváltoztatott adatokkal.
Alapvetően ezt az OO nyelvekben másoló konstruktorral oldjuk meg.
A rekord típusnál ennél is továbbmenve másoló kifejezést használhatunk.

``` csharp hl_lines="2-4"
var watson4 = new DogRec(Guid.Empty, "Watson", DateTime.Now.AddYears(-1));
var watson5 = watson4 with { Name = "Sherlock" };
WriteLine(watson4);
WriteLine(watson5);
```

Futtatáskor a konzolban gyönyörködjünk a rekord típusok alapértelmezetten is olvasható szöveges kiírásában.

A másoló kifejezésben a `with` operátor előtt megadjuk, melyik példányt klónoznánk, majd az **inicializáció részeként** milyen értékeket állítanánk át, ehhez az objektum inicializációs szintaxist használhatjuk.
Fontos eszünkbe vésni, hogy a másolás eredményeként új példány jön létre, új memóriaterület foglalódik le.
Gondoljunk erre akkor, amikor egy ciklusban használjuk ezt a módszert sok egymást követő módosításra.

!!! note "Mire is jó a rekord típus"
    Mire jó a rekord típus, az immutabilitás? Az immutábilis típussokkal való hatékony és eredményes munka másfajta, az imperatív nyelvekhez szokott fejlesztők számára szokatlan módszereket kíván. Vannak területek, ahol ez a befektetés megtérül, ilyen például a többszálú környezet. A legtöbb szálkezeléssel kapcsolatos probléma ugyanis a szálak által közösen használt adatstruktúrák mutációjára vezethető vissza (ún. *race condition*, versenyhelyzet). Nincs mutáció - nincs probléma. (*No mutation - no cry*)

### Kitérő: a szótár visszavág

A rekord típus által biztosított kellemes tulajdonságok csak akkor érvényesek, ha nem keverjük hagyományos referencia típusokkal.

A szokásos módszerrel ellenőrizzük le, hogy a `watson5 == watson6` kifejezés igaz-e. Igen, hiszen minden kitöltött adatuk egyezik.

``` csharp hl_lines="3 6"
var watson4 = new DogRec(Guid.Empty, "Watson", DateTime.Now.AddYears(-1));
var watson5 = watson4 with { Name = "Sherlock" };
var watson6 = watson4 with { Name = "Sherlock" };
WriteLine(watson4);
WriteLine(watson5);
WriteLine(watson6);
```

Vigyünk be egy ártatlan inicializációt a `Metadata` propertyre.

``` csharp hl_lines="2-3"
var watson4 = new DogRec(Guid.Empty, "Watson", DateTime.Now.AddYears(-1));
var watson5 = watson4 with { Name = "Sherlock", Metadata = new Dictionary<string, object>() };
var watson6 = watson4 with { Name = "Sherlock", Metadata= new Dictionary<string, object>() };
WriteLine(watson4);
WriteLine(watson5);
WriteLine(watson6);
```

Ezzel eléggé illogikus módon hamisra változik a `watson5 == watson6` kifejezés.
Az oka az, hogy a `Metadata` szótár egy klasszikus referencia típus, az összehasonlításnál a klasszikus memóriacím-összehasonlítás történik, viszont az a két új szótár példány esetében eltérő lesz.
A formázott szöveges kiírásba is belerondít a szótár, mert ott is a szótár típus alapértelmezett szöveges reprezentációja jut érvényre, ami a típus neve.

Klónozzunk tovább, aztán próbáljunk mutációt végrehajtani a `Metadata` szótáron.

``` csharp hl_lines="2-3"
var watson6 = watson4 with { Name = "Sherlock", Metadata = new Dictionary<string, object>() };
var watson7 = watson6 with { Name = "Watson" };
watson7.Metadata.Add("Chip azonosító", "12345QQ");
WriteLine(watson4);
```

Ez lefordul, pedig ez mutáció.
A **Locals** ablakban figyeljük meg a `watson6` és `watson7` szótárait: **mindkettőbe** bekerült a chip azonosító.
Ez az ún. *shallow copy* jelenség, amikor nem a szótár memóriaterülete klónozódik, csak a rá mutató referencia, ami azt eredményezi, hogy a két példánynak közös szótára lesz.

Összességében az adatstruktúránkban megjelenő klasszikus referencia típus elrontja:

* az immutabilitást
* az érték szerinti összehasonlítást
* a formázott szöveges megjelenést
* a klónozást

!!! warning "Immutabilitás"
    Immutábilis környezetben törekedjünk arra, hogy a **teljes** adatstruktúránk támogassa az immutábilis kezelést.

### Normál megadás

Ha nincs szükségünk a kikényszerített immutabilitásra, akkor használhatjuk a rekord normál megadását.
Fogjuk a `Dog` osztályt, másoljuk le a kódját, adjunk neki más nevet és `class` helyett `record` jelölőt.

A `Dog` osztály fölé:

``` csharp
public record DogRecExt
{
    public string Name { get; init; }
    public Guid Id { get; } = Guid.Empty;
    public DateTime? DateOfBirth { get; set; }
    public Dictionary<string, object> Metadata { get; } = new();

    private int? AgeInDays => (-DateOfBirth?.Subtract(DateTime.Now))?.Days;
    public int? Age => AgeInDays / 365;
    public int? AgeInDogYears => AgeInDays * 7 / 365;

    public object this[string key]
    {
        get { return Metadata[key]; }
        set { Metadata[key] = value; }
    }
}
```

!!! note "ToString"
    A `ToString` implementációját elhagytuk az előző szakaszban említettek miatt.

A `Program.cs`-be:

``` csharp
var watson8 = new DogRecExt { Name = "Watson" };
watson8.DateOfBirth = DateTime.Now.AddYears(-15);
var watson9 = watson8 with { };
WriteLine(watson8);
WriteLine(watson9);
```

Ellenőrizzük le a rekord tulajdonságokat:

* A konzol kimeneten a formázást, továbbá a mutáció működését, azaz a `watson8` születési dátuma a beállított lesz.
* Ez nem csoda, hiszen a property deklarációban engedtük a mutációt.
* A konzol kimeneten megfigyelt példányadatokon a klónozó kifejezés működését. Semmi különös, ugyanúgy működik, mint a tömör formánál.
* A **Watch** ablakban `watson8 == watson9` egyenlőséget. Ez igaz, mert minden adattagjuk egyezik.

!!! tip "record struct"
    A rekordoknak további válfajai vannak, ugyanis struktúra is lehet rekord, ilyenkor a `record struct` kulcsszó párt használjuk a típus deklarációjánál. Sőt, a `readonly record struct` egy immutábilis `record struct`. Ezen válfajok nyilván különbözőképpen viselkednek, mely viselkedéseket itt most nem részletezzük, de a [dokumentációban](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record) megtalálhatók.
