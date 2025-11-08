# Freeview EPG Splitter

Fetch the Freeview XMLTV feed and publish **per-channel JSON** files you can point Home Assistant REST sensors at (with a compatibility block mimicking the old Freesat/Freeview card shape).

## Preview

![Home Assistant EPG card](https://github.com/elyobelyob/freely_tv_guide/blob/1cfc53f9a20dbb4fdef2301da48d4b39d1229a4f/docs/img/readme/Screenshot%202025-10-03%20at%2016.27.32.png)

## Home Assistant Integration

### Quick Start
Replace the following placeholders in the examples below:
- `<CHANNEL_ID>` with the channel ID from the [Channel list](#channel-list) (e.g., `BBCNews.uk`)
- `<SENSOR_NAME>` with your desired sensor name (e.g., `EPG_BBCNews`)
- `<sensor_name>` with the entity ID (lowercased by Home Assistant, e.g., `sensor.epg_bbcnews`)

### REST Sensor Configuration

Add this to your `sensors.yaml` or under `sensor:` in `configuration.yaml`:

```yaml
# Replace <CHANNEL_ID> with the channel ID (e.g., BBCNews.uk)
# Replace <SENSOR_NAME> with your desired sensor name (e.g., EPG_BBCNews)
- platform: rest
  resource: "https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/docs/channels/<CHANNEL_ID>.json"
  scan_interval: 28800  # 8 hours
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

### Lovelace (compact today's schedule view)

Shows unique programme names for the rest of today (until 6am next day), removing duplicates for long-running or repeated shows:

```yaml
type: vertical-stack
cards:
  - type: entities
    entities:
      - type: custom:hui-element
        card_type: markdown
        content: >-
          {% set evts = state_attr('sensor.epg_bbc_news', 'event') or [] %}
          {% set nowts = now() %}
          {% set cutoff = now().replace(hour=6, minute=0, second=0, microsecond=0) + timedelta(days=1) %}
          {% set ns = namespace(prev_name='', count=0, shown=0, out='') %}
          {% set limit = 20 %}

          {% for e in evts | sort(attribute='startTime') %}
            {% if ns.shown >= limit %}
              {% break %}
            {% endif %}

            {% set st = as_local(as_datetime(e.startTime)) %}
            {% set en = st + timedelta(minutes=(e.duration | int(0))) %}

            {% if en > nowts and st < cutoff %}
              {% if ns.prev_name != e.name %}
                {% if ns.prev_name != '' %}
                  {% set ns.out = ns.out + '<br>' %}
                {% endif %}
                {% set ns.out = ns.out + st.strftime("%H:%M") ~ ' : ' ~ e.name %}
                {% set ns.prev_name = e.name %}
                {% set ns.count = ns.count + 1 %}
                {% set ns.shown = ns.shown + 1 %}
              {% endif %}
            {% endif %}
          {% endfor %}

          {% if ns.count == 0 %}
            No programmes scheduled for the rest of today.
          {% else %}
            {{ ns.out | safe }}
          {% endif %}
```

**Note:** Replace `sensor.epg_bbc_news` with your actual sensor entity ID (e.g., `sensor.epg_bbcnews`). Adjust the `limit` variable (default: 20) to show more or fewer programmes.

### Lovelace (featured programme with expandable schedule)

Displays the currently airing or next programme with full details (description, synopsis, image) and an expandable list of upcoming programmes:

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
| ID | Name | Logo | JSON |
|---:|---|:---:|---|
| 4seven.uk | 4Seven | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="4Seven logo" width="40"> | [channels/4seven.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/4seven.uk.json) |
| 5.uk | 5 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5 HD logo" width="40"> | [channels/5.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5.uk.json) |
| 5USA.uk | 5 USA | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5 USA logo" width="40"> | [channels/5USA.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5USA.uk.json) |
| 5Plus1.uk | 5+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5+1 logo" width="40"> | [channels/5Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5Plus1.uk.json) |
| 5Action.uk | 5ACTION | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5ACTION logo" width="40"> | [channels/5Action.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5Action.uk.json) |
| 5Select.uk | 5SELECT | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5SELECT logo" width="40"> | [channels/5Select.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5Select.uk.json) |
| 5Star.uk | 5STAR | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5STAR logo" width="40"> | [channels/5Star.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5Star.uk.json) |
| 5StarPlus1.uk | 5STAR+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5STAR+1 logo" width="40"> | [channels/5StarPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5StarPlus1.uk.json) |
| 5USAPlus1.uk | 5USA+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="5USA+1 logo" width="40"> | [channels/5USAPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/5USAPlus1.uk.json) |
| AlJazeeraArabic.qa | Al Jazeera Arabic | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Al Jazeera Arabic logo" width="40"> | [channels/AlJazeeraArabic.qa.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/AlJazeeraArabic.qa.json) |
| AlJazeeraEnglish.qa | Al Jazeera HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Al Jazeera HD logo" width="40"> | [channels/AlJazeeraEnglish.qa.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/AlJazeeraEnglish.qa.json) |
| ArirangTV.kr | Arirang TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Arirang TV logo" width="40"> | [channels/ArirangTV.kr.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ArirangTV.kr.json) |
| AriseNews.uk | Arise News | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Arise News logo" width="40"> | [channels/AriseNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/AriseNews.uk.json) |
| AsharqNews.uk | Asharq News | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Asharq News logo" width="40"> | [channels/AsharqNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/AsharqNews.uk.json) |
| BBCAlba.uk | BBC ALBA HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC ALBA HD logo" width="40"> | [channels/BBCAlba.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCAlba.uk.json) |
| BBCRAsianNetwork.uk | BBC Asian Network | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Asian Network logo" width="40"> | [channels/BBCRAsianNetwork.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRAsianNetwork.uk.json) |
| BBCFourHD.uk | BBC Four HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Four HD logo" width="40"> | [channels/BBCFourHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCFourHD.uk.json) |
| BBCNews.uk | BBC NEWS HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC NEWS HD logo" width="40"> | [channels/BBCNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCNews.uk.json) |
| BBCOneChannelIslands.uk | BBC One CI HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One CI HD logo" width="40"> | [channels/BBCOneChannelIslands.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneChannelIslands.uk.json) |
| BBCOneEast.uk | BBC One East HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One East HD logo" width="40"> | [channels/BBCOneEast.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneEast.uk.json) |
| BBCOneEastMidlands.uk | BBC One East Midlands HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One East Midlands HD logo" width="40"> | [channels/BBCOneEastMidlands.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneEastMidlands.uk.json) |
| BBCOneLondonHD.uk | BBC One London HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One London HD logo" width="40"> | [channels/BBCOneLondonHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneLondonHD.uk.json) |
| BBCOneNorthernIreland.uk | BBC One NI HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One NI HD logo" width="40"> | [channels/BBCOneNorthernIreland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneNorthernIreland.uk.json) |
| BBCOneNorthEastCumbria.uk | BBC One North East HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One North East HD logo" width="40"> | [channels/BBCOneNorthEastCumbria.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneNorthEastCumbria.uk.json) |
| BBCOneNorthWest.uk | BBC One North West HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One North West HD logo" width="40"> | [channels/BBCOneNorthWest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneNorthWest.uk.json) |
| BBCOneScotHD.uk | BBC One Scotland HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One Scotland HD logo" width="40"> | [channels/BBCOneScotHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneScotHD.uk.json) |
| BBCOneSouthEast.uk | BBC One South East HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One South East HD logo" width="40"> | [channels/BBCOneSouthEast.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneSouthEast.uk.json) |
| BBCOneSouth.uk | BBC One South HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One South HD logo" width="40"> | [channels/BBCOneSouth.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneSouth.uk.json) |
| BBCOneSouthWest.uk | BBC One South West HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One South West HD logo" width="40"> | [channels/BBCOneSouthWest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneSouthWest.uk.json) |
| BBCOneWalesHD.uk | BBC One Wales HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One Wales HD logo" width="40"> | [channels/BBCOneWalesHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneWalesHD.uk.json) |
| BBCOneWest.uk | BBC One West HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One West HD logo" width="40"> | [channels/BBCOneWest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneWest.uk.json) |
| BBCOneWestMidlands.uk | BBC One West Midlands HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One West Midlands HD logo" width="40"> | [channels/BBCOneWestMidlands.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneWestMidlands.uk.json) |
| BBCOneYorksLincs.uk | BBC One Yorks & Lincs HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One Yorks & Lincs HD logo" width="40"> | [channels/BBCOneYorksLincs.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneYorksLincs.uk.json) |
| BBCOneYorkshire.uk | BBC One Yorkshire HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC One Yorkshire HD logo" width="40"> | [channels/BBCOneYorkshire.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCOneYorkshire.uk.json) |
| BBCParliament.uk | BBC Parliament HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Parliament HD logo" width="40"> | [channels/BBCParliament.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCParliament.uk.json) |
| BBCR1.uk | BBC Radio 1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 1 logo" width="40"> | [channels/BBCR1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR1.uk.json) |
| BBCR1X.uk | BBC Radio 1 Xtra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 1 Xtra logo" width="40"> | [channels/BBCR1X.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR1X.uk.json) |
| BBCR2.uk | BBC Radio 2 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 2 logo" width="40"> | [channels/BBCR2.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR2.uk.json) |
| BBCR3.uk | BBC Radio 3 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 3 logo" width="40"> | [channels/BBCR3.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR3.uk.json) |
| BBCR4EX.uk | BBC Radio 4 Extra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 4 Extra logo" width="40"> | [channels/BBCR4EX.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR4EX.uk.json) |
| BBCR4FM.uk | BBC Radio 4 FM | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 4 FM logo" width="40"> | [channels/BBCR4FM.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR4FM.uk.json) |
| BBCR4LW.uk | BBC Radio 4 LW | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 4 LW logo" width="40"> | [channels/BBCR4LW.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR4LW.uk.json) |
| BBCR5L.uk | BBC Radio 5 Live | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 5 Live logo" width="40"> | [channels/BBCR5L.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR5L.uk.json) |
| BBCR5SX.uk | BBC Radio 5 Sports Xtra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 5 Sports Xtra logo" width="40"> | [channels/BBCR5SX.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR5SX.uk.json) |
| BBCR6M.uk | BBC Radio 6 Music | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio 6 Music logo" width="40"> | [channels/BBCR6M.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCR6M.uk.json) |
| BBCRadioBerkshire.uk | BBC Radio Berkshire | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Berkshire logo" width="40"> | [channels/BBCRadioBerkshire.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioBerkshire.uk.json) |
| BBCRadioCornwall.uk | BBC Radio Cornwall | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Cornwall logo" width="40"> | [channels/BBCRadioCornwall.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioCornwall.uk.json) |
| BBCRadioCumbria.uk | BBC Radio Cumbria | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Cumbria logo" width="40"> | [channels/BBCRadioCumbria.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioCumbria.uk.json) |
| BBCRadioCymru.uk | BBC Radio Cymru | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Cymru logo" width="40"> | [channels/BBCRadioCymru.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioCymru.uk.json) |
| BBCRadioCymru2.uk | BBC Radio Cymru 2 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Cymru 2 logo" width="40"> | [channels/BBCRadioCymru2.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioCymru2.uk.json) |
| BBCRadioFoyle.uk | BBC Radio Foyle | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Foyle logo" width="40"> | [channels/BBCRadioFoyle.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioFoyle.uk.json) |
| BBCRadioHumberside.uk | BBC Radio Humberside | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Humberside logo" width="40"> | [channels/BBCRadioHumberside.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioHumberside.uk.json) |
| BBCRadioKent.uk | BBC Radio Kent | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Kent logo" width="40"> | [channels/BBCRadioKent.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioKent.uk.json) |
| BBCRadioLancashire.uk | BBC Radio Lancashire | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Lancashire logo" width="40"> | [channels/BBCRadioLancashire.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioLancashire.uk.json) |
| BBCRadioLeeds.uk | BBC Radio Leeds | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Leeds logo" width="40"> | [channels/BBCRadioLeeds.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioLeeds.uk.json) |
| BBCRadioLincolnshire.uk | BBC Radio Lincolnshire | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Lincolnshire logo" width="40"> | [channels/BBCRadioLincolnshire.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioLincolnshire.uk.json) |
| BBCRadioLondon.uk | BBC Radio London | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio London logo" width="40"> | [channels/BBCRadioLondon.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioLondon.uk.json) |
| BBCRadioManchester.uk | BBC Radio Manchester | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Manchester logo" width="40"> | [channels/BBCRadioManchester.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioManchester.uk.json) |
| BBCRadioNewcastle.uk | BBC Radio Newcastle | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Newcastle logo" width="40"> | [channels/BBCRadioNewcastle.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioNewcastle.uk.json) |
| BBCRScotland.uk | BBC Radio Scotland | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Scotland logo" width="40"> | [channels/BBCRScotland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRScotland.uk.json) |
| BBCRadioSheffield.uk | BBC Radio Sheffield | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Sheffield logo" width="40"> | [channels/BBCRadioSheffield.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioSheffield.uk.json) |
| BBCRadioSurrey.uk | BBC Radio Surrey | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Surrey logo" width="40"> | [channels/BBCRadioSurrey.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioSurrey.uk.json) |
| BBCRadioThreeCounties.uk | BBC Radio Three Counties | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Three Counties logo" width="40"> | [channels/BBCRadioThreeCounties.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioThreeCounties.uk.json) |
| BBCRUlster.uk | BBC Radio Ulster | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Ulster logo" width="40"> | [channels/BBCRUlster.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRUlster.uk.json) |
| BBCRWales.uk | BBC Radio Wales | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio Wales logo" width="40"> | [channels/BBCRWales.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRWales.uk.json) |
| BBCRadioYork.uk | BBC Radio York | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio York logo" width="40"> | [channels/BBCRadioYork.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioYork.uk.json) |
| BBCRadioGaidhael.uk | BBC Radio nan Gaidhael | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Radio nan Gaidhael logo" width="40"> | [channels/BBCRadioGaidhael.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRadioGaidhael.uk.json) |
| BBCRedButton1.uk | BBC Red Button 1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Red Button 1 logo" width="40"> | [channels/BBCRedButton1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCRedButton1.uk.json) |
| BBCScotland.uk | BBC Scotland HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Scotland HD logo" width="40"> | [channels/BBCScotland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCScotland.uk.json) |
| BBCThreeHD.uk | BBC Three HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Three HD logo" width="40"> | [channels/BBCThreeHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCThreeHD.uk.json) |
| BBCTwoHD.uk | BBC Two HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Two HD logo" width="40"> | [channels/BBCTwoHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCTwoHD.uk.json) |
| BBCTwoNorthernIreland.uk | BBC Two NI HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Two NI HD logo" width="40"> | [channels/BBCTwoNorthernIreland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCTwoNorthernIreland.uk.json) |
| BBCTwoWales.uk | BBC Two Wales HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC Two Wales HD logo" width="40"> | [channels/BBCTwoWales.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCTwoWales.uk.json) |
| BBCWorldService.uk | BBC World Service | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BBC World Service logo" width="40"> | [channels/BBCWorldService.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BBCWorldService.uk.json) |
| BFBSRadio.uk | BFBS Radio | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="BFBS Radio logo" width="40"> | [channels/BFBSRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BFBSRadio.uk.json) |
| BestDirect.uk | Best Direct | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Best Direct logo" width="40"> | [channels/BestDirect.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BestDirect.uk.json) |
| Blaze.uk | Blaze | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Blaze logo" width="40"> | [channels/Blaze.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Blaze.uk.json) |
| BloombergTVEurope.uk | Bloomberg HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Bloomberg HD logo" width="40"> | [channels/BloombergTVEurope.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/BloombergTVEurope.uk.json) |
| CBBC.uk | CBBC | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="CBBC logo" width="40"> | [channels/CBBC.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CBBC.uk.json) |
| CBeebies.uk | CBeebies | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="CBeebies logo" width="40"> | [channels/CBeebies.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CBeebies.uk.json) |
| CNAInternational.sg | CNA | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="CNA logo" width="40"> | [channels/CNAInternational.sg.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CNAInternational.sg.json) |
| CNBCEurope.uk | CNBC HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="CNBC HD logo" width="40"> | [channels/CNBCEurope.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CNBCEurope.uk.json) |
| CNNInternational.uk | CNN HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="CNN HD logo" width="40"> | [channels/CNNInternational.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CNNInternational.uk.json) |
| CapitalRadio.uk | Capital | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Capital logo" width="40"> | [channels/CapitalRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CapitalRadio.uk.json) |
| CapitalXtra.uk | Capital Xtra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Capital Xtra logo" width="40"> | [channels/CapitalXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CapitalXtra.uk.json) |
| Challenge.uk | Challenge | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Challenge logo" width="40"> | [channels/Challenge.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Challenge.uk.json) |
| Challenge.ie | Challenge ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Challenge ROI logo" width="40"> | [channels/Challenge.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Challenge.ie.json) |
| Channel4London.uk | Channel 4 London HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 London HD logo" width="40"> | [channels/Channel4London.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4London.uk.json) |
| Channel4Midlands.uk | Channel 4 Midlands HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 Midlands HD logo" width="40"> | [channels/Channel4Midlands.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Midlands.uk.json) |
| Channel4North.uk | Channel 4 North HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 North HD logo" width="40"> | [channels/Channel4North.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4North.uk.json) |
| Channel4ROI.uk | Channel 4 ROI HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 ROI HD logo" width="40"> | [channels/Channel4ROI.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4ROI.uk.json) |
| Channel4Scotland.uk | Channel 4 Scotland HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 Scotland HD logo" width="40"> | [channels/Channel4Scotland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Scotland.uk.json) |
| Channel4South.uk | Channel 4 South HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 South HD logo" width="40"> | [channels/Channel4South.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4South.uk.json) |
| Channel4Ulster.uk | Channel 4 Ulster HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4 Ulster HD logo" width="40"> | [channels/Channel4Ulster.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Ulster.uk.json) |
| Channel4Plus1London.uk | Channel 4+1 London | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4+1 London logo" width="40"> | [channels/Channel4Plus1London.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Plus1London.uk.json) |
| Channel4Plus1North.uk | Channel 4+1 North | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4+1 North logo" width="40"> | [channels/Channel4Plus1North.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Plus1North.uk.json) |
| Channel4Plus1ROI.uk | Channel 4+1 ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4+1 ROI logo" width="40"> | [channels/Channel4Plus1ROI.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Plus1ROI.uk.json) |
| Channel4Plus1Scotland.uk | Channel 4+1 Scotland | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4+1 Scotland logo" width="40"> | [channels/Channel4Plus1Scotland.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Plus1Scotland.uk.json) |
| Channel4Plus1South.uk | Channel 4+1 South | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channel 4+1 South logo" width="40"> | [channels/Channel4Plus1South.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channel4Plus1South.uk.json) |
| Channels24.ng | Channels 24 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Channels 24 logo" width="40"> | [channels/Channels24.ng.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Channels24.ng.json) |
| ClassicFM.uk | Classic FM | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Classic FM logo" width="40"> | [channels/ClassicFM.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ClassicFM.uk.json) |
| ClublandTV.uk | Clubland TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Clubland TV logo" width="40"> | [channels/ClublandTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ClublandTV.uk.json) |
| CourtTV.uk | Court TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Court TV logo" width="40"> | [channels/CourtTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/CourtTV.uk.json) |
| DMAX.uk | DMAX | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="DMAX logo" width="40"> | [channels/DMAX.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/DMAX.uk.json) |
| DMAX.ie | DMAX ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="DMAX ROI logo" width="40"> | [channels/DMAX.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/DMAX.ie.json) |
| DMAXUKPlus1.uk | DMAX+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="DMAX+1 logo" width="40"> | [channels/DMAXUKPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/DMAXUKPlus1.uk.json) |
| E4.uk | E4 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 logo" width="40"> | [channels/E4.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4.uk.json) |
| E4Plus1.uk | E4 +1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 +1 logo" width="40"> | [channels/E4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4Plus1.uk.json) |
| E4Plus1.ie | E4 +1 ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 +1 ROI logo" width="40"> | [channels/E4Plus1.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4Plus1.ie.json) |
| E4Extra.uk | E4 Extra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 Extra logo" width="40"> | [channels/E4Extra.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4Extra.uk.json) |
| E4HD.uk | E4 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 HD logo" width="40"> | [channels/E4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4HD.uk.json) |
| E4.ie | E4 ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="E4 ROI logo" width="40"> | [channels/E4.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/E4.ie.json) |
| EuronewsEnglish.fr | Euronews | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Euronews logo" width="40"> | [channels/EuronewsEnglish.fr.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/EuronewsEnglish.fr.json) |
| Film4Plus1.uk | Film 4+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Film 4+1 logo" width="40"> | [channels/Film4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Film4Plus1.uk.json) |
| Film4.uk | Film4 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Film4 logo" width="40"> | [channels/Film4.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Film4.uk.json) |
| Film4HD.uk | Film4 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Film4 HD logo" width="40"> | [channels/Film4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Film4HD.uk.json) |
| Film4.ie | Film4 ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Film4 ROI logo" width="40"> | [channels/Film4.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Film4.ie.json) |
| FoodNetwork.uk | Food Network | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Food Network logo" width="40"> | [channels/FoodNetwork.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/FoodNetwork.uk.json) |
| FoodNetwork.ie | Food Network ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Food Network ROI logo" width="40"> | [channels/FoodNetwork.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/FoodNetwork.ie.json) |
| FoodNetworkPlus1.uk | Food Network+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Food Network+1 logo" width="40"> | [channels/FoodNetworkPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/FoodNetworkPlus1.uk.json) |
| Foodxp.uk | Foodxp | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Foodxp logo" width="40"> | [channels/Foodxp.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Foodxp.uk.json) |
| France24English.fr | France 24 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="France 24 HD logo" width="40"> | [channels/France24English.fr.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/France24English.fr.json) |
| GBNews.uk | GB News HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GB News HD logo" width="40"> | [channels/GBNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GBNews.uk.json) |
| GIGS.uk | GIGS | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GIGS logo" width="40"> | [channels/GIGS.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GIGS.uk.json) |
| GreatMoviesAction.uk | GREAT! action | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! action logo" width="40"> | [channels/GreatMoviesAction.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatMoviesAction.uk.json) |
| GreatMoviesActionPlus1.uk | GREAT! action+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! action+1 logo" width="40"> | [channels/GreatMoviesActionPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatMoviesActionPlus1.uk.json) |
| GreatMovies.uk | GREAT! movies | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! movies logo" width="40"> | [channels/GreatMovies.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatMovies.uk.json) |
| GreatMoviesPlus1.uk | GREAT! movies+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! movies+1 logo" width="40"> | [channels/GreatMoviesPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatMoviesPlus1.uk.json) |
| GreatMystery.uk | GREAT! mystery | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! mystery logo" width="40"> | [channels/GreatMystery.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatMystery.uk.json) |
| GreatReal.uk | GREAT! real | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! real logo" width="40"> | [channels/GreatReal.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatReal.uk.json) |
| GreatRealPlus1.uk | GREAT! real+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! real+1 logo" width="40"> | [channels/GreatRealPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatRealPlus1.uk.json) |
| GreatXmas.uk | GREAT! romance | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! romance logo" width="40"> | [channels/GreatXmas.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatXmas.uk.json) |
| GreatXmasPlus1.uk | GREAT! romance+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! romance+1 logo" width="40"> | [channels/GreatXmasPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatXmasPlus1.uk.json) |
| GreatTV.uk | GREAT! tv | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! tv logo" width="40"> | [channels/GreatTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatTV.uk.json) |
| GreatTVPlus1.uk | GREAT! tv+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="GREAT! tv+1 logo" width="40"> | [channels/GreatTVPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GreatTVPlus1.uk.json) |
| GemsTV.uk | Gemporia TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Gemporia TV logo" width="40"> | [channels/GemsTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GemsTV.uk.json) |
| GoldRadio.uk | Gold | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Gold logo" width="40"> | [channels/GoldRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/GoldRadio.uk.json) |
| HGTV.uk | HGTV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="HGTV logo" width="40"> | [channels/HGTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HGTV.uk.json) |
| HGTV.ie | HGTV ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="HGTV ROI logo" width="40"> | [channels/HGTV.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HGTV.ie.json) |
| HGTVPlus1.uk | HGTV+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="HGTV+1 logo" width="40"> | [channels/HGTVPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HGTVPlus1.uk.json) |
| Heart80s.uk | Heart 80s | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Heart 80s logo" width="40"> | [channels/Heart80s.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Heart80s.uk.json) |
| Heart90s.uk | Heart 90s | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Heart 90s logo" width="40"> | [channels/Heart90s.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Heart90s.uk.json) |
| HeartDance.uk | Heart Dance | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Heart Dance logo" width="40"> | [channels/HeartDance.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HeartDance.uk.json) |
| HeartRadio.uk | Heart Radio | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Heart Radio logo" width="40"> | [channels/HeartRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HeartRadio.uk.json) |
| HighStreetTV.uk | High Street TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="High Street TV logo" width="40"> | [channels/HighStreetTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HighStreetTV.uk.json) |
| HobbyMaker.uk | Hobby Maker HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Hobby Maker HD logo" width="40"> | [channels/HobbyMaker.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/HobbyMaker.uk.json) |
| ITVQuiz.uk | ITV Quiz | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV Quiz logo" width="40"> | [channels/ITVQuiz.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITVQuiz.uk.json) |
| ITV1Plus1Granada.uk | ITV+1 Granada | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV+1 Granada logo" width="40"> | [channels/ITV1Plus1Granada.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Plus1Granada.uk.json) |
| ITV1Plus1London.uk | ITV+1 London | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV+1 London logo" width="40"> | [channels/ITV1Plus1London.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Plus1London.uk.json) |
| ITV1Plus1Meridian.uk | ITV+1 Meridian | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV+1 Meridian logo" width="40"> | [channels/ITV1Plus1Meridian.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Plus1Meridian.uk.json) |
| ITV1Plus1Midlands.uk | ITV+1 Midlands | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV+1 Midlands logo" width="40"> | [channels/ITV1Plus1Midlands.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Plus1Midlands.uk.json) |
| ITV1AngliaWest.uk | ITV1 Anglia (West) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Anglia (West) HD logo" width="40"> | [channels/ITV1AngliaWest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1AngliaWest.uk.json) |
| ITV1Anglia.uk | ITV1 Anglia HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Anglia HD logo" width="40"> | [channels/ITV1Anglia.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Anglia.uk.json) |
| ITV1Border.uk | ITV1 Border HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Border HD logo" width="40"> | [channels/ITV1Border.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Border.uk.json) |
| ITV1CentralE.uk | ITV1 Central (East) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Central (East) HD logo" width="40"> | [channels/ITV1CentralE.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1CentralE.uk.json) |
| ITV1CentralW.uk | ITV1 Central (West) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Central (West) HD logo" width="40"> | [channels/ITV1CentralW.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1CentralW.uk.json) |
| ITV1Granada.uk | ITV1 Granada HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Granada HD logo" width="40"> | [channels/ITV1Granada.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Granada.uk.json) |
| ITV1London.uk | ITV1 London HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 London HD logo" width="40"> | [channels/ITV1London.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1London.uk.json) |
| ITV1MeridianE.uk | ITV1 Meridian (East) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Meridian (East) HD logo" width="40"> | [channels/ITV1MeridianE.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1MeridianE.uk.json) |
| ITV1MeridianN.uk | ITV1 Meridian (North) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Meridian (North) HD logo" width="40"> | [channels/ITV1MeridianN.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1MeridianN.uk.json) |
| ITV1MeridianS.uk | ITV1 Meridian (South) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Meridian (South) HD logo" width="40"> | [channels/ITV1MeridianS.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1MeridianS.uk.json) |
| ITV1North.uk | ITV1 North HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 North HD logo" width="40"> | [channels/ITV1North.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1North.uk.json) |
| ITV1TyneTees.uk | ITV1 Tyne Tees HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Tyne Tees HD logo" width="40"> | [channels/ITV1TyneTees.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1TyneTees.uk.json) |
| ITV1Wales.uk | ITV1 Wales HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Wales HD logo" width="40"> | [channels/ITV1Wales.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1Wales.uk.json) |
| ITV1WestCountry.uk | ITV1 West Country HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 West Country HD logo" width="40"> | [channels/ITV1WestCountry.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1WestCountry.uk.json) |
| ITV1West.uk | ITV1 West HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 West HD logo" width="40"> | [channels/ITV1West.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1West.uk.json) |
| ITV1YorkshireEast.uk | ITV1 Yorkshire (East) HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Yorkshire (East) HD logo" width="40"> | [channels/ITV1YorkshireEast.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1YorkshireEast.uk.json) |
| ITV1YorkshireWest.uk | ITV1 Yorkshire HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV1 Yorkshire HD logo" width="40"> | [channels/ITV1YorkshireWest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV1YorkshireWest.uk.json) |
| ITV2.uk | ITV2 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV2 HD logo" width="40"> | [channels/ITV2.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV2.uk.json) |
| ITV2Plus1.uk | ITV2+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV2+1 logo" width="40"> | [channels/ITV2Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV2Plus1.uk.json) |
| ITV3.uk | ITV3 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV3 HD logo" width="40"> | [channels/ITV3.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV3.uk.json) |
| ITV3Plus1.uk | ITV3+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV3+1 logo" width="40"> | [channels/ITV3Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV3Plus1.uk.json) |
| ITV4.uk | ITV4 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV4 HD logo" width="40"> | [channels/ITV4.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV4.uk.json) |
| ITV4Plus1.uk | ITV4+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="ITV4+1 logo" width="40"> | [channels/ITV4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ITV4Plus1.uk.json) |
| IdealWorld.uk | Ideal World HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Ideal World HD logo" width="40"> | [channels/IdealWorld.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/IdealWorld.uk.json) |
| JewelleryMaker.uk | Jewellery Maker | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Jewellery Maker logo" width="40"> | [channels/JewelleryMaker.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/JewelleryMaker.uk.json) |
| LBCRadio.uk | LBC | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="LBC logo" width="40"> | [channels/LBCRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/LBCRadio.uk.json) |
| LBCNews.uk | LBC News | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="LBC News logo" width="40"> | [channels/LBCNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/LBCNews.uk.json) |
| Legend.uk | Legend | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Legend logo" width="40"> | [channels/Legend.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Legend.uk.json) |
| LegendXtra.uk | Legend Xtra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Legend Xtra logo" width="40"> | [channels/LegendXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/LegendXtra.uk.json) |
| LegendXtraPlus1.uk | Legend Xtra+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Legend Xtra+1 logo" width="40"> | [channels/LegendXtraPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/LegendXtraPlus1.uk.json) |
| LondonLive.uk | London Live | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="London Live logo" width="40"> | [channels/LondonLive.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/LondonLive.uk.json) |
| More4.uk | More4 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="More4 logo" width="40"> | [channels/More4.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/More4.uk.json) |
| More4HD.uk | More4 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="More4 HD logo" width="40"> | [channels/More4HD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/More4HD.uk.json) |
| More4.ie | More4 ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="More4 ROI logo" width="40"> | [channels/More4.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/More4.ie.json) |
| More4Plus1.uk | More4+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="More4+1 logo" width="40"> | [channels/More4Plus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/More4Plus1.uk.json) |
| Movies24.uk | Movies24 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Movies24 logo" width="40"> | [channels/Movies24.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Movies24.uk.json) |
| Movies24Plus.uk | Movies24+ | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Movies24+ logo" width="40"> | [channels/Movies24Plus.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Movies24Plus.uk.json) |
| NDTVWorld.in | NDTV World | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="NDTV World logo" width="40"> | [channels/NDTVWorld.in.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/NDTVWorld.in.json) |
| NHKWorldJapan.jp | NHK World HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="NHK World HD logo" width="40"> | [channels/NHKWorldJapan.jp.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/NHKWorldJapan.jp.json) |
| NewsmaxTV.uk | Newsmax | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Newsmax logo" width="40"> | [channels/NewsmaxTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/NewsmaxTV.uk.json) |
| Now70s.uk | Now 70s | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Now 70s logo" width="40"> | [channels/Now70s.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Now70s.uk.json) |
| Now80s.uk | Now 80s | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Now 80s logo" width="40"> | [channels/Now80s.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Now80s.uk.json) |
| Now90s.uk | Now 90s & 00s | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Now 90s & 00s logo" width="40"> | [channels/Now90s.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Now90s.uk.json) |
| NowRock.uk | Now Rock | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Now Rock logo" width="40"> | [channels/NowRock.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/NowRock.uk.json) |
| PBSAmerica.uk | PBS America | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="PBS America logo" width="40"> | [channels/PBSAmerica.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/PBSAmerica.uk.json) |
| PopPlus1.uk | POP+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="POP+1 logo" width="40"> | [channels/PopPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/PopPlus1.uk.json) |
| Pop.uk | Pop | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Pop logo" width="40"> | [channels/Pop.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Pop.uk.json) |
| QuestPlus1.uk | QUEST+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="QUEST+1 logo" width="40"> | [channels/QuestPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestPlus1.uk.json) |
| QVCBeautyUK.uk | QVC Beauty | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="QVC Beauty logo" width="40"> | [channels/QVCBeautyUK.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QVCBeautyUK.uk.json) |
| QVCExtraUK.uk | QVC Extra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="QVC Extra logo" width="40"> | [channels/QVCExtraUK.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QVCExtraUK.uk.json) |
| QVCStyleUK.uk | QVC Style UK | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="QVC Style UK logo" width="40"> | [channels/QVCStyleUK.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QVCStyleUK.uk.json) |
| QVCUK.uk | QVC UK | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="QVC UK logo" width="40"> | [channels/QVCUK.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QVCUK.uk.json) |
| Quest.uk | Quest | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest logo" width="40"> | [channels/Quest.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Quest.uk.json) |
| QuestHD.uk | Quest HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest HD logo" width="40"> | [channels/QuestHD.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestHD.uk.json) |
| QuestHD.ie | Quest HD ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest HD ROI logo" width="40"> | [channels/QuestHD.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestHD.ie.json) |
| QuestRed.uk | Quest Red | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest Red logo" width="40"> | [channels/QuestRed.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestRed.uk.json) |
| QuestRed.ie | Quest Red ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest Red ROI logo" width="40"> | [channels/QuestRed.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestRed.ie.json) |
| QuestRedPlus1.uk | Quest Red+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Quest Red+1 logo" width="40"> | [channels/QuestRedPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/QuestRedPlus1.uk.json) |
| RTENews.ie | RTÉ News | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉ News logo" width="40"> | [channels/RTENews.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTENews.ie.json) |
| RTEOne.ie | RTÉ One HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉ One HD logo" width="40"> | [channels/RTEOne.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTEOne.ie.json) |
| RTEOnePlus1.ie | RTÉ One+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉ One+1 logo" width="40"> | [channels/RTEOnePlus1.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTEOnePlus1.ie.json) |
| RTE2.ie | RTÉ2 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉ2 HD logo" width="40"> | [channels/RTE2.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTE2.ie.json) |
| RTE2Plus1.ie | RTÉ2+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉ2+1 logo" width="40"> | [channels/RTE2Plus1.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTE2Plus1.ie.json) |
| RTEjr.ie | RTÉjr | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="RTÉjr logo" width="40"> | [channels/RTEjr.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RTEjr.ie.json) |
| RadioX.uk | Radio X | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Radio X logo" width="40"> | [channels/RadioX.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RadioX.uk.json) |
| Really.uk | Really | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Really logo" width="40"> | [channels/Really.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Really.uk.json) |
| Really.ie | Really ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Really ROI logo" width="40"> | [channels/Really.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Really.ie.json) |
| RewindTV.uk | Rewind TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Rewind TV logo" width="40"> | [channels/RewindTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/RewindTV.uk.json) |
| S4C.uk | S4C | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="S4C logo" width="40"> | [channels/S4C.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/S4C.uk.json) |
| STVCentralPlus1.uk | STV Central +1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="STV Central +1 logo" width="40"> | [channels/STVCentralPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/STVCentralPlus1.uk.json) |
| STVCentral.uk | STV Central HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="STV Central HD logo" width="40"> | [channels/STVCentral.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/STVCentral.uk.json) |
| STVNorthPlus1.uk | STV North +1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="STV North +1 logo" width="40"> | [channels/STVNorthPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/STVNorthPlus1.uk.json) |
| STVNorth.uk | STV North HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="STV North HD logo" width="40"> | [channels/STVNorth.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/STVNorth.uk.json) |
| SkyArts.uk | Sky Arts | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Sky Arts logo" width="40"> | [channels/SkyArts.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SkyArts.uk.json) |
| SkyArts.ie | Sky Arts ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Sky Arts ROI logo" width="40"> | [channels/SkyArts.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SkyArts.ie.json) |
| SkyMix.uk | Sky Mix HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Sky Mix HD logo" width="40"> | [channels/SkyMix.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SkyMix.uk.json) |
| SkyMix.ie | Sky Mix HD ROI | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Sky Mix HD ROI logo" width="40"> | [channels/SkyMix.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SkyMix.ie.json) |
| SkyNews.uk | Sky News HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Sky News HD logo" width="40"> | [channels/SkyNews.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SkyNews.uk.json) |
| SmoothRadio.uk | Smooth Radio | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Smooth Radio logo" width="40"> | [channels/SmoothRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/SmoothRadio.uk.json) |
| TBNUK.uk | TBN UK | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="TBN UK logo" width="40"> | [channels/TBNUK.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TBNUK.uk.json) |
| TG4.ie | TG4 HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="TG4 HD logo" width="40"> | [channels/TG4.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TG4.ie.json) |
| TJC.uk | TJC HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="TJC HD logo" width="40"> | [channels/TJC.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TJC.uk.json) |
| TRTWorld.tr | TRT World | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="TRT World logo" width="40"> | [channels/TRTWorld.tr.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TRTWorld.tr.json) |
| TVCNews.ng | TVC News | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="TVC News logo" width="40"> | [channels/TVCNews.ng.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TVCNews.ng.json) |
| TalkingPicturesTV.uk | Talking Pictures TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Talking Pictures TV logo" width="40"> | [channels/TalkingPicturesTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TalkingPicturesTV.uk.json) |
| ThatsDance.uk | That's Dance | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's Dance logo" width="40"> | [channels/ThatsDance.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsDance.uk.json) |
| ThatsFabulous.uk | That's Fabulous | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's Fabulous logo" width="40"> | [channels/ThatsFabulous.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsFabulous.uk.json) |
| ThatsMelody.uk | That's Melody | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's Melody logo" width="40"> | [channels/ThatsMelody.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsMelody.uk.json) |
| ThatsTV.uk | That's TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's TV logo" width="40"> | [channels/ThatsTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsTV.uk.json) |
| ThatsTV2.uk | That's TV 2 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's TV 2 logo" width="40"> | [channels/ThatsTV2.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsTV2.uk.json) |
| ThatsTV3.uk | That's TV 3 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="That's TV 3 logo" width="40"> | [channels/ThatsTV3.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/ThatsTV3.uk.json) |
| TogetherTV.uk | Together TV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Together TV logo" width="40"> | [channels/TogetherTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TogetherTV.uk.json) |
| Travelxp.in | Travelxp | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Travelxp logo" width="40"> | [channels/Travelxp.in.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/Travelxp.in.json) |
| TrueCrime.uk | True Crime | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="True Crime logo" width="40"> | [channels/TrueCrime.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TrueCrime.uk.json) |
| TrueCrimeXtra.uk | True Crime Xtra | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="True Crime Xtra logo" width="40"> | [channels/TrueCrimeXtra.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TrueCrimeXtra.uk.json) |
| TrueCrimePlus1.uk | True Crime+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="True Crime+1 logo" width="40"> | [channels/TrueCrimePlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TrueCrimePlus1.uk.json) |
| UAndDave.uk | U&Dave | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Dave logo" width="40"> | [channels/UAndDave.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndDave.uk.json) |
| UAndDavejavu.uk | U&Dave ja vu | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Dave ja vu logo" width="40"> | [channels/UAndDavejavu.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndDavejavu.uk.json) |
| UAndDrama.uk | U&Drama | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Drama logo" width="40"> | [channels/UAndDrama.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndDrama.uk.json) |
| UAndDramaPlus1.uk | U&Drama+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Drama+1 logo" width="40"> | [channels/UAndDramaPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndDramaPlus1.uk.json) |
| UAndEden.uk | U&Eden | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Eden logo" width="40"> | [channels/UAndEden.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndEden.uk.json) |
| UAndEdenPlus1.uk | U&Eden+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Eden+1 logo" width="40"> | [channels/UAndEdenPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndEdenPlus1.uk.json) |
| UAndW.uk | U&W | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&W logo" width="40"> | [channels/UAndW.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndW.uk.json) |
| UAndWPlus1.uk | U&W+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&W+1 logo" width="40"> | [channels/UAndWPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndWPlus1.uk.json) |
| UAndYesterday.uk | U&Yesterday | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Yesterday logo" width="40"> | [channels/UAndYesterday.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndYesterday.uk.json) |
| UAndYesterdayPlus1.uk | U&Yesterday+1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="U&Yesterday+1 logo" width="40"> | [channels/UAndYesterdayPlus1.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UAndYesterdayPlus1.uk.json) |
| UTV.uk | UTV | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="UTV logo" width="40"> | [channels/UTV.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/UTV.uk.json) |
| VirginMediaFour.ie | Virgin Media Four | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Media Four logo" width="40"> | [channels/VirginMediaFour.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginMediaFour.ie.json) |
| VirginMediaOne.ie | Virgin Media One | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Media One logo" width="40"> | [channels/VirginMediaOne.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginMediaOne.ie.json) |
| VirginMediaOnePlus1.ie | Virgin Media One +1 | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Media One +1 logo" width="40"> | [channels/VirginMediaOnePlus1.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginMediaOnePlus1.ie.json) |
| VirginMediaThree.ie | Virgin Media Three | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Media Three logo" width="40"> | [channels/VirginMediaThree.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginMediaThree.ie.json) |
| VirginMediaTwo.ie | Virgin Media Two | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Media Two logo" width="40"> | [channels/VirginMediaTwo.ie.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginMediaTwo.ie.json) |
| VirginRadio.uk | Virgin Radio | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="Virgin Radio logo" width="40"> | [channels/VirginRadio.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/VirginRadio.uk.json) |
| WION.in | WION HD | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="WION HD logo" width="40"> | [channels/WION.in.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/WION.in.json) |
| TalkSport.uk | talkSPORT | <img src="https://raw.githubusercontent.com/elyobelyob/freely_tv_guide_new/main/img/channels/placeholder.svg" alt="talkSPORT logo" width="40"> | [channels/TalkSport.uk.json](https://elyobelyob.github.io/freely_tv_guide_new/channels/TalkSport.uk.json) |
<!-- CHANNELS_END -->

## Stats
See counts of channels and images here:  
https://elyobelyob.github.io/freely_tv_guide_new/stats/
