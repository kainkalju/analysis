# Claude

Vaatan supadata skilli ja tõmban transkriptsiooni.Sain transkriptsiooni kätte. Siin on eestikeelne kokkuvõte videost:

## Kokkuvõte: "Yellow Key" haavatavus ja BitLocker bypass

**Taustaks:** Video räägib turvateadlasest nimega Nightmare Eclipse, kes on varem avalikustanud mitmeid Microsofti zero-day haavatavusi ja sai seetõttu GitHubist (ja hiljem ka GitLabist) kontolt välja visatud. Tegu on tema vihaga Microsofti turvalisuse boonusprogrammi (MSRC) vastu, kuna Microsoft ei maksa paljude turvaprobleemide eest, mida teadlased peavad oluliseks (nt admin-õigustelt kerneli õigusteni jõudmine).

**Mis on Yellow Key:**
Yellow Key on äsja levinud exploit, mis võimaldab mööda minna BitLockeri (Windowsi kõvaketta krüpteerimise) kaitsest, kasutades ära nõrkust Windowsi taastekeskkonnas (Windows Recovery Environment).

**Kuidas see töötab:**
- BitLocker krüpteerib kõvaketta ja kasutab võtit, mis on seotud arvuti TPM-kiibiga (Trusted Platform Module), mis kinnitab krüptograafiliselt, et arvuti on "õiges" olekus (turvaline alglaadimine jne).
- Exploit kasutab ära NTFS-i (Windowsi failisüsteemi) "transaktsioonilist" omadust — funktsiooni, mis on mõeldud failisüsteemi vigade taastamiseks.
- Pahaloomuline USB-pulk sisaldab faili (FSTX), mis simuleerib transaktsiooni, mis kustutab `winpe.ini` faili, mis ütleb taastekeskkonnale, mida käivitada.
- Kui see fail kustutatakse, kukub süsteem vaikimisi käsurea kesta (shell) sisse — ja kuna TPM on selleks hetkeks juba dekrüpteerimisvõtme operatsioonisüsteemile loovutanud, saab ründaja ligipääsu dekrüpteeritud BitLocker partitsioonile.

**Tähtis nüanss:** Autori arvates ei ole see tegelikult viga BitLockeris ega TPM-is — TPM käitub täpselt nii, nagu peab. Tegelik viga on taastekeskkonnas, mis lubab neid failisüsteemi transaktsioone automaatselt käivitada (Windows 11-l see funktsioon on sees, Windows 10-l ei oli, mistõttu Windows 10 ei ole haavatav).

**Leevendusvõimalused:**
1. Registris keelata FSTX-binaari automaatne käivitamine taastekeskkonnas.
2. Lubada **TPM PIN** — kui kasutajal on PIN-kood, ei saa rünnak (väidetavalt) tavapärasel kujul töötada.

**Aga oodake:** Nightmare Eclipse väitis blogipostituses, et isegi TPM+PIN ei kaitse täielikult — exploit töötab tema sõnul ka PIN-iga kaitstud süsteemides, kuid ta ei avalikustanud, kuidas. See viitab eraldi, veel avalikustamata haavatavusele.

**Lõppjäreldus videos:** Autor ei ole kindel, kas see on tahtlik tagauks (backdoor) või lihtsalt viga — kaldub arvama, et tegu on bugiga, mis on eksisteerinud juba Windows Vista ajast, mitte tahtliku tagauksega.
