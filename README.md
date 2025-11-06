# Freeview EPG Splitter


Fetch the Freeview XMLTV feed and publish **per-channel JSON** files you can point Home Assistant REST sensors at (with a compatibility block mimicking the old Freesat/Freeview card shape).

## Preview

![Home Assistant EPG card](https://github.com/elyobelyob/freely_tv_guide/blob/1cfc53f9a20dbb4fdef2301da48d4b39d1229a4f/docs/img/readme/Screenshot%202025-10-03%20at%2016.27.32.png)

## Home Assistant examples
Replace:

<CHANNEL_ID> with e.g. 37124

<SENSOR_NAME> with e.g. EPG_BBCOne

<sensor_name> in the Lovelace card with the entity id (lowercased by HA, e.g. sensor.epg_bbcone).

### REST sensor (drop in `sensors.yaml` or under `sensor:` in `configuration.yaml`)
```yaml
# Replace <CHANNEL_ID> with the JSON you want (e.g. 37124)
# Replace <SENSOR_NAME> if you want a different entity id
- platform: rest
  resource: "https://elyobelyob.github.io/freely_tv_guide_new/channels/<CHANNEL_ID>.json"
  scan_interval: 28800
  name: <SENSOR_NAME>
  value_template: >-
    {%- macro to_minutes(d) -%}
      {%- if d is number -%}{{ d }}
      {%- elif d is string and d.startswith('PT') -%}
        {% set h = ('H' in d) and (d.split('T')[-1].split('H')[0] | int) or 0 %}
        {% set m = ('M' in d) and (d.split('H')[-1].split('M')[0] | int) or 0 %}
        {% set s = ('S' in d) and (d.split('M')[-1].split('S')[0] | int) or 0 %}
        {{ h*60 + m + (s // 60) }}
      {%- else -%}0{%- endif -%}
    {%- endmacro -%}
    {% set evts = value_json.compat.freesat_card[0].event | default([], true) %}
    {% set nowts = now() %}
    {% for e in (evts | sort(attribute='startTime')) %}
      {% set st = as_local(as_datetime(e.startTime)) %}
      {% set en = st + timedelta(minutes=(to_minutes(e.duration) | int)) %}
      {% if en > nowts %}
        {{ st.strftime("%H:%M") }}  : {{ e.name }}
        
        {{ e.description or "" }}
        {% break %}
      {% endif %}
    {% else %}
      No EPG
    {% endfor %}
  json_attributes_path: "$.compat.freesat_card.0"
  json_attributes:
    - event
```

### Lovelace (featured programme with expandable schedule)
```yaml
type: vertical-stack
cards:
  - type: entities
    entities:
      - type: custom:hui-element
        card_type: markdown
        content: |-
          {% set evts = state_attr('sensor.<sensor_name>','event') or [] %}
          {% set nowts = now() %}
          {% set sorted = evts | sort(attribute='startTime') %}
          {% if not sorted %}
            No upcoming programmes
          {% else %}
            {% set featured = sorted[0] %}
            {% set f_start = as_local(as_datetime(featured.startTime)) %}
            {% set f_end = f_start + timedelta(minutes=featured.duration | int(0)) %}
            **{{ f_start.strftime("%H:%M") }} – {{ featured.name }}**

            {% if featured.description %}
            {{ featured.description }}
            {% endif %}

            {% if featured.synopsis and featured.synopsis != featured.description %}
            _{{ featured.synopsis }}_
            {% endif %}

            {% if featured.image %}
            ![{{ featured.name }}]({{ featured.image }})
            {% endif %}

            {% set remaining = sorted[1:] %}
            {% if remaining %}
            <details>
              <summary>Upcoming schedule (click to expand)</summary>
              {% for e in remaining %}
                {% set st = as_local(as_datetime(e.startTime)) %}
                {% set en = st + timedelta(minutes=e.duration | int(0)) %}
                - {{ st.strftime("%H:%M") }} – {{ e.name }}
              {% endfor %}
            </details>
            {% endif %}
          {% endif %}
```

## Channel list
<!-- CHANNELS_START -->
| ID | Name | JSON |
|---:|---|---|
| 4seven.uk | 4Seven | [channels/4seven.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/4seven.uk.json) |
| 5.uk | 5 HD | [channels/5.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5.uk.json) |
| 5USA.uk | 5 USA | [channels/5USA.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5USA.uk.json) |
| 5Plus1.uk | 5+1 | [channels/5Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5Plus1.uk.json) |
| 5Action.uk | 5ACTION | [channels/5Action.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5Action.uk.json) |
| 5Select.uk | 5SELECT | [channels/5Select.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5Select.uk.json) |
| 5Star.uk | 5STAR | [channels/5Star.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5Star.uk.json) |
| 5StarPlus1.uk | 5STAR+1 | [channels/5StarPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5StarPlus1.uk.json) |
| 5USAPlus1.uk | 5USA+1 | [channels/5USAPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/5USAPlus1.uk.json) |
| AlJazeeraArabic.qa | Al Jazeera Arabic | [channels/AlJazeeraArabic.qa.json](https://elyobelyob.github.io/freely_tv_guide/channels/AlJazeeraArabic.qa.json) |
| AlJazeeraEnglish.qa | Al Jazeera HD | [channels/AlJazeeraEnglish.qa.json](https://elyobelyob.github.io/freely_tv_guide/channels/AlJazeeraEnglish.qa.json) |
| ArirangTV.kr | Arirang TV | [channels/ArirangTV.kr.json](https://elyobelyob.github.io/freely_tv_guide/channels/ArirangTV.kr.json) |
| AriseNews.uk | Arise News | [channels/AriseNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/AriseNews.uk.json) |
| AsharqNews.uk | Asharq News | [channels/AsharqNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/AsharqNews.uk.json) |
| BBCAlba.uk | BBC ALBA HD | [channels/BBCAlba.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCAlba.uk.json) |
| BBCRAsianNetwork.uk | BBC Asian Network | [channels/BBCRAsianNetwork.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRAsianNetwork.uk.json) |
| BBCFourHD.uk | BBC Four HD | [channels/BBCFourHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCFourHD.uk.json) |
| BBCNews.uk | BBC NEWS HD | [channels/BBCNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCNews.uk.json) |
| BBCOneChannelIslands.uk | BBC One CI HD | [channels/BBCOneChannelIslands.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneChannelIslands.uk.json) |
| BBCOneEast.uk | BBC One East HD | [channels/BBCOneEast.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneEast.uk.json) |
| BBCOneEastMidlands.uk | BBC One East Midlands HD | [channels/BBCOneEastMidlands.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneEastMidlands.uk.json) |
| BBCOneLondonHD.uk | BBC One London HD | [channels/BBCOneLondonHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneLondonHD.uk.json) |
| BBCOneNorthernIreland.uk | BBC One NI HD | [channels/BBCOneNorthernIreland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneNorthernIreland.uk.json) |
| BBCOneNorthEastCumbria.uk | BBC One North East HD | [channels/BBCOneNorthEastCumbria.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneNorthEastCumbria.uk.json) |
| BBCOneNorthWest.uk | BBC One North West HD | [channels/BBCOneNorthWest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneNorthWest.uk.json) |
| BBCOneScotHD.uk | BBC One Scotland HD | [channels/BBCOneScotHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneScotHD.uk.json) |
| BBCOneSouthEast.uk | BBC One South East HD | [channels/BBCOneSouthEast.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneSouthEast.uk.json) |
| BBCOneSouth.uk | BBC One South HD | [channels/BBCOneSouth.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneSouth.uk.json) |
| BBCOneSouthWest.uk | BBC One South West HD | [channels/BBCOneSouthWest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneSouthWest.uk.json) |
| BBCOneWalesHD.uk | BBC One Wales HD | [channels/BBCOneWalesHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneWalesHD.uk.json) |
| BBCOneWest.uk | BBC One West HD | [channels/BBCOneWest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneWest.uk.json) |
| BBCOneWestMidlands.uk | BBC One West Midlands HD | [channels/BBCOneWestMidlands.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneWestMidlands.uk.json) |
| BBCOneYorksLincs.uk | BBC One Yorks & Lincs HD | [channels/BBCOneYorksLincs.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneYorksLincs.uk.json) |
| BBCOneYorkshire.uk | BBC One Yorkshire HD | [channels/BBCOneYorkshire.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCOneYorkshire.uk.json) |
| BBCParliament.uk | BBC Parliament HD | [channels/BBCParliament.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCParliament.uk.json) |
| BBCR1.uk | BBC Radio 1 | [channels/BBCR1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR1.uk.json) |
| BBCR1X.uk | BBC Radio 1 Xtra | [channels/BBCR1X.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR1X.uk.json) |
| BBCR2.uk | BBC Radio 2 | [channels/BBCR2.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR2.uk.json) |
| BBCR3.uk | BBC Radio 3 | [channels/BBCR3.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR3.uk.json) |
| BBCR4EX.uk | BBC Radio 4 Extra | [channels/BBCR4EX.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR4EX.uk.json) |
| BBCR4FM.uk | BBC Radio 4 FM | [channels/BBCR4FM.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR4FM.uk.json) |
| BBCR4LW.uk | BBC Radio 4 LW | [channels/BBCR4LW.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR4LW.uk.json) |
| BBCR5L.uk | BBC Radio 5 Live | [channels/BBCR5L.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR5L.uk.json) |
| BBCR5SX.uk | BBC Radio 5 Sports Xtra | [channels/BBCR5SX.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR5SX.uk.json) |
| BBCR6M.uk | BBC Radio 6 Music | [channels/BBCR6M.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCR6M.uk.json) |
| BBCRadioBerkshire.uk | BBC Radio Berkshire | [channels/BBCRadioBerkshire.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioBerkshire.uk.json) |
| BBCRadioCornwall.uk | BBC Radio Cornwall | [channels/BBCRadioCornwall.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioCornwall.uk.json) |
| BBCRadioCumbria.uk | BBC Radio Cumbria | [channels/BBCRadioCumbria.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioCumbria.uk.json) |
| BBCRadioCymru.uk | BBC Radio Cymru | [channels/BBCRadioCymru.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioCymru.uk.json) |
| BBCRadioCymru2.uk | BBC Radio Cymru 2 | [channels/BBCRadioCymru2.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioCymru2.uk.json) |
| BBCRadioFoyle.uk | BBC Radio Foyle | [channels/BBCRadioFoyle.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioFoyle.uk.json) |
| BBCRadioHumberside.uk | BBC Radio Humberside | [channels/BBCRadioHumberside.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioHumberside.uk.json) |
| BBCRadioKent.uk | BBC Radio Kent | [channels/BBCRadioKent.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioKent.uk.json) |
| BBCRadioLancashire.uk | BBC Radio Lancashire | [channels/BBCRadioLancashire.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioLancashire.uk.json) |
| BBCRadioLeeds.uk | BBC Radio Leeds | [channels/BBCRadioLeeds.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioLeeds.uk.json) |
| BBCRadioLincolnshire.uk | BBC Radio Lincolnshire | [channels/BBCRadioLincolnshire.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioLincolnshire.uk.json) |
| BBCRadioLondon.uk | BBC Radio London | [channels/BBCRadioLondon.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioLondon.uk.json) |
| BBCRadioManchester.uk | BBC Radio Manchester | [channels/BBCRadioManchester.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioManchester.uk.json) |
| BBCRadioNewcastle.uk | BBC Radio Newcastle | [channels/BBCRadioNewcastle.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioNewcastle.uk.json) |
| BBCRScotland.uk | BBC Radio Scotland | [channels/BBCRScotland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRScotland.uk.json) |
| BBCRadioSheffield.uk | BBC Radio Sheffield | [channels/BBCRadioSheffield.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioSheffield.uk.json) |
| BBCRadioSurrey.uk | BBC Radio Surrey | [channels/BBCRadioSurrey.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioSurrey.uk.json) |
| BBCRadioThreeCounties.uk | BBC Radio Three Counties | [channels/BBCRadioThreeCounties.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioThreeCounties.uk.json) |
| BBCRUlster.uk | BBC Radio Ulster | [channels/BBCRUlster.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRUlster.uk.json) |
| BBCRWales.uk | BBC Radio Wales | [channels/BBCRWales.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRWales.uk.json) |
| BBCRadioYork.uk | BBC Radio York | [channels/BBCRadioYork.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioYork.uk.json) |
| BBCRadioGaidhael.uk | BBC Radio nan Gaidhael | [channels/BBCRadioGaidhael.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRadioGaidhael.uk.json) |
| BBCRedButton1.uk | BBC Red Button 1 | [channels/BBCRedButton1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCRedButton1.uk.json) |
| BBCScotland.uk | BBC Scotland HD | [channels/BBCScotland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCScotland.uk.json) |
| BBCThreeHD.uk | BBC Three HD | [channels/BBCThreeHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCThreeHD.uk.json) |
| BBCTwoHD.uk | BBC Two HD | [channels/BBCTwoHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCTwoHD.uk.json) |
| BBCTwoNorthernIreland.uk | BBC Two NI HD | [channels/BBCTwoNorthernIreland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCTwoNorthernIreland.uk.json) |
| BBCTwoWales.uk | BBC Two Wales HD | [channels/BBCTwoWales.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCTwoWales.uk.json) |
| BBCWorldService.uk | BBC World Service | [channels/BBCWorldService.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BBCWorldService.uk.json) |
| BFBSRadio.uk | BFBS Radio | [channels/BFBSRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BFBSRadio.uk.json) |
| BestDirect.uk | Best Direct | [channels/BestDirect.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BestDirect.uk.json) |
| Blaze.uk | Blaze | [channels/Blaze.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Blaze.uk.json) |
| BloombergTVEurope.uk | Bloomberg HD | [channels/BloombergTVEurope.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/BloombergTVEurope.uk.json) |
| CBBC.uk | CBBC | [channels/CBBC.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CBBC.uk.json) |
| CBeebies.uk | CBeebies | [channels/CBeebies.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CBeebies.uk.json) |
| CNAInternational.sg | CNA | [channels/CNAInternational.sg.json](https://elyobelyob.github.io/freely_tv_guide/channels/CNAInternational.sg.json) |
| CNBCEurope.uk | CNBC HD | [channels/CNBCEurope.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CNBCEurope.uk.json) |
| CNNInternational.uk | CNN HD | [channels/CNNInternational.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CNNInternational.uk.json) |
| CapitalRadio.uk | Capital | [channels/CapitalRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CapitalRadio.uk.json) |
| CapitalXtra.uk | Capital Xtra | [channels/CapitalXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CapitalXtra.uk.json) |
| Challenge.uk | Challenge | [channels/Challenge.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Challenge.uk.json) |
| Challenge.ie | Challenge ROI | [channels/Challenge.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/Challenge.ie.json) |
| Channel4London.uk | Channel 4 London HD | [channels/Channel4London.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4London.uk.json) |
| Channel4Midlands.uk | Channel 4 Midlands HD | [channels/Channel4Midlands.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Midlands.uk.json) |
| Channel4North.uk | Channel 4 North HD | [channels/Channel4North.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4North.uk.json) |
| Channel4ROI.uk | Channel 4 ROI HD | [channels/Channel4ROI.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4ROI.uk.json) |
| Channel4Scotland.uk | Channel 4 Scotland HD | [channels/Channel4Scotland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Scotland.uk.json) |
| Channel4South.uk | Channel 4 South HD | [channels/Channel4South.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4South.uk.json) |
| Channel4Ulster.uk | Channel 4 Ulster HD | [channels/Channel4Ulster.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Ulster.uk.json) |
| Channel4Plus1London.uk | Channel 4+1 London | [channels/Channel4Plus1London.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Plus1London.uk.json) |
| Channel4Plus1North.uk | Channel 4+1 North | [channels/Channel4Plus1North.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Plus1North.uk.json) |
| Channel4Plus1ROI.uk | Channel 4+1 ROI | [channels/Channel4Plus1ROI.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Plus1ROI.uk.json) |
| Channel4Plus1Scotland.uk | Channel 4+1 Scotland | [channels/Channel4Plus1Scotland.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Plus1Scotland.uk.json) |
| Channel4Plus1South.uk | Channel 4+1 South | [channels/Channel4Plus1South.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channel4Plus1South.uk.json) |
| Channels24.ng | Channels 24 | [channels/Channels24.ng.json](https://elyobelyob.github.io/freely_tv_guide/channels/Channels24.ng.json) |
| ClassicFM.uk | Classic FM | [channels/ClassicFM.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ClassicFM.uk.json) |
| ClublandTV.uk | Clubland TV | [channels/ClublandTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ClublandTV.uk.json) |
| CourtTV.uk | Court TV | [channels/CourtTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/CourtTV.uk.json) |
| DMAX.uk | DMAX | [channels/DMAX.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/DMAX.uk.json) |
| DMAX.ie | DMAX ROI | [channels/DMAX.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/DMAX.ie.json) |
| DMAXUKPlus1.uk | DMAX+1 | [channels/DMAXUKPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/DMAXUKPlus1.uk.json) |
| E4.uk | E4 | [channels/E4.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4.uk.json) |
| E4Plus1.uk | E4 +1 | [channels/E4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4Plus1.uk.json) |
| E4Plus1.ie | E4 +1 ROI | [channels/E4Plus1.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4Plus1.ie.json) |
| E4Extra.uk | E4 Extra | [channels/E4Extra.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4Extra.uk.json) |
| E4HD.uk | E4 HD | [channels/E4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4HD.uk.json) |
| E4.ie | E4 ROI | [channels/E4.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/E4.ie.json) |
| EuronewsEnglish.fr | Euronews | [channels/EuronewsEnglish.fr.json](https://elyobelyob.github.io/freely_tv_guide/channels/EuronewsEnglish.fr.json) |
| Film4Plus1.uk | Film 4+1 | [channels/Film4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Film4Plus1.uk.json) |
| Film4.uk | Film4 | [channels/Film4.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Film4.uk.json) |
| Film4HD.uk | Film4 HD | [channels/Film4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Film4HD.uk.json) |
| Film4.ie | Film4 ROI | [channels/Film4.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/Film4.ie.json) |
| FoodNetwork.uk | Food Network | [channels/FoodNetwork.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/FoodNetwork.uk.json) |
| FoodNetwork.ie | Food Network ROI | [channels/FoodNetwork.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/FoodNetwork.ie.json) |
| FoodNetworkPlus1.uk | Food Network+1 | [channels/FoodNetworkPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/FoodNetworkPlus1.uk.json) |
| Foodxp.uk | Foodxp | [channels/Foodxp.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Foodxp.uk.json) |
| France24English.fr | France 24 HD | [channels/France24English.fr.json](https://elyobelyob.github.io/freely_tv_guide/channels/France24English.fr.json) |
| GBNews.uk | GB News HD | [channels/GBNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GBNews.uk.json) |
| GIGS.uk | GIGS | [channels/GIGS.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GIGS.uk.json) |
| GreatMoviesAction.uk | GREAT! action | [channels/GreatMoviesAction.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatMoviesAction.uk.json) |
| GreatMoviesActionPlus1.uk | GREAT! action+1 | [channels/GreatMoviesActionPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatMoviesActionPlus1.uk.json) |
| GreatMovies.uk | GREAT! movies | [channels/GreatMovies.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatMovies.uk.json) |
| GreatMoviesPlus1.uk | GREAT! movies+1 | [channels/GreatMoviesPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatMoviesPlus1.uk.json) |
| GreatMystery.uk | GREAT! mystery | [channels/GreatMystery.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatMystery.uk.json) |
| GreatReal.uk | GREAT! real | [channels/GreatReal.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatReal.uk.json) |
| GreatRealPlus1.uk | GREAT! real+1 | [channels/GreatRealPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatRealPlus1.uk.json) |
| GreatXmas.uk | GREAT! romance | [channels/GreatXmas.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatXmas.uk.json) |
| GreatXmasPlus1.uk | GREAT! romance+1 | [channels/GreatXmasPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatXmasPlus1.uk.json) |
| GreatTV.uk | GREAT! tv | [channels/GreatTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatTV.uk.json) |
| GreatTVPlus1.uk | GREAT! tv+1 | [channels/GreatTVPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GreatTVPlus1.uk.json) |
| GemsTV.uk | Gemporia TV | [channels/GemsTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GemsTV.uk.json) |
| GoldRadio.uk | Gold | [channels/GoldRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/GoldRadio.uk.json) |
| HGTV.uk | HGTV | [channels/HGTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HGTV.uk.json) |
| HGTV.ie | HGTV ROI | [channels/HGTV.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/HGTV.ie.json) |
| HGTVPlus1.uk | HGTV+1 | [channels/HGTVPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HGTVPlus1.uk.json) |
| Heart80s.uk | Heart 80s | [channels/Heart80s.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Heart80s.uk.json) |
| Heart90s.uk | Heart 90s | [channels/Heart90s.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Heart90s.uk.json) |
| HeartDance.uk | Heart Dance | [channels/HeartDance.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HeartDance.uk.json) |
| HeartRadio.uk | Heart Radio | [channels/HeartRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HeartRadio.uk.json) |
| HighStreetTV.uk | High Street TV | [channels/HighStreetTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HighStreetTV.uk.json) |
| HobbyMaker.uk | Hobby Maker HD | [channels/HobbyMaker.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/HobbyMaker.uk.json) |
| ITVQuiz.uk | ITV Quiz | [channels/ITVQuiz.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITVQuiz.uk.json) |
| ITV1Plus1Granada.uk | ITV+1 Granada | [channels/ITV1Plus1Granada.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Plus1Granada.uk.json) |
| ITV1Plus1London.uk | ITV+1 London | [channels/ITV1Plus1London.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Plus1London.uk.json) |
| ITV1Plus1Meridian.uk | ITV+1 Meridian | [channels/ITV1Plus1Meridian.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Plus1Meridian.uk.json) |
| ITV1Plus1Midlands.uk | ITV+1 Midlands | [channels/ITV1Plus1Midlands.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Plus1Midlands.uk.json) |
| ITV1AngliaWest.uk | ITV1 Anglia (West) HD | [channels/ITV1AngliaWest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1AngliaWest.uk.json) |
| ITV1Anglia.uk | ITV1 Anglia HD | [channels/ITV1Anglia.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Anglia.uk.json) |
| ITV1Border.uk | ITV1 Border HD | [channels/ITV1Border.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Border.uk.json) |
| ITV1CentralE.uk | ITV1 Central (East) HD | [channels/ITV1CentralE.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1CentralE.uk.json) |
| ITV1CentralW.uk | ITV1 Central (West) HD | [channels/ITV1CentralW.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1CentralW.uk.json) |
| ITV1Granada.uk | ITV1 Granada HD | [channels/ITV1Granada.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Granada.uk.json) |
| ITV1London.uk | ITV1 London HD | [channels/ITV1London.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1London.uk.json) |
| ITV1MeridianE.uk | ITV1 Meridian (East) HD | [channels/ITV1MeridianE.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1MeridianE.uk.json) |
| ITV1MeridianN.uk | ITV1 Meridian (North) HD | [channels/ITV1MeridianN.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1MeridianN.uk.json) |
| ITV1MeridianS.uk | ITV1 Meridian (South) HD | [channels/ITV1MeridianS.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1MeridianS.uk.json) |
| ITV1North.uk | ITV1 North HD | [channels/ITV1North.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1North.uk.json) |
| ITV1TyneTees.uk | ITV1 Tyne Tees HD | [channels/ITV1TyneTees.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1TyneTees.uk.json) |
| ITV1Wales.uk | ITV1 Wales HD | [channels/ITV1Wales.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1Wales.uk.json) |
| ITV1WestCountry.uk | ITV1 West Country HD | [channels/ITV1WestCountry.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1WestCountry.uk.json) |
| ITV1West.uk | ITV1 West HD | [channels/ITV1West.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1West.uk.json) |
| ITV1YorkshireEast.uk | ITV1 Yorkshire (East) HD | [channels/ITV1YorkshireEast.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1YorkshireEast.uk.json) |
| ITV1YorkshireWest.uk | ITV1 Yorkshire HD | [channels/ITV1YorkshireWest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV1YorkshireWest.uk.json) |
| ITV2.uk | ITV2 HD | [channels/ITV2.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV2.uk.json) |
| ITV2Plus1.uk | ITV2+1 | [channels/ITV2Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV2Plus1.uk.json) |
| ITV3.uk | ITV3 HD | [channels/ITV3.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV3.uk.json) |
| ITV3Plus1.uk | ITV3+1 | [channels/ITV3Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV3Plus1.uk.json) |
| ITV4.uk | ITV4 HD | [channels/ITV4.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV4.uk.json) |
| ITV4Plus1.uk | ITV4+1 | [channels/ITV4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ITV4Plus1.uk.json) |
| IdealWorld.uk | Ideal World HD | [channels/IdealWorld.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/IdealWorld.uk.json) |
| JewelleryMaker.uk | Jewellery Maker | [channels/JewelleryMaker.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/JewelleryMaker.uk.json) |
| LBCRadio.uk | LBC | [channels/LBCRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/LBCRadio.uk.json) |
| LBCNews.uk | LBC News | [channels/LBCNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/LBCNews.uk.json) |
| Legend.uk | Legend | [channels/Legend.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Legend.uk.json) |
| LegendXtra.uk | Legend Xtra | [channels/LegendXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/LegendXtra.uk.json) |
| LegendXtraPlus1.uk | Legend Xtra+1 | [channels/LegendXtraPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/LegendXtraPlus1.uk.json) |
| LondonLive.uk | London Live | [channels/LondonLive.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/LondonLive.uk.json) |
| More4.uk | More4 | [channels/More4.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/More4.uk.json) |
| More4HD.uk | More4 HD | [channels/More4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/More4HD.uk.json) |
| More4.ie | More4 ROI | [channels/More4.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/More4.ie.json) |
| More4Plus1.uk | More4+1 | [channels/More4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/More4Plus1.uk.json) |
| Movies24.uk | Movies24 | [channels/Movies24.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Movies24.uk.json) |
| Movies24Plus.uk | Movies24+ | [channels/Movies24Plus.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Movies24Plus.uk.json) |
| NDTVWorld.in | NDTV World | [channels/NDTVWorld.in.json](https://elyobelyob.github.io/freely_tv_guide/channels/NDTVWorld.in.json) |
| NHKWorldJapan.jp | NHK World HD | [channels/NHKWorldJapan.jp.json](https://elyobelyob.github.io/freely_tv_guide/channels/NHKWorldJapan.jp.json) |
| NewsmaxTV.uk | Newsmax | [channels/NewsmaxTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/NewsmaxTV.uk.json) |
| Now70s.uk | Now 70s | [channels/Now70s.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Now70s.uk.json) |
| Now80s.uk | Now 80s | [channels/Now80s.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Now80s.uk.json) |
| Now90s.uk | Now 90s & 00s | [channels/Now90s.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Now90s.uk.json) |
| NowRock.uk | Now Rock | [channels/NowRock.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/NowRock.uk.json) |
| PBSAmerica.uk | PBS America | [channels/PBSAmerica.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/PBSAmerica.uk.json) |
| PopPlus1.uk | POP+1 | [channels/PopPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/PopPlus1.uk.json) |
| Pop.uk | Pop | [channels/Pop.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Pop.uk.json) |
| QuestPlus1.uk | QUEST+1 | [channels/QuestPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestPlus1.uk.json) |
| QVCBeautyUK.uk | QVC Beauty | [channels/QVCBeautyUK.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QVCBeautyUK.uk.json) |
| QVCExtraUK.uk | QVC Extra | [channels/QVCExtraUK.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QVCExtraUK.uk.json) |
| QVCStyleUK.uk | QVC Style UK | [channels/QVCStyleUK.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QVCStyleUK.uk.json) |
| QVCUK.uk | QVC UK | [channels/QVCUK.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QVCUK.uk.json) |
| Quest.uk | Quest | [channels/Quest.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Quest.uk.json) |
| QuestHD.uk | Quest HD | [channels/QuestHD.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestHD.uk.json) |
| QuestHD.ie | Quest HD ROI | [channels/QuestHD.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestHD.ie.json) |
| QuestRed.uk | Quest Red | [channels/QuestRed.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestRed.uk.json) |
| QuestRed.ie | Quest Red ROI | [channels/QuestRed.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestRed.ie.json) |
| QuestRedPlus1.uk | Quest Red+1 | [channels/QuestRedPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/QuestRedPlus1.uk.json) |
| RTENews.ie | RTÉ News | [channels/RTENews.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTENews.ie.json) |
| RTEOne.ie | RTÉ One HD | [channels/RTEOne.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTEOne.ie.json) |
| RTEOnePlus1.ie | RTÉ One+1 | [channels/RTEOnePlus1.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTEOnePlus1.ie.json) |
| RTE2.ie | RTÉ2 HD | [channels/RTE2.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTE2.ie.json) |
| RTE2Plus1.ie | RTÉ2+1 | [channels/RTE2Plus1.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTE2Plus1.ie.json) |
| RTEjr.ie | RTÉjr | [channels/RTEjr.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/RTEjr.ie.json) |
| RadioX.uk | Radio X | [channels/RadioX.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/RadioX.uk.json) |
| Really.uk | Really | [channels/Really.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/Really.uk.json) |
| Really.ie | Really ROI | [channels/Really.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/Really.ie.json) |
| RewindTV.uk | Rewind TV | [channels/RewindTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/RewindTV.uk.json) |
| S4C.uk | S4C | [channels/S4C.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/S4C.uk.json) |
| STVCentralPlus1.uk | STV Central +1 | [channels/STVCentralPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/STVCentralPlus1.uk.json) |
| STVCentral.uk | STV Central HD | [channels/STVCentral.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/STVCentral.uk.json) |
| STVNorthPlus1.uk | STV North +1 | [channels/STVNorthPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/STVNorthPlus1.uk.json) |
| STVNorth.uk | STV North HD | [channels/STVNorth.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/STVNorth.uk.json) |
| SkyArts.uk | Sky Arts | [channels/SkyArts.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/SkyArts.uk.json) |
| SkyArts.ie | Sky Arts ROI | [channels/SkyArts.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/SkyArts.ie.json) |
| SkyMix.uk | Sky Mix HD | [channels/SkyMix.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/SkyMix.uk.json) |
| SkyMix.ie | Sky Mix HD ROI | [channels/SkyMix.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/SkyMix.ie.json) |
| SkyNews.uk | Sky News HD | [channels/SkyNews.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/SkyNews.uk.json) |
| SmoothRadio.uk | Smooth Radio | [channels/SmoothRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/SmoothRadio.uk.json) |
| TBNUK.uk | TBN UK | [channels/TBNUK.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TBNUK.uk.json) |
| TG4.ie | TG4 HD | [channels/TG4.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/TG4.ie.json) |
| TJC.uk | TJC HD | [channels/TJC.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TJC.uk.json) |
| TRTWorld.tr | TRT World | [channels/TRTWorld.tr.json](https://elyobelyob.github.io/freely_tv_guide/channels/TRTWorld.tr.json) |
| TVCNews.ng | TVC News | [channels/TVCNews.ng.json](https://elyobelyob.github.io/freely_tv_guide/channels/TVCNews.ng.json) |
| TalkingPicturesTV.uk | Talking Pictures TV | [channels/TalkingPicturesTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TalkingPicturesTV.uk.json) |
| ThatsDance.uk | That's Dance | [channels/ThatsDance.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsDance.uk.json) |
| ThatsFabulous.uk | That's Fabulous | [channels/ThatsFabulous.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsFabulous.uk.json) |
| ThatsMelody.uk | That's Melody | [channels/ThatsMelody.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsMelody.uk.json) |
| ThatsTV.uk | That's TV | [channels/ThatsTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsTV.uk.json) |
| ThatsTV2.uk | That's TV 2 | [channels/ThatsTV2.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsTV2.uk.json) |
| ThatsTV3.uk | That's TV 3 | [channels/ThatsTV3.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/ThatsTV3.uk.json) |
| TogetherTV.uk | Together TV | [channels/TogetherTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TogetherTV.uk.json) |
| Travelxp.in | Travelxp | [channels/Travelxp.in.json](https://elyobelyob.github.io/freely_tv_guide/channels/Travelxp.in.json) |
| TrueCrime.uk | True Crime | [channels/TrueCrime.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TrueCrime.uk.json) |
| TrueCrimeXtra.uk | True Crime Xtra | [channels/TrueCrimeXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TrueCrimeXtra.uk.json) |
| TrueCrimePlus1.uk | True Crime+1 | [channels/TrueCrimePlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TrueCrimePlus1.uk.json) |
| UAndDave.uk | U&Dave | [channels/UAndDave.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndDave.uk.json) |
| UAndDavejavu.uk | U&Dave ja vu | [channels/UAndDavejavu.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndDavejavu.uk.json) |
| UAndDrama.uk | U&Drama | [channels/UAndDrama.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndDrama.uk.json) |
| UAndDramaPlus1.uk | U&Drama+1 | [channels/UAndDramaPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndDramaPlus1.uk.json) |
| UAndEden.uk | U&Eden | [channels/UAndEden.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndEden.uk.json) |
| UAndEdenPlus1.uk | U&Eden+1 | [channels/UAndEdenPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndEdenPlus1.uk.json) |
| UAndW.uk | U&W | [channels/UAndW.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndW.uk.json) |
| UAndWPlus1.uk | U&W+1 | [channels/UAndWPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndWPlus1.uk.json) |
| UAndYesterday.uk | U&Yesterday | [channels/UAndYesterday.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndYesterday.uk.json) |
| UAndYesterdayPlus1.uk | U&Yesterday+1 | [channels/UAndYesterdayPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UAndYesterdayPlus1.uk.json) |
| UTV.uk | UTV | [channels/UTV.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/UTV.uk.json) |
| VirginMediaFour.ie | Virgin Media Four | [channels/VirginMediaFour.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginMediaFour.ie.json) |
| VirginMediaOne.ie | Virgin Media One | [channels/VirginMediaOne.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginMediaOne.ie.json) |
| VirginMediaOnePlus1.ie | Virgin Media One +1 | [channels/VirginMediaOnePlus1.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginMediaOnePlus1.ie.json) |
| VirginMediaThree.ie | Virgin Media Three | [channels/VirginMediaThree.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginMediaThree.ie.json) |
| VirginMediaTwo.ie | Virgin Media Two | [channels/VirginMediaTwo.ie.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginMediaTwo.ie.json) |
| VirginRadio.uk | Virgin Radio | [channels/VirginRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/VirginRadio.uk.json) |
| WION.in | WION HD | [channels/WION.in.json](https://elyobelyob.github.io/freely_tv_guide/channels/WION.in.json) |
| TalkSport.uk | talkSPORT | [channels/TalkSport.uk.json](https://elyobelyob.github.io/freely_tv_guide/channels/TalkSport.uk.json) |
<!-- CHANNELS_END -->

## Stats
See counts of channels and images here:  
https://elyobelyob.github.io/freely_tv_guide_new/stats/
