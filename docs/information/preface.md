---
author: kszicsillag,tibitoth
---

# Tudnivalók

## A jegyzet célja és célközönsége

Ezen jegyzet elsődlegesen a BME Villamosmérnöki és Informatikai Karán oktatott [Szoftverfejlesztés .NET platformra](https://www.aut.bme.hu/Course/dotnet) című tárgyhoz készült, célja, hogy segítséget nyújtson egyrészt a gyakorlatvezetőnek a gyakorlat megtartásában, másrészt a kurzus hallgatóinak a gyakorlat otthoni utólagos megismétléséhez, a tanult ismeretek átismétléséhez.

Ebből kifolyólag nem tekinthető egy teljesen kezdő szintű bevezető C# tankönyvnek, hiszen erőteljesen épít más kari tárgyak (pl. Szoftvertechnikák, Adatbázisok) által lefedett ismeretekre, de még inkább a Szoftverfejlesztés .NET platformra című tárgy előadásaira.

A feltételezett előismeretek:

* C# és objektumorientált nyelvi alapok
    * operátorok, változók, tömbök, struktúrák, függvények fogalma
    * operátor felüldefiniálás és függvényváltozatok
    * alapvető memóriakezelés (heap, stack), mutatók fogalma, érték és referencia típusok
    * alapvető vezérlési szerkezetek (ciklus, elágazás, stb.), érték- és referencia szerinti paraméterátadás, rekurzió
    * osztály, osztálypéldány fogalma, static, `new` operátor, osztály szintű változók, generikus típusok
    * leszármazás, virtuális tagfüggvények
    * C# esemény, delegate típusok és delegate példányok
    * Visual Studio használatának alapjai
    * operációs rendszer kapcsolatok, folyamatok, szálak, parancssor, parancssori argumentumok, környezeti változók
* SQL nyelvi alapok (SELECT, UPDATE, INSERT, DELETE utasítások), valamint alapvető relációs adatmodell ismeretek (táblák, elsődleges- és idegen kulcsok)

A fentiek elsajátításához segítséget nyújthatnak Reiter István ingyenesen [letölthető könyvei](https://reiteristvan.wordpress.com).

A szövegben megtalálhatók a gyakorlatvezetőknek szóló kitételek („Röviden mondjuk el…", „Mutassuk meg…", stb.). Ezeket mezei olvasóként érdemes figyelmen kívül hagyni, illetve szükség esetén a kapcsolódó elméleti ismereteket az előadásanyagból átismételni.

## A jegyzet naprakészsége

Az anyag gerincét adó .NET Core / .NET 5,6 platform jelenleg igen gyors ütemben fejlődik. A .NET Core 1.0-s verzió óta a készítők törekednek a visszafelé kompatibilitásra, azonban az eszközkészlet és a korszerűnek és ajánlottnak tekinthető módszerek folyamatosan változnak, finomodnak.

A jegyzet elsődlegesen az alábbi technológiai verziókhoz készült:

* C# **12**
* .NET **8**
* ASP.NET Core **8**
* Visual Studio **2022**

Ahogyan a fenti verziók változnak, úgy avulhatnak el a jegyzetben mutatott eljárások.

## Szoftverkörnyezet

A gyakorlatok az alábbi szoftverekből álló környezethez készültek:

* Windows 11 operációs rendszer
* [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/) (az ingyenes Community verzió elég) az alábbi workloadokkal:
    * .NET desktop development
    * Data storage and processing
    * ASP.NET and web development
    * Azure Development
* [Telerik Fiddler Classic](https://www.telerik.com/fiddler/fiddler-classic)
* [Postman](https://www.postman.com/)

A .NET (korábban .NET Core) széleskörű platformtámogatása miatt bizonyos nem Windows platformokon is elvégezhetők a gyakorlatok Visual Studio helyett [Visual Studio Code](https://code.visualstudio.com/) használatával - azonban a gyakorlatok szövege a Visual Studio használatát feltételezi.

## Kódrészletek változáskövetése

Az egyes gyakorlatok során gyakori eset, hogy a C# kód egy részét továbbfejlesztjük, megváltoztatjuk.
Ilyen esetben a változó sorokat a jegyzetben kiemelt háttérrel rendelkeznek.
A törölt kódrészleteket (amennyiben van segíti a megértést) kommentezéssel jelezzük.
Jelöljük még a meglévő, de a jegyzetben nem megjelenített kódrészleteket komment és ... (`//...`) jellel.

``` csharp hl_lines="2 7 9"
using System; //ez egy korábban meglévő kódsor, változatlan
using static System.Console; //ez új kódsor

//... meglévő kódrészlet az előző feladatokból

foreach (var dog in dogs)    //ez egy korábban meglévő kódsor, változatlan
  /* Console.*/WriteLine(dog); //ez a sor megváltozott, az elejéről kód törlődött

/* Console.*/ReadLine();     //ez a sor megváltozott, az elejéről kód törlődött
```

!!! warning "JSON kommentek"
    A JSON formátum alapértelmezésben (RFC szerint) nem támogatja a kommenteket, így ha JSON kódrészletet másolunk, győződjünk meg arról, hogy nem maradt-e a beillesztett kódban komment, mert problémát okozhat.
