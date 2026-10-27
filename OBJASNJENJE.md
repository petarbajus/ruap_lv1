# Objašnjenje rješenja — RUAP LV1

Bilježnica `RUAP_LV1.ipynb` rješava tri zadatka iz odjeljka 5. Ovaj dokument opisuje što svaki dio radi i zašto je postupak takav.

Izvorni odgovori stoje u `RUAP_LV1_podaci/izvorno`, a izdvojeni zapisi u `RUAP_LV1_podaci/obradeno`. Ta je podjela iz odjeljka 1.2: sirovi odgovor ostaje netaknut, pa se tablica ili citati mogu ponovno izgraditi ako se promijeni samo obrada. Funkcija `spremi_odgovor` uz tijelo odgovora piše i `.meta.json` s vremenom dohvata i točnom adresom zahtjeva. Metapodatak bilježi odakle je sadržaj došao, što kasnije služi i kao `source_url` za citate.

## Zadatak 1. HTTP zahtjevi i odgovori

Svi pozivi idu na JSONPlaceholder, ogledni API iz odjeljka 2. Zaglavlje `Accept: application/json` traži JSON, a `timeout=30` ograničava čekanje na vezu i na čitanje, kako je opisano u odjeljku 2.5. Status se provjerava s `raise_for_status()` prije nego što se tijelo tumači kao traženi zapis.

### 1.a) Zbirka i pojedinačni resurs

Uspoređena su dva GET zahtjeva:

- `/todos` s parametrima `userId=1` i `id=1`
- `/todos/1`

Parametri idu kroz rječnik `params`. Requests ih sam kodira u upitni niz, pa se adresa ne slaže ručno i ostaje čitljivo koji je filter primijenjen.

Oba odgovora imaju status 200 i `Content-Type: application/json`. Razlikuju se po obliku. Filtrirana zbirka vraća JSON listu s jednim objektom. Putanja s identifikatorom vraća jedan JSON objekt. To je razlika iz odjeljka 2.2: prvi zahtjev bira iz zbirke, drugi dohvaća jedan resurs. Usporedba `odgovor_zbirka.json()[0] == odgovor_resurs.json()` pokazuje da je riječ o istom zadatku, samo u drugom obliku. Zato kod koji očekuje objekt ne smije jednako obraditi odgovor zbirke, i obrnuto.

### 1.b) Nepostojeći zadatak

`GET /todos/999999` vraća 404. Tijelo je `{}`, što je valjan JSON, pa `response.json()` uspije i kad zahtjev nije uspio. Zato se status gleda prije čitanja podataka: poruka o pogrešci može i sama biti JSON, a uspješan parse ne znači da je stigao zadatak.

`raise_for_status()` za statuse 4xx i 5xx diže `requests.HTTPError`. Iznimka je uhvaćena, pa bilježnica nastavlja, a pogreška ostaje vidljiva u ispisu. Tijelo se ispisuje i kao tekst i kao JSON upravo zato da se vidi ta zamka.

### 1.c) Zadaci korisnika 2

Prvi zahtjev traži sve zadatke korisnika 2 (`userId=2`). Drugi dodaje `completed` s vrijednošću `"true"`. Tekst `"true"` traži i zadatak i sam API: boolean `True` u Pythonu Requests bi u upit stavio kao `True`, a ovaj poslužitelj filtrira po slovima `true`.

Odgovor ima 20 zadataka, od kojih je 8 dovršeno. Provjera nije samo broj zapisa. Svaki zapis mora imati `userId` 2, dovršeni moraju imati `completed == True`, a njihovi `id` moraju biti podskup svih zadataka tog korisnika. Time se vidi da je filter stvarno primijenjen, a ne da je stigao neki drugi skup iste duljine.

### 1.d) HEAD i GET

Za isti resurs `/todos/1` šalju se `requests.get` i `requests.head`. HEAD je, po odjeljku 2.4, ista radnja kao GET bez tijela odgovora. Zato se uspoređuju status, `Content-Type`, `etag`, `Transfer-Encoding` i duljina tijela.

Status, vrsta sadržaja i `etag` se podudaraju, pa oba odgovora opisuju istu reprezentaciju. Tijelo GET-a je JSON zadatka (83 bajta). Tijelo HEAD-a je prazno. GET ima `Transfer-Encoding: chunked` jer se tijelo šalje u dijelovima. Kod HEAD-a tog zaglavlja nema jer tijela nema.

