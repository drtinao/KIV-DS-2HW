# KIV-DS-2HW
1) NOVĚ implementované algoritmy v programu (oproti předchozí SP)

Předmětem následujících podkapitol je popis algoritmů, jež byly implementovány v rámci druhé SP.

1.a) Princip fce „failure detector“ - ze strany leadera

Leader považuje za odpojeného klienta takový uzel, jež nereportoval svou barvu po čas delší než 35 sekund. Kontrola stavu uzlů probíhá každé 3 sekundy. Chování je možné změnit - viz sekce s konstantami.

Po zjištění výpadku uzlu začne leader periodicky vypisovat informaci o tom, jak dlouho je uzel odpojen - formát: "Node X is disconnected for N seconds!". Následně je proveden přepočet barev přiřazených jednotlivým uzlům tak, aby byl zachován kýžený poměr barev v síti. Pro kontrolu stavu uzlů byla vytvořena funkce „master_check_for_disc_nodes“, jež je v kódu komentována.

K "přebarvení" uzlu dojde ihned poté, co je detekován výpadek jiného uzlu a následně je leader kontaktován uzlem, jež je stále aktivní.

1.b) Princip fce „failure detector“ - ze strany followera

Follower považuje leadera za invalidního, pokud se mu po odeslání jeho aktuální barvy od leadera vrátí zpráva, jež neodpovídá očekávanému formátu. Tedy leader nevrátí kód "GREEN" ani "RED__". V takovém případě je zahájen výběr nového leadera, který je vybrán stejným principem, který se využívá i při prvotní volbě vedoucího uzlu.

Tedy je proskenována síť, najde se uzel s nejvyšší IP a ten je považován za vedoucí uzel. Pokud uzel, jež inicioval prohledávání sítě má nejvyšší IP v síti, pak se stane leaderem a čeká na připojení ostatních uzlů. V opačném případě se snaží o připojení k uzlu s nejvyšší IP adresou v síti.

1. c) Implementace Docker healthcheck

Follower i leader obsahují ve svém FS soubor, který je označen jako "successful_operation_perf". Pokud je provedena operace, která charakterizuje správnou funkci uzlu, dojde k "touch" daného souboru - což je v podstatě změna času poslední modifikace souboru.

Za "healthy" je pak považován takový uzel, jež provedl validní operaci v posledních 20-ti sekundách - jinými slovy, v posledních 20 sekundách došlo k touch uvedeného souboru. Pokud během 20-ti sekund není provedena validní operace, nedojde k touch souboru a tím pádem je kontejner označen jako unhealthy - nekomunikuje. Kontrola je prováděna každých 5 sekund, chování je samozřejmě možno změnit v Dockerfile.

Z pohledu leadera je za validní operaci považováno každé přijetí zprávy / reportování akt. barvy od followera. Ze strany followera se jedná o přijetí validní zprávy (přiřazené barvy) od leadera.

Funkčnost lze vyzkoušet simulováním chyby - napojím se do kontejneru přes ssh a zastavím vykonávání skriptu -> kontejner přejde do stavu "unhealthy".

2) Konstanty umožňující konfiguraci programu

2.a) Změna času, po kterém je follower považován za odpojeného

„SLAVE_DISCONNECTED_SEC“ v souboru nodeTools.py. Jedná se o čas v sekundách, do kterého musí follower kontaktovat leadera, jinak je považován za neaktivního.

2.b) Změna frekvence kontroly odpojených uzlů

„SLAVE_DISCONNECTED_SEC_CHECK_SEC“ v souboru nodeTools.py. Čas v sekundách vyjadřující frekvenci kontroly stavu uzlů (aktivní / odpojený).

3) Komunikační náročnost implementovaného distribuovaného algoritmu

Followeři periodicky zasílají leaderovi svou barvu kódovým označením, jež má fixní délku 5B - jedná se o "GREEN", "RED__" nebo "NONE_". Leader odpovídá stejným formátem zpráv. Jelikož za každou zprávou následuje odpověď a délka zpráv je stejná, lze náročnost vyjádřit vztahem 2 \* n, kde n je délka zprávy. V tomto případě 2 \* 5 = 10 B.

Množství přenesených dat napříč uzly lze snadno redukovat snížením frekvence, jakou followeři kontaktují leadera - viz popis konstanty „MASTER_CONTACT_SEC“. Snížení přenosu dat by se dalo docílit i změnou protokolu, kdy by jednotlivé barvy měly kratší kód - např. "R", "G", "N".

