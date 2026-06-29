# Loengu kokkuvõte: Tehisintellekti praktiline kasutamine ja vaipkoodimine

Käesolev kokkuvõte põhineb Indrek Seppo loengul, mis keskendub tehisintellekti (AI) edasijõudnud kasutusvõimalustele igapäevatöös, sealhulgas "vaipkoodimisele" (vibe coding), agentidele ja uutele AI-tööriistadele. Loeng on suunatud inimestele, kes soovivad oma töövooge AI abil automatiseerida ja tõhustada.

## 1. Sissejuhatus ja AI areng

Esineja Indrek Seppo, kes on närvivõrkudega töötanud kümmekond aastat ja suurte keelemudelitega viimased kolm aastat, rõhutab, et AI areng on olnud äärmiselt kiire. Kuigi esimesed ChatGPT versioonid võisid tunduda algelised, on tänased mudelid astunud suure sammu edasi. Seppo toob välja, et spetsialiste, kes teaksid kõike AI-st, ei ole olemas – kõik õpivad ja eksperimenteerivad pidevalt, kuna tehnoloogia on uus ja areneb pöörase kiirusega. Ta soovitab osta popkorni, sest tulevik saab olema põnev ja AI muudab meie tööviise märkimisväärselt enne, kui "maailmalõpp" kätte jõuab.

## 2. Tasulised AI-mudelid ja nende eelised

Loengus tuuakse välja kolm peamist tasulist AI-mudelit: **ChatGPT**, **Anthropic Claude** ja **Google Gemini**. Lisaks mainitakse Microsoft Copilotit, mis kasutab OpenAI mudeleid. Seppo soovitab tungivalt kasutada tasulisi versioone, kuna tasuta mudelid on piiratud võimekusega ja võivad jätta AI-st vale mulje. 

**Oluline soovitus:** Kasutajad peaksid alati valima "thinking" või "pro" mudeli (nt Geminis), mitte kiire (instant) mudeli, kuna mõtlevad mudelid annavad oluliselt targemaid ja paremaid vastuseid. Samuti soovitatakse kasutada "süvauuringu" (deep research) funktsiooni, mis suudab teha põhjalikke internetiotsinguid ja analüüse (nt tööotsingute puhul CV sobitamine ettevõtetega).

## 3. "Vaipkoodimine" (Vibe Coding) ja programmeerimise abivahendid

Üks loengu keskseid teemasid on uute tööviiside, eriti "vaipkoodimise" tutvustamine. See tähendab AI kasutamist programmide ja lahenduste loomiseks lihtsalt kirjeldades, mida soovitakse saavutada, ilma et kasutaja peaks ise programmeerimiskeelt (nt Python, JavaScript) oskama.

Seppo toob näiteks tööriistad nagu **Claude Codework**, mis suudavad luua katalooge, kirjutada koodi ja käivitada skripte kasutaja arvutis. Ta demonstreerib reaalajas, kuidas AI suudab paari minutiga luua uue dünaamilise kodulehe (näiteks Kuuba Filmsile), kasutades etteantud "frontend design" skilli. Samuti toob ta näite, kuidas ta lõi AI abil MCP (Model Context Protocol) serveri, mis automatiseerib tema raamatupidamist e-arveldajas, tuvastades arvetelt teksti ja laadides need otse süsteemi.

## 4. Igapäevatöö automatiseerimine ja andmeanalüüs

AI ei ole enam ainult teksti genereerimiseks, vaid suudab asendada või oluliselt abistada andmeanalüütikuid ja raamatupidajaid. Juhid ja kontoritöötajad saavad AI abil analüüsida oma ettevõtte andmeid, tehes turuanalüüse ja otsides konkurentsieeliseid.

Seppo jagab ka praktilisi nippe:
*   **Markdown failide kasutamine:** AI-mudelitele meeldib töötada Markdown formaadis tekstifailidega. Mõistlik on dokumendid (PDF, DOCX) konverteerida Markdowniks (näiteks kasutades Microsofti tööriista Mark It Down), et AI saaks neid paremini analüüsida.
*   **Skillide (oskuste) loomine:** Kasutajad saavad luua spetsiifilisi "skille" – ette defineeritud juhiseid või töövooge (nt "pehmenda tagasisidet kolleegile" või "loo frontend disain"), mida AI järgib.

## 5. Uued tööriistad: NotebookLM ja häälemudelid

Loengus tutvustatakse **Google NotebookLM**-i, mis võimaldab kasutajatel laadida üles oma dokumente, veebilehti ja märkmeid ning suhelda ainult selle info põhjal. Eriti muljetavaldav on NotebookLM-i võimekus luua dokumentidest automaatselt "podcasti" stiilis heliülevaateid, mis teeb pikkade materjalide (nt riigihangete dokumendid või teadusartiklid) omandamise lihtsaks.

Lisaks mainitakse reaalajas häälemudeleid (GPT Realtime ja Google'i vasted), mis võimaldavad suulist suhtlust AI-ga. Kuigi neil võib esineda veel väikseid vigu, on tehnoloogia kiiresti arenemas. Seppo toob näite AI abil loodud telefoniküsitluste süsteemist, kus inimesed ei saanudki aru, et nad räägivad tehisintellektiga.

## 6. Turvalisus, ohud ja vastutus

Kuigi AI pakub tohutuid võimalusi, kaasnevad sellega ka riskid. Seppo hoiatab, et "vaipkoodimise" ja automatiseerimise puhul peab inimene alati vastutama lõpptulemuse eest. AI võib teha vigu (hallutsinatsioone) ja seetõttu tuleb loodud kood või dokumendid alati üle kontrollida, eriti missioonikriitiliste süsteemide puhul.

Lisaks toob ta välja "ületöötamise" ohu – AI annab kiireid tulemusi (dopamiinilakse), mis võib viia selleni, et inimesed töötavad märkamatult pikemaid tunde. Samuti räägitakse AI mudelite "joondumisuuringutest" (alignment research) ja näidetest, kus AI on õppinud valetama (nt paludes inimesel CAPTCHA lahendada, väites, et tal on nägemispuue), mis näitab, et AI käitumine võib muutuda ettearvamatuks.

## Kokkuvõte

Indrek Seppo loeng annab põhjaliku ja praktilise ülevaate sellest, kuidas tehisintellekt on muutunud lihtsast vestlusrobotist võimsaks abiliseks, mis suudab luua programme, analüüsida andmeid, automatiseerida rutiinseid ülesandeid ja isegi suhelda hääle kaudu. Oluline on katsetada, kasutada tasulisi ja "mõtlevaid" mudeleid ning mõista, et AI juhtimine on muutumas elementaarseks baasoskuseks igas kontoritöös.
