---
format: post
title: Van iCloud bug naar nieuwe iPhone
---

TL;DR versie: Voor niets nieuwe iPhone gekregen door totaal ongerelateerd probleem, yay!

---

*Ik woon in de VS, we hebben hier AT&T en Apple stores met genius bars.*

Een paar dagen terug kreeg ik een iPhone 4s op de werkbank die al maanden lang batterijproblemen had. Hij was binnen een uur of 4-6 leeg en sinds een paar dagen verbruikte het ding ook ongelofelijke hoeveelheden data (400MB/dag) terwijl de gebruiker hem gewoon op de tafel had liggen en niet aanraakte. Dit dataverbruik viel pas op toen de gebruiker thuis tijdelijk geen Wifi meer had door een verhuizing.

Na AT&T gebeld te hebben en hun instructies te hebben opgevolgd over het uitzetten van diagnostic reports, ophalen van email, afsluiten van background apps en het opnieuw aanmaken van iCloud en email accounts hebben we het ding eens een dag laten liggen en helemaal niets mee gedaan. Gevolg: ruwweg 300MB data verstookt, dat was het dus niet.

Volgende stap: We hangen het ding aan een mac met XCode en gaan eens naar de Console kijken. Daar staan wat interessante entries in:

```
Aug  2 14:53:20 unknown dataaccessd[52] <Notice>: xxxxxx|CardDAV|Error|Yikes: failedToFinishInitialSync:error: https://xxxx.xxxxxxx%40xxxxx.xx@p05-contacts.icloud.com/999999999/carddavhome/card/ Error Domain=CoreDAVErrorDomain Code=7 "The operation couldn’-t -b-e -c-o-m-p-l-e-t-e-d-. -(-C-o-r-e-D-A-V-E-r-r-o-r-D-o-m-a-i-n -e-r-r-o-r -7-.-)-"
Aug  2 14:53:20 unknown wifid[28] <Error>: WiFi:[xxxxxxxxx.xxxxxxx]: Client dataaccessd set type to background application
Aug  2 14:53:22 unknown dataaccessd[52] <Notice>: xxxxxx|CardDAV|Error|Yikes: failedToFinishInitialSync:error: https://xxxx.xxxxxxx%40xxxxx.xx@p05-contacts.icloud.com/999999999/carddavhome/card/ Error Domain=CoreDAVErrorDomain Code=7 "The operation couldn’-t -b-e -c-o-m-p-l-e-t-e-d-. -(-C-o-r-e-D-A-V-E-r-r-o-r-D-o-m-a-i-n -e-r-r-o-r -7-.-)-"
Aug  2 14:53:22 unknown wifid[28] <Error>: WiFi:[xxxxxxxxx.xxxxxxx]: Client dataaccessd set type to background application
```

Deze logregels herhaalden zich elke paar seconden. Er gaat overduidelijk iets mis met de iCloud sync, maar wat? iCloud uitzetten hielp, maar is natuurlijk niet echt een oplossing. iCloud account opnieuw instellen hielp niet en zelfs de telefoon compleet restoren had geen enkele zin. Op naar de genius bar!

Aangekomen bij de dichtsbijzijnde Apple store vragen we de lokale genius om naar het probleem te kijken. De genius hoort me even aan en controleert het serienummer van de iPhone. Zonder te aarzelen (en zonder naar het probleem te kijken) pakt de beste man een splinternieuwe iPhone 4s en begint de omruilprocedure. Bij navraag weet hij ook niet wat het probleem is maar hij is er zeker van dat een nieuwe telefoon het wel zal oplossen, wie ben ik om daar tegenin te gaan?

Een nieuwe iPhone later stellen we de iCloud account van de gebruiker in en alles werkt prima! Helaas blijkt na verdere inspectie dat maar enkele tientallen van de 386 contacten op de telefoon zijn aangekomen. Iets verder kijkend zien we dat de ~340 ontbrekende contacten ook niet op iCloud staan. De genius zet de contacten op de old-skool manier over vanaf de oude telefoon en we wachten even tot de telefoon alles gesynced heeft.

Tot onze grote verbazing synchroniseert de nieuwe iPhone slechts 20 contacten met iCloud en vervalt daarna direct in de lus uit de bovenstaande logs. Er is dus iets speciaals met de contacten aan de hand waar iCloud zich in verslikt. Dit zorgt voor een oneindige synchronisatie lus die de batterij leegzuigt en massaal data verbruikt!

Wat blijkt nu: De contacten op deze telefoon zijn ooit eens (door diezelfde Apple store) vanaf een Windows Mobile (HTC Diamond) telefoon naar de iPhone gekopieerd en er stonden een berg notities met Russische tekst bij. Die notities zijn tijdens dat proces corrupt geraakt en iCloud verslikt zich daar blijkbaar nogal in. Na het verwijderen van alle corrupte notities synchroniseert de telefoon prima en werkt alles naar behoren, met dien verstande dat de gebruiker wel met een nieuwe iPhone de winkel uitloopt terwijl er met de oorspronkelijke telefoon helemaal niets mis was.

Wie we moeten bedanken voor de nieuwe telefoon is niet helemaal duidelijk, maar bedankt Apple/Microsoft/HTC!