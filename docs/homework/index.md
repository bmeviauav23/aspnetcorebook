# Házi feladat tudnivalók

Az itteni szerkezetet kövessétek: [példa specifikáció](example-spec.md)

## Követelmények

Az alábbiak közül mindegyiknek teljesülnie kell az aláíráshoz:

* Két fő részből áll
    * szerver oldali HTTP alapú szolgáltatás
    * egy vastag vagy vékony kliens (szerveroldali renderelés nélkül) alkalmazás, ami a szolgáltatást hívja
        * elfogadható (példák): WPF, WinForms, MAUI, Swing, JavaFX, Blazor WebAssembly, Angular, React, Vue, Android (kotlin, java), iOS (swift, obj-c), stb.
        * nem elfogadható: ASP.NET Core MVC Razor generált weboldalakból álló webalkalmazás, JSP, PHP, Blazor Server, vagy Blazor Static Render, sima HTML+JS+CSS
        * kivétel: a felhasználókezeléshez szorosan kapcsolódó felületek (belépés, regisztráció, stb.) bármilyen felületi technológiával készülhetnek
    * a vastag/vékony kliens kiváltható Postman klienssel
* A kliens nem éri el közvetlenül az adatbázist
* A kliens nem csak egymástól független hívásokat csinál, hanem ténylegesen végre is lehet hajtani a felhasználói folyamatokat. Pl. Postman kliens esetében, nem csak különálló teszthívások vannak, hanem kollekciókba rendezve hívási sorozatok, ahol ez egyes hívások között változókban állapotot is tárolunk.
* Adatelérés: Entity Framework Core v8.x
* Kommunikáció: ASP.NET Core v8.x
    * Az előbb megadott verziókhoz képest későbbi verziók használhatók - saját felelősségre
* Minimum 3 összefüggő tábla használata, nem számolva a felhasználókezeléssel kapcsolatos táblákat
* A leadott specifikációnak megfelelő funkcionalitás
