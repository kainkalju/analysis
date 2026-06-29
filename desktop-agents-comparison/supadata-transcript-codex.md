# Codex

Native transcript leitud; saadaval oli ainult ingliskeelne transcript.

**Eestikeelne kokkuvõte**

Video räägib turvauurijast Nightmare Eclipse, kes avalikustas mitu Microsofti toodete nullpäeva haavatavust ning kelle kontod eemaldati GitHubist ja hiljem ka GitLabist. Taustaks on konflikt Microsoft Security Response Centeri ja bug bounty reeglitega.

Põhiteema on “Yellow Key” nimeline BitLockeri möödapääs. Video selgitab, et see ei paista olevat otseselt BitLockeri krüptograafia viga, vaid pigem Windows Recovery Environmenti probleem. Ründeidee seisneb selles, et pahatahtliku USB abil kasutatakse transactional NTFS-i logisid, mis taastamiskeskkonnas käivitudes kustutavad `winpeshl.ini` faili. Selle tulemusena avaneb tavapärase taasteliidese asemel käsurida.

Kuna süsteem on selleks hetkeks juba boot-protsessi läbinud ja TPM on võtme operatsioonisüsteemile andnud, saab käsurealt ligi BitLockeriga kaitstud kettale. See tähendab, et rünne töötab olukorras, kus BitLocker kasutab TPM-i ilma PIN-ita.

Video järgi Windows 11 on tõenäoliselt haavatav, sest selle recovery environment käivitab automaatselt vastavad NTFS-i tehingud; Windows 10 puhul seda käitumist väidetavalt ei ole. Võimalike leevendustena mainitakse recovery environmentis automaatse FSTX-käitamise keelamist registrivõtme kaudu ning TPM PIN-i kasutamist.

Autor rõhutab siiski, et Nightmare Eclipse on väitnud, et ka TPM + PIN ei pruugi aidata, kuid vastavat PoC-d pole avaldatud. Seega praegu tundub Yellow Key pigem Windowsi taastamiskeskkonna veana kui BitLockeri või TPM-i enda krüptograafilise murdmisena.

