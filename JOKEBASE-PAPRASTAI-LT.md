# JokeBase paprastai

![Žmogus ir dirbtinis intelektas kuria mažą duomenų saugyklą iš užantspauduotos skaitmeninės dėžės; jos veikimą tikrina bandymai, nuotraukos ir archyvas.](assets/jokebase-be-kodo-iliustracija-v1.png)

## Vienu sakiniu

JokeBase yra mažas duomenų saugojimo įrankis, kurį kuriame laikydamiesi keistos taisyklės: nei žmogus, nei dirbtinis intelektas negali skaityti ar rašyti jo pradinio programos teksto.

## Kokia čia idėja?

Paprastai programos kuriamos taip: žmogus ar dirbtinis intelektas pakeičia suprantamą programos tekstą, kompiuteris jį paverčia vykdoma programa, o radus klaidą tas tekstas vėl perskaitomas ir pataisomas.

Su JokeBase taip nedarome.

Turime tik mažą vykdomą failą. Jis parašytas **WebAssembly** formatu. Paprastai tariant, tai yra ne konkretaus kompiuterio procesoriaus komandos, o kompaktiškas tarpinis formatas, kurį naršyklė ar kita vykdymo aplinka vėliau pritaiko konkrečiam kompiuteriui.

Failo viduje nėra žmogui skirto pradinio teksto, kurį būtų galima atsiversti ir taisyti įprastu būdu. Yra tik baitų seka – labai žemo lygio skaitmeniniai duomenys.

Todėl JokeBase yra ir pokštas, ir rimtas tyrimas.

Pokštas todėl, kad „duomenų bazė, kurios niekas negali normaliai prižiūrėti“ skamba absurdiškai.

Rimtas tyrimas todėl, kad klausiame: ar galima atsargiai tobulinti tokią nepermatomą programą, jei kiekvienas pokytis yra mažas, kiekvienas teiginys patikrinamas, o kiekviena versija ir nesėkmė lieka užfiksuota?

## Ką ji jau moka?

Dabartinė vieša versija vadinasi 19 seka. Ji nėra apsimetimas ir negrąžina iš anksto paruoštų atsakymų į kelis klausimus.

Ji iš tikrųjų saugo kintančią būseną:

- gali sukurti 12 iš anksto apibrėžtų mažų lentelių;
- gali įrašyti skaičius, tekstą ir tuščią reikšmę;
- gali tikrinti, ar reikšmė jau yra lentelėje;
- gali saugoti unikalias reikšmes ir atmesti pasikartojimus ten, kur jų būti negali;
- gali kopijuoti duomenis tarp tam tikrų lentelių;
- gali išsaugoti savo būsenos kopiją ir vėliau ją tiksliai atkurti;
- prieš atkurdama būseną patikrina, ar pateikta kopija nėra sugadinta ar nelogiška;
- turi aiškias ribas: kiekvienoje lentelėje telpa iki 64 eilučių, o tekstas gali būti iki 32 simbolių baitų ilgio.

Ji taip pat supranta vieną sunkesnę duomenų bazių idėją: kartais atsakymas nėra vien „taip“ ar „ne“. Jei duomenyse yra nežinoma reikšmė, atsakymas irgi gali būti „nežinoma“.

## Iš kur žinome, kad tai veikia?

Kadangi negalime pasitikėti perskaitytu programos tekstu, pasitikime tuo, ką programa parodo iš išorės.

Tikriname ją daugeliu skirtingų būdų:

- ji be klaidų įvykdė vieną visą viešą SQLite bandymų failą: 27 paruošimo veiksmus ir 187 užklausas;
- 10 000 atsitiktinai sudarytų klausimų buvo palyginti su plačiai naudojama SQLite duomenų baze ir atsakymai sutapo;
- 40 000 skirtingų veiksmų eilių tikrinta pagal atskirą elgesio modelį;
- 50 000 anksčiau išsaugotų klausimų pakartota be neatitikimų;
- tūkstančiai tyčia sugadintų išsaugotų būsenų buvo atmestos nepakeičiant tikrų duomenų;
- viešai paskelbto failo skaitmeninis pirštų atspaudas buvo parsisiųstas iš GitHub ir sutapatintas su mūsų įrašu.

