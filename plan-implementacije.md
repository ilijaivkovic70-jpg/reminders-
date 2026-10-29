# Plan implementacije: Podsetnik sa pozivom

Lična aplikacija po uzoru na iOS Reminders, ali jednostavnija i brža. Glavno obaveštenje je telefonski poziv sa robotskim glasom, uz SMS kao rezervu.

## Ukratko

**Stack:** Next.js (TypeScript), Supabase (baza, login, minutni budilnik), Tailwind CSS, shadcn/ui, PWA na iPhone-u, Vercel, provajder poziva i SMS-a (prvo Twilio), tabela `call_log`.

**Odluke koje smo doneli:**
- Robot izgovara: "Podsetnik: sastanak u 16h", pa traži taster (1 = gotovo, 2 = podseti me za 10 minuta).
- Ako se ne javiš ili ne pritisneš taster: SMS stiže odmah, zove ponovo posle 2 minuta, najviše 3 poziva.
- Unos je što brži: naziv + vreme. Ponavljanje i brz unos tekstom dolaze u Fazi 2.
- Prijava je kodom iz emaila (6 cifara), a ne linkom, jer link ne radi dobro u PWA na iPhone-u (vidi korak 0.7).
- Pozive i SMS šalje server, pa ti ne treba prava iOS aplikacija ni Apple developer nalog.

### Kako radi jedan podsetnik

1. Uneseš "Zubar, 16:00". Aplikacija upisuje red u `reminders` i prvi red u `alarms`.
2. Supabase svakog minuta poziva rutu `/api/cron/tick`.
3. Ruta atomski preuzima alarme na redu i traži poziv od provajdera.
4. Telefon zvoni (do 30 sekundi). Kad se javiš, robot izgovara podsetnik i traži taster.
5. Taster 1 zatvara podsetnik, taster 2 ga pomera za 10 minuta.
6. Ako nije bilo potvrde: SMS odmah, novi alarm za 2 minuta, najviše 3 poziva ukupno.
7. Sve se upisuje u `call_log`, pa uvek možeš da vidiš šta se desilo.

Stanja alarma: `pending`, `processing`, `calling`, `confirmed`, `missed`, `failed`, `cancelled`.

### Troškovi i rizici (pročitaj pre početka)

| Stavka | Procena | Napomena |
|:---|:---|:---|
| Poziv ka srpskom mobilnom (Twilio) | oko 0,82 USD po minutu | prema Twilio stranici cena, proveriti u koraku 0.4 |
| 2 podignuta poziva dnevno | oko 50 USD mesečno | 2 x 30 x 0,82 |
| 5 podignutih poziva dnevno | oko 120 USD mesečno | 5 x 30 x 0,82 |
| Poziv koji niko ne podigne | po pravilu se ne naplaćuje | proveriti na računu posle probe |
| SMS ka Srbiji | nije proveren | izvori ukazuju da je skuplji od proseka, proveriti u 0.4 |
| Supabase, Vercel Hobby | besplatno | Vercel Hobby je samo za ličnu upotrebu |
| Claude API (korak 2.2) | zanemarljivo | |

Cena poziva je najveći rizik celog projekta. Zato u Fazi 0 poredimo provajdere pre izgradnje, sav kod za pozive držimo iza jednog fajla (`lib/telephony.ts`) da se provajder lako menja, i dodajemo dnevni limit poziva da greška ne napravi veliki račun.

Opcije za smanjenje troška: jeftiniji provajder (korak 0.4), režim "samo zvoni" gde poziv niko ne podiže a tekst nosi SMS (korak 2.4), kraći poziv.

---

# FAZA 0: Postavka

### 0.1 Alati i nalozi

**Opis:** Instaliramo sve što treba na računaru i otvaramo besplatne naloge. Twilio nalog otvaramo kasnije, u koraku 0.4.

**Zadaci:**
- Instaliraj Node.js (LTS verzija), Git i VS Code.
- Napravi naloge: GitHub, Vercel (prijava preko GitHub-a) i Supabase.
- U terminalu proveri: `node -v` i `git -v`.
- Napravi folder `podsetnik` na računaru.

**Rezultat:** Obe komande ispisuju verziju, nalozi postoje.

**Napomena:** Vercel Hobby plan je samo za lične, nekomercijalne projekte. Ako aplikaciju kasnije deliš ili naplaćuješ, treba plaćeni plan.

### 0.2 Skelet projekta

**Opis:** Pravimo prazan Next.js projekat sa Tailwind-om i shadcn/ui i stavljamo ga na GitHub.

**Zadaci:**
- U folderu `podsetnik` pokreni `npx create-next-app@latest .` Odgovori: TypeScript da, ESLint da, Tailwind da, `src/` folder ne, App Router da.
- Pokreni `npm run dev` i otvori `http://localhost:3000`.
- Pokreni `npx shadcn@latest init`, pa `npx shadcn@latest add button input card checkbox sheet sonner`.
- Napravi repozitorijum na GitHub-u i uradi `git push`.
- Napravi prazan fajl `.env.local` i proveri da je naveden u `.gitignore`.

**Rezultat:** Početna stranica radi na localhost-u, kod je na GitHub-u.

