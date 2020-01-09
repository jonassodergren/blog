+++
title = "Jobsearch API - Enkel marknadsanalys"
description = ""
date = 2019-11-03T19:24:42+01:00
weight = 20
draft = false
bref = "Vilken nytta kan skapas på 5 minuter"
toc = true
+++
Hösten 2019 lanserade det nya API:et jobstream för att hämta alla platsannonser från Platsbanken. Men hur enkelt är det egentligen?

Det är värdefull för många företag, myndigheter och personer att enkelt få tillgång till all annonsdata. För att prova behövde jag ett problem att försöka lösa.

_Hur många rekryteringssystem finns på marknaden som kräver att en CV laddas upp för att kunna ansöka om ett jobb._

```bash
curl -X GET "https://jobstream.api.jobtechdev.se/stream?date=2019-10-01" -H "accept: application/json" -H "api-key: {GET_YOUR_KEY:https://apirequest.jobtechdev.se/}" |
    jq '.[] | select(. | (.application_details.url // empty)) | [.application_details.url,.employer.url]' |
    jq '.|select((.[1]==null))' |
    jq '.[0]' |
    sed -e 's|^[^/]*//||' -e 's|/.*$||' |
    sed 's/\"//g' |
    sort |
    uniq > ansokningssystem_october.file
```
