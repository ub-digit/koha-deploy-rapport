# Koha Deploy

För deploy använder vi Capistrano, ett verktyg skrivet i Ruby som kan användas för att automatisera körning av uppgifter/rutiner i olika servermiljöer (tex. staging, production, lab osv).

Vi har utökat Capistranos arbetsflöde med ett som är anpassat för Koha och hur vi arbetar med git.

Repositoriet för detta finns på github: https://github.com/ub-digit/ub-koha-deploy

Vi använder Koha-deploy för att automatisera deploy av Koha från git men även till viss del för konfigurationshantering, serverkonfiguration, hantering av servertjänster med mera.

Man kan grovt dela in funktionaliteten i följande områden:

- Automatisering av arbetsflöde i Git
- Deploy
- Deploy av plugins
- Hantering av Koha-inställningar (i databas)
- Serverkonfiguration

# Arbetsflöde i Git

  För att förstå hur Koha-deploy automatiserar av arbetsflödet i Git börjar vi med att redogöra för hur själva arbetsflödet är utformat och varför. Man skulle i teorin kunna använda sig av detta arbetsflöde utan Koha-Deploy även om det skulle vara opraktiskt pga de många repetitiva momenten. Koha-deploy används för att automatisera merparten av dessa. Således ingår Koha-Deploy som en central del i hur vi jobbar med Koha och versionshantering.

När vi bestämnde oss för en modell för versionhantering fanns ett antal förutsättningar, vissa under vår kontroll och den del satta av Koha-gemenskapen.

Vi har själva ställt följande krav för hur utveckling i Koha skall ske:

1. Deploy får ej vara för krångligt och måste kunna ske ofta och från godtycklig punkt i vårt git-repo. Vi måste tex. snabbt kunna få upp akuta bugfixar.
2. Vi vill aktivt delta utvecklingen av Kohas och merparten av den kod vi producerar skall skriva så möjligt att få in i Koha.
3. Vi måsta kunna hantera förändringar i kodbasen som aldrig är tänkta att pushas upstream till Koha utan endast kommer användas av oss.
4. Vi har ambitionen att hålla oss någorlunda synkroniserade med Kohas master och INTE underhålla en egen Koha-fork. Vi vill kontinueligt integrera kodändringar både från oss och Koha upstream.

Med dessa krav blir det svårt använda sig av Kohas egna releases. Vi har helt enkelt inte tid att vänta på att den kod vi kontribuerar till Koha via Bugzilla accepteras och pushas upstream, vilket man heller inte alltid sker, vilket bryter mot 1). Om vi använder Koha-releases, som ofta utgår från en väldigt gammal master blir det också svårt att uppfylal 2) då utveckling måste ske mot en uppdaterad master för att undvika kod-konflikter i Bugzilla.

Vi har således valt att inte förhålla oss till Kohas officiella releases, utan skapar egna releases och driftsätta utifrån en godtycklig punkt i vårt Git-repo. Vi vill som sagt också inte göra en fork i klassisk bemärkelse, utan vi vill snarare använda oss av den officiell Koha-master plus ett godtyckligt antal egna patchar. Utifrån dessa krav har vi landad i följande arbetsflöde:

Vi har ett eget Koha-repo på github: https://github.com/ub-digit/Koha

Branchen "master" i detta repositorium pekar på den punkt i Kohas repo som vi för närvarande har som utgångspunkt för utveckling av nya branches, samt byggande av release branches. "master" flyttas fram med jämna mellarnrum, vanligtvis maximalt några månader.

Vi hanterar kodförändringar som inte ingår i Koha master i separata branches med vissa namnkonversioner. Branches som börjar med "gub-dev-" och följs av en beskrivning, innehåller kodförändringar som inte är tänkta bli en del av Koha, utan är specialanpassningar för Göteborgs Universitetsbibliotek. Branches som börjar med "gub-bug-" följt av ett bugnummer i bugzilla och sen en beskrivning är antingen kodförädrningar som vi själva skapat ärenden för i Bugzilla eller andras kodförändringar i Bugzilla som vi applicerat och skapat branches utifrån.