**Napomena:** Fajl `.env.local` nikad ne ide na GitHub, jer sadrži tajne ključeve. `create-next-app` ga već ignoriše, ali proveri.

### 0.3 Prvi deploy na Vercel

**Opis:** Postavljamo prazan sajt na internet i uključujemo automatsko objavljivanje na svaki `git push`.

**Zadaci:**
- Na Vercel-u: Add New Project, izaberi repozitorijum `podsetnik`, Deploy.
- Zapiši javnu adresu (npr. `podsetnik.vercel.app`).
- Promeni tekst u `app/page.tsx`, uradi `git push` i proveri da se sajt sam ažurirao.

**Rezultat:** Javni URL prikazuje tvoju stranicu, svaki push objavljuje novu verziju.

**Napomena:** Preview adrese na Vercel-u traže prijavu, pa spoljni servisi (Supabase, provajder poziva) dobijaju grešku 401. U svim podešavanjima koristi samo produkcioni domen.

### 0.4 Izbor provajdera poziva i probni poziv

**Opis:** Pre izgradnje proveravamo tri stvari od kojih sve zavisi: da poziv i SMS stvarno stižu na tvoj iPhone, da glas zvuči prihvatljivo na srpskom i koliko sve to košta.

**Zadaci:**
- Otvori Twilio nalog (probni) i potvrdi svoj broj (`+381...`) kao verifikovan.
- U Twilio konzoli uključi Srbiju u Voice i SMS geografskim dozvolama (Geo Permissions).
- Uzmi Twilio broj sa mogućnošću poziva i SMS-a.
- Na Twilio stranici cena proveri cenu poziva ka srpskom mobilnom i cenu SMS-a. Izračunaj mesečni trošak: podsetnika dnevno x 30 x cena po minutu.
- Napiši `scripts/test-call.mjs` koji zove tvoj broj sa ugrađenim TwiML-om: `<Say>Podsetnik: probni poziv</Say>`. Pokreni sa `node scripts/test-call.mjs`.
- Na Twilio stranici Text-to-Speech (Available voices and languages) potraži srpski. Ako ga nema, probaj najbliži jezik sa liste i oceni razumljivost.
- Napiši `scripts/test-sms.mjs` na isti način za SMS.
- Testiraj: zaključan telefon, režim Ne uznemiravaj, mobilna mreža i Wi-Fi. Zabeleži koji se broj prikazuje i sačuvaj ga u kontakte kao "PODSETNIK".
- Ako je cena previsoka, uradi isti test sa još jednim provajderom (npr. Plivo ili Telnyx) i uporedi cene i kvalitet.

**Rezultat:** Imaš odluku o provajderu, procenu mesečnog troška, potvrdu da poziv i SMS stižu i potvrdu glasa (ili odluku da ide ElevenLabs iz koraka 2.5).

**Napomena:** (1) Probni Twilio nalog pre tvoje poruke pušta najavu da je nalog probni, a nestaje posle nadogradnje na plaćeni. (2) Naplata je po minutu, pa i poziv od 10 sekundi može koštati ceo minut. (3) Ako SMS sa stranog broja kasni ili ne stiže, kod nekih provajdera SMS ka Srbiji traži registraciju pošiljaoca. (4) Ako glas ne zvuči dobro, korak 2.5 radi pre koraka 1.7.

### 0.5 Supabase projekat i veza sa aplikacijom

**Opis:** Pravimo bazu i povezujemo je sa aplikacijom iz tri konteksta: pregledač, server i administratorski pristup.

