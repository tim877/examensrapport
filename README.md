---
#

Datamodellering i MySQL och dess påverkan på prestanda och användarupplevelse i en React‐applikation
Hybridprojekt - utveckling och teknisk analys
Tim Hagman - 2026
---

Hybridprojekt - utveckling och teknisk analys

Tim Hagman - 2026

### Abstract

This thesis investigates how data modeling choices in MySQL affect end‐to‐end system performance and perceived user experience in a React‐based web application. Modern feed‐oriented interfaces require presentation‐ready data, often including aggregated values such as comment counts and author information. In a normalized relational schema, such views typically require `JOIN` operations combined with `COUNT` and `GROUP BY` at read time. In contrast, a denormalized read‐projection stores precomputed, presentation‐ready data, shifting computational cost from reads to maintenance.

A prototype system was implemented using MySQL, a Node.js and Express backend, and a React frontend. Two database models were evaluated on identical datasets: a normalized schema with separate tables for users, posts, and comments, and a denormalized schema using a flat projection table. Performance was measured across multiple layers, including database query execution, backend processing, frontend fetch, and rendering. Measurements were repeated across multiple batches to account for cache and system state variance.

Results show that data modeling decisions are a dominant factor in overall latency. The normalized model exhibited database query times on the order of seconds per request, while the denormalized model reduced database time to approximately one millisecond. When the primary objective is read performance, this reduction propagates through the system, significantly improving frontend responsiveness and aligning system behavior with established UX response-time thresholds. The study concludes that denormalized read-projections are highly effective for read-heavy feed workloads where read speed is the primary concern, but introduce trade-offs in data integrity and maintenance complexity that must be handled explicitly.

Keywords: MySQL, normalization, denormalization, performance, React, user experience

### Lista över förkortningar och begrepp

### 1. Inledning

#### 1.1 Bakgrund och problemformulering

Valet av ämne för detta examensarbete grundar sig i ett återkommande mönster i moderna, datadrivna webbapplikationer: listor och feeds upplevs ofta som långsamma trots att frontend‐koden i sig är relativt enkel och korrekt implementerad. I praktiken leder detta ofta till att fokus hamnar på optimering av React‐komponenter, rendering och state‐hantering, trots att användaren inte upplever var fördröjningen uppstår - endast att applikationen känns seg.

För användaren är prestanda ett helhetsintryck. Så länge datan inte har levererats till klienten spelar det ingen roll hur optimerad rendering eller komponentstruktur är, eftersom användargränssnittet ännu inte har något meningsfullt innehåll att visa. Detta är särskilt tydligt i feed‐baserade vyer där presentationen är helt beroende av att data först hämtas från backend.

Detta examensarbete tar därför sin utgångspunkt i ett systemperspektiv på frontend‐prestanda. I stället för att anta att långsam upplevelse beror på frontend‐logik undersöks hur bakomliggande datamodellering i MySQL påverkar hela kedjan från databas till användarupplevelse.

#### 1.2 Relevans för frontendutveckling

Inom frontendutveckling läggs ofta stor vikt vid rendering, komponentlivscykler och state‐uppdateringar. I praktiken visar dock många applikationer att den faktiska flaskhalsen inte ligger i UI‐lagret utan i hur snabbt och effektivt data kan levereras till frontend. Detta gäller särskilt datadrivna applikationer där vyer är beroende av sammanfogad och aggregerad data.

Frontend‐prestanda blir därmed inte en isolerad frontend‐fråga utan ett systemproblem där databas, backend, nätverk och klient samverkar. En snabb frontend kan inte kompensera för en långsam databas, men en väloptimerad datamodell kan däremot göra frontenden upplevt snabb utan att någon frontend‐kod förändras.

#### 1.3 Syfte och forskningsfrågor

Syftet med detta examensarbete är att analysera hur olika datamodeller i MySQL påverkar prestanda och användarupplevelse i en React‐baserad webbapplikation. Arbetet fokuserar på jämförelse mellan en normaliserad och en denormaliserad datamodell för samma feed‐vy.

Arbetet utgår från följande forskningsfrågor:

Hur stor prestandaskillnad uppstår mellan en normaliserad och en denormaliserad datamodell när samma vy hämtas?

Var i systemkedjan uppstår den dominerande tidskostnaden: i databasen, backend‐bearbetningen eller i frontend?

När övergår uppmätta tidsvärden från att vara en teknisk detalj till att bli ett problem för användarupplevelsen?

#### 1.4 Avgränsningar

Arbetet fokuserar på läs-prestanda för feed-vyer i en kontrollerad testmiljö. Detta innebär att resultatet främst är relevant för applikationer där samma vy läses frekvent av många användare, snarare än system med hög skrivintensitet eller komplex affärslogik.

React-komponenter optimeras inte medvetet, utan hålls enkla och identiska mellan båda testerna. Syftet med detta är att säkerställa att observerade skillnader i prestanda inte kan härledas till skillnader i frontend-logik, rendering eller komponentstruktur, utan enbart till hur data hämtas och bearbetas.