Ispisuju se samo ta zaglavlja. Ostala, poput onih koja dodaje posrednik, ne govore ništa o razlici između dviju metoda. `requests.head()` po zadanim postavkama ne prati preusmjeravanja. Ova adresa vraća 200 izravno, pa ta postavka na usporedbu ne utječe.

## Zadatak 2. Dohvat vremenskih podataka

Izvor je Open-Meteo Historical Weather API, model ERA5, adresa iz odjeljka 3.2. Razdoblje je 1.–7. siječnja 2024., uključivo, zona `UTC`, model `era5`. Tražene koordinate su one iz odjeljka 3.1: Osijek (45.55, 18.70), Zagreb (45.81, 15.98) i Split (43.51, 16.44).

ERA5 vraća točku mreže blizu tražene lokacije, ne očitanje meteorološke postaje. Zato se u ispisu vide i tražene i vraćene koordinate. Stupac `location` i dalje nosi ime grada koji smo tražili, jer redak opisuje taj upit, a ne ime najbliže točke mreže.

Podaci su pod licencom CC BY 4.0. Uz skup su navedeni izvor, poveznica na licencu i promjene: tri grada, jedan tjedan, odabrane satne varijable, te spajanje lista iz `hourly` u tablicu sa stupcem `location`. Besplatni pristup ovdje je nastavni.

Zajednička funkcija `parametri_vremena` drži razdoblje, zonu i model na jednom mjestu. Time sva tri grada, i kasniji upit s vjetrom, dobivaju iste ostale parametre. `dohvati_grad` šalje zahtjev, provjerava status i odmah sprema sirovi odgovor pod zasebnim imenom. `tablica_iz_hourly` iz jednako dugih lista u `hourly` radi redake, preimenuje `time` u `time_utc`, označava zonu s `utc=True` i na početak dodaje grad. `utc=True` je potreban jer je u upitu tražena zona UTC, a CSV tu zonu ne pamti sam.

### 2.a) Tri grada, zaseban izvorni odgovor

Za svaki grad nastaju datoteke `{grad}.json` i `{grad}.meta.json`. Zaseban naziv traži odjeljak 3.3: svaki upit ima vlastiti izvorni odgovor. Jedna zajednička datoteka brisala bi vezu između grada i točnog URL-a kojim je dohvaćen.

Svaki grad ima 168 sati, od `2024-01-01T00:00` do `2024-01-07T23:00`. To je sedam uključenih dana po 24 sata. Jedinice u ovom odgovoru su °C, % i mm.

### 2.b) Zajednička tablica `vrijeme.csv`

Tablica se slaže iz spremljenih JSON datoteka, a ne iz odgovora koji je još u memoriji. Tako se odmah vidi da izdvajanje može ići iz arhive. `pd.concat(..., ignore_index=True)` spaja tri okvira u jedan i daje nove redne brojeve, jer stari indeksi 0–167 iz svakog grada inače ostaju ponovljeni.

Stupci su oni iz odjeljka 3.4: `location`, `time_utc`, `temperature_2m`, `relative_humidity_2m`, `precipitation`. Ukupno je 504 retka, po 168 za svaki grad.

`vrijeme.csv` koristi zarez kao razdjelnik i ne piše indeks, kao u odjeljku 3.5. Nakon `read_csv` stupac `time_utc` ponovno se pretvara s `utc=True`, jer CSV čuva tekst, ne vremenski tip. `DataFrame.equals` potvrđuje da su vrijednosti i vremenske oznake sačuvane.

### 2.c) Brzina vjetra

Upit se proširuje varijablom `wind_speed_10m` i podaci se za sva tri grada dohvaćaju ponovno. Novi dohvat traži zadatak: brzina vjetra mora doći iz istog odgovora kao ostale satne liste, na istim položajima. Novi odgovori spremaju se kao `{grad}_vjetar.json`, pa izvorni odgovori iz 2.a ostaju dostupni, a dopunjena tablica ima vlastiti izvor.

Zajednička tablica opet ima 504 retka, s dodatnim stupcem `wind_speed_10m`, i sprema se u `vrijeme_vjetar.csv`. Datoteka `vrijeme.csv` ostaje tablica iz 2.b, jer zadatak 2.b izričito traži taj naziv.

Jedinica se čita iz `hourly_units`, a ne upisuje unaprijed. Odgovor daje `wind_speed_10m` u km/h, uz već poznate °C, % i mm. Taj opis ide u ispis i u `obradeno/opis_vremena.json`, da jedinice ostanu uz podatke i kad se bilježnica ne gleda.

### Ponovno izdvajanje