**Zadaci:**
- U Supabase-u napravi novi projekat (region Frankfurt) i sačuvaj lozinku baze.
- U `.env.local` upiši: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`.
- Pokreni `npm install @supabase/supabase-js @supabase/ssr`.
- Napravi `lib/supabase/client.ts` (pregledač), `lib/supabase/server.ts` (server, sa kolačićima) i `lib/supabase/admin.ts` (service role, samo server).
- Iste tri promenljive unesi u Vercel (Settings, Environment Variables) i uradi redeploy.

**Rezultat:** Aplikacija se povezuje na bazu iz sva tri konteksta bez greške.

**Napomena:** `SUPABASE_SERVICE_ROLE_KEY` nikad ne sme da dobije prefiks `NEXT_PUBLIC_`, jer bi postao vidljiv svima i zaobilazi zaštitu baze. Besplatni Supabase projekat se pauzira posle otprilike nedelju dana neaktivnosti, a to zaustavlja budilnik. Proveri aktuelne uslove i vidi korak 2.3.

### 0.6 Tabele u bazi i zaštita (RLS)

**Opis:** Pravimo četiri tabele i pravila da svaki korisnik vidi samo svoje redove.

**Zadaci:**
- Napravi `db/001_init.sql` u projektu i izvrši ga u Supabase SQL Editor-u.
- `profiles`: `id` (isti kao auth korisnik), `phone_e164`, `timezone` (podrazumevano `Europe/Belgrade`), `created_at`.
- `reminders`: `id`, `user_id`, `title`, `due_at` (`timestamptz`), `status` (`open` ili `done`), `repeat_rule` (za sada prazno), `notify_call`, `notify_sms`, `completed_at`, `created_at`, `updated_at`, `deleted_at`.
- `alarms`: `id`, `reminder_id`, `user_id`, `fire_at`, `attempt` (1 do 3), `status`, `claimed_at`, `created_at`.
- `call_log`: `id`, `alarm_id`, `reminder_id`, `user_id`, `channel` (`call` ili `sms`), `provider_sid`, `status`, `duration_s`, `pressed_key`, `reason`, `error`, `created_at`.
- Uključi RLS na svim tabelama i dodaj pravilo `user_id = auth.uid()` (za `profiles`: `id = auth.uid()`).
- Dodaj indeks na `alarms (status, fire_at)`.

**Rezultat:** Tabele su vidljive u Table Editor-u, RLS je uključen na sve četiri.

**Napomena:** Bez RLS-a svako ko ima javni ključ može da čita tvoje podatke. Vreme uvek čuvaj kao `timestamptz`, nikad kao `timestamp`.

### 0.7 Prijava kodom iz emaila

**Opis:** Login bez lozinke: uneseš email, stigne kod od 6 cifara, uneseš ga. Radi i u instaliranoj PWA.

**Zadaci:**
- U Supabase Authentication ručno dodaj svog korisnika (Add user), pa isključi javnu registraciju (Allow new users to sign up).
- U email šablonu (Magic Link) umesto linka stavi kod: `{{ .Token }}`.
- Napravi `app/login/page.tsx`: polje za email poziva `supabase.auth.signInWithOtp({ email, options: { shouldCreateUser: false } })`, zatim polje za kod poziva `supabase.auth.verifyOtp({ email, token, type: 'email' })`.
- Dodaj `middleware.ts` (u novijim verzijama Next.js zove se `proxy.ts`, prati šta kaže dokumentacija) koji osvežava sesiju i šalje neprijavljene na `/login`.

**Rezultat:** Možeš da se prijaviš kodom, a neprijavljen korisnik ne vidi stranicu `/`.

**Napomena:** Link iz emaila na iPhone-u otvara Safari, a instalirana PWA ima odvojenu sesiju, pa prijava "nestane". Zato koristimo kod. Ugrađeno slanje emailova u Supabase-u ima mali limit po satu, što je dovoljno za jednog korisnika. Ako dobiješ grešku o limitu, sačekaj ili poveži Resend.

### 0.8 Dizajn osnova i PWA

**Opis:** Definišemo izgled (boje, font, razmake) i pretvaramo sajt u aplikaciju koja stoji na početnom ekranu iPhone-a.

**Zadaci:**
- U `app/globals.css` definiši boje i polumere iz sekcije "Dizajn sistem" kao CSS promenljive. Font Inter učitaj preko `next/font` u `app/layout.tsx`.
- Napravi `app/manifest.ts`: `name`, `short_name: "Podsetnik"`, `display: "standalone"`, `start_url: "/"`, `theme_color`, ikone 192 i 512.
- Dodaj `public/icon-192.png`, `public/icon-512.png` i `public/apple-touch-icon.png` (180 x 180).
- U `app/layout.tsx` podesi `viewport` (`viewportFit: cover`) i centriran okvir maksimalne širine 480 px (mobile first).
- Na iPhone-u u Safari-ju: Share, Add to Home Screen. Otvori aplikaciju sa početnog ekrana i prijavi se.

**Rezultat:** Ikona na početnom ekranu otvara aplikaciju preko celog ekrana, bez Safari trake, i prijava radi.

**Napomena:** iOS traži `apple-touch-icon`, ikone iz manifesta nisu dovoljne. Dodaj `padding-bottom: env(safe-area-inset-bottom)` da donja traka ne prekrije dugmad.

---

# FAZA 1: MVP (srž)

### 1.1 Podešavanja: telefon i vremenska zona

**Opis:** Aplikacija mora da zna koji broj da zove. Pravimo stranicu `/settings`.

**Zadaci:**
- Napravi `app/settings/page.tsx` sa poljem "Broj telefona", prikazom vremenske zone (`Europe/Belgrade`) i dugmetom "Odjavi se".
- Napiši `lib/phone.ts` sa funkcijom `toE164(unos)`: `0641234567` postaje `+381641234567`.
- Napravi server akciju `saveProfile` u `app/actions/profile.ts`: validira broj i radi upsert u `profiles`.

**Rezultat:** Broj se čuva u `profiles` u formatu `+381...`, pogrešan unos daje jasnu poruku.

**Napomena:** Kod srpskih brojeva se početna nula izbacuje i dodaje `+381`. Ako nula ostane, poziv ide na pogrešan broj.

### 1.2 Brz unos podsetnika

**Opis:** Glavna stranica sa jednim poljem i izborom vremena. Pri čuvanju se pravi i prvi alarm.

**Zadaci:**
- U `app/page.tsx` dodaj `components/quick-add.tsx`: polje za naziv, izbor vremena (`input type="datetime-local"`, podrazumevano sledeći pun sat) i prečice "Za 10 min", "Za sat", "Sutra 9:00".
- Napiši `lib/time.ts` (pomoć: `date-fns` i `date-fns-tz`) za pretvaranje lokalnog vremena u UTC.
- Napravi server akciju `createReminder` u `app/actions/reminders.ts`: upisuje red u `reminders` i red u `alarms` (`attempt = 1`, `status = pending`, `fire_at = due_at`).
- Posle čuvanja polje se prazni, a fokus ostaje u njemu.

**Rezultat:** Uneseš "Zubar" i vreme, a u bazi se pojave red u `reminders` i red u `alarms`.

**Napomena:** Server radi u UTC, a `datetime-local` daje vreme bez zone. Konverzija mora da ide iz zone `Europe/Belgrade`, inače podsetnik stiže sat ili dva ranije ili kasnije, a pomeranje na letnje vreme dodatno komplikuje.

### 1.3 Lista podsetnika

**Opis:** Prikazujemo otvorene podsetnike grupisane po danima, sa "Danas" na vrhu.

**Zadaci:**
- Napravi `components/reminder-list.tsx` koji čita `reminders` gde je `status = open` i `deleted_at` je prazno, sortirano po `due_at`.
- Dodaj `groupByDay` u `lib/time.ts` sa grupama: Zakasnelo, Danas, Sutra, Kasnije.
- Red prikazuje naziv, vreme (`HH:mm`) i mali status alarma (Čeka, Potvrđeno, Propušteno).
- Prazno stanje: "Nema obaveza. Dodaj prvu."

**Rezultat:** Lista pokazuje obaveze po grupama, Zakasnelo i Danas su na vrhu.

**Napomena:** "Danas" računaj u svojoj vremenskoj zoni, ne u UTC, inače obaveza u 00:30 završi u pogrešnom danu.

### 1.4 Gotovo, izmena i brisanje

**Opis:** Možeš da završiš podsetnik, promeniš vreme ili ga obrišeš, a alarmi prate svaku promenu.

**Zadaci:**
- Dodaj akcije `completeReminder`, `updateReminder` i `deleteReminder` u `app/actions/reminders.ts`.
- Gotovo: `status = done`, `completed_at = now()`, svi `pending` alarmi tog podsetnika postaju `cancelled`.
- Izmena vremena: stari `pending` alarmi postaju `cancelled`, pravi se novi sa `attempt = 1`.
- Brisanje je meko (`deleted_at = now()`), alarmi postaju `cancelled`.
- Checkbox levo od naziva za "gotovo", a klik na red otvara panel za izmenu (`components/ui/sheet`).

**Rezultat:** Završen ili obrisan podsetnik više ne zove, a izmena vremena pomera poziv.

**Napomena:** Ako zaboraviš da otkažeš alarme, zvaće te za obaveze koje si već završio. Zato tick u koraku 1.5 dodatno proverava status podsetnika neposredno pre poziva.

### 1.5 Tick ruta (probni režim bez poziva)

**Opis:** Srce sistema: ruta koja pronalazi alarme na redu. Za sada samo upisuje u `call_log`, bez pravog poziva, da logiku testiramo besplatno.

**Zadaci:**
- Napiši SQL funkciju `claim_due_alarms(batch int)` u `db/002_claim.sql`. Jednim upitom menja `pending` u `processing` za alarme čiji je `fire_at <= now()`, koristeći `for update skip locked`, i vraća ih.
- Napravi `app/api/cron/tick/route.ts` (POST): proveri zaglavlje `x-cron-secret` u odnosu na `CRON_SECRET`, pozovi `claim_due_alarms` preko `lib/supabase/admin.ts`, za svaki alarm učitaj podsetnik i profil.
- Ako je podsetnik gotov ili obrisan, alarm postaje `cancelled`. Inače upiši red u `call_log` (`reason = dry_run`) i alarm prebaci u `calling`.
- Dnevni limit: promenljiva `MAX_CALLS_PER_DAY` (npr. 15). Kad se prekorači, ruta ne zove i upisuje grešku u `call_log`.
- Čuvar: alarmi u stanju `processing` ili `calling` duže od 10 minuta postaju `failed`.
- Dodaj `CRON_SECRET` (duga slučajna reč) u `.env.local` i Vercel, pa uradi deploy.
- Testiraj sa `curl -X POST` i zaglavljem `x-cron-secret`.

**Rezultat:** Alarm sa vremenom u prošlosti posle poziva rute dobija `calling` i red u `call_log`, a drugi poziv rute ga ne uzima ponovo.

**Napomena:** Bez `skip locked` dva preklopljena poziva rute uzmu isti alarm i dobiješ dupli poziv. Ruta mora brzo da vrati odgovor 200.

### 1.6 Minutni budilnik u Supabase-u

**Opis:** Podešavamo da Supabase svakog minuta zove rutu `tick`. Vercel-ov besplatni cron radi samo jednom dnevno, pa ga ne koristimo.

**Zadaci:**
- U Supabase-u (Database, Extensions) uključi `pg_cron` i `pg_net`.
- Sačuvaj `CRON_SECRET` u Supabase Vault.
- U SQL Editor-u napravi zadatak: `select cron.schedule('tick-every-minute', '* * * * *', $$ select net.http_post(url := 'https://TVOJ-DOMEN/api/cron/tick', headers := jsonb_build_object('x-cron-secret', '<tajna>')) $$);`
- Proveri u tabeli `cron.job_run_details` da se zadatak izvršava svakog minuta.
- Napravi probni podsetnik za 2 minuta i gledaj da li se u `call_log` pojavi `dry_run`.

**Rezultat:** Svakog minuta stiže poziv rute, a podsetnik u prošlosti dobija `dry_run` zapis u roku od oko minut.

**Napomena:** URL mora biti produkcioni (vidi 0.3). `pg_net` ne čeka odgovor, pa greške tražiš u tabeli `net._http_response`. Budilnik radi na nivou minuta, pa poziv može stići do minut posle zadatog vremena.

### 1.7 Prvi pravi poziv

**Opis:** Zamenjujemo probni režim pravim pozivom: telefon zvoni i robot izgovara podsetnik.

**Zadaci:**
- Instaliraj biblioteku izabranog provajdera (za Twilio: `npm install twilio`).
- Napravi `lib/telephony.ts` sa funkcijama `placeCall({ to, alarmId })` i `sendSms({ to, body })`. Ostatak koda zove samo njih.
- Dodaj promenljive: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`, `APP_URL` (lokalno i na Vercel-u).
- Napravi `app/api/twilio/voice/route.ts` (POST) koja vraća TwiML sa `<Say>`: "Podsetnik: {naziv}, u {vreme}" (glas i jezik iz koraka 0.4).
- Tick sada poziva `placeCall` (zvonjenje 30 sekundi, `statusCallback` ka `/api/twilio/status`), upisuje `provider_sid` u `call_log` i alarm prebacuje u `calling`.
- Proveravaj potpis svakog zahteva sa `twilio.validateRequest` i zaglavljem `X-Twilio-Signature`, nevalidne odbij.
- Napravi podsetnik za 2 minuta i javi se kad zazvoni.