Mätningarna genomförs som så kallade warm runs, vilket innebär att systemet redan är igång och att cold-start-effekter såsom uppstart av Node-process, databaskopplingar och cache-uppvärmning inte ingår. Detta val görs för att bättre spegla ett realistiskt steady-state-scenario där en applikation redan är i drift.

### 2. Teoretisk grund

Detta kapitel presenterar den teoretiska bakgrund som ligger till grund för arbetets designval och analys. Fokus ligger på relationsdatabasdesign, påverkan på prestanda av olika datamodeller samt hur dessa val påverkar användarupplevelsen i datadrivna webbapplikationer.

#### 2.1 Normalisering och normalformer

Genom att dela upp data i flera tabeller som hör ihop lagras varje typ av information på ett tydligt ställe. Det gör uppdateringar enklare och minskar risken för att olika delar av systemet innehåller olika värden.

Normalisering beskrivs genom olika nivåer som kallas normalformer. Varje nivå ställer högre krav på hur data ska struktureras.

Första normalformen (1NF) innebär att varje fält bara innehåller ett värde och att samma typ av information inte upprepas i flera kolumner.

Andra normalformen (2NF) bygger vidare på detta och innebär att all information i en tabell ska höra till hela primärnyckeln, och inte bara en del av den.

Tredje normalformen (3NF) innebär slutligen att information inte ska bero på andra icke-nyckelvärden, utan endast på tabellens primärnyckel.

Den normaliserade datamodell som används i detta arbete följer tredje normalformen. Användare, inlägg och kommentarer lagras i olika tabeller, vilket gör att data hålls konsekvent och att uppdateringar blir enkla att hantera. Ur ett teoretiskt perspektiv är normalisering ofta att föredra, särskilt i system där korrekthet och datakonsistens är viktiga.

Nackdelen med denna typ av design märks vid läsning av sammanhängande vyer. För att visa en feed behöver databasen hämta data från flera tabeller med `JOIN`-operationer och samtidigt räkna exempelvis antal kommentarer med hjälp av `COUNT` och `GROUP BY`. När detta görs vid varje request kan kostnaden bli hög, särskilt vid stora datamängder. I praktiken innebär detta att normalisering passar bäst i system där skrivoperationer sker ofta och där läsningar inte är den dominerande belastningen, eller där mer komplexa vyer hanteras med andra lösningar, som cache eller materialiserade vyer.

#### 2.2 Denormalisering, read-projections och CQRS-inspirerat tänk

Denormalisering innebär att medvetet avvika från strikt normaliserad design genom att duplicera eller sammanställa data för att optimera läsprestanda. I stället för att konstruera komplexa vyer vid varje request lagras data i ett format som redan är anpassat för hur den ska konsumeras av applikationen.

Den denormaliserade tabell som används i detta arbete kan ses som en förenklad read-projection, där varje rad innehåller all information som krävs för presentation i feed-vyn. Det innebär att data som författarnamn och antal kommentarer redan finns med i samma rad, vilket gör att läsfrågan kan besvaras utan `JOIN`s och utan runtime-aggregering.

Ett liknande synsätt återfinns i arkitekturmönster som CQRS (Command Query Responsibility Segregation), där läs- och skrivmodeller separeras för att möjliggöra optimering för respektive användningsfall. I CQRS optimeras läsmodellen ofta för snabba frågor och presentation, medan skrivmodellen optimeras för korrekthet och uppdateringar.

Fördelen med detta angreppssätt är att läsningar blir mycket snabba och förutsägbara. Databasen behöver inte utföra `JOIN`s eller tunga sammanräkningar, vilket kan minska CPU-belastning och risken för låsning i databasen. Nackdelen är att uppdateringar blir mer komplexa, eftersom förändringar i grunddata måste propageras till den denormaliserade representationen. Det innebär att applikationen (eller ett underhållsflöde) måste ta ansvar för synkronisering och datakonsistens.

Denormalisering lämpar sig därför särskilt väl för read-tunga system såsom sociala flöden, dashboards och publika listor, där samma vy läses ofta men uppdateras relativt sällan. Valet innebär dock ett tydligt ansvar att hantera datakonsistens på applikationsnivå.

#### 2.3 Databasprestanda, `JOIN`s och aggregeringar

`JOIN`-operationer och aggregeringar är kraftfulla verktyg i SQL, men de är inte gratis ur ett prestandaperspektiv. Vid en `JOIN` måste databasen matcha rader mellan tabeller baserat på nycklar, vilket vid stora tabeller kan innebära att stora mängder data behöver läsas och jämföras.

Aggregeringar såsom `COUNT` och `GROUP BY` innebär dessutom att databasen måste bearbeta stora delar av resultatmängden innan ett slutgiltigt värde kan beräknas. Även om index kan förbättra prestanda i vissa fall, kan de sällan eliminera behovet av att läsa mycket data vid aggregering, särskilt när aggregeringen behöver göras per post i en feed. Detta gör att normaliserade modeller med runtime-aggregering ofta skalar sämre för read-tunga vyer.