Zadnja ćelija zadatka 2 čita samo tri `*_vjetar.json` datoteke i gradi tablicu iznova. Jednakost s tablicom iz 2.c pokazuje da mrežni zahtjev za tu provjeru nije potreban.

## Zadatak 3. Izdvajanje citata

Izvor je [Quotes to Scrape](https://quotes.toscrape.com/), stranica Web Scraping Sandboxa namijenjena učenju. Licenca tekstova na stranici nije navedena, pa je to uz skup zapisano. Zahtjev na `/robots.txt` vraća 404: poslužitelj ne daje posebne upute za putanje. `robots.txt` ionako ne bi odredio licencu, samo pristup putanjama.

Polja zapisa su `text`, `author`, `tags` i `source_url`. Skup je JSON jer `tags` mora ostati lista. CSV bi tu listu spljoštio u tekst.

### 3.a) Prve dvije stranice

Obilazak kreće od `https://quotes.toscrape.com/`. Petlja radi najviše dva prolaza i staje i ako poveznice Next nema. Zadatak traži točno to ograničenje: dvije stranice, ili kraj niza ako je kraći.

Nakon svakog uspjelog odgovora spremaju se HTML i metapodaci (`quotes_page_1.html`, `quotes_page_2.html`). Sirovi dokument ostaje pa se izdvajanje može ponoviti. Sljedeća adresa čita se selektorom `li.next a`. `href` je relativan (`/page/2/`), pa `urljoin` od njega i adrese trenutačne stranice radi potpuni URL. Ručno lijepljenje domene puklo bi čim se promijeni oblik relativne putanje.

Na obje stranice status je 200. Druga stranica i dalje ima poveznicu na treću, ali se obilazak tu zaustavlja jer su dvije stranice dovoljne.

### 3.b) Tekst, autor, oznake i adresa

`citati_iz_html` prima već spremljeni HTML. Svaki `div.quote` daje:

- tekst iz `span.text`
- autora iz `small.author`
- oznake iz `.tags a.tag`
- `source_url` iz metapodatka, dakle iz adrese kojom je stranica stvarno dohvaćena

Selektori su oni iz odjeljka 4.3, jer stranica razredima označuje upravo te dijelove citata. `get_text(..., strip=True)` skida suvišne razmake s ruba. Ako na stranici nema nijednog bloka, funkcija diže `ValueError`: prazan rezultat tada znači promijenjenu stranicu, a ne „nema citata”.

Svaka stranica ima 10 citata, ukupno 20. Svi idu u `obradeno/quotes.json`. Ispis autora po stranici služi usporedbi sa sadržajem: isti su ljudi koji stoje u spremljenom HTML-u, jer se zapisi grade iz tih datoteka.

### 3.c) Jedna oznaka

Odabrana je oznaka `life`. Među prikupljenim oznakama pojavljuje se na objema stranicama, pa datoteka pokazuje i filtriranje i to da zapisi dolaze s više adresa. Prije filtriranja provjerava se da oznaka uopće postoji u skupu. Inače bi prazna datoteka izgledala kao uspjeh.

U `quotes_life.json` spremljeno je 6 citata. Zadržana su sva četiri polja, uključujući `source_url`, da se vidi s koje je stranice citat uzet.

### Ponovno izdvajanje

Završna ćelija ponovno čita samo dva HTML-a i njihove metapodatke te uspoređuje listu sa `quotes.json`. Rezultat je jednak. Time je ispunjen zahtjev da se izdvajanje može ponoviti iz spremljenih odgovora, bez novog dohvata.

## Sažetak datoteka

| Datoteka | Uloga |
| --- | --- |
| `izvorno/osijek.json`, `zagreb.json`, `split.json` | Izvorni ERA5 odgovori bez vjetra, uz `.meta.json`. |
| `izvorno/*_vjetar.json` | Isti upit proširen s `wind_speed_10m`. |
| `obradeno/vrijeme.csv` | Zajednička tablica tri grada, stupci iz odjeljka 3.4. |
| `obradeno/vrijeme_vjetar.csv` | Ista tablica s brzinom vjetra. |
| `obradeno/opis_vremena.json` | Izvor, licenca, promjene i jedinice pročitane iz `hourly_units`. |
| `izvorno/quotes_page_1.html`, `quotes_page_2.html` | Spremljene stranice i metapodaci. |
| `obradeno/quotes.json` | Svih 20 citata. |
| `obradeno/quotes_life.json` | Citati s oznakom `life`. |

Isti skup je i u `RUAP_LV1_podaci.zip`.