**Rezultat:** U zakazano vreme (uz odstupanje do oko minut) telefon zvoni i čuješ naziv podsetnika.

**Napomena:** Provera potpisa pada ako URL koji koristiš nije identičan javnom (https, isti domen). Koristi `APP_URL`. Naziv iz baze moraš da escape-uješ (`&`, `<`) pre ubacivanja u TwiML, inače poziv ćuti.

### 1.8 Tasteri 1 i 2

**Opis:** Da bismo znali da si stvarno čuo podsetnik, tražimo pritisak tastera.

**Zadaci:**
- U TwiML dodaj `<Gather numDigits="1" timeout="6" action="/api/twilio/gather?alarmId=...">` oko `<Say>`: "Pritisni 1 za gotovo, ili 2 da te podsetim za 10 minuta."
- Napravi `app/api/twilio/gather/route.ts`: taster 1 postavlja podsetnik na `done` i alarm na `confirmed`. Taster 2 postavlja alarm na `confirmed` i pravi novi alarm sa `fire_at = now() + 10 min`. Drugi tasteri ponavljaju pitanje jednom.
- Upisuj `pressed_key` u `call_log`.
- Ako nema pritiska, na kraj TwiML-a dodaj kratko "Nisam čuo odgovor", pa `<Hangup/>`.

