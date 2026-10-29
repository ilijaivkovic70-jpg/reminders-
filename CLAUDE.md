# Podsetnik sa pozivom: uputstvo za Claude Code

## Šta gradimo

Lična aplikacija za podsetnike po uzoru na iOS Reminders, ali jednostavnija i brža. Glavno obaveštenje je **telefonski poziv sa robotskim glasom** ("Podsetnik: sastanak u 16h"), uz SMS kao rezervu. Vlasnik je jedini korisnik (možda kasnije deli sa drugima). Mora da radi pouzdano, jer vlasnik često zaboravlja, a obična poruka mu nije dovoljna.

Pun plan implementacije je u `docs/plan-implementacije.md`. To je glavni izvor istine za korake, rute, bazu i dizajn.

## Kako radiš sa mnom

Ja sam početnik. Vodi me korak po korak.

- Piši na srpskom (latinica). Nikad ne koristi duple crtice između reči. Kratke rečenice, bez žargona. Tehnički pojam objasni jednom rečenicom kad se prvi put pojavi.
- Radi **jedan korak plana po sesiji**. Na početku reci koji korak radimo (npr. 1.2) i šta će postojati kad se završi (polje "Rezultat" iz plana). Na kraju proveri Rezultat i reci mi kako da ga sam isprobam.
- Ne preskači korake i ne radi unapred ono što zavisi od neodrađenih koraka (vidi "Napomene o zavisnostima" u planu).
- Pre izvršavanja pročitaj odgovarajući korak u `docs/plan-implementacije.md`, uključujući "Napomena" (zamke).
- Komande koje ja treba da pokrenem napiši tačno, jednu po jednu, i reci šta treba da vidim.
- Pitaj me za potvrdu pre: instaliranja paketa, brisanja fajlova, izmene baze i svega što troši novac.
- Budi iskren i kritičan. Ako postoji bolji pristup ili plan ne odgovara stvarnosti, reci to i predloži izmenu plana. Ne improvizuj tiho.
- Kad završimo korak, ažuriraj listu "Napredak" ispod, ali tek posle moje potvrde.

## Stack

- Next.js (App Router, TypeScript), Tailwind CSS, shadcn/ui
- Supabase: Postgres baza, Auth (kod iz emaila), `pg_cron` + `pg_net` za minutni budilnik
- PWA (aplikacija sa početnog ekrana iPhone-a, bez App Store-a)
- Vercel (hosting)
- Provajder poziva i SMS-a: Twilio (izbor se potvrđuje u koraku 0.4, zato sav kod za pozive ide kroz `lib/telephony.ts`)
- Kasnije: Claude API (brz unos tekstom), ElevenLabs (glas, po potrebi), Healthchecks.io (nadzor)

## Struktura projekta

```
app/
  login/page.tsx
  page.tsx                  brz unos + lista
  log/page.tsx              istorija poziva
  settings/page.tsx
  actions/                  server akcije (reminders.ts, profile.ts)
  api/
    cron/tick/route.ts
    twilio/voice|gather|status/route.ts
    health/route.ts
    parse-reminder/route.ts
components/                 quick-add, reminder-list, bottom-nav, ui/ (shadcn)
lib/
  supabase/client.ts, server.ts, admin.ts
  telephony.ts              jedino mesto koje zna za provajdera poziva
  time.ts, phone.ts, recurrence.ts, tts.ts
db/                         SQL fajlovi (001_init.sql, 002_claim.sql)
scripts/                    test-call.mjs, test-sms.mjs
docs/plan-implementacije.md
```

## Ključna pravila arhitekture (ne krši ih)

1. **Podsetnik i alarm su odvojeni.** `reminders` je šta, `alarms` je kada i koji pokušaj. Stanja alarma: `pending`, `processing`, `calling`, `confirmed`, `missed`, `failed`, `cancelled`.
2. **Vreme:** uvek `timestamptz` u bazi (UTC). Pretvaranje iz zone `Europe/Belgrade` ide preko `lib/time.ts`. "Danas" i grupisanje po danima računaj u korisnikovoj zoni, nikad u UTC servera.
3. **Telefon:** čuvaj samo u E.164 formatu (`+3816...`). Normalizacija je u `lib/phone.ts` (početna nula se izbacuje).
4. **Atomsko preuzimanje alarma:** tick koristi SQL funkciju `claim_due_alarms` sa `for update skip locked`. Nikad ne čitaj pa ažuriraj alarme u dva odvojena koraka (dupli pozivi).
5. **Potvrda je taster, ne podignut poziv.** Taster 1 = gotovo, taster 2 = odloži 10 minuta. Govorna pošta se računa kao nepotvrđen poziv.
6. **Odluka o ponovnom pokušaju je na jednom mestu:** `app/api/twilio/status/route.ts`. Pravilo: ako nije `confirmed` i `attempt < 3`, novi alarm za 2 minuta; inače `missed`. SMS se šalje odmah posle prvog neuspelog poziva.
7. **Webhook rute (`/api/twilio/*`):** uvek proveri potpis (`X-Twilio-Signature`, URL iz `APP_URL`), budi idempotentan, vrati 200 brzo. Tekst iz baze koji ulazi u TwiML mora da se escape-uje (`&`, `<`).
8. **Meko brisanje:** `deleted_at`, nikad pravi `delete`. Pri završetku, izmeni vremena ili brisanju podsetnika otkaži njegove `pending` alarme, a tick pre poziva proverava status podsetnika.
9. **Sve se beleži u `call_log`** (kanal, status, taster, razlog, greška).
10. **Prijava je kodom od 6 cifara iz emaila**, ne magic linkom (link ne radi u PWA na iPhone-u). `shouldCreateUser: false`, javna registracija je isključena.
11. **RLS je uključen na svim tabelama** (`user_id = auth.uid()`). Service role klijent (`lib/supabase/admin.ts`) koristi se samo u serverskim rutama (tick, webhook-ovi).
12. **Promene podataka iz UI-ja idu preko server akcija** (`app/actions`), ne preko API ruta.

