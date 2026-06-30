# GNSS ja aikasynkronointi

Tämä dokumentti kuvaa GNSS-vastaanottimen (GPS/Galileo/GLONASS) roolia ja toteutusta **RF Intelligence Hub** -laitteessa. Dokumentti käsittelee aikasynkronoinnin tarvetta offline-olosuhteissa, tietosuojaa (GDPR) sekä virranhallinnan haasteita.

---

## 1. Käyttötarkoitus ja tarve

Laitteen ensisijainen tavoite on toimia kenttäkelpoisena ja itsenäisenä RF-sensorina ([design_principles.md:P1](file:///home/larry/Projects/rf-intelligence-hub/spec/design_principles.md#L41)). Monissa tapauksissa laite sijoitetaan alueille, joilla ei ole pääsyä Internetiin (NTP-aikapalvelimiin) tai yhteystunneliin mobiilisovelluksen kautta.

Aika on kuitenkin kriittinen havaintojen laadun kannalta:
- **Lokitietojen korrelaatio:** Useiden skannausten ja eri laitteiden keräämien tietojen vertailu vaatii sekunnintarkat aikaleimat.
- **Kelloaikaryömintä (Drift):** Laitteen sisäinen reaaliaikakello (RTC) ryömii ilman ulkoista synkronointia (tyypillisesti useita sekunteja tai minuutteja kuukaudessa riippuen lämpötilasta).
- **GNSS-ratkaisu:** GNSS-satelliittien lähettämä atomikellotarkka aikasignaali tarjoaa luotettavan tavan synkronoida laitteen RTC missä päin maailmaa tahansa ilman verkkoyhteyttä.

---

## 2. Tietosuoja ja GDPR (Privacy by Design)

Sijaintitiedon (GPS-koordinaatit) kerääminen ja tallentaminen muodostaa suuren GDPR-riskin ([gdpr.md:L15](file:///home/larry/Projects/rf-intelligence-hub/docs/gdpr.md#L15)), sillä se mahdollistaa laitteen kantajan liikkumisprofiilin seurannan.

Tämän vuoksi GNSS-vastaanotinta käytetään **ensisijaisesti vain aikasynkronointiin**:
- **Aika vs. Sijainti:** Laite lukee GNSS-moduulin NMEA-lauseista (esim. `$GPRMC` tai `$GPZDA`) ainoastaan UTC-ajan ja päivämäärän ja synkronoi laitteen sisäisen RTC-piirin.
- **Koordinaattien hylkääminen:** Sijaintikoordinaatit (leveys- ja pituusasteet) hylätään suoraan ohjelmistotasolla, eikä niitä kirjoiteta lokitiedostoihin tai välitetä eteenpäin.
- **Opt-in sijaintitiedolle:** Sijainnin lokitus otetaan käyttöön vain, jos käyttäjä erikseen aktivoi sen asetuksista (esimerkiksi tarkkaa RF-kartoitusta varten), ja silloinkin koordinaatit voidaan karkeistaa ([gdpr.md:L35](file:///home/larry/Projects/rf-intelligence-hub/docs/gdpr.md#L35)).

---

## 3. Virranhallinta (Power Management)

GNSS-vastaanottimet kuluttavat tyypillisesti huomattavan paljon virtaa aktiivisen seurannan aikana (noin 15–25 mA @ 3.3V). Tämä on liikaa laitteen "Always-on"-virtadomainille ([device.md:L137](file:///home/larry/Projects/rf-intelligence-hub/spec/device.md#L137)), jonka tavoite on toimia alle 100 mA kokonaiskulutuksella.

Tämän vuoksi GNSS-virranhallinta toteutetaan seuraavasti:
- **Jaksottainen synkronointi (Duty-cycling):** GNSS-moduuli pidetään oletusarvoisesti kokonaan sammutettuna (virta katkaistuna kuormakytkimellä tai EN-pinillä).
- **Synkronointisykli:** Moduuli käynnistetään vain kerran vuorokaudessa tai uuden RF-skannaussession alkaessa.
- **Pikasynkronointi (Hot/Warm Start):** Moduuli hakee satelliittiyhteyden (lock), päivittää RTC-kellon ja sammutetaan välittömästi sen jälkeen (kesto tyypillisesti alle 30 sekuntia, jos käytössä on RTC-varmistettu warm start).
- **Varmistusakku/Superkondensaattori:** Pieni apujännite pitää GNSS-moduulin sisäisen muistin (efemeridityypit) ja RTC-lohkon aktiivisena, mikä nopeuttaa seuraavaa käynnistystä (TTFF, Time-To-First-Fix).

---

## 4. Laitteistovaihtoehdot (Candidates)

Seuraavia erittäin vähävirtaisia ja pienikokoisia GNSS-moduuleja arvioidaan prototyyppivaiheessa:

| Moduuli | Liitäntä | Antenni | Huomiot |
|---|---|---|---|
| **Quectel L80-M39** | UART | Integroitu patch-antenni | Helppo layout, mutta patch-antenni vaatii tilaa ja hyvän maatason. |
| **u-blox MAX-M10** | I2C / UART | Ulkoinen passiivi/aktiivi | Erittäin pieni virrankulutus (~25mW), u-blox M10 -alustan korkea herkkyys. |
| **Gtop (GlobalTop) breakoutit** | I2C | Integroitu tai ulkoinen | Edullinen vaihtoehto testaukseen. |

### Layout-huomiot:
GNSS-antenni on sijoitettava riittävän kauas korkeataajuuksisista häiriölähteistä (kuten ESP32-C6:n WiFi/BLE-antenni ja Compute SoC:n hakkuriregulaattorit) riittävän herkkyyden varmistamiseksi.

---

## 5. Avoimet suunnittelupäätökset

1. **Moduulin valinta:** Valitaanko integroitu antenni (esim. L80-M39 tilan ja helppouden takia) vai erillinen siru ja keraaminen antenni (esim. u-blox MAX-M10 paremman herkkyyden ja pienemmän koon vuoksi)?
2. **Käynnistysaika kylmässä:** Miten kylmä käynnistys (Cold Start) ja aikalukituksen saaminen toimivat Finland-talviolosuhteissa (-25°C) lumisateessa?
3. **GNSS-avustus (A-GPS):** Siirretäänkö matkapuhelimesta (companion app) avustustietoja (almanakka/efemeridit) BLE:n yli laitteelle aikalukituksen nopeuttamiseksi (TTFF)?