**Rezultat:** Pritisak na 1 zatvara podsetnik u aplikaciji, pritisak na 2 pomera poziv za 10 minuta.

**Napomena:** Poziv koji preuzme govorna pošta operatera izgleda kao da si se javio. Zato potvrdu merimo tasterom, a ne time da je poziv podignut.

### 1.9 Propušteni pozivi, ponovni pokušaji i SMS

**Opis:** Ako nisi potvrdio, zovemo ponovo posle 2 minuta (najviše 3 poziva) i šaljemo SMS odmah posle prvog neuspelog poziva.

**Zadaci:**
- Napravi `app/api/twilio/status/route.ts` (POST). Kad poziv završi, ažuriraj `call_log` (status, trajanje).
- Na jednom mestu odluči: ako alarm nije `confirmed` (status `no-answer`, `busy`, `failed`, `canceled` ili je podignut bez tastera): ako je `attempt < 3` napravi novi alarm (`attempt + 1`, `fire_at = now() + 2 min`), inače postavi `missed`.
- Posle neuspelog prvog poziva pošalji SMS "Podsetnik: {naziv} ({HH:mm})" preko `sendSms` (samo ako je `notify_sms` uključen) i upiši ga u `call_log` (`channel = sms`).
- Zaštita od duplikata: ako je alarm već obrađen, ruta samo vrati 200.
- Podsetnik ostaje `open` i u listi se vidi kao "Propušteno".

**Rezultat:** Ne javiš se: SMS stiže odmah, drugi poziv posle 2 minuta, treći posle još 2, a posle trećeg podsetnik je označen kao "Propušteno".

**Napomena:** Provajder ponovo šalje obaveštenje ako ruta ne odgovori brzo, pa bez provere duplikata dobiješ više poziva. Uvek odmah vrati 200. Svaki podignut ponovni poziv je novi trošak.

### 1.10 Istorija poziva

**Opis:** Stranica na kojoj vidiš šta se dogodilo sa svakim podsetnikom. To je tvoj način da proveriš da li sve radi "besprekorno".

**Zadaci:**
- Napravi `app/log/page.tsx` sa poslednjih 50 redova iz `call_log` (vreme, naziv, kanal, status, taster, greška).
- Dodaj `components/bottom-nav.tsx` sa tri stavke: Podsetnici, Istorija, Podešavanja.
- Redove sa statusom `failed` ili `missed` obeleži crvenom bojom.

