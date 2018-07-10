# Koha Deploy

För deploy använder vi Capistrano, ett verktyg skrivet i Ruby som kan användas för att automatisera körning av uppgifter/rutiner i olika servermiljöer (tex. staging, production, lab osv).

Vi har utökat Capistranos arbetsflöde med ett som är anpassat för Koha och hur vi arbetar med git.

Repositoriet för detta finns på github: https://github.com/ub-digit/ub-koha-deploy

Vi använder Koha-deploy för att automatisera deploy av Koha från git men även till viss del för konfigurationshantering, serverkonfiguration, hantering av servertjänster med mera.

Man kan grovt dela in funktionaliteten i följande områden:

- Automatisering av arbetsflöde i Git
- Deploy
- Hantering av Koha-inställningar (som ligger i databas)
- Serverkonfiguration

# Arbetsflöde i Git

För att förstå hur Koha-deploy automatiserar av arbetsflödet i Git börjar vi med att redogöra för hur själva arbetsflödet är utformat och varför. Man skulle i teorin kunna använda sig av detta arbetsflöde utan Koha-Deploy även om det skulle vara hemskt opraktiskt pga de många repetitiva ingående momenten. Koha-deploy används för att automatisera merparten av dessa. Således ingår Koha-Deploy som en central del i hur vi jobbar med Koha och versionshantering.

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

Vissa branches kan också ha ett release-prefix, som sedan följs av någon av ovanstående prefix, och dessa är akriverade branches som tillhör en viss release, mer om detta senare angående hur vi hanterar releases.

Git är flexiblelt när det gäller vilket arbetsflöde man väljer att använda. Ett exempel på ett vanligt förekommande arbetsflöde finns detaljerat beskrivet här: https://nvie.com/posts/a-successful-git-branching-model. I detta arbetsflöde används en "master"- och en "devel/integration" branch. "master" är alltid i ett stabilt, produktionssättningsbart tillstånd, "devel" innehåller de senaste kodförändringarna från utvecklarnas egna branches som integrerats i den gemensamma kodbasen och betraktas som sinstabil. Inför ny release skapas en tillfällig release-branch utifrån "devel" där eventuella småförändringar kan göras (vilka även måste propageras tillbaka till "devel") tills denna betraktas som stabil och mergas in i master då release av ny stabil version sker.

Att använda detta arbetsflöde för Koha-utveckling utfrån de ovan ställda kraven är inte lämpligt.

Ett av problemet är att de förändringar som vi arbetar med i egna feature-branches och som med ovanstående arbetsflöde senare mergas in i devel och master också kan komma in Koha upstream master (via ärenden i Bugzilla). Då kan dessuotm ytterligare commits eller till med rebasningar tillkommit. Samma, eller liknande, kodförädrninga kan således komma in kodbasen från två olika håll. Detta leder till återkommande svårlösta mergekonflikter med stor risk för handhavandefel och resulterar också i en väldigt trasslig och svårtolkad git-historik.

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

`git checkout upstream-master`
`git pull`
`git checkout master`
`git merge upstream-master`
`git push`

### Rebasa feature-branches mot ny master

För att minimera risken för merge-konflikter i ett senare skede av branch-byggandet (då alla feature-branches rebasas in) rebasas alla feature-branches mot ny master. Detta sker genom att lokalt checka ut aktuella branches, rebasa var och en mot master, och sedan force-pusha upp till vårt Koha-repo:

`git checkout gub-bug-123-fixa-bug-i-koha`
`git rebase master`
`git push -f`

`git checkout gub-dev-egna-anspassnigar-1`
`git rebase master`
`git push -f`

`git checkout gub-dev-egna-anspassnigar-2`
`git rebase master`
`git push -f`

I och med att vi har rebasat och force-pushat har vi nu i praktiken en helt ny uppsättning feature-branches i vår remote, men med samma eller liknande ändringar (beroende på om vi fick merge-konflikter eller ej).

### Skapa ny "tom" release-branch

Sedan skapas den branch som senare kommer utgöra startpunkten för vår nya release:

`git checkout master`
`git checkout -b release-2018.07-20180710.1130`

Vi följer en namnkonversion för release branches med ett statiskt prefix "release-" följt av år och månad för den commit branchbygget utgått ifrån separerade med punkt (2018.07) följt av datum och tid för branch-bygget.

### Rebasa in alla feature branches

Ståendes i den nya tomma release branchen rebasas i tur och ordning alla feature branches in:

`git rebase gub-bug-123-fixa-bug-i-koha`
`git rebase gub-dev-egna-anspassnigar-1`
`git rebase gub-dev-egna-anspassnigar-2`

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

Inställningen "branch" är namnet på den branch som deployas av Capistrano/Kona-deploy. Den kod som deployas på servern är alltså tillståndet för nuvarande HEADför denna branch.

För att deploya körs sedan:

`cap lab deploy`

När kommandot kört klart är aktuell release deployad på lab-servern.
