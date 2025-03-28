# ASP.NET Core alapszolgáltatások

## Projekt létrehozása

Ezen a gyakorlaton nem a beépített API projektsablont fogjuk felhasználni, hanem egy üres ASP.NET Core projektből próbáljuk felépíteni és megérteni azt a funkcionalitást, amit egyébként az előre elkészített VS projektsablonok adnának készen a kezünkbe.

### Generálás

Hozzunk létre a Visual Studioban egy új, C# nyelvű projektet az *ASP.NET Core Empty* sablonnal, a neve legyen *HelloAspNetCore*.
Megcélzott keretrendszerként adjuk meg a *.NET 8*-at.
Minden extra opció legyen kikapcsolva, a docker és a HTTPS is (a laborgépek miatt).

??? note "Kitérő: NuGet és a keretrendszert alkotó komponensek helye"

    A .NET 8 és az ASP.NET Core gyakorlatilag teljes mértékben publikusan elérhető komponensekből épül fel.
    A komponensek kezelésének infrastruktúráját a NuGet csomagkezelő szolgáltatja.
    A csomagkezelőn keresztül elérhető csomagokat a [nuget.org](https://www.nuget.org/) listázza és igény esetén a NuGet kliens, illetve a .NET Core eszközök (dotnet.exe, Visual Studio) is innen töltik le.
    
    A fejlesztőknek teljesítményszempontból nem érné meg az alap keretrendszert alkotó csomagokat állandóan letöltögetni, így a klasszikus keretrendszerekhez hasonlóan a .NET 8 telepítésekor egy könyvtárba (Windows-on ide: **C:\Program Files (x86)\dotnet**, illetve **C:\Program Files\dotnet**) bekerülnek az alap keretrendszert alkotó komponensek - lényegében egy csomó .dll különböző alkönyvtárakban.
    
    A futtatáshoz szükséges szerelvények a **shared** alkönyvtárba települnek, ezek az ún. **Shared Framework**-ök.
    A gépen futó különböző .NET Core/8 alkalmazások közösen használhatják ezeket.
    
    A fejlesztéshez az alapvető függőségeket a **packs** alkönyvtárból hivatkozhatjuk.

    Nem fejlesztői, például végfelhasználói vagy szerver környezetben- ahol nem is biztos, hogy fel van telepítve az SDK, nem feltétlenül így biztosítjuk a függőségeket, de ennek a boncolgatása nem témája ennek a gyakorlatnak.

### Eredmény

Nézzük meg, milyen projekt generálódott:

* **.csproj**: (menu:Projekten jobb gomb\[Edit Project File\]) a projekt fordításához szükséges beállításokat tartalmazza. Előző verziókhoz képest itt erősen építenek az alapértelmezett értékekre, hogy minél karcsúbbra tudják fogni ezt az állományt.
    * **Project SDK**: projekt típusa ([Microsoft.NET.Sdk.Web](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/web-sdk)), az eszközkészlet funkcióit szabályozza, meghatározza a futtatáshoz használatos shared framework-öt, illetve meghatározza a megcélzott keretrendszert is(lásd lentebb).
    * **TargetFramework**: *net8.0*. Ezzel jelezzük, hogy .NET 8-os API-kat használunk az alkalmazásban.
* **Connected Services**: külső szolgáltatások, amiket használ a projektünk, most nincs ilyenünk.
* **Dependencies**: a keretrendszer alapfüggőségei és egyéb NuGet csomagfüggőségek szerepelnek itt. Egyelőre csak keretrendszer függőségeink vannak.
    * **Frameworks**: két alkönyvtárat (Microsoft.AspNetCore.App, Microsoft.NETCore.App) hivatkozunk a .NET SDK **packs** alkönyvtárából. Ezek a függőségek külső NuGet csomagként is elérhetőek, de ahogy fentebb jeleztük, nem érdemes úgy hivatkozni őket.
    * **Analyzers**: speciális komponensek, amik kódanalízist végzenek, de egyébként ugyanúgy külső függőségként (NuGet csomag) kezelhetjük őket. Ha kibontjuk az egyes analizátorokat, akkor láthatjuk, hogy miket ellenőriznek. Ezek a függőségek a futáshoz nem szükségesek.
* **Properties**: duplakattra előjön a klasszikus projektbeállító felület.
    - **launchSettings.json:** a különböző indítási konfigurációkhoz tartozó beállítások (lásd később).
* **appsettings.json**: futásidejű beállítások helye. Kibontható, kibontva a különböző környezetekre specifikus konfigurációk találhatóak (lásd később).

#### Legfelső szintű kód, minimál API

Az előző ASP.NET verzióval ellentétben, itt már az ASP.NET Core alkalmazások a születésüktől fogva klasszikus konzolos alkalmazásként is indíthatók, ekkor az alkalmazás alapértelmezett belépési pontja a legfelső szintű kód (esetleg a `Main` metódus).
Az ASP.NET Core 6-os verzióban megjelent ún. **minimál API** segítségével már nem csak a konfigurációt tartalmazhatja ez a kód, hanem (egyszerű) kiszolgáló logikát is.

Esetünkben a következő lépéseket végzi el a generált kód:

* a hosztolási környezetet és az alkalmazás alapszolgáltatásait konfiguráló **builder** objektum összeállítása (`CreateBuilder` függvényhívás)
* a **builder** objektum alapján a hosztolási környezet és az alkalmazás szerkezetének felállítása (`Build` függvényhívás)
* végpontot definiál az alkalmazás gyökércímére minimál API segítségével. A végpont a meghívására a **Hello World!** szöveget adja vissza.
* a felállított szerkezet futtatása (`Run` függvényhívás)

Az igazán munkás feladat a builder megalkotása lenne, igen sok mindent lehetne benne konfigurálni, ez a kódban a `CreateBuilder`-ben történik, ami egy szokványos, az egész webalkalmazás működési környezetét meghatározó beállításokat elvégző kiinduló buildert állít elő.
Ha valamit a kiinduló builderben megadottól eltérően szeretnénk, vagy új beállításokat adnánk meg, akkor a kiinduló builder objektumon történő függvényhívásokkal tehetnénk meg.

Mivel a kiinduló builderen nem végzünk semmilyen utólagos konfigurálást, így akár egy utasítással is megkaphatnánk az alkalmazásszerkezetet reprezentáló `WebApplication` példányt.

``` csharp
//var builder = WebApplication.CreateBuilder(args);
//var app = builder.Build();

var app = WebApplication.Create();
```

## Végrehajtási pipeline, middleware-ek

Az ASP.NET Core-ban egy kérés kiszolgálása úgy történik, hogy a kérés egy csővezetéken halad (végig).
A csővezeték middleware-ekből (MW) áll. Az alábbi ábra szemlélteti a middleware pipeline működését.

<figure markdown>
![ASP.NET Core pipeline](images/aspnetcore1-pipeline.png)
<figurecaption>ASP.NET Core pipeline [Forrás](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware)</figurecaption>
</figure>

Az ASP.NET Core alkalmazás alapszerkezete, hogy a befutó HTTP kérés (végig)fusson a middleware-ekből álló csővezetéken és valamelyik (alapesetben az utolsó) middleware előállítja a választ, ami visszairányban halad végig a csővezetéken.
A csővezeték adja tehát az alkalmazás szerkezetét.
A kiinduló csővezetéket a `WebApplication.Create` vagy a `builder.Build` építi fel, ezt utána `app.UseX` (X= MW neve) hívásokkal testreszabhatjuk, kiegészíthetjük.

Esetünkben a kiinduló csővezetékben három MW van:

* kivételkezelő middleware (`UseDeveloperExceptionPage`), ami az őt követő middleware-ek hibáit képes elkapni és ennek megfelelően egy a fejlesztőknek szóló hibaoldalt jelenít meg. Ez csak opcionálisan kerül beregisztrálásra attól függően, hogy most éppen *Development módban* futtatjuk-e az alkalmazást vagy sem. (lásd később)
* routing middleware (`UseRouting`), aminek a feladata, hogy a bejövő kérés és a végpontok (lásd lentebb) által adott információk alapján kitalálja, hogy melyik endpoint felé továbbítsa a bejövő kérést.
* végpontok middleware (`UseEndpoints`), ami a kiválasztott endpoint definíciójában megadott logika tényleges lefuttatásáért felel

!!! tip "Példa middleware regisztrációra"
    A kiinduló csővezeték regisztrálását megfigyelhetjük a [`WebApplicationBuilder` forráskódjában](https://github.com/dotnet/aspnetcore/blob/c911002ab43b7b989ed67090f2a48d9073d5118d/src/DefaultBuilder/src/WebApplicationBuilder.cs#L232) - keressük az `app.UseX` sorokat.

A kiinduló projekt nem változtat a kiinduló csővezetéken, csak egy végpont definíciót ad meg (`app.MapGet` sor).

!!! warning "Fontos a sorrend"
    A middleware-ek sorrendje fontos. Ha nem megfelelő sorrendben regisztráljuk őket, nem megfelelő működés lehet az eredmény. A dokumentáció általában tartalmazza, hogy melyik middleware [hova illeszthető be](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware#built-in-middleware).

## Hosztolási lehetőségek a fejlesztői gépen

Próbáljuk ki IIS Expressen keresztül futtatva, azaz a VS-ben az indítógomb (zöld nyíl) mellett az IIS Express felirat legyen! Ha nem ez a felirat van, állítsuk át az indítógomb jobb szélén lévő menüt lenyitva.

Két dolog is történik: az alkalmazásunk IIS Express webkiszolgálóban hosztolva kezd futni és egy böngésző is elindul, hogy ki tudjuk próbálni.
Figyeljük meg az értesítési területen (az óra mellett) megjelenő IIS Express ikont, és azon jobbklikkelve a hosztolt alkalmazás címét (menu:jobbklikk\[Show All Applications\]).

A böngésző az alkalmazás gyökércímére navigál (a cím csak **localhost:port**-ból áll), így a **Hello World!** szöveg jelenik meg.

!!! tip "Böngésző választása"
    A indítógomb legördülőjében a böngésző típusát is állíthatjuk.

!!! note "IIS Express"
    Az IIS Express a Microsoft webszerverének (IIS) fejlesztői célra optimalizált változata. Alapvetően csak ugyanarról a gépről érkező (localhost) kéréseket szolgál ki.

A másik lehetőség, ha közvetlenül a konzolos alkalmazást szeretnénk futtatni, akkor ezt az indítógombot lenyitva a projekt nevét kiválasztva tehetjük meg.
Ebben az esetben egy beágyazott webszerverhez (*Kestrel*) futnak be a kérések.
Próbáljuk ki a Kestrelt közvetlenül futtatva!

Most is két dolog történik: az alkalmazásunk konzolos alkalmazásként kezd futni, illetve az előző esethez hasonlóan a böngésző is elindul.
Figyeljük meg a konzolban megjelenő naplóüzeneteket.

!!! tip "Éles hosting"
    Bár ezek a hosztolási opciók fejlesztői környezetben nagyon kényelmesek, érdemes áttekinteni az éles hosztolási opciókat [itt](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers).
    A Kestrel ugyan jelenleg már alkalmas arra, hogy kipublikáljuk közvetlenül a világhálóra, de mivel nem rendelkezik olyan széles konfigurációs és biztonsági beállításokkal, mint a már bejáratott webszerverek, így érdemes lehet egy ilyen webszervert a Kestrel elé rakni proxy gyanánt, például az IIS-t vagy nginx-et.

Rakjunk most a kiszolgáló logikánkba egy kivétel dobást a kiírás helyett, hogy kipróbáljuk a hibakezelő MW-t.

``` csharp
app.MapGet("/", () =>
{
    throw new Exception("hiba");
    //return "Hello World!"
});
```

Próbáljuk ki debugger nélkül (++ctrl+f5++)!

Láthatjuk, hogy a kivételt a hibakezelő middleware elkapja és egy hibaoldalt jelenítünk meg, sőt még a konzolon is megjelenik naplóbejegyzésként.

## Alkalmazásbeállítások vs. indítási profilok

Figyeljük meg, hogy most **Development** konfigurációban fut az alkalmazás (konzolban a *Hosting environment* kezdetű sor).
Ezt az információt a keretrendszer környezeti változó alapján állapítja meg.
Ha a **lauchSettings.json** állományt megnézzük, akkor láthatjuk, hogy az `ASPNETCORE_ENVIRONMENT` környezeti változó `Development`-re van állítva.

Próbáljuk ki Visual Studio-n kívülről futtatni. menu:Projekten jobb klikk\[Open Folder in File Explorer\].
Ezután a címsorba mindent kijelölve `cmd` + ++enter++, a parancssorba `dotnet run`.

Ugyanúgy fog indulni, mint VS-ből, mert az újabb .NET verziókban már a *dotnet run* is figyelembe veszi a **launchSettings.json**-t.
A böngészőt magunknak kell indítani ([most még](https://github.com/dotnet/sdk/issues/9038)) és elnavigálni a naplóban szereplő címre (**Now listening on: [http://localhost:port](http://localhost:port)** üzenetet keressünk).

Ha nem akarjuk ezt, akkor a `--no-launch-profile` kapcsolót használhatjuk a *dotnet run* futtatásánál.

Most az alkalmazásunk Production módban indul el, és ha a *localhost:5000*-es oldalt megnyitjuk a böngészőben, akkor nem kapunk hibaoldalt, de a konzolon továbbra is megjelenik a naplóbejegyzés.

!!! tip "Leállítás"
    A **dotnet run** futását ++ctrl+c++-vel állíthatjuk le.

!!! tip "Környezeti változók felvétele"
    A konzolban a `setx ENV_NAME Value` utasítással tudunk [felvenni](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/setx) környezeti változót úgy, hogy az permanensen megmaradjon, és ne csak a konzolablak bezárásáig maradjon érvényben. (Admin/nem admin, illetve powershell konzolok különbözőképpen viselkednek)

Az eredeti logikánkat kommentezzük vissza.

``` csharp hl_lines="3-4"
app.MapGet("/", () =>
{
    //throw new Exception("hiba");
    return "Hello World!";
});
```

Az alkalmazás számára a különböző beállításokat JSON állományokban tárolhatjuk, amelyek akár környezetenként különbözőek is lehetnek.
A generált projektünkben ez az **appsettings.json**, nézzünk bele - főleg naplózási beállítások vannak benne.
A fájl a **Solution Explorer** ablakban kinyitható, alatta megtaláljuk az **appsettings.Development.json**-t.
Ebben a *Development* nevű konfigurációra vonatkozó beállítások vannak.
Alapértelmezésben az **appsettings.\<indítási konfiguráció neve\>.json** beállításai jutnak érvényre, felülírva a sima **appsettings.json** egyező értékeit (a pontosabb logikát lásd lentebb).

Állítsunk Development módban részletesebb naplózást. Az **appsettings.Development.json**-ben minden naplózási szintet írjunk `Debug`-ra.

``` json hl_lines="4-5"
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Debug"
    }
  }
}
```

!!! tip "Naplózási szintek"
    A naplózási szintek sorrendje [itt található](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging#configure-logging).

Próbáljuk ki, hogy így az alkalmazásunk futásakor minden böngészőbeli frissítésünk (++f5++) megjelenik a konzolon.

VS-ből is tudjuk állítani a környezeti változókat, nem kell a **launchSettings.json**-ben kézzel varázsolni.
A projekt tulajdonságok *Debug* lapján az *Open debug launch profiles UI* szövegre kattintva egy dialógusablak ugrik fel, itt tudunk új indítási profilt megadni, illetve a meglévőeket módosítani.
Válasszuk ki az aktuálisan használt profilunkat (projektneves), majd írjuk át az `ASPNETCORE_ENVIRONMENT` környezeti változó értékét az *Environment Variables* részen mondjuk *Production*-re.

Indítsuk ezzel a profillal és figyeljük meg, hogy már nem jelennek meg az egyes kérések a naplóban, bárhogy is frissítgetjük a böngészőt.
Oka: nincs **appsettings.Production.json**, így az általános **appsettings.json** jut érvényre.

!!! tip "Konfigurációk forrása"
    Számos forrásból lehet konfigurációt megadni: parancssor, környezeti változó, fájl (ezt láttuk most), felhő (**Azure Key Vault**) stb.
    Ezek közül többet is használhatunk egyszerre, a különböző források konfigurációja a közös kulcsok mentén összefésülődik.
    A források (*configuration provider*-ek) között sorrendet adhatunk meg, amikor regisztráljuk őket, a legutolsóként regisztrált provider konfigurációja a legerősebb.
    Az alapértelmezett provider-ek regisztrációját elintézi a korábban látott kiinduló builder.

### Statikus fájl MW

Hozzunk létre a projekt gyökerébe egy `wwwroot` nevű mappát (menu:jobbklikk a projekten\[Add \> New Folder\]) és tegyünk egy képfájlt bele. (Ellophatjuk pl. a <http://www.bme.hu> honlap bal felső sarkából a logo-t)

A statikus fájlkezelést a teljes modularitás jegyében egy külön middleware-ként implementálták a *Microsoft.AspNetCore.StaticFiles* osztálykönyvtárban (az AspNetCore.App már függőségként tartalmazza, így nem kell külön hivatkoznunk), csak hozzá kell adnunk a pipeline-hoz.

``` csharp hl_lines="1"
app.UseStaticFiles();
app.MapGet("/", () => "Hello World!");
```

Próbáljuk ki!

Láthatjuk hogy a *localhost:port* címen még mindig a *Hello World!* szöveg tűnik fel, de amint a *localhost:port/\[képfájlnév\]*-vel próbálkozunk, a kép töltődik be.
A static file MW megszakítja a pipeline futását, ha egy általa ismert fájltípusra hivatkozunk, egyébként továbbhív a következő MW-be.
Az ilyen MW-eket ún. **termináló** MW-eknek hívjuk.

!!! note "Breakpoint a lambdában"
    Ezt az egysoros endpoint logikára tett törésponttal is szemléltethetjük. Figyeljünk arra, hogy csak a **Hello World!** szövegre kerüljön a töréspont és ne az egész `MapGet` sorra, illetve csak akkor nézzük, hogy mi fut le, amikor a kép URL-re hívunk.

## Web API

Minden API-nál nagyon magas szinten az a cél, hogy egy kérés hatására egy szerveroldali kódrészlet meghívódjon.
ASP.NET Core-ban a Minimap API megközelítés mellett alkalmazható az MVC keretrendszer is, ahol a kódrészleteket függvényekbe írjuk, a függvények pedig ún. *kontrollerek*-be kerülnek.
Egy controller általában az egy erőforrástípushoz kapcsolódó műveleteket fogja össze. Összességében tehát a cél, hogy a webes kérés hatására egy kontroller egy függvénye meghívódjon.

### DummyController

Hozzunk létre egy új mappát *Controllers* néven.
A mappába hozzunk létre egy kontrollert (menu:jobbklikk a Controllers mappán\[Add \> Controller… \> a bal oldali fában Common \> API \> jobb oldalon API Controller with read/write actions\]) `DummyController` néven.
A generált kontrollerünk a *Microsoft.AspNetCore.Mvc.Core* csomagban található `ControllerBase` osztályból származik.
(Ezt a csomagot sem kell feltennünk, mivel az **AspNetCore.App** függősége)

Adjuk hozzá a szolgáltatásokhoz a kontrollertámogatás szolgáltatást, és adjuk hozzá a csővezetékhez a kontroller kezelő MW-t.
Az egysoros MW-t kommentezzük ki.
Így néz ki a teljes legfelső szintű kód:

``` csharp hl_lines="2 4 6"
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers(); // (1)!
var app = builder.Build();
/*var app = WebApplication.Create();*/ // (2)!
app.UseStaticFiles();
/*app.MapGet("/", () => "Hello World!");*/ // (3)!
app.MapControllers();
app.Run();
```

1. Kontrollertámogatás szolgáltatás regisztrálása
2. Mivel kell a kiinduló builder, így ezt az egysoros app inicializációt nem alkalmazhatjuk
3. Egysoros MW kikommentezve

Próbáljuk ki.

Az alapoldal üres, viszont ha az `/api/Dummy` címre hívunk, akkor megjelenik a `DummyController.Get` által visszaadott érték.
A routing szabályok szabályozzák, hogy hogyan jut el a HTTP kérés alapján a végrehajtás a függvényig.
Itt attribútum alapú routing-ot használunk, azaz a kontroller osztályra és a függvényeire biggyesztett attribútumok határozzák meg, hogy a HTTP kérés adata (pl. URL) alapján melyik függvény hívódik meg.

A `DummyController` osztályon lévő `Route` attribútum az `"api/[controller]"` útvonalat definiálja, melyből a `[controller]` úgynevezett token, ami jelen esetben a controller nevére cserélődik.
Ezzel összességében megadtuk, hogy az `api/Dummy` útvonal a `DummyController`-t választja ki, de még nem tudjuk, hogy a függvényei közül melyiket kell meghívni - ez a függvényekre tett attribútumokból következik.
A `Get` függvényen levő `HttpGet` mutatja, hogy ez a függvény akkor hívandó, ha a GET kérés URL-je nem folytatódik - ellentétben a `Get(int id)` függvénnyel, ami az URL-ben még egy további szegmenst vár (ezért van egy `"{id}"` paraméter megadva az attribútum konstruktorban), amit az `id` nevű függvényparaméterként használ fel.

!!! tip "Routing lehetőségek"
    Az API-t publikáló alkalmazásoknál az attribútum alapú routing az ajánlott, de emellett vannak más megközelítések is, például **Razor** alapú weboldalaknál konvenció alapú routing az ajánlott.
    Bővebben a témakörről általánosan [itt](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing), illetve specifikusan webes API-k vonatkozásában [itt](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing#attribute-routing-for-rest-apis) lehet olvasni.
    A dokumentáció mennyiségéből látható, hogy a routing alrendszer nagyon szofisztikált és sokat tud, szerencsére az alap működés elég egyszerű és gyorsan megszokható.

Ha van időnk, próbáljuk ki az `/api/Dummy/[egész szám]` címet is. A `Get(int id)` függvény kódjának megfelelően, bármit adunk meg, az eredmény a *value* szöveg lesz.

## Konfigurációs beállítások, Dependency Injection

Bővítsük az *appsettings.json*-t egy saját beállításcsoporttal (`DummySettings`):

``` json hl_lines="9-14"
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*", // a sor végére bekerült egy vessző
  "DummySettings": {
    "DefaultString": "My Value",
    "DefaultInt": 23,
    "SuperSecret":  "Spoiler Alert!!!"
  }
}
```

Kérjük le a beállításokat a `DummyController`-ben, a `Get` függvényekben írjuk ki a `DefaultString` és `DefaultInt` értékét.

Ehhez viszont el kell kérjük a `IConfiguration` interfészt a konstruktorban a Dependency Injection (DI) mechanizmuson keresztül a DI konténertől.
Ez többek között lehetővé teszi, hogy az alkalmazáson belül konstruktorban paraméterként igényeljük a szolgáltatást.
A paraméter értékét a DI alrendszer automatikusan tölti ki a regisztrált szolgáltatások alapján.

``` csharp
private readonly IConfiguration _configuration;

public DummyController(IConfiguration configuration)
{
    _configuration = configuration;
}

[HttpGet]
public string Get()
{
    return $"string: {_configuration.GetValue<string>("DummySettings:DefaultString")}" +
           $"int: {_configuration.GetValue<int>("DummySettings:DefaultInt")}" +
           $"secret: {_configuration.GetValue<string>("DummySettings:SuperSecret")}";
}
```

!!! tip "DI minta preferálása"
    ASP.NET Core környezetben (is) törekedjünk arra, hogy lehetőleg minden osztályunk minden függőségét a DI minta szerint a DI konténer kezelje.
    Ez nagyban hozzájárul a komponensek közötti laza csatolás és a jobb tesztelhetőség eléréséhez.
    Bővebb információ az ASP.NET Core DI alrendszeréről a [dokumentációban](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) található.


## Típusos beállítások, `IOptions<T>`

Fentebb láttuk, hogy a konfigurációt ki tudtuk olvasni az `IConfiguration` interfészen keresztül, de még jobb lenne, ha csoportosítva és csoportonként külön C# osztályokon keresztül látnánk őket.

Hozzunk létre egy új mappát `Options` néven.

A mappába hozzunk létre egy sima osztályt `DummySettings` néven, a szerkezete feleljen meg a JSON-ben leírt beállításcsoportnak:

``` csharp
public class DummySettings
{
    public string? DefaultString { get; set; }
    public int DefaultInt { get; set; }
    public string? SuperSecret { get; set; }
}
```

Regisztráljuk szolgáltatásként a `DummySettings` kezelését, és adjuk meg, hogy a példányt mi alapján kell inicializálni - a konfiguráció megfelelő szekciójára hivatkozzunk:

``` csharp
builder.Services.Configure<DummySettings>(
    builder.Configuration.GetSection(nameof(DummySettings)));
```

A `builder.Services`-ben regisztrált szolgáltatások valójában egy dependency injection (DI) konténerbe kerülnek regisztrálásra.

Igényeljünk `DummySettings`-t a `DummyController` konstruktorban az `IConfiguration` helyett:

``` csharp
private readonly DummySettings _options;

public DummyController(IOptions<DummySettings> options)
{
    _options = options.Value;
}
```

!!! tip "IOptions<> és társai"
    Látható, hogy a beállítás `IOptions`-ba burkolva érkezik. Vannak az `IOptions`-nál okosabb burkolók is (pl. `IOptionsMonitor`), ami például jelzi, ha megváltozik valamilyen beállítás.
    Bővebb információ az `IOptions` és társairól a hivatalos dokumentációban [található](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options).

Az egész számot váró `Get` változatban használjuk fel az értékeket:

``` csharp hl_lines="4-6"
[HttpGet]
public string Get()
{
    return $"string: {_options.DefaultString}" +
           $"int: {_options.DefaultInt}" +
           $"secret: {_options.SuperSecret}";
}
```

Próbáljuk ki, hogy a végpont meghívásakor a megfelelő értékeket kapjuk-e vissza.

## User Secrets

A projekt könyvtára gyakran valamilyen verziókezelő (pl. Git) kezelésében van.
Ilyenkor gyakori probléma, hogy a konfigurációs fájlokba írt szenzitív információk (API kulcsok, adatbázis jelszavak) bekerülnek a verziókezelőbe.
Ha egy publikus projekten dolgozunk, például publikus GitHub projekt, akkor ez komoly biztonsági kockázat lehet.

!!! warning "Szenzitív információk"
    Ne tegyünk a verziókezelőbe szenzitív információkat még privát repó esetében sem.
    Gondoljunk arra is, hogy a verziókezelő nem felejt! Ami egyszer már bekerült, azt vissza is lehet nyerni belőle (history).

Ennek a problémának megoldására egy eszköz a *User Secrets* tároló.
Jobbklikkeljünk a projekten a Solution Explorer ablakban, majd válasszuk a *Manage User Secrets* menüpontot.
Ennek hatására megnyílik egy **secrets.json** nevű fájl.
Vizsgáljuk meg, hol is van ez a fájl: vigyük az egeret a fájlfül fölé - azt láthatjuk, hogy a fájl a felhasználónk saját könyvtárán belül van és az útvonal része egy GUID is.
A projektfájlba (.csproj) bekerült ugyanez a GUID (a *UserSecretsId* címkébe).

Vegyünk fel egy új beállítást a **secrets.json**-ba, ami a `SuperSecret` értékét írja felül:

``` json
{
  "DummySettings": {
    "SuperSecret": "SECRET"
  }
}
```

!!! tip "Részleges felülírás"
    A **secrets.json**-ban csak azokat a json levél elemeket kell felvenni, amiket felül akarunk írni. Ez a módszer működik a sima **appsettings.json** környezetfüggő változóira is.

Töréspontot letéve (pl. a `DummyController` konstruktorának végén) ellenőrizzük, hogy a titkos érték melyik fájlból jön.
Ehhez meg kell hívnunk böngészőből az `api/dummy` címet.

!!! warning "User Secrets csak Development módban"
    Fontos tudni, hogy a *User Secrets* tároló csak **Development** mód esetén jut érvényre, így figyeljünk rá, hogy a megfelelő módot indítsuk és a környezeti változók is jól legyenek beállítva.

Ez az eljárás tehát a futtató felhasználó saját könyvtárából a GUID alapján kikeresi a projekthez tartozó **secrets.json**-t, annak tartalmát pedig futás közben összefésüli az **appsettings.json** tartalmával. Így szenzitív adat nem kerül a projekt könyvtárába.

!!! tip "Titkok éles környezetben"
    Mivel a *User Secrets* tároló csak Development mód esetén jut érvényre, így ha az éles változatnak szüksége van ezekre a titkos értékekre, akkor további trükkökre van szükség.
    Ilyen megoldás lehet, ha a felhős hosztolás esetén a felhőből (pl. [Azure App Service Configuration](https://docs.microsoft.com/en-us/azure/app-service/configure-common#configure-app-settings)) vagy felhőbeli titoktárolóból (pl. [Azure Key Vault](https://docs.microsoft.com/en-us/aspnet/core/security/key-vault-configuration)) vagy a DevOps eszközből (pl. [Azure DevOps Pipeline Secrets](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#secret-variables)) töltjük be a szenzitív beállításokat.

## Epilógus - WebApplicationBuilder

Az eddigiekből látható, hogy számos alapszolgáltatás már a `CreateBuilder` hívás által visszaadott kiinduló builderben konfigurálva van. Ilyen az alap (`IOptions` nélküli) alkalmazásbeállítások kezelése vagy a naplózás. A `CreateBuilder` a `WebApplicationBuilder` internal konstruktorát hívja.

A `WebApplicationBuilder` elődje az `IWebHostBuilder`, ez utóbbinak a [dokumentációját](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host) tanulmányozva érthetjük meg, hogy mi mindent tud a kiinduló builder.