**Rezultat:** Za svaki podsetnik vidiš tok: poziv, SMS, ponovni pokušaji, taster, greške.

### 1.11 Test na pravom telefonu

**Opis:** Pre nego što se osloniš na aplikaciju, nedelju dana je koristiš uz kontrolnu listu.

**Zadaci:**
- Proveri iPhone podešavanja: Settings, Phone, Silence Unknown Callers (isključi ili sačuvaj broj kao kontakt "PODSETNIK"), i u režimima Fokus i Ne uznemiravaj dozvoli pozive od tog kontakta.
- Testiraj: zaključan ekran, utišan telefon (sa prekidačem za zvuk), slab signal, noć, Wi-Fi poziv.
- U provajderu podesi mesečni limit potrošnje ili upozorenje na potrošnju.
- Napravi 10 podsetnika u različita doba dana i beleži koliko stigne u roku od minut.
- Svakog dana pogledaj `/log` i zapiši propuste.

**Rezultat:** Imaš listu prođenih testova i znaš kada aplikacija zakaže, a limit potrošnje je podešen.

**Napomena:** iPhone može ćutke da odbije poziv nepoznatog broja ili da ga prikaže samo kao propušten (Silence Unknown Callers, Fokus). Ako je telefon utišan prekidačem, poziv samo vibrira.

---

# FAZA 2: Proširenja i poliranje

### 2.1 Ponavljanje

**Opis:** Podsetnik koji se ponavlja (svako jutro u 8h, radnim danima, nedeljno, mesečno). Ako ti treba ranije, ovaj korak može ispred 1.11.

**Zadaci:**
- Popuni `repeat_rule` vrednostima `daily`, `weekdays`, `weekly`, `monthly`.
- Napiši `nextOccurrence(dueAt, rule, tz)` u `lib/recurrence.ts`.
- Kad podsetnik završiš (taster 1 ili checkbox) ili se označi kao `missed`, dobija sledeći `due_at` i novi alarm.
- U `quick-add.tsx` i panelu za izmenu dodaj izbor "Ponovi".

**Rezultat:** Podsetnik "svako jutro u 8h" zove svakog jutra bez ponovnog unosa.

**Napomena:** Sledeće pojavljivanje računaj u lokalnoj zoni, da 8:00 ostane 8:00 i posle prelaska na letnje ili zimsko vreme.

### 2.2 Brz unos tekstom (Claude API)

**Opis:** Napišeš "sutra u 9 zubar", a aplikacija sama prepoznaje naziv i vreme.

**Zadaci:**
- Instaliraj `@anthropic-ai/sdk` i dodaj `ANTHROPIC_API_KEY` samo na serveru.
- Napravi `app/api/parse-reminder/route.ts`: šalje tekst, trenutno vreme i zonu, traži JSON `{ title, due_at, repeat_rule }`. Koristi mali, brz model (npr. `claude-haiku-5-5`).
- U `quick-add.tsx` prikaži prepoznato ("Zubar, sutra u 9:00") i čuvaj tek posle tvoje potvrde.

**Rezultat:** Jednom rečenicom praviš podsetnik, a ako API ne radi, običan unos i dalje radi.

**Napomena:** Model može pogrešiti datum, zato uvek prikaži prepoznato pre čuvanja.

### 2.3 Nadzor budilnika

**Opis:** Ako budilnik stane (pauziran projekat, greška), dobijaš obaveštenje pre nego što propustiš važan podsetnik.

**Zadaci:**
- Napravi `app/api/health/route.ts` koja vraća vreme poslednjeg tick-a.
- Na kraju svakog tick-a pozovi `HEALTHCHECK_URL` (besplatna usluga tipa Healthchecks.io).
- Podesi da servis pošalje email ako signal izostane 5 minuta.

**Rezultat:** Zaustavljen budilnik te obaveštava emailom u roku od nekoliko minuta.

### 2.4 Podešavanja obaveštenja

**Opis:** Biraš kako te aplikacija obaveštava, po podsetniku i globalno.

**Zadaci:**
- Po podsetniku prekidači Poziv i SMS (kolone `notify_call`, `notify_sms` već postoje).
- U `/settings` dodaj: razmak ponavljanja (2 min), broj pokušaja (1 do 5), odlaganje (10 min), dnevni limit poziva.
- Dodaj režim "samo zvoni": poziv se ne podiže, tekst nosi SMS, trošak je minimalan.
- Sačuvaj vrednosti u `profiles` i čitaj ih u tick-u i status ruti.

**Rezultat:** Ponašanje poziva menjaš iz aplikacije, bez izmene koda.

### 2.5 Bolji glas (po potrebi)

**Opis:** Ako glas provajdera ne zvuči dobro na srpskom, koristimo ElevenLabs koji generiše zvučni zapis, a provajder ga samo pusti.

**Zadaci:**
- Napravi `lib/tts.ts` koji preko ElevenLabs API-ja generiše mp3 za tekst podsetnika.
- Čuvaj zapise u javnom Supabase Storage bucket-u `tts`, imenovane po hešu teksta, da ne plaćaš dvaput isti tekst.
- U TwiML-u zameni `<Say>` sa `<Play>`.
- Generiši zapis unapred, pri čuvanju podsetnika.

