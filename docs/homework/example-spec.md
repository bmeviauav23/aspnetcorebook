---
title: Példa kisHF specifikáció
---

# Kisdoktori generátor

## Feladat [^1]

Egy olyan alkalmazás készítése, mely törzsanyagokból disszertációt állít elő. Törzsanyagokat lehet felvinni a rendszerbe, egy szerkesztőfelületen megadni a generálás paramétereit, majd a generált dokumentumot különböző formátumokba menteni.

## A kisháziban elérhető funkciók [^2]

- törzsanyagok feltöltése (csak .txt formátumban), törlése - a törzsanyag témája megadható.
- egyszerű szerkesztőfelület - téma megadása, elvárt szószám
- egyszerű generálás - a téma alapján szövegek összeválogatása véletlenszerűen
- mentés XML formátumban, az egyes szövegrészekhez megadva, hogy melyik forrásműből származnak
- az előbbi XML betöltése és tartalmának megjelenítése (az egyes szövegrészekhez jelenjen meg, hogy melyik forrásműből való)

## Adatbázis entitások [^3]

- törzsanyag
- dolgozat
- forráshivatkozás

## Alkalmazott alaptechnológiák [^4]

- adatelérés: Entity Framework Core v8
- kommunikáció, szerveroldal: ASP.NET Core v8
- kliensoldal: Blazor WebAssembly

## Továbbfejlesztési tervek [^5]

- hosztolás Azure-ban
- HiLo elsődleges kulcs alkalmazása
- logikai törlés (soft delete) globális szűrőkkel
- OData szolgáltatás megvalósítása

[^1]: 2-3 mondatban foglald össze a feladatot!
[^2]: Adatmódosítással járó is legyen benne
[^3]: min. 3 db.
[^4]: a szerver oldal mindenkinek ugyanez lesz, kliensoldal választható. Verziószámok lehetnek nagyobbak, mint a lentiek
[^5]: opcionális, a pontrendszerből érdemes válogatni. Célja, hogy KHF bemutatáskor a felmerülő kérdéseket megbeszélhessük