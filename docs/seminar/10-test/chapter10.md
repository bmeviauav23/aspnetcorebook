# Automatizált tesztelés

## Segédeszközök

- kapcsolódó GitHub repo: <https://github.com/bmeviauav23/WebApiLab-kiindulo>

## Bevezetés

Az automatizált tesztelés az alkalmazásfejlesztés egyik fontos lépése, mivel ezzel tudunk meggyőződni arról, hogy egy-egy funkció akkor is helyesen működik, ha az alkalmazás egy másik részén valamit módosítunk.
Hogy ezt az ellenőrzést ne kelljen minden egyes alkalommal manuálisan végrehajtani az alkalmazáson, programozott teszteket szoktunk írni, amelyek futtatását CI/CD folyamatokban automatizálhatjuk.

A tesztek több típusát ismerhetjük:

- **Unit test (egységteszt)** célja, hogy egy adott osztály egy metódusának a viselkedését önmagába vizsgáljuk úgy, hogy a függőségeit [mock/fake objektumokkal](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices#lets-speak-the-same-language) helyettesítjük, hogy azok a tesztesetnek megfelelően viselkedjenek vagy megfigyelhetőek legyenek.
- **Integrációs teszt / End-2-end teszt / funkcionális teszt** esetében a célunk, hogy a teljes rendszert meghajtsuk úgy, hogy az integrációk (SQL kapcsolat, egyéb szolgáltatások) is tesztelésre kerülnek, illetve a BE szempontjából vizsgáljuk azt is, hogy a rendszer interfésze helyesen válaszol-e a különböző kérésekre.
- **UI teszt** esetében azt vizsgáljuk, hogy a felhasználói felület a különböző felhasználói interakciókra, eseményekre helyesen rajzolja-e ki az elvárt felületeket.

A fenti tesztelési módok mindegyike fontos, de érdemes egy olyan egészséges egyensúlyt megtalálni, ahol a lehető legjobban lefedhetőek a legfontosabb funkcionalitások különböző tesztesetekkel.

## Automatizált tesztelés .NET környezetben

Automatizált tesztelésre több keretrendszer is használható .NET környezetben, de ASP.NET Core alkalmazások esetében a legelterjedtebb ilyen könyvtár az [**xUnit**](https://xunit.net/).
Ebben a keretrendszerben lehetőségünk van tesztesetek definiálására, akár a bemenetek variálásával is, illetve kellően rugalmas, ahhoz, hogy a tesztek feldolgozási mechanizmusa kiterjeszthető legyen.

Unit tesztek esetében az osztályok függőségeit le kell cseréljük, amire több library is lehetőséget nyújt.
A legelterjedtebbek a [**Moq**](https://github.com/moq) és az [**NSubstitute**](https://nsubstitute.github.io/).

Gyakran szükséges funkció, hogy a bemenő adatok előállítása során szeretnénk a valóságra hasonlító véletlenszerű/generált példaadatokat megadni.
Ehhez egy bevált osztálykönyvtár a [**Bogus**](https://github.com/bchavez/Bogus).

A tesztesetek elvárt eredményének a vizsgálatát asszertálásnak nevezzük (*assert*), aminek az írásához nagy segítséget tud nyújtani a [**Fluent Assertions**](https://fluentassertions.com) könyvtár.
Ez nem csak a szintaktikát teszi olvashatóbbá fluent szintakszissal, hanem több olyan beépített segédlogikát tartalmaz, amivel tömörebbé tehető az *assert* logika (pl.: objektumok mélységi összehasonlítása érték szerint).

## Integrációs tesztelés

Ezen gyakorlat keretében csak integrációs teszteket fogunk készíteni.

### Teszt projekt

Vegyünk fel a solutionbe egy új xUnit (.NET 8) típusú projektet `WebApiLab.Tests` néven. A létrejövő tesztosztályt és fájlját nevezzük át `ProductControllerTests` névre. Ide fogjuk a `ProductController`-hez kapcsolódó műveletekre vonatkozó integrációs teszteket készíteni.

Vegyük fel az alábbi NuGet csomagokat a teszt projektbe.
A *Bogus*ról és a *Fluent Assertions*ről már volt szó.
A *Microsoft.AspNetCore.Mvc.Testing* csomag olyan segédszolgáltatásokat nyújt, amivel integrációs tesztekhez egy in-process teszt szervert tudunk futtatni, és ennek a meghívásában is segítséget nyújt.
A projektfájlban a többi `PackageReference` mellé (menu:a projekten jobbklikk\[Edit Project File\]):

``` xml
<PackageReference Include="Bogus" Version="35.5.1" />
<PackageReference Include="FluentAssertions" Version="6.12.0" />
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.4" />
```

Vegyük fel az **Api** projektet projekt referenciaként a teszt projektbe. A projektfájlban egy másik `ItemGroup` mellé:

``` xml
<ItemGroup>
  <ProjectReference Include="..\WebApiLab.Api\WebApiLab.Api.csproj" />
</ItemGroup>
```

### Teszt szerver

A tesztszervernek meg kell tudnunk mondani, hogy melyik osztály adja az alkalmazásunk belépési pontját.
Viszont mivel top level statement szintaktikájú a `Program` osztályunk, annak láthatósága internal, ami a tesztelés szempontjából nem szerencsés (a hasonló esetekben alkalmazott `InternalsVisibleTo` sem lenne [ebben az esetben megoldás](https://stackoverflow.com/a/69483450/1406798)).
Helyette tegyük a `Program` osztályt publikussá egy `partial` deklarációval.
Vegyük fel az alábbi partial kiegészítést az API projektben a legfelső szintű kód végére:

``` csharp
public partial class Program { }
```

Az integrációs tesztünkhöz az in-process teszt szervert egy `WebApplicationFactory<TEntryPoint>` leszármazott osztály fogja létrehozni.
Ez a segéd ősosztály a fenti Microsoft.AspNetCore.Mvc.Testing csomagból jön.
Itt lehetőségünk van a teszt szerverünket konfigurálni, így akár a DI konfigurációt is.

Hozzunk létre egy osztályt a teszt projektbe `CustomWebApplicationFactory` néven, ami származzon a `WebApplicationFactory<Program>` osztályból és definiáljuk felül a `CreateHost` metódusát.

``` csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override IHost CreateHost(IHostBuilder builder)
    {
        builder.UseEnvironment("Development");
        builder.ConfigureServices(services =>
        {
            services.AddScoped(sp => new DbContextOptionsBuilder<AppDbContext>()
                .UseSqlServer(@"connection string")
                .UseApplicationServiceProvider(sp)
                .Options);
        });

        var host = base.CreateHost(builder);

        using var scope = host.Services.CreateScope();
        scope.ServiceProvider.GetRequiredService<AppDbContext>()
            .Database.EnsureCreated();

        return host;
    }
}
```

Megfigyelhetjük, hogy itt is LocalDB-t használunk (mivel integrációs teszt), de a connection stringet lecseréjük a DI konfigurációban. A connection string alapvetően egyezhet a tesztelendő projektben használttal, csak az adatbázisnevet változtassuk meg.
Az adatbázis automatikusan létrejön és a migrációk is lefutnak az `EnsureCreated` meghívásával - az első lefutáskor.

!!! warning "DI Scope létrehozása"
    Mivel az `AppDbContext` Scoped életciklussal van regisztrálva a DI-ba, szükséges létrehozni egy scope-ot, hogy el tudjuk kérni a DI konténertől.
    Ezt természetesen ha HTTP kérés közben lennénk az ASP.NET Core automatikusan megtenné.

### Kontrollertesztek előkészítése

Alakítsuk át a `ProductControllerTests` osztályt.
Az osztály valósítsa meg az `IClassFixture<CustomWebApplicationFactory>` interfészt, amivel azt tudjuk jelezni az xUnit-nak, hogy kezelje a `CustomWebApplicationFactory` életciklusát (tesztek között [megosztott objektum](https://xunit.net/docs/shared-context#class-fixture) lesz), illetve pluszban lehetőségünk van ezt a tesztosztályokban konstruktoron keresztül elkérni.

``` csharp
public partial class ProductControllerTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly WebApplicationFactory<Program> _appFactory;

    public ProductControllerTests(CustomWebApplicationFactory appFactory)
    {
        _appFactory = appFactory;
    }
}
```

!!! warning "xUnit Fixture"
    Az xUnit nem tartalmaz DI konténert.
    Csak azok a konstruktorparaméterek töltődnek ki, amelyek a dokumentációban megtalálhatók.
    A `CustomWebApplicationFactory` típusú paraméter azért töltődik ki, mert az osztály az interfészében jelzi, hogy megosztott kontextusként `CustomWebApplicationFactory`-t vár.

Hozzunk létre a Bogus könyvtárral egy olyan `Faker<Product>` objektumot, amivel az API-nak küldendő DTO objektum generálását végezzük el.
Azonosítóként küldjünk 0 értéket, mivel a létrehozás műveletet fogjuk tesztelni, kategória esetében pedig az 1-et, mivel a migráció által létrehozott 1-es kategóriát fogjuk tudni csak használni.
A többi esetben használjuk a Bogus beépített lehetőségeit a név és a szám értékek random generálásához.

``` csharp
// ...
private readonly Faker<Product> _dtoFaker;

public ProductControllerTests(CustomWebApplicationFactory appFactory)
{
    // ...
    _dtoFaker = new Faker<Product>()
        .RuleFor(p => p.Id, 0)
        .RuleFor(p => p.Name, f => f.Commerce.Product())
        .RuleFor(p => p.UnitPrice, f => f.Random.Int(200, 20000))
        .RuleFor(p => p.ShipmentRegion,
                 f => f.PickRandom<Dal.Entities.ShipmentRegion>())
        .RuleFor(p => p.CategoryId, 1)
        .RuleFor(p => p.RowVersion, f => f.Random.Bytes(5));
}
```

A kliensoldali JSON sorosítást a szerveroldallal kompatibilisen kell megtegyük.
Ehhez készítsünk egy `JsonSerializerOptions` objektumot, amibe beállítjuk, hogy a felsorolt típusokat szöveges értékként kezelje.
Mivel ugyanazt a példányt akarjuk használni a tesztekben, ezért a példányt a `CustomWebApplicationFactory` (mint tesztek közötti megosztott objektum) készítse el és ajánlja ki.

``` csharp
public JsonSerializerOptions SerializerOptions { get; }

public CustomWebApplicationFactory()
{
    JsonSerializerOptions jso = new(JsonSerializerDefaults.Web);
    jso.Converters.Add(new JsonStringEnumConverter());
    SerializerOptions = jso;
}
```

A `ProductControllerTests` a kiajánlott `JsonSerializerOptions`-t vegye át.

``` csharp
// ...
private readonly JsonSerializerOptions _serializerOptions;

public ProductControllerTests(CustomWebApplicationFactory appFactory)
{
    // ...
    _serializerOptions = appFactory.SerializerOptions;
}
```

!!! warning "Sorosítás beállításai"
    Sajnos ezt a `JsonSerializerOptions` példányt minden sorosítást igénylő műveletnél majd át kell adnunk, mivel az alapértelmezett JSON sorosítónak [nincs publikusan elérhető API-ja](https://github.com/dotnet/runtime/issues/31094) alapértelmezett sorosítási beállítások megadásához.
    Ugyanakkor fontos, hogy kerüljük a `JsonSerializerOptions` [felesleges példányosítását](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/configure-options?pivots=dotnet-6-0#reuse-jsonserializeroptions-instances).
    Ugyanolyan beállításokat igénylő műveletek lehetőleg ugyanazt a példányt használják.
    Ezt most az XUnit megosztott kontextusával oldottuk meg.

### POST művelet alapműködés tesztelése

Készítsük el az első tesztünket a `ProductController` `Post` műveletéhez.
Érdemes azt az osztálystruktúrát követni, hogy minden művelethez / függvényhez külön teszt osztályokat hozunk létre, ami akár több tesztesetet is tartalmazhat.
Ez a teszt osztályt beágyazott osztályként (`Post`) hozzuk létre egy külön partial fájlban (**ProductIntegrationTests.Post.cs**) a nagyobb egységhez tartozó tesztosztályon belül.
Ezzel szépen strukturáltan tudjuk tartani a **Test Explorerben** (lásd később) is a teszteseteinket.
Pluszban még származtassuk le a tartalmazó osztályból, hogy a tesztesetek elérhessék a fentebb létrehozott osztályváltozókat.

!!! tip "Láthatóság beágyazott osztályoknál"
    Érdekesség, hogy nem kell `protected` láthatóságúaknak lenniük a fenti osztályváltozóknak, ha beágyazott osztály akarja elérni azokat.

``` csharp
public partial class ProductControllerTests
{
    //...
    public class Post : ProductControllerTests
    {
        public Post(CustomWebApplicationFactory appFactory)
            : base(appFactory)
        {
        }
    }
}
```

A tesztesetek a teszt osztályban metódusok fogják reprezentálni, amelyek `[Fact]` vagy `[Theory]` attribútummal rendelkeznek.
A fő különbég az, hogy a `Fact` egy statikus tesztesetet reprezentál, míg a `Theory` bemenő paraméterekkel rendelkezhet.

Elsőként az egyenes ágat teszteljük le, hogy a beszúrás helyesen lefut-e, és a megfelelő HTTP válaszkódot, a *location* HTTP fejlécet, és válasz DTO-t adja-e vissza. 
Hozzunk létre egy függvényt `Fact` attribútummal `Should_Succeded_With_Created` néven.

A teszteset az [AAA (Arrange, Act, Assert)](https://learn.microsoft.com/en-us/visualstudio/test/unit-test-basics?view=vs-2022#write-your-tests) mintát követi, ahol 3 részre tagoljuk magát a tesztesetet.

1. Az *Arrange* fázisban előkészítjük a teszteset körülményeit.
2. Az *Act* fázisban elvégezzük a tesztelendő műveletet.
3. Az *Assert* fázisban pedig megvizsgáljuk a végrehajtott művelet eredményeit, mellékhatásait.

``` csharp
[Fact]
public async Task Should_Succeded_With_Created()
{
    // Arrange

    // Act

    // Assert
}
```

Az *Arrage*-ben kérjünk el egy a teszt szerverhez kapcsolódó `HttpClient` objektumot, illetve hozzunk létre egy felküldendő DTO-t.

``` csharp
// Arrange
var client = _appFactory.CreateClient();
var dto = _dtoFaker.Generate();
```

Az *Act* fázisban küldjünk el egy POST kérést a megfelelő végpontra a megfelelő sorosítási beállításokkal és olvassuk ki a választ.

``` csharp
// Act
var response = await client.PostAsJsonAsync("/api/products", dto, _serializerOptions);
var p = await response.Content.ReadFromJsonAsync<Product>(_serializerOptions);
```

Az *Assert* fázisban pedig fogalmazzuk meg a FluentValidation könyvtár segítségével az elvárt eredmény szabályait.
Gondoljunk arra is, hogy a `Category`, `Order`, `Id` és `RowVersion` property-k esetében nem az az elvárt válasz, amit felküldünk a szerverre, ezért ezeket szűrjük le az összehasonlításból és vizsgáljuk őket külön szabállyal.

``` csharp
// Assert
response.StatusCode.Should().Be(HttpStatusCode.Created);
response.Headers.Location
    .Should().Be(
        new Uri(_appFactory.Server.BaseAddress, $"/api/Products/{p.Id}")
    );

p.Should().BeEquivalentTo(
    dto,
    opt => opt.Excluding(x => x.Category)
        .Excluding(x => x.Orders)
        .Excluding(x => x.Id)
        .Excluding(x => x.RowVersion));
p.Category.Should().NotBeNull();
p.Category.Id.Should().Be(dto.CategoryId);
p.Orders.Should().BeEmpty();
p.Id.Should().BeGreaterThan(0);
p.RowVersion.Should().NotBeEmpty();
```

!!! warning "Fluent Assertions és Nullable Reference Types"
    A Fluent Assertions (nem preview verziója) [jelenleg még nem működik együtt](https://github.com/fluentassertions/fluentassertions/issues/1115) a nem nullozható referencia típusokkal kapcsolatos ellenőrzési logikákkal, így az *Assert* részen kaphatunk ennek kapcsán figyelmeztetéseket `Should().NotBeNull()` hívások után is.

A POST művelet megváltoztatná az adatbázis állapotát, amit célszerű lenne elkerülni.
Ezt legegyszerűbben úgy érhetjük el, hogy nyitunk egy tranzakciót a tesztben, amit nem commitolunk a teszt lefutása során. Ehhez vegyük fel az alábbi utasításokat az *Arrange* fázisban.

``` csharp hl_lines="2-3"
// Arrange
_appFactory.Server.PreserveExecutionContext = true;
using var tran = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

var client = _appFactory.CreateClient();
var dto = _dtoFaker.Generate();
```

Tranzakciót a .NET `TransactionScope` osztállyal fogunk most nyitni, amin engedélyezzük az aszinkron támogatást is.
Ahhoz pedig, hogy a tesztben létrehozott tranzakció érvényre jusson a teszt szerveren is, a `PreserveExecutionContext` tulajdonságot be kell kapcsoljuk.

Próbáljuk ki a menu:Test\[Run All Test\] menüpont segítségével.
A [Test Explorerben](https://learn.microsoft.com/en-us/visualstudio/test/run-unit-tests-with-test-explorer?view=vs-2022#run-tests-in-test-explorer) figyeljük meg az eredményt.

### POST művelet hibaág tesztelése

Készítsünk egy tesztesetet, ami a hibás terméknév ágat teszteli le.
Mivel ez két esetet is magában foglal (null, üres string), használjunk paraméterezhető tesztesetet, tehát `Theory`-t.
A teszteset bemenő paramétereit többféleképpen is meg lehet adni.
Mi most válasszuk az `InlineData` megközelítést, ahol attribútumokkal a teszteset fölött közvetlenül megadhatóak a bemenő paraméter értékei.
Ilyen esetben az attribútumban megadott értékeket a teszt metódus paraméterlistáján kell elkérjük.
Esetünkben a név hibás értékeit várjuk első paraméterként, második paraméterként pedig az elvárt hibaüzenetet.

``` csharp
[Theory]
[InlineData("", "Product name is required.")]
[InlineData(null, "Product name is required.")]
public async Task Should_Fail_When_Name_Is_Invalid(string name, string expectedError)
{
    // Arrange

    // Act

    // Assert
}
```

Az előző tesztesethez hasonlóan hozzunk létre a teszt szervert és a DTO-t, de most a nevet a paraméter alapján töltsük fel.
Bár elvileg nem lenne szükséges tranzakciókezelés, hiszen nem szabadna adatbázis módosításnak történnie, a biztonság kedvéért implementáljuk itt is a tranzakciókezelést.

``` csharp
// Arrange
 _appFactory.Server.PreserveExecutionContext = true;
using var tran = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
var client = _appFactory.CreateClient();
var dto = _dtoFaker.RuleFor(x => x.Name, name).Generate();
```

Az *Act* fázisban annyi a különbség, hogy most `ValidationProblemDetails` objektumot várunk a válaszban.

``` csharp
// Act
var response = await client.PostAsJsonAsync("/api/products", dto, _serializerOptions);
var p = await response.Content
    .ReadFromJsonAsync<ValidationProblemDetails>(_serializerOptions);
```

Az *Assert* fázisban pedig a HTTP státuszkódot és a `ProblemDetails` tartalmára vizsgáljunk.

``` csharp
// Assert
response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

p.Status.Should().Be(400);
p.Errors.Should().HaveCount(1);
p.Errors.Should().ContainKey(nameof(Product.Name));
p.Errors[nameof(Product.Name)].Should().ContainSingle(expectedError);
```

Próbáljuk ki a menu:Test\[Run All Test\] menüpont segítségével.
Figyeljük meg a tesztek hierarchiáját is, a POST művelethez kapcsolódó tesztek egy csoportba lettek összefogva a beágyazott osztály mentén.

!!! tip "Transzakciókezelés kódduplikáció"
    Észrevehetjük, hogy a tranzakciókezeléssel kapcsolatos kódot duplikáltuk, ennek elkerülésére például [például tesztfüggvényre tehető attribútumot](https://github.com/xunit/samples.xunit/blob/main/AutoRollbackExample/AutoRollbackAttribute.cs) vezethetünk be.

## Naplózás

A tesztek üzeneteket naplózhatnak egy speciális tesztkimenetre.
Ehhez minden tesztosztály példány kap(hat) egy saját `ITestOutputHelper` példányt a konstruktoron keresztül.
Vezessük be az új konstruktorparamétert a tesztosztályban és az ősosztályában is.

``` csharp hl_lines="1 5 8"
private readonly ITestOutputHelper _testOutput;

public ProductControllerTests(
    CustomWebApplicationFactory appFactory,
    ITestOutputHelper output)
{
    //...
    _testOutput = output;
}
```

``` csharp title="Post beágyazott típus konstruktora"
public Post(CustomWebApplicationFactory appFactory, ITestOutputHelper output)
    : base(appFactory, output)
{ }
```

Próbaképp írjunk ki egy üzenetet a `ProductControllerTests` konstruktorában.

``` csharp
output.WriteLine("ProductControllerTests ctor");
```

Ellenőrizzük, hogy a tesztek lefuttatása után *Test Explorer*-ben megjelennek-e az üzenetek a [*Test Detail Summary*](https://learn.microsoft.com/en-us/visualstudio/test/run-unit-tests-with-test-explorer?view=vs-2022#view-test-details) ablakrész *Standard output* szekciójában.
Ebből láthatjuk, hogy minden tesztfüggvény, sőt minden tesztfüggvény változat (a *Theory* minden bemeneti adatsora egy külön változat) meghívásakor lefut a konstruktor.

Ugyanerre a kimenetre kössük rá a szerveroldali naplózást, hogy a tesztek lefutása mellett ezek a naplóüzenetek is megjelenjenek.
Ehhez telepítsünk egy segédcsomagot a tesztprojektbe.

``` xml
<PackageReference Include="MartinCostello.Logging.XUnit" Version="0.3.0" />
```

A `ProductControllerTests` konstruktorában kössük össze a két paramétert, a `CustomWebApplicationFactory` és az `ITestOutputHelper` példányt a fenti segédcsomag (`AddXUnit` metódus) segítségével.
A tesztszerver naplózó alrendszerének adjuk meg kimenetként az xUnit tesztkimenetét.

``` csharp
_appFactory = appFactory
    .WithWebHostBuilder(builder =>
    {
        builder.ConfigureLogging(logging =>
        {
            logging.ClearProviders();
            logging.AddXUnit(output);
        });
    });
```

Ellenőrizzük, hogy a tesztek lefuttatása után *Test Explorer*-ben megjelennek-e a szerveroldali üzenetek is.

A végállapot elérhető a kapcsolódó GitHub repóban.
