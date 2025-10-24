#  Zendure - Home Assistant
**Om in 2️⃣ stappen je batterij werkend te krijgen in Home Assistant**.

Gebaseerd op de [zenSDK RESTful API](https://github.com/Zendure/zenSDK) voor Home Assistant. Deze setup maakt lokaal verbinding met één Zendure Solarflow 2400 AC / Zendure Solarflow 800 Pro (geen aangesloten zonnepanelen) zonder gebruik te maken van integraties maar werkt met **één automatisering**. Voor de gene die graag de batterij 100% lokaal in eigen beheer wilt zonder updates van derden en netjes in Home Assistant.

Vind je dit project leuk en wil je mij steunen? Trakteer mij dan op een kopje koffie ☕️ – ik codeer beter met cafeïne!

<a href="https://www.buymeacoffee.com/gielz" target="_blank">
  <img src="https://github.com/Gielz1986/Zendure-zenSDK-HA/blob/main/Images/bmc.png?raw=true" width="180" alt="Buy Me a Coffee">
</a><br><br>

## 1️⃣ Configuration.yaml

> ⚠️ Let op: Zorg ervoor dat **HEMS is uitgeschakeld** in de Zendure-app.

Daarna gaan wij alles aanmaken voor de RESTful integratie (zit standaard in HA). Hiervoor heb ik een bijna plug-n-play Configuration.yaml gemaakt.
#### ℹ️ Benodigde hardware

- Homewizard P1 (of een andere P1/CT-meter die data per seconde levert (- wattage voor export en + voor import)).
- één Zendure Solarflow 2400 AC / Zendure Solarflow 800 Pro (geen aangesloten zonnepanelen).

---

### #️⃣ Configuratie en herstart

1. Maak eerst een **backup** van je `configuration.yaml`.
2. Pas daarna je `configuration.yaml` aan door gebruik te maken van de Github `configuration.yaml`.
3. Herstart Home Assistant.
5. Vul nu bij de onderstaande entiteiten de juiste gegevens in en herstart Home Assistant nogmaals.

| Configuratie| Extra info|
|-|-|
| `input_text.zendure_2400_ac_ip_adres`      | In de Zendure app onder device Information |
| `input_text.homewizard_p1_ip_adres`    | In de Homewizard app (lokale API aanzetten)  |
| `input_text.afwijkende_p1_sensor` | **Optioneel** een afwijkende P1 sensor toevoegen |
| `input_text.dynamisch_nordpool_sensor` | **Optioneel** de sensor van Nordpool toevoegen |

![Preview](Images/instellingen.png) 


*Zelf toe te voegen entiteiten op een dashboard.
![Preview](Images/Rest2.png) 

| Categorie              | Entiteiten                              | State                                                        |
| ---------------------- | -------------------------------------- | ------------------------------------------------------------ |
| Configuratie           | Homewizard P1 IP-adres                 | bijv. 192.168.0.192                                          |
|                        | Zendure 2400 AC IP-adres               | bijv. 192.168.0.172                                          |
|                        | Afwijkende P1 Sensor                   | bijv. sensor.eigen_P1                                        |
|                        | Dynamisch Nordpool Sensor              | bijv. sensor.nordpool_kwh_nl_eur_3_09_0                      |
| Aansturing             | Handmatig Vermogen                     | Gebruikt in modus Handmatig                                  |
|                        | Modus Selecteren                       | Zie **Modus uitleg bij ✅ Batterij mag aan de slag**        |
|                        | P1 Aansturing Vermogen                 | Homewizard P1 of eigen P1 vermogen                           |
| Informatie             | Aantal Batterijen                      | 1-6                                                          |
|                        | Serienummer                            | Serienummer Omvormer                                         |
|                        | Batterijspanning                       | Voltage                                                      |
|                        | Batterij 1-6 Temperatuur               | 0-100 °C                                                     |
|                        | Omvormer Temperatuur                   | 0-100 °C                                                     |
|                        | Laadpercentage                         | 5-100%                                                       |
|                        | Maximale Laadpercentage                | 5-100%                                                       |
|                        | Minimale Laadpercentage                | 5-100%                                                       |
|                        | Resterende Ontlaad Tijd                | uur - minuten                                                |
|                        | Opslagmodus                            | Opslaan in RAM of Opslaan in Flash                           |
| Vermogen & Energie     | Ingesteld Oplaadvermogen               | 0-2400 watt                                                  |
|                        | Ingesteld Ontlaadvermogen              | 0-2400 watt                                                  |
|                        | Vermogen Aansturing                    | -2400-2400 watt                                              |
|                        | Vermogen Import                        | 0-2400 watt                                                  |
|                        | Vermogen Export                        | 0-2400 watt                                                  |
|                        | Vermogen Import (DC)                   | 0-2400 watt                                                  |
|                        | Vermogen Export (DC)                   | 0-2400 watt                                                  |
|                        | Energie Import                         | KWH                                                          |
|                        | Energie Export                         | KWH                                                          |
| Status & Foutmeldingen | Modus                                  | Opladen of Ontladen                                          |
|                        | Error                                  | Geen meldingen of Zie Zendure APP                            |
|                        | Signaalsterkte                         | Uitstekend, Goed, Zwak of Slecht                             |
|                        | Indicatie Beschikbare Energie          | 0 - 16,42 kwh                                                |
|                        | SOC Status                             | Goed of Kalibreren                                           |
|                        | Relais Schakelingen Totaal Vandaag     | Beperk deze tot ±50 per dag. Bij bewolkte dagen ±100 per dag |
| Efficiëntie & RTE      | RTE Totaal                             | 0-100%                                                       |
|                        | Efficiëntie Import                     | 0-100%                                                       |
|                        | Efficiëntie Export                     | 0-100%                                                       |
|                        | Efficiëntie Import (24u gemiddelde)    | 0-100%                                                       |
|                        | Efficiëntie Export (24u gemiddelde)    | 0-100%                                                       |
| Dynamische Aansturing  | Dynamisch Nordpool                     | Nordpool prijzen in 15min en 1uur                            |
|                        | Dynamisch 15 Minuten                   | Prijzen in 15 minuten                                        |
|                        | Dynamisch Handmatige Periode           | bijv. G11:00;D12:00;G15:00 of 'Geen'                         |
|                        | Dynamisch Handmatige Periode Morgen    | bijv. G11:00;D12:00;G15:00 of 'Geen'                         |
|                        | Dynamisch Minimale Spread              | Gebruikt in modus Dynamisch NOM. Boven minimale spread laden |
|                        | Dynamisch Spread Indicatie             | Berekening spread                                            |
|                        | Dynamisch Spread Indicatie NOM         | Berekening spread NOM, duurste na eerste laadactie           |
|                        | Dynamisch Spread Indicatie Morgen      | Berekening spread                                            |
|                        | Dynamisch Spread Indicatie NOM Morgen  | Berekening spread NOM, duurste na eerste laadactie           |
|                        | Dynamisch Goedkoopste Periode          | Ja of Nee                                                    |
|                        | Dynamisch Duurste Periode              | Ja of Nee                                                    |
|                        | Dynamisch Goedkoopste X Periode        | 1-24                                                         |
|                        | Dynamisch Duurste X Periode            | 1-24                                                         |
|                        | Dynamisch Goedkoopste X Periode Morgen | 1-24                                                         |
|                        | Dynamisch Duurste X Periode Morgen     | 1-24                                                         |
|                        | Dynamisch Recent Geladen               | Gebruikt in modus Dynamisch NOM. Om te voorkomen dat er laadgedrag van 99>100>99>100 SOC ontstaat                                                      |


## 2️⃣ Zendure zenSDK (Gielz) automatisering
De motor van alles. Deze zal slim opladen en slim ontladen en samen dansen tot één geheel. Heb je bij het bovenstaande geen namen aangepast dan is het een kwestie van een nieuwe automatisering aanmaken.

1. Maak een nieuwe automatisering aan.
2. Klik rechtsboven op **Bewerken in YAML**.
3. Plak de YAML-code `(zie automatisering bestand)`.
4. Sla op, en start de automatisering.

![Preview](Images/Automation1.gif)   

![Preview](Images/Automation2.gif) 

<br><br>

## ✅ Batterij mag aan de slag
Het is dan eindelijk zo ver de batterij mag eens laten zien wat hij kan.

1. Voeg de entiteit **Zendure 2400 AC Modus Selecteren** toe aan je dashboard.
2. Voeg eventueel andere entiteiten toe die je via de `configuration.yaml` hebt aangemaakt.
3. De modus zal op **Standby** staan.
4. Kies hier je gewenste modus om de **Zendure zenSDK (Gielz) automatisering** te activeren.
5. De batterij zal nu aan de slag gaan.

![Preview](Images/Modus.gif) 

| Modus                         | Werking                                | 
| ----------------------------- | -------------------------------------- | 
| Standby                | Zet volledig in standby en `sensor.zendure_2400_ac_opslagmodus` zal naar **Opslaan in Flash** gaan. Hierdoor is het ook 0 watt op een KWH MID Meter.                                       |
| Handmatig              | Via `input_number.zendure_2400_ac_handmatig_vermogen` kun je zelf aangeven wat de batterij doet.       |
| Nul op de meter        | Constant 0 op de meter behouden (-40 watt bij opladen en -2 watt bij ontladen).                |
| Dynamisch NOM          | Wanneer `sensor.dynamisch_goedkoopste_periode` op JA staat zal er opgeladen worden indien `sensor.dynamisch_spread_indicatie_nom` hoger is dan `input_number.dynamisch_minimale_spread`. Wanneer `sensor.dynamisch_goedkoopste_periode` erg lang op JA staat zal hij pas weer gaan laden wanneer de batterij ontladen is tot 90%. Hij zal na laden constant 0 op de meter behouden (-40 watt bij opladen en -2 watt bij ontladen).                                        |
| Alleen slim ontladen   | Identiek aan **Nul op de meter** alleen zonder opladen.                                                |
| Alleen slim opladen    | Identiek aan **Nul op de meter** alleen zonder ontladen.                                               |
| Opladen met 2400 watt  | Snel opladen op maximaal vermogen.                                                                     |
| Ontladen met 2400 watt | Snel ontladen op maximaal vermogen.                                                                    |

<br><br>

## 🔃 (Optioneel) Nordpool
Wanneer je ook het Dynamisch Nordpool gedeelte in gebruik gaat nemen moet je voor dat je deze in gebruik neemt bij input_text.dynamisch_handmatige_periode en
input_text.dynamisch_handmatige_periode_morgen even unknown weghalen. Hierna zal het dynamisch gedeelte werken. Alles wat in de forecast gezet word zal overgenomen worden om 00:00 via de automatisering.

Dynamisch Goedkoopste Periode en Dynamisch Duurste Periode geven JA en NEE aan. deze kun je vervolgens in je eigen automatisering gebruiken waarmee je de modus van de batterij veranderd.

### #️⃣ Apexcharts
Je kunt om het visueel aantrekkelijk te maken de Apexcharts `Nordpool_Apexcharts_Vandaag` en `Nordpool_Apexcharts_Morgen` gebruiken `(zie Github bestanden)`.