**Rezultat:** Poziv pušta prirodniji glas.

**Napomena:** Provajder čeka odgovor rute samo kratko (oko 15 sekundi), pa zapis ne smeš da generišeš u trenutku poziva. Ako korak 0.4 pokaže loš glas, uradi ovaj korak pre 1.7.

### 2.6 Poliranje izgleda

**Opis:** Aplikacija izgleda i deluje završeno.

**Zadaci:**
- Tamna tema preko `prefers-color-scheme`.
- Animacije iz sekcije "Dizajn sistem".
- `loading.tsx` i `error.tsx` za glavne stranice, poruke o greškama preko `sonner`.
- Prazna stanja na svim listama.

**Rezultat:** Brza, čista aplikacija bez praznih ili "slomljenih" ekrana.

### 2.7 Priprema za deljenje sa drugima (samo ako odlučiš)

**Opis:** Danas aplikacija ima jednog korisnika. Ako je deliš, najveća prepreka nije kod nego cena poziva.

**Zadaci:**
- Uključi registraciju i dodaj potvrdu broja telefona SMS kodom pre prvog poziva.
- Dodaj limite po korisniku (poziva dnevno).
- Odaberi model naplate (pretplata ili krediti) i pređi na plaćeni nalog provajdera.
- Dodaj stranice sa uslovima korišćenja i politikom privatnosti, i brisanje naloga.
- Pređi na plaćeni Vercel plan.

**Rezultat:** Aplikaciju može bezbedno da koristi više ljudi.

**Napomena:** Pri ceni od oko 0,82 USD po minutu, jedan korisnik sa 5 podignutih poziva dnevno košta više od 100 USD mesečno. Pre deljenja reši cenu (jeftiniji provajder, režim "samo zvoni", plaćanje po pozivu).

---

# Definicija gotovog po fazama

| Faza | Gotovo kad |
|:---|:---|
| Faza 0 | Aplikacija je objavljena na Vercel-u, prijava kodom radi, instalirana je na iPhone-u, baza ima tabele sa RLS-om, a probni poziv i SMS stižu na tvoj telefon uz poznatu cenu. |
| Faza 1 | Uneseš podsetnik, u zakazano vreme (uz odstupanje do oko minut) telefon zvoni i robot ga izgovara, tasteri rade, nepotvrđeno se ponavlja do 3 puta uz SMS, sve se vidi u istoriji, a 10 probnih podsetnika je prošlo. |
| Faza 2 | Ponavljanje, brz unos tekstom, nadzor i podešavanja obaveštenja rade, a izgled je dotjeran. |

# Mapa stranica i ruta

### Stranice

| Putanja | Namena | Korak |
|:---|:---|:---|
| `/login` | Prijava kodom iz emaila | 0.7 |
| `/` | Brz unos i lista podsetnika | 1.2, 1.3 |
| `/log` | Istorija poziva i SMS-ova | 1.10 |
| `/settings` | Broj telefona, zona, odjava, kasnije podešavanja obaveštenja | 1.1, 2.4 |

### API rute

| Ruta | Ko je zove | Namena | Korak |
|:---|:---|:---|:---|
| `POST /api/cron/tick` | Supabase cron | Pronalazi alarme na redu i pokreće pozive | 1.5 |
| `POST /api/twilio/voice` | Provajder poziva | Vraća TwiML (govor + tasteri) | 1.7 |
| `POST /api/twilio/gather` | Provajder poziva | Obrađuje taster 1 ili 2 | 1.8 |
| `POST /api/twilio/status` | Provajder poziva | Ažurira `call_log`, ponovni pokušaji, SMS | 1.9 |
| `GET /api/health` | Ti, nadzor | Vreme poslednjeg tick-a | 2.3 |
| `POST /api/parse-reminder` | Aplikacija | Prepoznavanje teksta preko Claude API-ja | 2.2 |

Čuvanje, izmena, završavanje i brisanje podsetnika idu preko server akcija (`createReminder`, `updateReminder`, `completeReminder`, `deleteReminder`, `saveProfile`), ne preko API ruta.

# Dizajn sistem

Aplikacija je vizuelna i pretežno se koristi na telefonu. Princip: mirno, jasno, velike dodirne površine.

### Boje

| Namena | Svetla tema | Tamna tema |
|:---|:---|:---|
| Pozadina | `#FFFFFF` | `#0B0B10` |
| Površina (kartice) | `#F6F6F9` | `#16161D` |
| Tekst | `#14141A` | `#F2F2F7` |
| Sporedni tekst | `#6B6B78` | `#A0A0AE` |
| Okvir | `#E5E5EC` | `#2A2A35` |
| Primarna (indigo) | `#4F46E5` | `#818CF8` |
| Uspeh (potvrđeno) | `#16A34A` | `#4ADE80` |
| Upozorenje (čeka, zakasnelo) | `#D97706` | `#FBBF24` |
| Greška (propušteno) | `#DC2626` | `#F87171` |

### Tipografija

- Font: Inter (`next/font`).
- Naslov 28/34, podnaslov 20/28 (semibold), tekst 16/24, sitno 13/18.
- Tekst u poljima nikad manji od 16 px (inače iOS zumira stranicu).
- Vreme sa `font-variant-numeric: tabular-nums`.