Tai neįrodo, kad JokeBase moka viską arba kad joje niekada nebus klaidų. Tačiau tai duoda rimtą pagrindą teigti, kad ji veikia taip, kaip pažadėta, aiškiai nurodytose ribose.

## Kodėl tai taip sunku?

Paprastoje programoje randi eilutę, kuri atrodo įtartina, ją pakeiti ir paleidi bandymus.

Čia vienas neteisingas baitas gali padaryti taip, kad visa programa nebeužsikrauna. Dar blogiau: ji gali užsikrauti, bet tyliai duoti neteisingus atsakymus.

Todėl bandymai čia tampa ne paskutiniu patikrinimu, o pagrindiniu darbo įrankiu. Jie yra tarsi prožektorius, kuriuo apšviečiame uždarytos dėžės elgesį.

Kiekvienas žingsnis vyksta taip:

1. Aiškiai pasakome, ką turėtų pakeisti vienas mažas pokytis.
2. Pakeičiame tik mažą vykdomo failo dalį.
3. Patikriname, ar programa vis dar gali būti paleista.
4. Išbandome naują savybę ir senas artimas savybes.
5. Išbandome daug platesnius atvejus: atsitiktinius duomenis, sugadintas būsenas, pilnas lenteles ir neleidžiamas užklausas.
6. Užrašome, kas iš tikrųjų įvyko – net jei pradinis spėjimas buvo neteisingas.

## Kaip neprarandame darbo ir pamokų?

Kiekviena priimta versija turi unikalų skaitmeninį pirštų atspaudą. Ji išsaugoma kartu su ankstesne versija, todėl visada galima grįžti atgal.

Taip pat vedame papildomą, tik papildomą kūrimo žurnalą. Senų įrašų netaisome ir netriname. Jei vėliau paaiškėja, kad kažką supratome ne taip, parašome naują įrašą su pataisymu. Taip nesėkmės tampa žiniomis, o ne dingusia istorija.

Vieša saugykla GitHub yra ne tik vitrina, bet ir atsarginė kopija: joje yra vykdomas failas, jo pirštų atspaudai, bandymų duomenys, įrodymai ir ankstesnės versijos.

## Ko ši istorija neturėtų žadėti?

JokeBase nėra SQLite pakaitalas ir nėra paruošta tikriems svarbiems duomenims.

Ji nemoka laisvai kurti bet kokių lentelių, jungti lentelių, rūšiuoti, skaičiuoti suvestinių, saugiai dirbti keliems žmonėms vienu metu ar pati rašyti duomenų į kompiuterio diską.

Svarbu ir tai, kad negalime nuotoliniu būdu įrodyti pačios taisyklės „niekas neskaitė ir nerašė pradinio teksto“. Galime tik viešai parodyti saugyklos istoriją, taisykles ir visus paliktus pėdsakus.

## Svarbiausias klausimas

JokeBase nėra bandymas pasakyti: „žiūrėkite, dirbtinis intelektas gali išspjauti baitus“.

Klausimas yra gilesnis:

> Kiek aiškių bandymų, viešų įrodymų ir sąžiningai nurodytų ribų reikia, kad būtų protinga pasitikėti programa, kurios vidaus negalime perskaityti?

Štai kodėl vertingiausia čia yra ne vien mažoji duomenų bazė. Vertingiausia yra metodas, žurnalas ir įrodymai, kurie lieka po kiekvieno žingsnio.

## Kur rasti daugiau

- Vieša saugykla: https://github.com/bitkojine/JokeBase
- Išsamūs techniniai įrodymai anglų kalba: `JokeBase-v1-evidence.json`
- Papildomas kūrimo žurnalas anglų kalba: `DEVLOG.md`
- Naudoti įrankiai ir pamokos anglų kalba: `TOOLS-AND-LEARNINGS.md`