I kontrast innebär en denormaliserad read-projection att databasen i större utsträckning "bara läser färdiga rader". Då flyttas kostnaden från läsning till underhåll/uppdatering, vilket i praktiken är själva trade-offen mellan modellerna.

#### 2.4 Prestanda och användarupplevelse

Prestanda i webbapplikationer är nära kopplad till hur användare upplever interaktion och responsivitet. Responstider under cirka 100 ms upplevs ofta som omedelbara, medan svarstider över en sekund blir tydligt märkbara och ofta uppfattas som störande. Vid svarstider på flera sekunder riskerar användaren att tappa fokus eller uppleva applikationen som opålitlig.

I datadrivna applikationer är användarupplevelsen starkt beroende av när data blir tillgänglig snarare än hur snabbt gränssnittet kan renderas. En snabb rendering är av begränsat värde om datan levereras sent, eftersom användaren ändå väntar på att vyn ska visa något meningsfullt. Detta gör att beslut som fattas i databasskiktet - exempelvis om aggregeringar sker vid varje request eller om data lagras presentationsfärdigt - får direkta och mätbara konsekvenser för upplevd UX.

Ur detta perspektiv blir prestanda inte enbart en teknisk fråga utan även en designfråga: val som görs i databasskiktet kan avgöra om en vy upplevs som snabb eller långsam, oavsett hur väloptimerad frontend-koden är.

### 3. Metod och genomförande

#### 3.1 Övergripande upplägg

Utvecklingsarbetet har genomförts med hjälp av versionshantering för att säkerställa spårbarhet och kontroll över förändringar. Källkoden har versionshanterats med Git och lagrats i ett GitHub-repository. Detta har gjort det möjligt att följa förändringar över tid, förenkla felsökning och vid behov återgå till tidigare fungerande versioner.

Versionshantering har även spelat en viktig roll i samband med prestandamätningarna, där förändringar i mätlogik och beräkningsmetodik kunnat isoleras och verifieras över tid.

För att kunna isolera effekten av datamodellering byggdes en enkel och kontrollerad prototyp bestående av tre tydliga lager: databas, backend och frontend. Arkitekturen bygger på en enkel och välkänd uppdelning mellan dessa lager, vilket gör att resultaten är lätta att överföra till verkliga webbapplikationer.