# Níže pokračuje dokumentace dodaná k předchozí úloze.
1) Algoritmy použité v programu

Předmětem následujících podkapitol je popis algoritmů, jež byly implementovány v rámci první SP.

1.a) Volba master uzlu, princip fce aplikace

Volba master uzlu je zajištěna implementací tzv. „bully algoritmu“. Za master uzel je tedy považován ten uzel v síti, jež má nejvyšší IP adresu.

Řídící program při spuštění přijímá argument vyjadřující počet očekávaných uzlů v síti. Předpokládá se, že veškeré uzly v síti mají hostname „node-1“, „node-2“ až „node-x“. Při spuštění uzlu se tedy zahájí prohledávání sítě funkcí „retrieve_network_nodes“, jež zkouší překládat hostname na IP adresy. Pokud proběhne překlad hostname → IP v pořádku, funkce start_node_ping ověří, že nalezený uzel je schopný pingovat. Prohledávání sítě probíhá dokud není nalezen očekávaný počet uzlů (parametr programu) či dokud nevyprší timeout.

Po získání seznamu ostatních uzlů v síti je vyhodnocena nejvyšší IP adresa v síti. Pokud má spouštěný uzel danou IP, pak je prohlášen za mastera a očekává připojení od ostatních uzlů v síti – funkce „master_node_listen“. Před samotným umožněním připojení ostatních uzlů je vypočítán počet zelených / červených uzlů funkcí „master_count_colors“.

Pokud spouštěný uzel nemá nejvyšší IP v síti, pak je uzel prohlášen za slave uzel a pokouší se o připojení k masteru – funkce „slave_node_connect“.

1.b) Nastavení barvy uzlů

Master uzel po spuštění a získání dat o stavu strojů v síti vypočítá potřebný počet červených a zelených uzlů – funkce „master_count_colors“. Informace získané výpočtem (= odpovídající barvy) jsou poté zasílány jakožto odpovědi jednotlivým uzlům.

Slave uzly pravidelně zasílají masteru zprávy, ve kterých sdělují svou aktuální barvu. Master odpovídá barvou, kterou by uzly měly mít nastaveny. Zde je zřejmé, že barva se může v průběhu běhu programu měnit, jelikož některý z uzlů může přestat komunikovat atp. V takovém případě by byl nutný přepočet, aby byly zachovány požadované poměry barev v síti.

Kontakt masteru slave uzlem probíhá pravidelně každých 15s, interval je možno změnit – viz sekce „Konstanty umožňující konfiguraci programu“.

1.c) Kontrola stavu uzlů masterem

Slave uzly pravidelně informují mastera o své barvě – funkce „slave_send_color_mes“. Tímto způsobem masteru sdělují i informaci o tom, že jsou stále aktivní a nedošlo k ukončení funkčnosti uzlu. //To se může hodit v dalších úlohách, kdy bude potřeba řešit výpadky uzlů v síti.

2) Konstanty umožňující konfiguraci programu

2.a) Změna počtu uzlů

Počet uzlů lze změnit přímo ve Vagrantfile, jež se nachází v kořenovém adresáři. Řádka 19. „NODES_COUNT“ – defaultní hodnota je 6 uzlů. Tedy 4 uzly červené, 2 zelené.

2.b) Timeout – nalezení všech uzlů v síti

„DISCOVERY_TIMEOUT_SEC“ v souboru supportTools.py. Vyjadřuje čas v sekundách, do kterého musí uzel nalézt očekávaný počet uzlů v síti (tzn. úspěšně přeložit hostname na IP a pingnout uzel).

2.c) Timeout – připojení slave -> master

„MASTER_CONNECT_SEC“ v souboru nodeTools.py. Počet sekund, do kterých musí slave navázat spojení s masterem.

2.d) Interval odeslání aktuální barvy slave -> master

„MASTER_CONTACT_SEC“ v souboru nodeTools.py. Interval v sekundách, jež vyjadřuje frekvenci kontaktování mastera slave uzlem.

3) Spuštění aplikace

Pro spuštění je potřeba mít nainstalován Vagrant a Docker. Poté stačí v kořenovém adresáři pouze zadat příkaz „vagrant up“ a aplikace bude sestavena / spuštěna. Program následně běží do té doby než je ukončen (např. ctrl + c). Do terminálu jsou každých 15s (pokud není změna, viz konstanty) vypisovány informace od každého z uzlů.