Vissa branches kan också ha ett release-prefix, som sedan följs av någon av ovanstående prefix, och dessa är akriverade branches som tillhör en viss release, mer om detta senare angående hur vi hanterar major releases.

Git är flexiblelt när det gäller vilket arbetsflöde man väljer att använda. Ett exempel på ett vanligt förekommande arbetsflöde finns detaljerat beskrivet här: https://nvie.com/posts/a-successful-git-branching-model. I detta arbetsflöde används en "master"- och en "devel/integration" branch. "master" är alltid i ett stabilt, produktionssättningsbart tillstånd, "devel" innehåller de senaste kodförändringarna från utvecklarnas egna branches som integrerats i den gemensamma kodbasen och betraktas som stabil. Inför ny release skapas en tillfällig release-branch utifrån "devel" där eventuella småförändringar kan göras (vilka även måste propageras tillbaka till "devel") tills denna betraktas som stabil och mergas in i master då release av ny stabil version sker.

Att använda detta arbetsflöde för Koha-utveckling utfrån de ovan ställda kraven är inte lämpligt.

Ett av problemet är att de förändringar som vi arbetar med i egna feature-branches och som med ovanstående arbetsflöde senare mergas in i devel och master också kan komma in Koha upstream master (via ärenden i Bugzilla). Då kan dessutom ytterligare commits eller till med rebasningar tillkommit. Samma, eller liknande, kodförädrninga kan således komma in kodbasen från två olika håll. Detta leder till återkommande svårlösta mergekonflikter med stor risk för handhavandefel och resulterar också i en väldigt trasslig och svårtolkad git-historik.

Våra feature-branches är alltså inte helt under vår kontroll då andra utvecklare kan lägga till commits eller utföra rebasningar mot Koha master. Som konsekvens av detta kan vi inte merga in feature-branches någon annan branch, eftersom den branch mot vilken mergningen sker får ett beroende till en branch för vilken historiken kan skrivas om.

För att undvika problematiken med branchberoeden skapar vi först inför varje release en branch där där startpunkten är en commit i Koha upstream master. För att få de kodförändringar som ligger i feature-branches rebasas dessa sedan in in i denna branch. Konflikter kan dyka upp och måste då lösas. Git-verktyget Rerere (reuse recorded solution) används för att slippa lösa samma konflikt flera gånger. Resultatet blir en release-branch som inte har några beroenden till andra branches, utan i princip bara innehåller en uppsättning cherry-pickade commits (som återfinns i aktuelle feature-branches). Detta då rebase både ignorerar eventuella merge-commits i feature-brancharna, samt spelar upp commits i tur och ordning vilket även resulterar i en helt linjär historik.

Man kan säga att vi valt att förhålla oss till feature-branches mer som en uppsättning patchar (där varje branch, som kan innehålla en eller flera commits, motsvarar en patch) som appliceras appliceras i en branch som utgår från punkt i en ren Koha master. Detta har fördelen att när en kodförändring som har en motsvarande featurebranch hos oss kommer in i Koha upstream, kan denna helt enkelt plockas bort hos oss utan sidoeffekter. Om en branch ändras i Bugzilla kan man på samma sätt bara ta bort den ersätta med ny. Vid nästa branchbygge kommer aktuella kodförändringar in via den nya uppsättningen branches.

För att förklara vad som händer när vi bygger en release-branch kan vi utgå från följande exempel med ett antal fiktiva feature-branches:

Vi har följande branches:

- master
- gub-bug-123-fixa-bug-i-koha
- gub-dev-egna-anspassnigar-1
- gub-dev-egna-anpassningar-2

## master

Följer Koha upstream master, men synkroniseras manuellt, och kommer nästan aldrig innehålla de absolut senaste ändringarna. Kohas upstream master mergas in i denna med jämna mellanrum, i regel i samband med förberedelser inför ny release.

## Feature-branches