## Bezbednost i trošak (stroga pravila)

- **Pozivi su skupi:** poziv ka srpskom mobilnom košta oko 0,82 USD po minutu (Twilio, proveriti u koraku 0.4). Zato:
  - Nikad ne pokreći pravi poziv ili SMS u testu bez moje izričite potvrde.
  - Dok traje korak 1.5, koristi probni režim (`reason = dry_run`) bez provajdera.
  - Test pozive šalji samo na moj verifikovani broj.
  - `MAX_CALLS_PER_DAY` mora da postoji i da se proverava u tick-u. Ponovni pokušaji uvek imaju gornju granicu (3). Nikad petlja bez granice.
- `.env.local` nikad ne ide u git. Ne ispisuj vrednosti ključeva u razgovoru ni u logovima.
- `SUPABASE_SERVICE_ROLE_KEY` nikad ne dobija prefiks `NEXT_PUBLIC_` i nikad ne ide u kod koji se izvršava u pregledaču.
- `ANTHROPIC_API_KEY`, `TWILIO_AUTH_TOKEN`, `CRON_SECRET` i ElevenLabs ključ su samo serverski.
- Spoljni servisi (Supabase cron, Twilio) zovu samo **produkcioni Vercel URL**, nikad preview adrese.
- Pre svake izmene šeme baze pokaži mi SQL i sačekaj potvrdu. SQL fajlovi se čuvaju u `db/`.

## Promenljive okruženja (samo nazivi)

`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `CRON_SECRET`, `APP_URL`, `MAX_CALLS_PER_DAY`, `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`, kasnije `ANTHROPIC_API_KEY`, `HEALTHCHECK_URL`, ElevenLabs ključ. Nove promenljive dodaj i lokalno (`.env.local`) i na Vercel-u (posle promene na Vercel-u treba redeploy).

## Komande

- `npm run dev`: pokretanje lokalno na `http://localhost:3000`
- `npm run build`: provera da li projekat može da se objavi
- `npm run lint`: provera koda
- `git push`: objavljuje na Vercel (automatski deploy)

## Stil koda

- TypeScript strogo, bez `any`. Mali, fokusirani fajlovi.
- Imena fajlova, promenljivih i funkcija na engleskom. Komentari i tekstovi u aplikaciji na srpskom (latinica).
- UI: shadcn/ui komponente i dizajn tokeni iz `app/globals.css` (boje, polumeri, senke, animacije iz sekcije "Dizajn sistem" u planu). Mobile first, jedna kolona do 480 px, dodirne površine najmanje 44 px, tekst u poljima najmanje 16 px, poštuj `prefers-reduced-motion` i donju bezbednu zonu iPhone-a.
- Greške prikazuj korisniku jasnom porukom (sonner), ne praznim ekranom.

## Poznate zamke po koracima

- **0.2:** ako `create-next-app` odbije da radi u folderu zbog fajla `CLAUDE.md`, privremeno premesti `CLAUDE.md` van foldera, pokreni komandu, pa ga vrati.
- **0.7:** `middleware.ts` se u novijim verzijama Next.js zove `proxy.ts`. Prati dokumentaciju instalirane verzije.
- **1.6:** budilnik radi na nivou minuta, pa poziv može stići do minut kasnije od zadatog vremena. Besplatni Supabase projekat se pauzira posle nedelju dana neaktivnosti, a to zaustavlja budilnik.
- **1.7:** ako potpis webhook-a ne prolazi, prvo proveri da `APP_URL` tačno odgovara javnom https URL-u.
- **0.4 / 2.5:** srpski glas u Twilio-u nije potvrđen. Ako ne zvuči dobro, korak 2.5 (ElevenLabs) ide pre 1.7.

## Napredak

Označi korak tek kad ja potvrdim da radi.

**Faza 0: Postavka**
- [x] 0.1 Alati i nalozi
- [ ] 0.2 Skelet projekta
- [ ] 0.3 Prvi deploy na Vercel
- [ ] 0.4 Izbor provajdera poziva i probni poziv
- [ ] 0.5 Supabase projekat i veza sa aplikacijom
- [ ] 0.6 Tabele u bazi i zaštita (RLS)
- [ ] 0.7 Prijava kodom iz emaila
- [ ] 0.8 Dizajn osnova i PWA

**Faza 1: MVP**
- [ ] 1.1 Podešavanja: telefon i vremenska zona
- [ ] 1.2 Brz unos podsetnika
- [ ] 1.3 Lista podsetnika
- [ ] 1.4 Gotovo, izmena i brisanje
- [ ] 1.5 Tick ruta (probni režim bez poziva)
- [ ] 1.6 Minutni budilnik u Supabase-u
- [ ] 1.7 Prvi pravi poziv
- [ ] 1.8 Tasteri 1 i 2
- [ ] 1.9 Propušteni pozivi, ponovni pokušaji i SMS
- [ ] 1.10 Istorija poziva
- [ ] 1.11 Test na pravom telefonu

**Faza 2: Proširenja i poliranje**
- [ ] 2.1 Ponavljanje
- [ ] 2.2 Brz unos tekstom (Claude API)
- [ ] 2.3 Nadzor budilnika
- [ ] 2.4 Podešavanja obaveštenja
- [ ] 2.5 Bolji glas (po potrebi)
- [ ] 2.6 Poliranje izgleda
- [ ] 2.7 Priprema za deljenje sa drugima (samo ako odlučim)

**Trenutni korak:** 0.2