### Razmaci i oblici

- Skala razmaka: 4, 8, 12, 16, 24 px. Bočne margine 16 px.
- Dodirne površine najmanje 44 x 44 px.
- Zaobljenje: 12 px za kartice i polja, potpuno okruglo za oznake statusa.

### Senke

- Mala: `0 1px 2px rgba(0,0,0,0.06)`.
- Srednja: `0 4px 12px rgba(0,0,0,0.08)` (paneli i izbornici).
- U tamnoj temi bez senki, samo okvir.

### Animacije

- Pritisak na dugme: 150 ms, blago smanjenje na 0,98.
- Dodavanje i uklanjanje reda: 200 ms, nestajanje uz pomeranje od 8 px.
- Checkbox: 120 ms, kratko "iskakanje".
- Nijedna animacija duža od 300 ms, a sve se isključuju za `prefers-reduced-motion`.

### Oznake statusa

Čeka (žuta), Zove (indigo), Potvrđeno (zelena), Propušteno (crvena). Uz boju uvek stoji i tekst, da status ne zavisi samo od boje.

### Raspored

Jedna kolona, maksimalna širina 480 px. Brz unos na vrhu stranice `/`. Donja navigacija sa tri stavke (Podsetnici, Istorija, Podešavanja), uz poštovanje donje bezbedne zone iPhone-a.

# Česte greške i rešenja

| Problem | Rešenje |
|:---|:---|
| Podsetnik stiže sat ili dva ranije ili kasnije | Čuvaj `timestamptz`, konvertuj iz zone `Europe/Belgrade`, ne iz UTC servera. |
| Dupli pozivi | `claim_due_alarms` sa `skip locked`, a webhook rute neka budu idempotentne. |
| Poziv ne stiže na iPhone | Proveri Ne uznemiravaj, Fokus i Silence Unknown Callers, sačuvaj broj kao kontakt, proveri da je Srbija uključena u geo dozvolama provajdera. |
| Zahtev provajdera dobija 403 (nevalidan potpis) | `APP_URL` mora tačno da odgovara javnom https URL-u. |
| Cron ne radi ili vraća 401 | Koristi produkcioni URL (ne preview), proveri `CRON_SECRET` i `cron.job_run_details`, proveri da projekat nije pauziran. |
| Link za prijavu ne radi u PWA | Koristi kod od 6 cifara, ne link (korak 0.7). |
| Tuđi podaci vidljivi | RLS na svim tabelama, a service role ključ samo na serveru. |
| Poziv ćuti ili se prekida | Escape-uj `&` i `<` u nazivu pre ubacivanja u TwiML. |
| Govorna pošta se računa kao "javio se" | Potvrda tasterom (korak 1.8). |
| Podsetnik zove i posle "gotovo" | Otkaži `pending` alarme pri završetku, a tick proverava status pre poziva. |
| Neočekivano veliki račun | `MAX_CALLS_PER_DAY`, najviše 3 pokušaja po podsetniku, limit ili upozorenje na potrošnju kod provajdera. |
| Uvodna najava probnog naloga pre poruke | Nadogradi na plaćeni nalog provajdera. |
| Glas muca ili pogrešno čita | Kratke rečenice, vreme rečima, a ako ne pomaže, ElevenLabs (korak 2.5). |
| Izmena promenljive na Vercel-u nema efekta | Uradi redeploy. |
| Supabase pauzira projekat | Nadzor iz koraka 2.3, po potrebi plaćeni plan. |
| `.env.local` završi na GitHub-u | Proveri `.gitignore`, a ako je procureo, odmah promeni sve ključeve. |

# Napomene o zavisnostima

| Korak | Zavisi od |
|:---|:---|
| 0.1 | nema |
| 0.2 | 0.1 |
| 0.3 | 0.2 |
| 0.4 | 0.2 (skripte i `.env.local`) |
| 0.5 | 0.2 |
| 0.6 | 0.5 |
| 0.7 | 0.5 |
| 0.8 | 0.2, 0.3 (PWA se instalira sa javnog https sajta) |
| 1.1 | 0.6, 0.7, 0.8 |
| 1.2 | 0.6, 1.1 |
| 1.3 | 1.2 |
| 1.4 | 1.3 |
| 1.5 | 0.6, 1.4 |
| 1.6 | 0.3, 1.5 |
| 1.7 | 0.4, 1.1, 1.6 |
| 1.8 | 1.7 |
| 1.9 | 1.8 |
| 1.10 | 1.5 (a pun smisao dobija posle 1.9) |
| 1.11 | 1.9, 1.10 |
| 2.1 | 1.4, 1.9 |
| 2.2 | 1.2 |
| 2.3 | 1.6 |
| 2.4 | 1.1, 1.9 |
| 2.5 | 0.5, 1.2 (menja govor iz 1.7, može i ispred njega) |
| 2.6 | 1.3 |
| 2.7 | 1.11, 2.4 |

Najvažniji lanac: 0.4 (provajder i cena), pa 0.6 (baza), pa 1.5 (tick), pa 1.6 (budilnik), pa 1.7 do 1.9 (poziv, tasteri, ponovni pokušaji). Ako bilo koji od njih zakaže, aplikacija ne ispunjava svoju glavnu svrhu.
