---
authors: kszicsillag,tibitoth
---

# LINQ

## Előkészítés

A gyakorlat kezdetén klónozzuk le a [kiinduló projektet](https://github.com/bmeviauav23/Linq-lab) az alábbi paranccsal:

```cmd
git clone https://github.com/bmeviauav23/Linq-lab.git
```

Nyissuk meg Visual Studio-ban a *HelloLinq.sln* solution fájlt.

Megnyitás után tekintsük át a kiinduló projektben levő fájlokat:

* **Program.cs**: a legfelső szintű kódot tartalmazó osztály. Található benne egy `Dogs` változó, ami a `Dog` osztály statikus `Repository` tulajdonságába hív át.
* **Dog.cs**: a korábbi gyakorlatokon használt adatmodell (apróbb módosításokkal).
    * Bekerült egy `Siblings` tulajdonság, a `ToString` pedig kiírja a kutyához tartozó testvérek számát is (ehhez a `TrimPad` bővítő metódust használja).
    * A statikus `Repository` tulajdonság mögött egy lustán inicializált `Lazy<T> RepositoryHolder` található, ami egy megfelelően formázott bemeneti CSV fájlból elkészíti számunkra az adatmodellt, amivel a későbbiekben dolgozunk. Ennek implementációját elég a gyakorlat végén megnézni. Az `Import` és `Export` függvények a kutyák sorosítását végzik el mindkét irányban.
* **Extensions/StringExtensions.cs**: ez az osztály tartalmaz egy segédmetódust a formázott kiíráshoz. A `Dog` `ToString` metódusa használja fel. A bővítő metódusos részben lesz jelentősége.
* **dogs.csv**: egy pontosvesszővel tagolt adathalmaz, amelyben 100 darab előre felvett kutya adata található. Innen puskázhatunk, ha ellenőrizni akarjuk, hogy helyesek-e a programunk eredményei.

A kiinduló projektben a globális implicit névtérhivatkozások ki vannak kapcsolva. A **csproj** fájlban megnézhetjük (menu:jobb klikk a projekten\[Edit Project File\]):

``` xml
<ImplicitUsings>disable</ImplicitUsings>
```

## Lambda kifejezések, delegátok

Gyakori feladat, hogy objektumok kollekciójával kell dolgoznunk.
Képesek vagyunk olyan jellegű segédfüggvényeket készíteni, amik például egy kollekcióban kikeresik az összes olyan elemet, amely egy megadott feltételnek eleget tesz.

A `Program.cs` fájlban látható ennek a kezdeti naiv változata, szemrevételezzük:

``` csharp
static List<Dog> ListDogsByNamePrefix(IEnumerable<Dog> dogs, string prefix)
{
    var result = new List<Dog>();
    foreach (var dog in dogs)
    {
        if (dog.Name.StartsWith(prefix, StringComparison.OrdinalIgnoreCase))
        {
            result.Add(dog);
        }
    }
    return result;
}
```

Próbáljuk ki!

A kód működik, viszont nem újrahasznosítható. 
Ha bármi más alapján szeretnénk keresni a kutyák között (pl. a neve tartalmaz-e egy adott szövegrészt), mindig egy új segédfüggvényt kell készítenünk, ami rontja a kód újrahasznosíthatóságát.

Oldjuk meg úgy, hogy az általános problémát is megoldjuk!
Ehhez az szükséges, hogy a kollekciónk egyes elemein kiértékelhessünk egy, a hívó által megadott predikátumot.
Készítsük el az általánosabb változatot, ehhez felhasználhatjuk a `ListDogsByNamePrefix` kódját.

``` csharp hl_lines="2 7"
static List<Dog> ListDogsByPredicate(
    IEnumerable<Dog> dogs, Predicate<Dog> predicate)
{
    var result = new List<Dog>();
    foreach (var dog in dogs)
    {
        if (predicate(dog))
        {
            result.Add(dog);
        }
    }
    return result;
}
```

A legfelső szintű kódban így hívhatjuk meg (felhasználhatjuk az eredeti ciklust):

``` csharp
foreach(var dog in ListDogsByPredicate(Dogs, delegate (Dog d) 
    {
        return d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
    })
)
{
    Console.WriteLine(dog);
}
```

Egy egy bemenő paraméterű és egy logikai (`bool`) értéket visszadó függvényt definiálunk helyben (*inline*) és ezt (illetve a referenciáját) adjuk át.
Használjunk inkább lambda kifejezést, az jóval rövidebben leírható - egyelőre csak nézzük meg, de ne integráljuk a kódba:

``` csharp
d => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
```

!!! tip "Lambda kifejezések szintaktikája"
    Lambda kifejezéssel az egyetlen kifejezésből álló függvényeket adhatjuk meg nagyon kompakt módon. A `=>`-tól balra elnevezzük a bemenő paramétereket, jobbra pedig felhasznál(hat)juk. A `return`, `{}` és egyéb sallangokat elhagyhatjuk.

Vessük össze, hogy az első esetben explicit megadtuk, hogy a bemenő paraméterünk `Dog`, most viszont nem.
Ezt a fordító statikus kódanalízis alapján el tudja dönteni: a `d` változónk nem lehet más, csak `Dog` (statikus) típusú, ezért csak így használhatjuk, viszont nem kell kiírnunk a típust.

A lambda kifejezések egy lehetséges módja a delegátok leírásának. A delegát kódot reprezentál, viszont a kódot kezelhetjük adatként is.

Próbáljuk meg a delegátunkat kivenni egy implicit típusú változóba a ciklus előtt:

``` csharp hl_lines="1-4"
// fordítási hiba!
var predicate = 
    d => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
foreach (var dog in ListDogsByPredicate(Dogs, predicate))
{
    Console.WriteLine(dog);
}
```

Fordítási hibát kapunk, lambda kifejezés típusa nem lehet implicit eldönthető az inicializációs sorban: sem a bemenő paraméter pontos típusát nem tudjuk (`Dog`? `Puppy`?), sem a visszatérési értéket (`bool`? `object`? `void`?). Tehát explicit meg kell adnunk a típust:

``` csharp hl_lines="1"
Predicate<Dog> predicate =
    d => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
```

Ezután fordul és fut is az alkalmazásunk.

!!! tip "Predicate beépített deleagate típus"
    Ehhez tudnunk kellett, hogy a `Predicate<T>` megfelelő szignatúrájú. Mutassuk meg ezen típus dokumentációját vagy tegyük a kurzort a típusra és nyomjunk ++f12++-t.

## `Func<>`, `Action<>`

Ismerkedjünk meg a `Func` és `Action` általános delegáttípusokkal.
Ezzel a két generikus típussal (pontosabban a változataikkal) gyakorlatilag az összes gyakorlatban előforduló függvényszignatúrát le lehet fedni.
Például a fenti szűrőlogikát is átírhatnánk erre:

``` csharp
Func<Dog, bool> dogFunc =
    d => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
```

A `dogFunc` és a `predicate` kompatibilisnek tűnhetnek (elvégre a jobboldaluk ugyanaz), ám ha lecserélnénk pl. a `ListDogsByPredicate(Dogs, predicate)` hívásban a `predicate`-et `dogFunc`-ra, a kód nem fordulna, ugyanis a két delegáttípus nem kompatibilis.

Az `Action<>` hasonló elven működik, visszatérési érték nélküli függvényekre.

!!! note "Egyéb delegate típusok"
    Ha minden esetre jók, miért vannak használatban `Action<>` és `Func<>`-on kívül más delegáttípusok?
    Egyrészt történelmi okok miatt. Később jelentek meg, mint a specifikusak, például a `Predicate<T>`. Másrészt a specifikusabbak a nevükkel kifejezőbbek lehetnek.

!!! warning "Tiszta függvények"
    A fenti predikátumváltozataink mind nem tiszta függvények (pure function), ugyanis olyan adattól is függ a visszatérési értéke, ami nem szerepel a paraméterlistáján - ez esetünkben a `searchText` változó.
    A kódunk azért működik, mert a delegát megadásakor a `searchText` aktuális értékét [elkapjuk (capture)](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions), belerakjuk a függvénylogikába.

Próbáljuk a `dogFunc`-ot `var`-ként deklarálni.

``` csharp
//Fordítási hiba!
var dogFunc =
    d => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
```

A fordító nem tudja meghatározni a `d` paraméter típusát, ezért kapjuk a fordítási hibát. Adjuk meg explicit a paraméter típusát.

``` csharp
var dogFunc =
    (Dog d) => d.Name.StartsWith(searchText, StringComparison.OrdinalIgnoreCase);
```

Debugger-rel ellenőrizhetjük, hogy a `dogFunc` valódi típusa `Func<Dog, bool>` lesz.

## IEnumerable\<T\> bővítő metódusok

Vigyük tovább az általánosítást.
Írjunk olyan logikákat, mely nem csak kutyák listájára, hanem bármilyen felsorolható (enumerálható) kollekcióra működik.
Írjunk `IEnumerable<T>` típuson működő segédfüggvényeket.

Hozzunk létre egy `EnumerableExtensions` (I betű nélkül, az ugyanis interfészre utal) nevű fájlt az `Extensions` mappában!
Elsőként valósítsuk meg az összegző logikát.

``` csharp
namespace HelloLinq.Extensions.Enumerable;

public static class EnumerableExtensions
{
    public static int Sum<T> (IEnumerable<T>  source, Func<T, int>  sumSelector)
    {
        var result = 0;
        foreach (var elem in source)
        {
            result += sumSelector(elem);
        }
        return result;
    }
}
```

Hívjuk meg a legfelső szintű kódból.

``` csharp hl_lines="1 4-11"
using HelloLinq.Extensions.Enumerable;

IEnumerable<Dog> Dogs = Dog.Repository.Values;
foreach (var dog in Dogs)
{
    Console.WriteLine(dog);
}

Console.WriteLine(
    "Életkorok összege: " 
    + $"{EnumerableExtensions.Sum(Dogs, d => d.Age ?? 0)}");

string searchText;
```

A segédfüggvények hátránya, hogy ismernünk kell a segédosztály nevét.
Továbbá jobb lenne, ha a kollekción közvetlenül hívhatnánk az összegző függvényt.
Erre megoldás a bővítő metódus.

A bővítő metódusok:

* statikus osztályban definiálhatók
* statikus függvények
* első paramétere előtt `this` jelöli, hogy melyik típust bővítik

Az első paraméter elé tegyük be a `this` jelölőt.

``` csharp hl_lines="2"
public static int Sum<T> (
    this IEnumerable<T>  source,
    Func<T, int>  sumSelector)
    {
        // ...
    }
```

Most már használhatjuk azt a szintaxist, mintha a kollekciónak eleve lenne összegző függvénye:

``` csharp
Console.WriteLine($"Életkorok összege: {Dogs.Sum(d => d.Age ?? 0)}");
```

!!! warning "OO egységbezárási elv"
    A bővítő metódusok semmilyen módon nem bontják meg a típusok egységbezárási képességeit. A függvények implementációi a bővítendő típusok kívülről is elérhető függvényeit, propertyjeit használhatják, privát adattagokhoz, függvényekhez nem férnek hozzá.

!!! warning "Osztály nevének feloldása"
    A bővítő metódusok alkalmazásakor nagyon fontos, hogy bár a bővítő metódus osztályának nevét nem írjuk ki, az osztály nevének feloldhatónak kell lennie, azaz az osztály névterét `using` direktívával be kell hivatkoznunk.
    Egy próba erejéig kommentezzük ki a `using HelloLinq.Extensions.Enumerable;` sort és ellenőrizzük, hogy nem fordul a kódunk, a bővítő metódus nevét a fordító nem tudja feloldani.

Gyakorlásképpen írhatunk további gyakori adatfeldolgozási műveletekre függvényeket, mint amilyen az átlagszámítás, min-max keresés.

??? success "Megoldás"

    ``` csharp
    public static class EnumerableExtensions
    {
        //...

        public static double Average<T>(this IEnumerable<T> source, Func<T, int> sumSelector)
        {
            var result = 0.0; // Az osztás művelet miatt double
            var elements = 0;
            foreach (var elem in source)
            {
                elements++;
                result += sumSelector(elem);
            }
            return result / elements;
        }

        public static int Min<T>(this IEnumerable<T> source, Func<T, int> valueSelector)
        {
            int value = int.MaxValue;
            foreach (var elem in source)
            {
                var currentValue = valueSelector(elem);
                if (currentValue < value)
                {
                    value = currentValue;
                }
            }
            return value;
        }

        public static int Max<T> (this IEnumerable<T> source, Func<T, int> valueSelector)
            => -source.Min(e => -valueSelector(e));
    }
    ```

Próbáljuk ki az új függvényeket. Mivel a `Dogs` típusa `IEnumerable<Dog>`, így a bővítő metódusok bővítendő típusa illeszkedik rá.

``` csharp
Console.WriteLine($"Átlagos életkor: {Dogs.Average(d => d.Age ?? 0)}");
Console.WriteLine(
    $"Minimum-maximum életkor: " +
    $"{Dogs.Min(d => d.Age ?? 0)} | {Dogs.Max(d => d.Age ?? 0)}");
```

!!! note "StringExtensions.cs"
    A `StringExtensions` osztályban egy lambdaként megvalósított bővítő metódust láthatunk, ami egy szöveget adott hosszra (szélességre) egészít ki szóközökkel. A függvényt a `Dog` `ToString` metódusa használja fel.

## Gyakori lekérdező műveletek, yield return

Gyakran előfordul, hogy egy listát szűrni vagy projektálni szeretnénk. Írjunk saját generátort ezekhez a műveletekhez is az `EnumerableExtensions`-be:

``` csharp
public static IEnumerable<T> Where<T>(
    this IEnumerable<T>  source, Predicate<T>  predicate)
{
    foreach (var elem in source)
    {
        if (predicate(elem))
        {
            yield return elem;
        }
    }
}

public static IEnumerable<TValue> Select<T, TValue>(
    this IEnumerable<T>  source, Func<T, TValue> selector)
{
    foreach (var elem in source)
    {
        yield return selector(elem);
    }
}
```

Próbáljuk ki a legfelső szintű kód elején, válasszuk ki a 2010 előtt született kutyák nevét és korát egy stringbe:

``` csharp
foreach (var text in Dogs
    .Where(d => d.DateOfBirth?.Year < 2010)
    .Select(d => $"{d.Name} ({d.Age})"))
{
    Console.WriteLine(text);
}
```

!!! tip "A yield return haszna"
    A `yield return` egy hasznos eszköz, ha IEnumerable-t kell produkálnunk visszatérési értékként. Segítségével mindig csak akkor állítjuk elő a következő elemet, amikor a hívó kéri. A működését debuggerrel is figyeljük meg: tegyünk breakpointot a két `yield return` sorra, majd ++f10++-zel kövessük végig, ahogy a `foreach` elkéri a `Select`-től a következő elemet, ami emiatt elkéri a `Where`-től, majd újraindul a ciklus. A hívások állapotgépként működnek, a következő meghíváskor onnan folytatódnak, ahonnan az előző `yield return`-nél kiléptünk.

Nem nagy meglepetés, hogy az általunk megírt `Sum`, `Average` (melyek egyedi visszatérésűek), `Select` és `Where` (amik szekvenciális visszatérésűek, generátorok) metódusok mind a .NET keretrendszer részét képezik (a [`System.Linq.Enumerable`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable) statikus osztályban definiált bővítő metódusok). A **LINQ** — **L**anguage **IN**tegrated **Q**uery — ezeket a műveleteket teszi lehetővé `IEnumerable` interfészt megvalósító objektumokon. A LINQ függvények bővítő metódusként lettek hozzáadva meglevő funkcionalitáshoz (kollekciókhoz, lekérdezésekhez), sőt, külső library-k is adnak saját LINQ bővítő metódusokat.

Cseréljük le a `Program.cs`-ben a `using HelloLinq.Extensions.Enumerable` hivatkozást `using System.Linq`-re: az általunk megírt kód továbbra is ugyanazt az eredményt produkálja! Nézzük meg, hogy hol vannak definiálva ezek a függvények a keretrendszeren belül: a kurzort tegyük a kódban oda, ahol valamelyik korábban megírt függvényünket hívnánk, majd nyomjunk ++f12++-t. Próbáljuk ki, hogy továbbra is az elvárt módon működik-e a programunk.

!!! tip "Implicit usings"
    A névtércsere helyett bekapcsolhatjuk a globális implicit névtér funkciót, mert a `System.Linq` névtér is egy implicit hivatkozott névtér. Ehhez a projektfájlban az `<ImplicitUsings>disable</ImplicitUsings>` beállítást írjuk át `enable`-re, majd a `using HelloLinq` -en kívül minden névtérhivatkozást töröljünk a `Program.cs`-ből.

## Anonim típusok

Lekérdezéseknél gyakran használatosak az anonim típusok, amelyeket jellemzően lekérdezések eredményének ideiglenes, típusos tárolására használunk.
Az anonim típusokkal lehetőségünk van *inline* definiálni olyan osztályokat, amelyek jellemzően csak dobozolásra és adattovábbításra használtak.
Vegyük az alábbi példákat a legfelső szintű kód elején:

``` csharp
var dolog1 = new { Name = "Alma", Weight = 100, Size = 10 };
var dolog2 = new { Name = "Körte", Weight = 90 };
```

Korábban már említettük a `var` kulcsszót, amellyel implicit típusú, lokális változók definiálhatók.
Az értékadás jobb oldalán definiálunk egy-egy anonim típust, amelynek felveszünk néhány tulajdonságot.
A tulajdonságok mind típusosak maradnak, a típusrendszerünk továbbra is sértetlen.
Az implicit statikus típusosság nem csak a `var` kulcsszóban jelenik meg tehát, hanem az egyes tulajdonságok típusában is.

Az anonim típusok:

* csak referencia típusúak lehetnek (objektumok, nem pedig struktúrák),
* csak publikusan látható, csak olvasható tulajdonságokat tartalmazhatnak,
* eseményeket és metódusokat nem tartalmazhatnak (delegate példányokat tulajdonságban viszont igen),
* szerelvényen belül láthatók (`internal`) és nem származhat belőlük másik típus (`sealed`)
* típusnevét nem ismerjük, így hivatkozni sem tudunk rá, csak a `var`-t tudjuk használni
* nem használhatók ott, ahol a `var` típus se használható, többek között nem adhatjuk át függvénynek és nem lehet visszatérési érték sem

Ha az egeret a `var` kulcsszavak vagy egyes tulajdonságnevek fölé visszük, láthatjuk, hogy valóban fordítási idejű típusokról van szó.

!!! tip "IntelliSense"
    Figyeljük meg, hogy az **IntelliSense** is működik ezekre a típusokra, felkínálja a típus property-jeit.

A fordító újra is hasznosítja az egyes típusokat:

``` csharp
var dolgok = new { Name = "Gyümölcsök", Contents = new[] { dolog1, dolog2 } };
```

A `Contents` tulajdonság típusa a fenti anonim objektumaink tömbje, ezért nem is adhatnánk meg másképpen (nem tudjuk a nevét, amivel hivatkozhatunk rá).
A fordító most panaszkodik, ugyanis a két dolog típusa nem implicit következtethető.
Ha felvesszük a `Size` tulajdonságot a `dolog2` definíciójába, máris fordul.

``` csharp
var dolog2 = new { Name = "Körte", Weight = 90, Size = 12 };
```

Ha végeztünk az anonim típusokkal való ismerkedéssel, az ezekkel kapcsolatos kódsorokat kikommentezhetjük.

## LINQ szintaxisok

Az előző részben ismertetett jellegű lekérdezések nagyban hasonlítanak azokhoz, amiket adatbázis-lekérdezésekben alkalmazunk.
A különbség itt az, hogy imperatív szintaxist használunk, szemben pl. az SQL-lel, ami deklaratívat.
Ezért is van jelen a C# nyelvben az ún. *query syntax*, amely jóval hasonlatosabb az SQL szintaxisához, így az adatbázisokban jártas fejlesztők is könnyebben írhatnak lekérdezéseket.
Ugyanakkor nem minden lekérdezést tudunk query syntax-szal leírni.

!!! note "Miért nem lehet mindent megírni query syntaxban?"
    Ennek oka, hogy az operátorok bevezetése egy nyelvben elég drága - le kell péládul foglalni az operátor nevét, amit utána korlátozottan lehet csak használni másra. Ezért sem csinálták meg minden LINQ függvénynek az operátor párját, csak az SQL-ben gyakrabban használatosabbaknak.

Az előzőhöz hasonló lekérdezést megírhatunk az alábbi módon query syntax használatával:

``` csharp
using HelloLinq.Extensions;

//...
IEnumerable<Dog> Dogs = Dog.Repository.Values;

var query = from d in Dogs
            where d.DateOfBirth?.Year < 2010
            select new
            {
                Dog = d,
                AverageSiblingAge = d.Siblings.Average(s => s.Age ?? 0)
            };
foreach (var meta in query)
{
    Console.WriteLine(
        $"{meta.Dog.Name} - {meta.AverageSiblingAge}");
}
```

A query szintaxis végül a korábban is használt, ún. *fluent szintaxis*sá fordul. Ennek igazolására nézzük meg ++f12++-vel, hogy hol vannak definiálva az újonnan megismert operátorok (`select`, `where`).
A két szintaxist szokás ötvözni is, jellemzően akkor, ha query szintaxisban írjuk a lekérdezést, és a hiányzó funkcionalitást fluent szintaxissal pótoljuk.

!!! note "Fluent szintaxis"
    A fluent szintaxist olyan kialakítású API-knál alkalmazhatjuk, ahol a függvények a tartalmazó típust várják (egyik) bemenetként és azonos (vagy leszármazott) típust adnak vissza. A LINQ-nél ez a típus az `IEnumerable<>`.

Ezen az órán memóriabeli adatforrásokkal dolgoztunk (konkrétan a `Dogs` nevű `Dictionary<,>` típusú változóval), a LINQ operátorok közül a memóriabeli listákon dolgozókat használtuk, melyeket az `IEnumerable<>` interfészre biggyesztettek rá bővítő metódusként.
Ezt a LINQ API-t teljes nevén *LINQ-to-Objects*nek hívják, de gyakran csak LINQ-ként hivatkozzák.

## Kitekintő: Expression\<\>, LINQ providerek

Vegyük az alábbi nagyon egyszerű delegate-et és ennek `Expression<>`-ös párját.

``` csharp
Func<int, int>  f = x => x + 1;
Expression<Func<int, int>> e = x => x + 1;
```

Nézzük meg debuggolás közben a **Watch** ablakban a fenti két változót. Az `f` egy delegate, lefordított *kód*ra mutató referencia, az `Expression` a jobb oldali kifejezésből épített (fa struktúrájú) *adat*.

A fát kóddá fordíthatjuk a `Compile` metódus segítségével, mely a lefordított függvény referenciáját (delegát példány) adja vissza, amit a függvényhívás szintaxissal hívhatunk meg. Ebből áll össze az alábbi fura kinézetű kifejezés:

``` csharp
Console.WriteLine(e.Compile()(5));
```

Bár az `Expression<>` emiatt okosabb választásnak tűnik, ám a LINQ-to-Objects alapinterfészének (ami a lekérdezőfüggvényeket biztosítja) függvényei `Func<>` / `Action<>` delegátokat várnak. Ami nem csoda, hiszen memóriabeli listákat általában sima programkóddal dolgozunk fel, nincs értelme felépíteni kifejezésfát csak azért, hogy utána egyből kóddá fordítsuk. Emellett más, memóriabeli adatokon dolgozó LINQ technológia is létezik, pl. LINQ-to-XML saját API-val (nem `IEnumerable<>` alaptípussal).

A nem memóriabeli adatokon, hanem például külső adatbázisból dolgozó LINQ provider-ek viszont `IQueryable<>`-t valósítanak meg. Az `IQueryable<>` az `IEnumerable<>`-ból származik, így neki is vannak `Func<>` / `Action<>`-ös függvényei, de emellett `Expression<>`-ösek is. Ez utóbbiak teszik lehetővé, hogy ne csak .NET kódot generáljanak a lambda kifejezésekből, hanem helyette pl. SQL kifejezést - hiszen egy relációs adatbázis adatfeldolgozó nyelve nem .NET, hanem valamilyen SQL dialektus.

### A LINQ providerek általános működése

Bemenetük: query függvényeknek (`IQueryable<>` vagy `IEnumerable<>` függvényei vagy pl. `XDocument`) paraméterül adott lambdák (`Func<>` vagy `Expression<>`)

Kimenetük: az adatforrásnak megfelelő nyelvű, a query-t végrehajtó kód (.NET kód vagy SQL).

LINQ-to-Objects esetén nincs valódi LINQ provider (a provider az `IQueryable.Provider`-en keresztül érhető el, de a `List<>` nem `IQueryable`!), hiszen nincs feladata: kódot kap bemenetül, ugyanazt kellene kimenetül adnia. A *LINQ-to-XML* is hasonló elven működik.

Valódi LINQ providert valósít meg például az *Entity Framework*, de ezt a technológiát később tárgyaljuk.