"gub-bug-123-fixa-bug-i-koha" har ett motsvarande ärende i Bugzilla (bugnummer 123), "gub-dev-egna-anspassnigar-1" och "gub-dev-egna-anspassnigar-2" innehåller interna anpassingar och finns ej i Bugzilla.

## Skapa ny release (manuellt)

### Flytta fram master

Flytta fram våran master till den punkt vi vill utgå ifrån för den nya releasen. Vanligtvis innebär detta att flytta fram master till senaste committen i Koha upstream master. I föjlande exempler är "upstream-master" en tracking-branch för Kohas master.

```
git checkout upstream-master
git pull
git checkout master
git merge upstream-master
git push
```

### Rebasa feature-branches mot ny master

För att minimera risken för merge-konflikter i ett senare skede av branch-byggandet (då alla feature-branches rebasas in) rebasas alla feature-branches mot ny master. Detta sker genom att lokalt checka ut aktuella branches, rebasa var och en mot master, och sedan force-pusha upp till vårt Koha-repo:

```
git checkout gub-bug-123-fixa-bug-i-koha
git rebase master
git push -f
```

```
git checkout gub-dev-egna-anspassnigar-1
git rebase master
git push -f`

```

```
git checkout gub-dev-egna-anspassnigar-2
git rebase master
git push -f
```

I och med att vi har rebasat och force-pushat har vi nu i praktiken en helt ny uppsättning feature-branches i vår remote, men med samma eller liknande ändringar (beroende på om vi fick merge-konflikter eller ej).

### Skapa ny "tom" release-branch

Sedan skapas den branch som senare kommer utgöra startpunkten för vår nya release:

```
git checkout master
git checkout -b release-2018.07-20180710.1130
```

Vi följer en namnkonversion för release branches med ett statiskt prefix "release-" följt av år och månad för den commit branchbygget utgått ifrån separerade med punkt (2018.07) följt av datum och tid för branch-bygget.

### Rebasa in alla feature branches

Ståendes i den nya tomma release branchen rebasas i tur och ordning alla feature branches in:

```
git rebase gub-bug-123-fixa-bug-i-koha
git rebase gub-dev-egna-anspassnigar-1
git rebase gub-dev-egna-anspassnigar-2
```

Här kan konflikter mellan olika branches uppstå och behöva lösas, men har man aktiverat git Rerere behöver en specifik konflikt bara lösas en gång.

### Pusha release-branch

Vi har nu en release-branch-kandidat som är redo att testas och utvärderas. Men innan detta sker måste den pushas till vårt Koha-repo:

`git push --set-upstream origin release-2018.07-20180710.1130`

### Deploy

Se sista steget under "Release-bygge med Koha-deploy".

## Skapa ny release (med Koha-deploy)

I Koha-deploy så är det mesta i ovanstående arbetsflöde automatiserat. Följande krävs för att utföra motsvarande mot lab-miljön:

### Konfiguration av Koha-deploy

"config/deploy.rb" innehåller generell konfiguration för alla stages (servrar/miljöer mot vilka Koha deployas).

De viktigaste inställningarna är:

### Repot som Koha-deploy använder vid deploy
`set :repo_url, 'https://github.com/ub-digit/Koha.git'`

### Den punkt som Koha-deploy utgår ifrån vid bygget av ny release-branch
`set :koha_deploy_release_branch_start_point, 'master'`

### De feature-branches som skall integreras (rebasas in) vid bygge av ny release-branch:
```
set :koha_deploy_rebase_branches, [
  'gub-bug-14957-marc-permissions',
  'gub-bug-123-fixa-bug-i-koha`',
  'gub-dev-egna-anspassnigar-1',
  'gub-dev-egna-anspassnigar-2'
]
```

"config/deploy/lab.rb" innehåller stage-specifik konfiguration. Så som serverns domännamn, användare, hurvida plack är aktiverat (för automatisk omstart) och vilken katalog på servern Koha skall deployas till med mera.

Koha deploy klonar automatiskt repot som angivits via "repo_url" lokalt i katalogen "./repo" och där sker sedan det lokala branch-byggandet sker. Det är också i detta repo lösning av eventuellt konflikter utförs, samt alla rutiner i Koha-deploys "build" och "branches" namespaces opererar mot.

### Flytta fram master

Det första steget "Flytta fram master" sker manuellt även och är identiskt med ovanstånde steg. Detta då det är en operation som utförs relativt sällan och är ej meningsfullt att automatisera.

### Rebasa feature-branches mot ny master

För att rebasa alla feature-branches mot ny master:

`cap lab koha:branches:rebase`

Vid mergekonflikt kommer Koha-Deploy att krasha, och konflikten måste lösas manuellt. Man upprepar sedan kommandot och löser eventulla konflikter tills dess att alla braches är rebasade.

### Bygg ny release branch

`cap lab koha:build`

Denna rutin automatiserar de manuella stegen ovan, och kommer automatiskt använda tidigare lösningar för eventuella konflikter sparade av Git Rerere. Om nya konflikter uppstår krashar Koha-Deploy varefter de måste lösas manuellt. När konflikten är löst och ändringarna stagade skörs `git rebase --continue`, det kan då uppstå nya konflikter och proceduren upprepas tills rebase är klar. Därefter körs `cap lab koha:build` igen och Koha-Deploy kommer förtsätta med nästa branch (eftersom den konfliktdrabbade redan är integrerad) och krasha på samma sätt om nya konflikter uppstår. När inga mer konflikter återstår att lösa (alternativt om inga nya konflikter uppstod) finns det en lokal branch med samma namnkonversion som i detta manuellt exemplet. Denna behöver dock på samma sätt pushas manuellt, det sista steget är således också identiskt:

`cd repo`
`git push --set-upstream origin release-2018.07-20180710.1130`

### Deploy av release-branch

Ändra inställning i "config/deploy.rb"

`set :branch, 'release-2018.05-20180531.1119'`

Inställningen "branch" är namnet på den branch som deployas av Capistrano/Kona-deploy. Den kod som deployas på servern är alltså tillståndet för nuvarande HEAD i denna branch.

För att deploya körs sedan:

`cap lab deploy`

När kommandot kört klart har aktuell release deployats till lab-servern.

## Versioneringsstrategi

Vi använder oss av en lite okonventionell versionering, som tidigare nämnt ser ett releasenamn ut på följande vis:

`release-YYYY.MM-YYYYMMDD.HHMM`

Där första delen är ett godtyckligt prefix, i vårt fall "release", `YYYY.MM` är år och månad för den commit i master som releasen utgår från, och `YYYYMMDD.HHMM` är det datum som våra lokala feature-branches (via rebase) integrerades i release branchen.

Varje ny release som innebär en framflyttning av master (vilken resulterar i nytt `YYYY.MM`) innehålla betydligt fler nya commits än om vi bygger en ny release från befintling utgångspunkt (som endast involverar ändringar i lokala feature-branches). Därför betraktar vi den första typen av release som en slags "major" release, medan ett branchbygge utan framflyttning av master mer är att betrakta som en minor/patch-version.

Det finns in inställning som vi hittils underlåtigt att nämna som används just för att sätta nytt prefix vid ny "major" release.

`set :koha_deploy_release_branch_prefix, 'release-2018.05-'`

Det finns ingen magi vad gäller datumet, utan det är i vårt fall endast en konvension och sätts manuellt. Varje gång master flyttas fram och feature-branches har rebasats mot den nya mastern behöver man manuellt sätta nytt prefix.

Det finns dock ett problem som uppstår i vakuumet mellan att man påbörjar arbetet med att bygga en ny stabil release på framflyttad master, och behovet kvarstår att kunna bygga releaser baserade på den tidigare startpunkten i master vilket kan illustreras med ett exempel:

Säg att vi har flyttat fram våran master så den ny inte har samma HEAD som då de tidigare releaserna byggdes, vi har även rebasat alla feature-branches så de nu utgår från den senaste committen i den framflyttade mastern. Om vi nu använder Koha-deploy för att bygga en ny release-branch med `cap lab koha:build`, så kommer denna använda startpuntken "master" (om inget annat angetts), samt de nyligen rebasade feature-brancherna. Vad vi nu har gjort är att bygga den nya, för närvaradande otestade och instabila, major releasen. Men säg att vi upptäcker ny bugg problem i produktionsmiljön som akut måste åtgärdas. Vi kan inte åtgärda buggen i aktuell feature-branch, bygga, och deploya då vi då har deployat den nya otestade versionen. Vi vill bygga mot den gamla startpunkten, och med de gamla feature-brancharna så som de såg ut innan rebase.

Vi löser detta genom att innan varje ny major-release arkivera branchen som används som startpunkt, samt alla feature-branches, och spara dessa med det gamla branch namnet som prefix. Att göra detta manuellt är opraktiskt, men som tur är har Koha-deploy funktionallitet för automatisering av denna procedur:

Checka ut alla feature-branches lokalt med release-branch-namn som prefix:

`cap lab branches:checkout[release-2018.07-]`

Pusha branches med detta prefix (de som precis skapats) till remote repo:

`cap lab branches:push[release-2018.07-]`

Skapa också en branch som pekar på den tidigare releasens med samma prefix:

`git branch release-2018.07-master <startpunkt>`

eller om detta sker innan master flytts fram:

`git branch release-2018.07-master master`

och pusha:

```
git checkout release-2018.07-master
git push origin
```
Nu kan vi bygga den "gamla" releasen, utifrån prefixad arkiverad startpunkt och prefixade feature-branches. Sätt sedan följande inställningar i "config/deploy/production.rb":

```
release_prefix = 'release-2018.07-'
set :koha_deploy_branches_prefix, release_prefix
set :koha_deploy_release_branch_prefix, release_prefix
set :koha_deploy_release_branch_start_point, release_prefix + 'master'
```
För att bygga:

`cap production koha:build`

Notera att vi bygger mot stage "production" istället för lab, vilket resulterar i att inställningna i "config/deploy/production.rb" används istället för "config/deploy/lab.rb". De inställningar som definieras i en stage-specifik konfigurationsfil skriver över de återfinns i "config/deploy.rb".

Med denna setup kan vi sedan backporta kritiska buggfixar i de prefixade feature-brancharna, och bygga nya versioner av den gamla major releasen, samtidigt arbetet kan fortsätta att ta fram en ny stabil major-release.

## Övrig funktionallitet

### Hantering av Koha-inställningar (i databas)

Mycket av Kohas funktionalitet beror inte endast på programkoden, utan konfiguration sparad databasen. Dessa inställningar ändrar Kohas beteende på precis samma sätt som kodförändringar gör, och bör därför betraktas som en del av det tillstånd man vill kunna deploya. Det behövs därför en mekanism för att omvandla inställningar i databaser till ett format som kan versionshanteras, eller i alla användas för att synkronisering av databastillstånd mellan olika Kohainstanser. Man måste skilja på data (tex. marc-poster) och inställningar (tex. system-preferences), vill man flytta inställnigar mellan instanser måste man kunna göra detta utan att skriva över övrig data. Vi använder ett ruby-program (https://github.com/ub-digit/extract-koha-config, tanken är att så småningom integrera detta i Koha-deploy) för att extrahera inställningar utifrån ett strukturat format över utvalda fält och tabeller. Denna data sparas i ett strukturerat yaml-format som Koha-deploy kan tolka och omvanlda till ett SQL-script sedan exekveras för att importera aktuella inställningar i valfri Koha-instans. Efter det inställningar extraherats kan desssa importeras i valfi Koha-miljö genom att lägga den exporterade datan i en mapp på servern, och sedan köra:

`cap lab koha:sync-managed-data[/path/to/data]`

Det finns utrymme för förbättring av denna funktionalitet, men det fungerar tillräckligt bra för att vara användbart.

### Serverkonfiguration

Vissa rutiner i Koha-Deploy är tänkte att köras då man sätter upp en ny Koha-instans, och tillhandahåller bland annat samma funktionalitet som Koha-gitify (https://github.com/mkfifo/koha-gitify), men med vissa problem som Koha-gitify lider av åtgärdade. Bland annat går det inte att köra Koha-gitfy mer än en gång, medans Koha-deploys server-konfiguration är designad för att vara idempotent, dvs man får samma resultat utifrån samma tillstånd även om operationen upprepas flera gånger. Ett par buggar är även åtgärdade som Koha-gitfy i alla fall led av vid tiden då denna funktionalitet i Koha-deploy skrevs. Det bör även vara möjligt att köra Koha-deploy upprepade tillfällen för att hämta in uppdaterad konfiguration med Koha-repot som källa, och samtidigt bevara egna ändringar. Det var dock ett tag sedan vi själva körde dessa rutiner, så det finns en risk koden befinner sig i ett trasigt tillstånd. Det fanns ursprungligen även en ambition att Koha-deploy skulle kunna sätta upp en Koha-instans helt från grunden, inklusive skapande av databas med mera, men då detta skrivs krävs att det finns färdig Koha-instans skapad. Koha-deploy tar däremot hand om att omvandla denna från en system-installerad Koha, till en "gitifierad" Koha, där koden som körs kan komma direkt från en utcheckning i Git. För att gitifiera en system-installerad Koha körs:

`cap lab koha:setup-instance`

Ett problem med att utgå från en systeminstallerad Koha är också att script som direkt från kommandoraden så som "koha-shell", "koha-mysql", "koha-plack" kommer från den system-installerade releasen. Man kan inte vara säker på att dessa är kompatibla med den Gitifierade Koha-instansen. Vill man instället använda de script som ligger i repot (under ./debian/scripts) körs följande:

`cap lab koha:adjust-package-installation-scripts`

som lägger till en första rad i varje packet-installerat script som kollar om det finns ett motsvarande script i katalogen Koha deployats till och i så fall använder detta istället.

### Plugins

Plugins är också en komponent i tillståndet för en Koha-instans och bör därför kunna deployas på ett enkelt sätt. Då plugins oftast versionshanteras i egna git-repos skulle deta vara opraktiskt att inkludera dessa som en del vårt Koha repo. Git submoduler skulle kunna vara ett alterativ, men det dåliga rykte de lider av är inte helt oförtjänt. Vi har försökt att välja enklast tänkbara lösning och resultat är att möjlighet finns att på ett strukturerat sätt ange en lista över plugins i en fil i Koha repot med en bestämd sökväg: "koha_deploy/plugins.yaml". Vår version av denna fil ser förnärvarande ut såhär:

```
---
# Koha plugins
- url: https://github.com/ub-digit/koha-plugin-libris-marc-import
  branch: master
- url: https://github.com/ub-digit/koha-plugin-edifact
  branch: master
- url: https://github.com/ub-digit/koha-plugin-get-print-data
  branch: master
- url: https://github.com/ub-digit/koha-plugin-format-facet
  branch: master
```

Formatet är enkelt och består helt enkelt av en listan över poster med två egenskaper, "url", url till git repo för plugin, och "branch", den branch som skall deployas.

Efter det att plugin-funktionalitet aktiverats i Koha kan man skriva följande för att deploya alla plugins via Git till Kohas plugin-katalog:

`cap <stage> koha:plugins-install`

### Bekvämlighetskommandon

#### För att komma direkt in i Koha-shell för aktuell stage
`cap <stage> koha:shell`

#### För att komma direkt in i Mysql-prompt för aktuell stage
`cap <stage> koha:mysql`

#### Ta backup av databas
`cap <stage> koha:stash-database`

#### Ladda ner backup lokalt
`cap <stage> koha:stash-database:download`

#### Tömma Kohas cache
`cap <stage> koha:clear-cache`

#### Starta om Apache
`cap <stage> apache:restart`