Frontend är utvecklad i React och presenterar en feed-vy bestående av ett större antal poster. Backend är implementerad i Node.js med Express och exponerar två separata API-endpoints, en per datamodell. Databasen utgörs av MySQL och innehåller samma underliggande dataset i båda fallen.
![Normaliserad SQL-fråga](https://raw.githubusercontent.com/tim877/examensrapport/629aa0cd9be87d05c49f63a1e1d7b460b525ddce/1.png)
Bilden som visas är hämtad från tidigare tester, men används för att illustrera hur vyerna ser ut i applikationen.

Det viktigaste metodvalet är att frontend-koden är exakt samma oavsett datamodell. Den enda skillnaden i systemet är vilken endpoint som anropas. På så sätt kan skillnader i uppmätt prestanda kopplas direkt till datamodelleringen och inte till frontend- eller backend-logik.

#### 3.2 Datamodeller

Två datamodeller implementerades och testades:

Den normaliserade modellen består av separata tabeller för användare, inlägg och kommentarer. När feed-vyn hämtas måste databasen slå samman data från flera tabeller med `JOIN`-operationer samt räkna antalet kommentarer per inlägg med `COUNT` och `GROUP BY`. Aggregeringen sker vid varje läsning, i realtid.

Den denormaliserade modellen använder i stället en read-projection i form av en tabell där varje rad redan innehåller all data som krävs för presentation i feeden, inklusive författarnamn och antal kommentarer. Vid läsning krävs varken `JOIN`s eller runtime-aggregering, vilket gör frågan betydligt enklare för databasen.

Det är viktigt att understryka att det som visas i frontend är exakt samma vy i båda fallen. Skillnaden ligger uteslutande i hur mycket arbete databasen behöver utföra för att leverera denna vy.

![Figur 7](https://raw.githubusercontent.com/tim877/examensrapport/974e17e0fa781cedd37a8510a3b5de201f5fc265/7.png)


#### 3.3 Dataset

Datasetet är konstruerat för att tydliggöra skillnaden mellan modellerna och består av:

50 användare

1 600 inlägg

4 291 650 kommentarer

![Figur 2](https://raw.githubusercontent.com/tim877/examensrapport/ae5e437b22eda3cbdbea012a4a01dde91fb719f8/2.png)

Den stora mängden kommentarer är ett medvetet val för att göra aggregering kostsam i den normaliserade modellen. På så sätt blir kontrasten mellan att räkna data vid varje request och att läsa färdig data tydlig.

#### 3.4 Mätstrategi

För att förstå var begränsningarna i systemet faktiskt uppstår delas mätningen upp i flera nivåer. Samtliga tidsmätningar genomförs med hjälp av `performance.now()`, som är en högupplöst och monoton timer som i Node.js används via modulen `perf_hooks`. Till skillnad från enklare tidtagning med exempelvis Date.now() erbjuder `performance.now()` högre precision och är särskilt avsedd för prestandamätning.

I backend startas en timer precis innan SQL-frågan skickas till MySQL och stoppas när databasen har returnerat sitt resultat. Därefter startas en ny mätning för JSON-serialisering, som avslutas när svaret är färdigbyggt. Dessa mätningar ger en tydlig uppdelning mellan databaskostnad och backend-bearbetning.

![Figur 3](https://raw.githubusercontent.com/tim877/examensrapport/68bb4700301d98ca04512e2b63bdf76f528c38c7/3.png)

I frontend används samma princip, där en timer startas precis innan `fetch()` anropas och stoppas först när JSON-svaret har mottagits och parsats av webbläsaren. Rendering mäts separat genom att kombinera state-uppdatering med `requestAnimationFrame` för att säkerställa att mätningen inkluderar faktisk visuell uppdatering.Genom att använda samma typ av högupplösta timers i både backend och frontend säkerställs att mätningarna är jämförbara och att små skillnader inte försvinner i avrundningsfel.

#### Testmiljö

Prestandatesterna genomfördes i en lokal utvecklingsmiljö med MySQL 8.4, där backend-servern (Node.js/Express) och databasen kördes på samma maskin, medan frontend-applikationen (React) exekverades i webbläsaren. Testsystemet bestod av en AMD Ryzen 7 9800X3D, 32 GB DDR5-minne (6000 MHz, CL30), NVIDIA GeForce RTX 4060 Ti samt Gen5 M.2 NVMe-baserad SSD-lagring. Eventuella effekter av systembelastning och testmiljö diskuteras vidare i avsnitt 5.4.

#### 3.5 Statistik och körsätt och körsätt

Varje datamodell testades genom upprepade batcher om 100 requests, upp till totalt 1 000 körningar per modell. För varje mått beräknades medelvärde, median och 95:e percentilen (p95). Medianen representerar ett typiskt fall, medan p95 ger en bild av hur långsamt det kan bli i värre men fortfarande realistiska scenarier.

Alla tester genomfördes som så kallade warm runs, vilket innebär att både Node-processen och databasen redan var igång. Syftet är att mäta steady-state-prestanda snarare än cold start-beteende.

Under arbetets gång upptäcktes ett metodfel som behövde korrigeras. I ett tidigt skede räknades backend-tiden dubbelt genom att frontendens fetch-tid summerades med databastiden, trots att fetch-tiden redan inkluderar hela backend-exekveringen. När detta identifierades justerades beräkningarna så att jämförelser i stället bygger på skillnader mellan mätpunkter, vilket ger tydliga och logiskt korrekta resultat.

#### 3.6 Förklaring av mätpunkter och hur de mäts

För att resultaten ska kunna tolkas korrekt krävs en tydlig förståelse för vad varje mätpunkt representerar och hur mätningen genomförs. Nedan beskrivs samtliga mätvärden i den ordning de uppstår i systemets dataflöde.

dbQuery
Mäter den tid databasen använder för att exekvera SQL-frågan och returnera resultatet till backend. Timern startas precis innan SQL-frågan skickas till MySQL och stoppas när databasen har returnerat hela resultatuppsättningen. Detta mått inkluderar därmed all kostnad för `JOIN`-operationer, aggregering (`COUNT`, `GROUP BY`) samt indexanvändning.

JSON-serialisering
Mäter tiden det tar för backend att omvandla databasresultatet till ett JSON-objekt som kan skickas till frontend. Timern startas efter att databasen svarat och stoppas när JSON-strängen är färdigbyggd. Detta mått representerar CPU-arbete i backend snarare än databaskostnad.

Backend total
Utgör summan av dbQuery och JSON-serialisering. Detta mått representerar den totala tiden backend aktivt arbetar med att ta fram ett svar.

Server total
Mäter den totala tiden på serversidan från att en HTTP-request tas emot tills dess att svaret är färdigbyggt och redo att skickas till klienten. Detta inkluderar routing, middleware, databasfråga och serialisering, men exkluderar nätverksöverföring.

Frontend fetch
Mäter tiden från att fetch-anropet initieras i frontend tills dess att JSON-svaret har mottagits och parsats i webbläsaren. Detta mått inkluderar hela backend-exekveringen, nätverkslatens samt webbläsarens JSON-parsning. Frontend fetch representerar därmed den tid användaren väntar innan data överhuvudtaget finns tillgänglig i klienten.

Fetch overhead
Beräknas som frontend fetch minus server total. Detta mått används för att isolera kostnaden för nätverk, protokoll och webbläsarens eget arbete från ren serverexekvering.

Frontend render
Mäter tiden det tar från att frontend uppdaterar sitt state med den mottagna datan tills dess att React har renderat komponenterna och webbläsaren har målat ut resultatet på skärmen. Mätningen sker med hjälp av `requestAnimationFrame` för att säkerställa att renderingen faktiskt har slutförts visuellt.

Frontend total
Utgör summan av frontend fetch och frontend render och representerar den totala tid det tar från användarinteraktion tills dess att färdig data visas i gränssnittet.

System total
Avser den sammanlagda kostnaden för backend och frontend tillsammans och används främst för att illustrera skillnader i helhetsbeteende mellan de två datamodellerna.

### 4. Resultat

För att tydligt kunna jämföra de två datamodellerna presenteras resultaten i tabellform. Samtliga värden baseras på samtliga körningar per modell och redovisas som medelvärde, median, 95:e percentilen (p95) samt total ackumulerad tid. Tider anges i millisekunder (ms) och är avrundade till en decimal där det är tillämpligt.

### 4.1 Normaliserad datamodell – sammanställning

| Mätpunkt               | Medelvärde (ms) | Median (ms) |    p95 (ms) |  Total tid (ms) |
| ---------------------- | --------------: | ----------: | ----------: | --------------: |
| Databasfråga (dbQuery) |         1 526,0 |     1 527,5 |     1 545,1 |     1 526 384,2 |
| JSON-serialisering     |             0,2 |         0,2 |         0,3 |           212,9 |
| Backend total          |         1 527,8 |     1 527,8 |     1 545,3 |     1 526 597,1 |
| Server total           |         1 530,2 |     1 527,8 |     1 545,4 |     1 529 981,5 |
| Frontend fetch         |         1 534,7 |     1 532,3 |     1 549,0 |     1 534 662,8 |
| Frontend render        |            85,3 |        82,6 |       110,8 |        85 417,9 |
| **Frontend total**     |     **1 620,0** | **1 614,9** | **1 659,8** | **1 620 080,7** |
| Fetch overhead         |             4,5 |         4,5 |         5,8 |         4 681,3 |

I den normaliserade modellen är det tydligt att databastiden dominerar alla efterföljande mätvärden. Frontendens fetch-tid ligger mycket nära backend- och server-tiderna, vilket visar att frontend i praktiken bara väntar på databasen.

### 4.2 Denormaliserad datamodell – sammanställning

| Mätpunkt               | Medelvärde (ms) | Median (ms) |  p95 (ms) | Total tid (ms) |
| ---------------------- | --------------: | ----------: | --------: | -------------: |
| Databasfråga (dbQuery) |             1,2 |         1,2 |       1,3 |        1 318,6 |
| JSON-serialisering     |             0,1 |         0,1 |       0,2 |          113,7 |
| Backend total          |             1,3 |         1,3 |       1,4 |        1 432,3 |
| Server total           |             1,4 |         1,3 |       1,4 |        1 469,8 |
| Frontend fetch         |            43,0 |        41,8 |      55,6 |       42 967,8 |
| Frontend render        |            52,2 |        49,1 |      85,5 |       39 501,6 |
| **Frontend total**     |        **95,2** |    **90,9** | **141,1** |   **82 469,4** |
| Fetch overhead         |            41,6 |        40,5 |      54,2 |       41 498,0 |

I den denormaliserade modellen blir databastiden mycket låg. I stället är det frontendens fetch- och render-tid som står för den största delen av den totala väntetiden. Detta visar att när databasen inte längre är flaskhalsen blir systemets övriga kostnader, som nätverk, JSON-parsning och rendering i webbläsaren, tydligare.

#### 4.3 Jämförande översikt

| Mätpunkt                    | Normaliserad medel (ms) | Denormaliserad medel (ms) | Förbättringsfaktor |
| --------------------------- | ----------------------- | ------------------------- | ------------------ |
| Databasfråga                | 1526,0                  | 1,2                       | 1272x              |
| Backend total               | 1527,8                  | 1,3                       | 1175x              |
| Server total                | 1530,2                  | 1,4                       | 1093x              |
| Frontend fetch              | 1534,7                  | 43,0                      | 35,7x              |
| Frontend Render             | 85,3                    | 52,2                      | 1,6x               |
| End-to-end (Frontend total) | 1620,0                  | 44,0                      | 17,0x              |
| Fetch overhead              | 4,5                     | 41,6                      | 0,1x               |

Den jämförande tabellen visar tydligt att prestandavinsten är mycket stor och sträcker sig över flera storleksordningar, särskilt i databasskiktet.

Varför "system total" inte summeras
Det är viktigt att notera att frontend fetch redan inkluderar hela backend‐exekveringen (databas, backend‐logik och serialisering), samt nätverksöverföring och webbläsarens JSON‐parsning. Att summera backend‐tid och frontend‐fetch skulle därför innebära dubbelräkning. I stället redovisas:

End‐to‐end som frontend total (fetch + render), vilket bäst representerar användarens faktiska väntetid.

Denormalisering gör att flaskhalsen flyttas från databasen till nätverk och klienten. Det innebär att svarstiderna hamnar långt under de nivåer som brukar upplevas som långsamma av användare. Även om fetch-overheaden blir större i förhållande till databastiden när databasen inte längre är flaskhalsen, minskar den totala väntetiden kraftigt, vilket ger en tydligt bättre användarupplevelse.

#### 4.1 Normaliserad datamodell

I den normaliserade modellen har databasen genomgående långa svarstider. Den genomsnittliga databastiden ligger runt 1,5-1,6 sekunder per anrop. Frontendens fetch-tid ligger nästan på samma nivå, vilket är väntat eftersom fetch-tiden redan inkluderar hela backend-exekveringen.

Detta visar att databasen är den tydliga flaskhalsen i systemet. Så länge databasen måste utföra `JOIN`-operationer och aggregeringar med `COUNT` och `GROUP BY` blockeras hela kedjan. Frontenden gör i praktiken inget annat än att vänta på att datan ska bli klar.

#### 4.2 Denormaliserad datamodell

I den denormaliserade modellen reduceras databastiden till omkring en millisekund per request. Databasen utför i princip inget arbete vid läsning utöver att läsa färdig data.

Frontendens fetch-tid hamnar här i stället runt 40-50 millisekunder. Detta beror på att när databasen inte längre är flaskhalsen blir nätverk, JSON-parsning och JavaScript-exekvering den dominerande kostnaden.

#### 4.3 Jämförande analys

Skillnaden mellan modellerna är flera storleksordningar. I den normaliserade modellen dominerar databastiden hela systemets beteende. I den denormaliserade modellen flyttas flaskhalsen bort från databasen, vilket blottlägger den faktiska overhead som finns i resten av systemet.

Detta innebär att backend- och frontend-tiderna blir mer lika varandra först när databasen är tillräckligt snabb. När databasen är långsam döljs dessa kostnader helt.

### 5. Diskussion

#### 5.4 Begränsningar i studien

En tydlig begränsning i detta arbete är att alla tester har genomförts på en personlig dator i en miljö som inte varit helt isolerad. Det innebär att andra program, bakgrundsprocesser och varierande nätverksförhållanden kan ha påverkat resultaten. Dessutom kördes databasen och applikationsservern på samma maskin, vilket inte motsvarar hur en produktionsmiljö vanligtvis är uppbyggd.

Arbetet har även fokuserat på läsprestanda för en specifik typ av vy, i detta fall en feed- eller listvy. Resultaten kan därför inte direkt överföras till alla typer av system, till exempel skrivintensiva eller transaktionskritiska applikationer.

Trots dessa begränsningar bedöms resultaten vara tillräckligt tydliga för att besvara forskningsfrågorna, eftersom skillnaderna mellan datamodellerna är mycket stora.

#### 5.0 Alternativa strategier för prestandaoptimering

Datamodellering är en av flera sätt att förbättra prestanda i datadrivna applikationer. I praktiken används ofta flera tekniker tillsammans för att uppnå en bra användarupplevelse.

En vanlig metod är caching, till exempel med in-memory-cache eller externa system som Redis. Genom att lagra färdig data i minnet kan svarstiderna minska kraftigt. Nackdelen är att cache innebär extra komplexitet, särskilt när det gäller att hålla datan uppdaterad och undvika inkonsistens.

Pagination och infinite scroll är andra tekniker som kan minska mängden data som skickas vid varje anrop. Det minskar både belastningen på databasen och mängden data som skickas över nätverket, men påverkar samtidigt användarupplevelsen och passar inte alla typer av vyer.

Ett annat alternativ är att utföra aggregeringar i backend eller att använda förberäknade räknare. Dessa lösningar kan minska belastningen på databasen, men innebär i praktiken samma avvägning som denormalisering, där uppdateringar blir mer komplexa.

I detta arbete valdes datamodellering som den huvudsakliga optimeringsstrategin för att tydligt isolera dess effekt och visa hur stora prestandaskillnader som kan uppstå enbart genom förändringar i databasskiktet.

#### 5.1 Tolkning av resultat

Vid en första anblick visar resultaten en mycket stor numerisk skillnad mellan den normaliserade och den denormaliserade datamodellen. För att förstå vad dessa siffror faktiskt innebär krävs dock en mer sammanhängande tolkning av hur systemet beter sig som helhet.

I den normaliserade modellen är databasen den tydligt dominerande flaskhalsen. Databastiden ligger konsekvent på runt 1,6 sekunder per request, vilket innebär att alla efterföljande steg i systemet - backend-logik, nätverksöverföring och frontend-rendering - i praktiken blockeras tills databasen är klar. Även om dessa steg i sig är relativt billiga, får de ingen möjlighet att påverka den totala tiden eftersom de helt enkelt väntar på databasen.

I den denormaliserade modellen sker ett tydligt skifte i systemets beteende. När databastiden reduceras till omkring en millisekund upphör databasen att vara begränsningen. I stället synliggörs den faktiska kostnaden för övriga delar av systemet, främst nätverk och webbläsarens arbete med att ta emot och parsa JSON. Detta innebär inte att frontend eller backend har blivit långsammare, utan att de alltid haft en viss kostnad som tidigare dolts av databasen.

Denna observation är central för arbetets slutsatser. Prestanda är inte en absolut egenskap hos ett enskilt lager, utan ett resultat av samspelet mellan flera lager. När ett lager är tillräckligt långsamt maskerar det alla andra. Först när den största flaskhalsen elimineras blir systemets verkliga fördelning av kostnader synlig.

#### 5.2 Varför frontend inte är problemet

Ett centralt resultat är att frontend i sig inte är problemet. I den normaliserade modellen följer frontendens fetch-tid nästan exakt databastiden, vilket visar att frontend inte är långsam utan blockerad.

När databasen blir snabb i den denormaliserade modellen blir frontend automatiskt snabbare, utan att någon frontend-kod ändras. Detta understryker att frontend-prestanda i datadrivna applikationer ofta är ett systemproblem snarare än ett frontendproblem.

#### 5.3 Trade-offs

Valet mellan normaliserad och denormaliserad datamodell innebär i praktiken ett val mellan olika typer av komplexitet. I den normaliserade modellen ligger komplexiteten främst vid läsning. Databasen måste utföra `JOIN`-operationer och aggregeringar vid varje request, vilket kan bli kostsamt vid stora datamängder. Fördelen är att uppdateringar är enklare och att datakonsistens naturligt upprätthålls.

Den normaliserade modellen lämpar sig därför väl i system där korrekthet är central, där data uppdateras ofta och där läsningar inte dominerar belastningen. Exempel på detta kan vara administrativa system eller applikationer med komplex affärslogik.

I den denormaliserade modellen flyttas komplexiteten i stället till uppdateringsfasen. Läsningar blir mycket snabba och förutsägbara, vilket gör modellen särskilt lämpad för read-tunga vyer såsom feeds, dashboards och publika listor. Nackdelen är att applikationen själv måste hantera synkronisering och säkerställa att den färdiga vyn hålls uppdaterad.

Vilken modell som är mest lämplig beror därför på användningsområde och belastningsmönster. Arbetets resultat visar att det inte finns en universell lösning, utan att datamodellering bör anpassas efter hur systemet faktiskt används.

### 5.7 Framtida förbättringar

Detta examensarbete har haft som mål att isolera effekten av datamodellering på prestanda och användarupplevelse. För att uppnå detta har flera medvetna avgränsningar gjorts, vilka samtidigt innebär begränsningar i hur resultaten kan generaliseras.

För det första har arbetet fokuserat uteslutande på läs-prestanda. I praktiska system är dock skrivoperationer ofta lika viktiga, särskilt i denormaliserade lösningar där uppdateringar måste propageras till flera fält eller tabeller. Att mäta write-kostnader och analysera hur denormaliserade tabeller hålls konsistenta över tid hade därför varit ett naturligt och värdefullt nästa steg.

För att ytterligare stärka tillförlitligheten hade testerna kunnat genomföras på en isolerad testmaskin, med databasen placerad på en separat server och kommunikationen över ett kontrollerat och isolerat nätverk. En sådan uppsättning hade minskat externa störfaktorer och gjort resultaten mer reproducerbara.

Utöver detta hade arbetet kunnat kompletteras med fler typer av mätningar. Verktygsbaserade frontend-analyser, exempelvis genom Lighthouse, hade kunnat koppla de uppmätta tiderna till etablerade UX-metriker såsom First Contentful Paint och Time to Interactive. På databassidan hade en djupare analys med `EXPLAIN` eller `EXPLAIN ANALYZE` kunnat ge ytterligare insikt i var databasen spenderar mest tid i den normaliserade modellen.

Trots dessa begränsningar bedöms arbetets metodval vara tillräckligt robusta för att besvara de uppställda forskningsfrågorna. Framför allt är skillnaderna mellan datamodellerna så pass stora att de inte rimligen kan förklaras av mätbrus eller miljövariationer.

#### 5.5 Metareflektion - lärdomar från arbetet

Utöver de tekniska resultaten har detta examensarbete gett flera viktiga insikter om hur frontendprestanda bör förstås och hanteras i praktiken. En central insikt är att prestandaproblem i datadrivna applikationer sällan kan lösas isolerat inom ett enskilt lager. Trots att arbetet genomförts ur ett frontendperspektiv har resultaten tydligt visat att användarupplevelsen i hög grad formas av beslut som fattas långt bortom användargränssnittet.

Arbetet har bidragit till en fördjupad förståelse för sambandet mellan datamodellering och frontendupplevelse. Som frontendutvecklare är det lätt att intuitivt fokusera på rendering, komponentstruktur och state-hantering när en applikation upplevs som långsam. Detta arbete har visat att sådana optimeringar i många fall riskerar att bli sekundära om den bakomliggande datamodellen kräver omfattande beräkningar vid varje läsning.

En annan viktig lärdom är vikten av korrekt och genomtänkt mätning. Under arbetets gång blev det tydligt hur lätt det är att dra felaktiga slutsatser om man inte fullt ut förstår vad som faktiskt mäts. Identifieringen och korrigeringen av den initiala dubbelräkningen av backend-tid illustrerar vikten av att kontinuerligt ifrågasätta både mätmetod och resultat. Detta har stärkt förståelsen för prestandamätning som en iterativ process snarare än ett statiskt moment.

Arbetet har även tydliggjort värdet av att tänka i termer av systembeteende snarare än enskilda optimeringar. När databasen i den normaliserade modellen var den dominerande flaskhalsen doldes kostnaderna i övriga delar av systemet. Först när denna flaskhals eliminerades blev det möjligt att se och resonera kring nätverksöverföring, JSON-parsning och rendering på ett meningsfullt sätt. Denna insikt är direkt överförbar till verkliga utvecklingsprojekt, där det ofta är avgörande att identifiera rätt nivå att optimera.

Sammanfattningsvis har examensarbetet inte bara resulterat i konkreta prestandamätningar utan även bidragit till en mer holistisk syn på frontendprestanda. Arbetet har stärkt förståelsen för hur tekniska beslut i databasskiktet kan få direkta och mätbara konsekvenser för användarupplevelsen, samt vikten av att som frontendutvecklare kunna resonera om och samarbeta kring hela systemets arkitektur.

#### 5.6 Slutsats

Hur data modelleras i databasen har mycket stor påverkan på systemets prestanda och användarupplevelse. Denormalisering kan ge kraftigt förbättrad läsprestanda i read-tunga vyer, men är ett designval som innebär ökade krav på korrekthet och underhåll.

Det övergripande bidraget i detta arbete är att visa att frontend-prestanda inte enbart är en frontendfråga. När rätt lager optimeras kan användarupplevelsen förbättras markant utan att frontend-koden förändras. Fokus bör därför ligga på att få rätt data till frontend så snabbt som möjligt, snarare än att enbart optimera rendering och komponentstruktur.

I praktiken innebär detta att datamodellen bör väljas utifrån hur systemet faktiskt används. För read-tunga vyer kan en read-projection vara ett effektivt sätt att nå stabila och låga svarstider, men lösningen kräver ett tydligt ansvar för hur den denormaliserade datan uppdateras och hålls konsekvent över tid.

#### 6 Referenslista

MYSQL 8.4 (officiella manualen)

-MySQL 8.4 Reference Manual (start/översikt) https://dev.mysql.com/doc/refman/8.4/en/

`SELECT`-statement (grundsyntax + hur `SELECT` hänger ihop med `JOIN`) https://dev.mysql.com/doc/en/select.html

`JOIN` Clause (`JOIN`-syntax)

https://dev.mysql.com/doc/en/join.html

Aggregate functions (`COUNT`, m.fl.) + koppling till `GROUP BY` https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions-and-modifiers.html

(och tabellen med aggregate functions) https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html

`EXPLAIN` + `EXPLAIN ANALYZE` (hur MySQL visar query plan och faktisk körning)

https://dev.mysql.com/doc/en/explain.html

(för output-format)

https://dev.mysql.com/doc/en/explain-output.html

React - Official Documentation.

https://react.dev

MÄTNING / TIMERS (officiella källor)

- Node.js `perf_hooks` / Performance APIs (performance.now i Node via `perf_hooks`)

https://nodejs.org/api/`perf_hooks`.html

- MDN: `performance.now()` (high resolution timestamp i webbläsaren)

https://developer.mozilla.org/en-US/docs/Web/API/Performance/now

- MDN: `requestAnimationFrame`() (callback före nästa repaint - bra för att få "visuell" render-mätning)

https://developer.mozilla.org/en-US/docs/Web/API/Window/`requestAnimationFrame`

- MDN: Fetch API (anrop + respons-hantering)

https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch

- MDN: Response.json() (att .json() läser streamen och parsar JSON)

https://developer.mozilla.org/en-US/docs/Web/API/Response/json

#### 7 Bilagor
![Figur 6](https://raw.githubusercontent.com/tim877/examensrapport/9413b692a3354b45fd8a93242d636e9d59897924/6.png)
Jag kopplar posts till users för att få författarnamn och LEFT `JOIN` mot comments för att kunna räkna hur många kommentarer varje inlägg har. `GROUP BY` används för att slå ihop kommentarerna per post.
![Figur 5](https://raw.githubusercontent.com/tim877/examensrapport/9413b692a3354b45fd8a93242d636e9d59897924/5.png)
Här gör jag en enkel `SELECT` mot en färdig, platt tabell. Det finns inga `JOIN`s eller beräkningar vid läsning - all data är redan förberedd, vilket gör frågan snabbare.

Här skapar jag en MySQL connection pool.
Pooling gör att anslutningar återanvänds istället för att skapas på nytt för varje request, vilket ger bättre prestanda och stabilare beteende vid många samtidiga anrop.
![Figur 4](https://raw.githubusercontent.com/tim877/examensrapport/9413b692a3354b45fd8a93242d636e9d59897924/4.png)
