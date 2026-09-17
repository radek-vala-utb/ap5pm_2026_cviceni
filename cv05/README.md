# CV5 – REST API a odpočet do svátku

## Cíl cvičení

V CV4 jste uložili historii počítadel do zařízení pomocí Capacitor Preferences. Nyní CounterApp rozšíříte o další způsob počítání: **počet dnů do vybraného českého svátku**. Termíny načtete ze skutečného REST API, uživatel si vybere svátek a aplikace vypočítá odpočet.

Po dokončení cvičení budete umět:

- odlišit lokální data od údajů získaných ze vzdáleného API,
- nastavit Angular `HttpClient` a oddělit HTTP komunikaci do služby,
- odeslat požadavek `GET` s parametry země, jazyka a období,
- převést jednorázový `Observable` na `Promise` pomocí `firstValueFrom()`,
- ověřit tvar JSON odpovědi za běhu,
- použít `computed()` pro odvozené hodnoty,
- vybrat svátek pomocí `ion-select` a vypočítat počet kalendářních dnů,
- rozlišit načítání, výsledek, prázdnou odpověď a chybu,
- prohlédnout HTTP požadavek v nástrojích pro vývojáře.

Použijeme [OpenHolidays API](https://www.openholidaysapi.org/en/), které nabízí také české svátky. Požadavky nevyžadují účet ani API klíč. Použití je zdarma a data jsou pod licencí ODbL; v aplikaci uvedeme zdroj a odkaz na licenci. [Podmínky a původ dat](https://www.openholidaysapi.org/en/faq/)

Povinná část běží v prohlížeči. Ruční počítadlo a historie z CV4 zůstávají zachované. Odpočet se počítá z data svátku, neukládá se jako ručně měněné číslo.

## 1. Výchozí stav

Navazujete na vlastní projekt `counter-app` dokončený v CV4. Před zahájením musí fungovat:

- `CounterComponent` a ukládání nového počítadla,
- `CounterService` se signals a Capacitor Preferences,
- záložka Historie včetně odstranění záznamů,
- zachování historie po obnovení stránky,
- záložka O aplikaci s vaším jménem,
- příkazy `npm run lint` a `ionic build`.

Přístup k API vyžaduje internet. Historie z CV4 musí dál fungovat i bez připojení.

## 2. Otevření projektu a vytvoření větve

Novou větev vytvořte ze stavu dokončeného CV4. Nejprve zkontrolujte čistý pracovní strom a poslední commity.

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty\counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv5/rest-api
code .
```

### macOS – Terminal

```bash
cd "$HOME/AP5PM-projekty/counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv5/rest-api
code .
```

Bez `nvm` příkaz `nvm use 26` vynechte. Pokud větev již existuje, použijte `git switch cv5/rest-api`.

Pokud na macOS nefunguje `code .`, otevřete projekt přes **File → Open Folder** ve VS Code nebo příkazem:

```bash
open -a "Visual Studio Code" .
```

## 3. Seznámení s OpenHolidays API

V prohlížeči otevřete [české svátky roku 2026](https://openholidaysapi.org/PublicHolidays?countryIsoCode=CZ&languageIsoCode=CS&validFrom=2026-01-01&validTo=2026-12-31).

```text
GET https://openholidaysapi.org/PublicHolidays?countryIsoCode=CZ&languageIsoCode=CS&validFrom=2026-01-01&validTo=2026-12-31
```

| Parametr | Význam |
| --- | --- |
| `countryIsoCode=CZ` | Svátky v České republice |
| `languageIsoCode=CS` | České názvy; jazyk má kód CS, nikoliv CZ |
| `validFrom` | Začátek období ve formátu YYYY-MM-DD |
| `validTo` | Konec období ve stejném formátu |

`GET` načítá data. `/PublicHolidays` určuje zdroj a část za otazníkem obsahuje parametry dotazu. API dovoluje nejvýše tříleté období. My budeme načítat od dneška do konce příštího roku, aby odpočet fungoval i na přelomu roku. [Dokumentace API](https://www.openholidaysapi.org/en/)

Odpověď je pole objektů. Pro aplikaci potřebujeme:

| Vlastnost | Význam |
| --- | --- |
| `id` | Identifikátor svátku |
| `startDate` | Datum začátku |
| `endDate` | Datum konce |
| `name` | Pole názvů, například `[{ "language": "CS", "text": "Štědrý den" }]` |

API vrací také další vlastnosti, které zatím nepoužijeme. Názvy zůstávají polem i při volbě jednoho jazyka. Jeden den může obsahovat více svátků, proto budeme položky rozlišovat podle `id`, ne podle data.

## 4. Nastavení HttpClient

`HttpClient` je součástí již nainstalovaného balíčku `@angular/common`. Nový npm balíček ani Capacitor plugin nepotřebujete.

Do `src/main.ts` přidejte import `provideHttpClient` z `@angular/common/http` a jeho volání do pole `providers`. Ostatní poskytovatele zachovejte. Výsledný soubor projektu z předchozích cvičení:

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import {
  PreloadAllModules,
  RouteReuseStrategy,
  provideRouter,
  withComponentInputBinding,
  withPreloading,
} from '@angular/router';
import { IonicRouteStrategy, provideIonicAngular } from '@ionic/angular';

import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    { provide: RouteReuseStrategy, useClass: IonicRouteStrategy },
    provideIonicAngular(),
    provideHttpClient(),
    provideRouter(routes, withPreloading(PreloadAllModules), withComponentInputBinding()),
  ],
});
```

V Angularu 22 je `HttpClient` dostupný i ve výchozím nastavení. Zde jeho konfiguraci uvádíme explicitně; později sem lze přidat například interceptory. [Nastavení HttpClient](https://angular.dev/guide/http/setup)

Pokud váš projekt používá `app.config.ts`, doplňte volání do jeho existujícího pole `providers`. Nevytvářejte druhé spuštění aplikace pomocí `bootstrapApplication()`.

## 5. Datový model svátku

Ve VS Code vytvořte `src/app/models/public-holiday.ts`:

```typescript
export interface PublicHoliday {
  id: string;
  startDate: string;
  endDate: string;
  name: { language: string; text: string }[];
}
```

Model popisuje vlastnosti odpovědi, které využíváme. `SavedCounter` z CV4 neměňte: uložené ruční počítadlo a svátek jsou odlišná data.

## 6. Práce s kalendářními dny

Vytvořte `src/app/models/holiday-date.ts`:

```typescript
// Kalendářní datum zařízení, nikoliv datum odvozené z UTC času.
export function localDateIso(date = new Date()): string {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
}

export function isCalendarDate(value: unknown): value is string {
  if (typeof value !== 'string' || !/^\d{4}-\d{2}-\d{2}$/.test(value)) {
    return false;
  }

  const date = new Date(`${value}T00:00:00Z`);
  return !Number.isNaN(date.getTime()) && date.toISOString().slice(0, 10) === value;
}

export function daysUntil(target: string, today: string): number {
  // Obě kalendářní data převedeme na UTC půlnoc, aby nevadila změna času.
  const targetTime = Date.parse(`${target}T00:00:00Z`);
  const todayTime = Date.parse(`${today}T00:00:00Z`);
  return Math.round((targetTime - todayTime) / 86_400_000);
}
```

Funkce mají tři úlohy:

- `localDateIso()` vytvoří dnešní datum podle zařízení. Nepoužíváme `new Date().toISOString().slice(0, 10)`, protože kolem půlnoci se datum v UTC může lišit.
- `isCalendarDate()` kontroluje formát i existenci dne; například 30. únor odmítne.
- `daysUntil()` počítá rozdíl kalendářních dnů. Obě data převede na UTC půlnoc, takže rozdíl nezkreslí 23hodinový nebo 25hodinový den při změně času.

`86_400_000` je počet milisekund v 24 hodinách. Funkci předáváme již ověřené datum svátku a datum vytvořené vlastní funkcí.

Očekávané výsledky:

```typescript
daysUntil('2026-09-28', '2026-09-16'); // 12
daysUntil('2026-09-28', '2026-09-28'); // 0
daysUntil('2027-01-01', '2026-12-31'); // 1
daysUntil('2026-03-30', '2026-03-28'); // 2, i přes změnu času
```

## 7. Vygenerování služby

V kořenovém adresáři projektu spusťte:

### Windows – PowerShell

```powershell
npx ng generate service services/holiday-api --type=service
```

### macOS – Terminal

```bash
npx ng generate service services/holiday-api --type=service
```

Parametr `--type=service` zachová příponu `.service.ts`, stejně jako v CV4.

## 8. Implementace HolidayApiService

Nahraďte obsah `src/app/services/holiday-api.service.ts`:

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom, timeout } from 'rxjs';
import { PublicHoliday } from '../models/public-holiday';
import { isCalendarDate } from '../models/holiday-date';

function isHolidayName(value: unknown): value is PublicHoliday['name'][number] {
  return typeof value === 'object' && value !== null &&
    'language' in value && typeof value.language === 'string' &&
    'text' in value && typeof value.text === 'string';
}

function isPublicHoliday(value: unknown): value is PublicHoliday {
  return typeof value === 'object' && value !== null &&
    'id' in value && typeof value.id === 'string' &&
    'startDate' in value && isCalendarDate(value.startDate) &&
    'endDate' in value && isCalendarDate(value.endDate) &&
    'name' in value && Array.isArray(value.name) &&
    value.name.length > 0 && value.name.every(isHolidayName);
}

@Injectable({
  providedIn: 'root',
})
export class HolidayApiService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = 'https://openholidaysapi.org/PublicHolidays';

  async getHolidays(validFrom: string, validTo: string): Promise<PublicHoliday[]> {
    const response = await firstValueFrom(
      this.http.get<unknown>(this.apiUrl, {
        params: {
          countryIsoCode: 'CZ',
          languageIsoCode: 'CS',
          validFrom,
          validTo,
        },
      }).pipe(timeout(10000)),
    );

    if (!Array.isArray(response) || !response.every(isPublicHoliday)) {
      throw new Error('API vrátilo svátky v neočekávaném formátu.');
    }

    return response.sort((a, b) => a.startDate.localeCompare(b.startDate));
  }
}
```

| Část | Význam |
| --- | --- |
| `inject(HttpClient)` | Získá HTTP klienta přes dependency injection. |
| `params` | Přidá zemi, jazyk a období do URL. |
| `get<unknown>()` | Přijme JSON, jehož tvar ještě není ověřený. |
| `firstValueFrom()` | Spustí odběr a převede první odpověď na Promise. |
| `timeout(10000)` | Ukončí čekání chybou po 10 sekundách bez odpovědi. |
| `isPublicHoliday()` | Ověří potřebné vlastnosti včetně vnořeného pole názvů. |
| `sort()` | Seřadí svátky od nejbližšího data. |

`HttpClient.get()` vrací `Observable`; požadavek odešle až odběr, který zde zajišťuje `firstValueFrom()`. Díky převodu můžeme použít `async/await` známé z CV4. Samotný zápis `get<PublicHoliday[]>()` by skutečný JSON nezkontroloval. [HTTP požadavky v Angularu](https://angular.dev/guide/http/making-requests)

Zápis `value is PublicHoliday` je typový predikát: po úspěšné kontrole TypeScript ví, jaké vlastnosti lze použít. Prázdné pole `[]` je platný výsledek. Chybu na prázdné pole nepřevádíme, aby stránka rozeznala „žádná data“ od „načtení selhalo“.

## 9. Logika třetí záložky

Nahraďte obsah `src/app/tab3/tab3.page.ts`:

```typescript
import { Component, computed, inject, signal } from '@angular/core';
import { DatePipe } from '@angular/common';
import { HttpErrorResponse } from '@angular/common/http';
import { TimeoutError } from 'rxjs';
import {
  IonButton,
  IonCard,
  IonCardContent,
  IonCardHeader,
  IonCardTitle,
  IonContent,
  IonHeader,
  IonSelect,
  IonSelectOption,
  IonSpinner,
  IonTitle,
  IonToolbar,
} from '@ionic/angular';
import { PublicHoliday } from '../models/public-holiday';
import { daysUntil, localDateIso } from '../models/holiday-date';
import { HolidayApiService } from '../services/holiday-api.service';

@Component({
  selector: 'app-tab3',
  templateUrl: 'tab3.page.html',
  styleUrls: ['tab3.page.scss'],
  imports: [
    DatePipe,
    IonButton,
    IonCard,
    IonCardContent,
    IonCardHeader,
    IonCardTitle,
    IonContent,
    IonHeader,
    IonSelect,
    IonSelectOption,
    IonSpinner,
    IonTitle,
    IonToolbar,
  ],
})
export class Tab3Page {
  private readonly holidayApi = inject(HolidayApiService);

  readonly holidays = signal<PublicHoliday[]>([]);
  readonly selectedId = signal('');
  readonly today = signal(localDateIso());
  readonly loading = signal(false);
  readonly errorMessage = signal('');

  readonly selectedHoliday = computed(() =>
    this.holidays().find((holiday) => holiday.id === this.selectedId()),
  );

  readonly remainingDays = computed(() => {
    const holiday = this.selectedHoliday();
    return holiday ? daysUntil(holiday.startDate, this.today()) : null;
  });

  async ionViewWillEnter(): Promise<void> {
    await this.loadHolidays();
  }

  holidayName(holiday: PublicHoliday): string {
    return holiday.name.find((name) => name.language === 'CS')?.text ??
      holiday.name[0]?.text ?? 'Svátek bez názvu';
  }

  async loadHolidays(): Promise<void> {
    if (this.loading()) {
      return;
    }

    const previousId = this.selectedId();
    const today = localDateIso();
    const nextYear = Number(today.slice(0, 4)) + 1;
    this.today.set(today);
    this.loading.set(true);
    this.errorMessage.set('');
    this.holidays.set([]);
    this.selectedId.set('');

    try {
      const holidays = await this.holidayApi.getHolidays(today, `${nextYear}-12-31`);
      const upcoming = holidays.filter((holiday) => holiday.startDate >= today);
      this.holidays.set(upcoming);
      this.selectedId.set(
        upcoming.find((holiday) => holiday.id === previousId)?.id ?? upcoming[0]?.id ?? '',
      );
    } catch (error) {
      console.error('Svátky se nepodařilo načíst.', error);

      if (error instanceof TimeoutError) {
        this.errorMessage.set('Server neodpověděl včas. Zkuste načtení znovu.');
      } else if (error instanceof HttpErrorResponse && error.status === 0) {
        this.errorMessage.set('API není dostupné. Zkontrolujte připojení a zkuste to znovu.');
      } else {
        this.errorMessage.set('Svátky se nepodařilo načíst. Zkuste to znovu později.');
      }
    } finally {
      this.loading.set(false);
    }
  }
}
```

`selectedId` ukládá volbu uživatele. `selectedHoliday` a `remainingDays` jsou odvozené signály vytvořené pomocí `computed()`: přepočítají se při změně vstupních signálů. Výběr jiného svátku proto nevyvolává nový HTTP požadavek.

Data načítáme v Ionic metodě `ionViewWillEnter()`, která se u stránky v router outletu volá při vstupu na stránku, i když ji Ionic ponechal v paměti. Při návratu se tak obnoví také dnešní datum. [Životní cyklus Ionic stránky](https://ionicframework.com/docs/angular/lifecycle)

Po načtení vybíráme nejbližší svátek. Při dalším načtení zachováme dosavadní výběr, pokud je stále v nabídce. Dnešní svátek má hodnotu `0`; minulé svátky odfiltrujeme. Data ve formátu YYYY-MM-DD lze chronologicky porovnávat jako řetězce.

Odpočet se aktualizuje při vstupu na záložku a po stisku tlačítka. Pokud ji necháte otevřenou přes půlnoc, použijte **Načíst znovu**. Automatická aktualizace přes půlnoc je bonusový úkol.

`finally` vypne spinner i po chybě. Kontrola `loading()` brání souběžným požadavkům a `holidayName()` najde český název nebo použije dostupnou náhradní hodnotu.

## 10. Šablona třetí záložky

Nahraďte obsah `src/app/tab3/tab3.page.html`. Doplňte vlastní jméno:

```html
<ion-header [translucent]="true">
  <ion-toolbar>
    <ion-title>Odpočet do svátku</ion-title>
  </ion-toolbar>
</ion-header>

<ion-content [fullscreen]="true" class="ion-padding">
  <div class="page-content">
    <section aria-labelledby="holidays-heading" [attr.aria-busy]="loading()">
      <h2 id="holidays-heading">Kolik dnů zbývá do…</h2>
      <p>Vyberte český svátek a zobrazte počet zbývajících kalendářních dnů.</p>

      <ion-button [disabled]="loading()" (click)="loadHolidays()">
        {{ loading() ? 'Načítám…' : 'Načíst znovu' }}
      </ion-button>

      @if (loading()) {
        <div class="state-message" role="status">
          <ion-spinner aria-hidden="true"></ion-spinner>
          <p>Načítám svátky…</p>
        </div>
      } @else if (errorMessage()) {
        <p class="state-message error-message" role="alert">{{ errorMessage() }}</p>
      } @else if (holidays().length === 0) {
        <p class="state-message">Pro vybrané období nejsou žádné nadcházející svátky.</p>
      } @else {
        <ion-select
          label="Vyberte svátek"
          label-placement="stacked"
          interface="alert"
          cancel-text="Zrušit"
          ok-text="Vybrat"
          [value]="selectedId()"
          (ionChange)="selectedId.set($event.detail.value)"
        >
          @for (holiday of holidays(); track holiday.id) {
            <ion-select-option [value]="holiday.id">
              {{ holiday.startDate + 'T00:00:00Z' | date: 'd. M. y' : 'UTC' }} – {{ holidayName(holiday) }}
            </ion-select-option>
          }
        </ion-select>

        @if (selectedHoliday(); as holiday) {
          <ion-card>
            <ion-card-header>
              <ion-card-title>{{ holidayName(holiday) }}</ion-card-title>
            </ion-card-header>
            <ion-card-content>
              <p>{{ holiday.startDate + 'T00:00:00Z' | date: 'd. M. y' : 'UTC' }}</p>
              <div aria-live="polite">
                <p class="countdown-value">{{ remainingDays() }}</p>
                @if (remainingDays() === 0) {
                  <p>Svátek je dnes!</p>
                } @else {
                  <p>Počet dnů do svátku</p>
                }
              </div>
              <p>Počítáno k {{ today() + 'T00:00:00Z' | date: 'd. M. y' : 'UTC' }}.</p>
            </ion-card-content>
          </ion-card>
        }
      }
    </section>

    <section>
      <h2>O aplikaci</h2>
      <p>Autor: Jméno a příjmení</p>
      <p>AP5PM 2026 – CounterApp</p>
      <p>
        Zdroj dat:
        <a href="https://www.openholidaysapi.org/en/">OpenHolidays API</a>
        (<a href="https://www.openholidaysapi.org/en/faq/">licence ODbL</a>).
      </p>
    </section>
  </div>
</ion-content>
```

`ion-select` předává vybrané ID přes událost `ionChange`. Hodnotu zapíšeme do signálu a `computed()` přepočítá odpočet. `FormsModule` zde není potřeba, protože nepoužíváme `ngModel`.

`DatePipe` je uveden v importech stránky. Pro výpis používáme časovou zónu `UTC`, aby kalendářní datum z API nebylo posunuto na předchozí den.

V `src/app/tabs/tabs.page.html` změňte text třetího `ion-label` z `O aplikaci` na `Odpočet`. Hodnoty `tab="tab3"`, `href="/tabs/tab3"` a ikonu zachovejte. Informace o autorovi zůstávají na třetí stránce pod odpočtem.

## 11. Styly stránky

Nahraďte obsah `src/app/tab3/tab3.page.scss`:

```scss
.page-content {
  max-width: 40rem;
  margin: 1rem auto;
}

section + section {
  margin-top: 2rem;
}

ion-select {
  margin-top: 1rem;
}

ion-card {
  margin: 1rem 0;
  text-align: center;
}

ion-card-title {
  font-size: 1.25rem;
  overflow-wrap: anywhere;
}

.countdown-value {
  margin: 1.5rem 0;
  font-size: 4rem;
  font-weight: 700;
  line-height: 1;
}

.state-message {
  margin: 2rem 0;
  color: var(--ion-color-medium);
  text-align: center;
}

.error-message {
  color: var(--ion-color-danger);
}
```

Velké číslo navazuje na vzhled ručního počítadla. Zde je však výsledkem výpočtu z data a uživatel ho nemění tlačítky +1 a −1.

## 12. Tok dat

```text
Tab3Page → HolidayApiService → GET /PublicHolidays → OpenHolidays
    ↑              │
    └── ověřené a seřazené svátky
    │
    └── výběr ID → selectedHoliday → remainingDays → odpočet
```

Služba zná API a kontroluje formát odpovědi. Stránka spravuje výběr a zobrazení. `CounterService` dál spravuje lokální historii z CV4. API poskytuje termín; počet dnů dopočítává aplikace.

## 13. Spuštění a ověření

V kořenovém adresáři projektu spusťte:

```bash
ionic serve
```

Na adrese `http://localhost:8100` ověřte:

1. Otevřete třetí záložku **Odpočet**.
2. Po načtení je vybraný nejbližší svátek a zobrazuje se český název, datum a počet dnů.
3. Vyberte jiný svátek a ověřte změnu čísla.
4. V Developer Tools → **Network → Fetch/XHR** klikněte na **Načíst znovu**.
5. Najděte `PublicHolidays` a ověřte metodu `GET`, stav `200`, parametry `CZ`, `CS` a rozsah dat.
6. V části **Response** porovnejte `startDate` a `name` s údaji v aplikaci.
7. Při pouhé změně vybraného svátku nevzniká nový požadavek.
8. Pod odpočtem je vaše jméno a uvedený zdroj dat.
9. Ověřte úzký mobilní displej, přepínání záložek a zachování historie z CV4 po obnovení stránky.

Pro zřetelný spinner dočasně zapněte pomalejší připojení v panelu Network. Potom vraťte **No throttling**. Datum dneška vypisujeme na kartě, takže lze výsledek zkontrolovat proti kalendáři.

## 14. Chyba sítě a opakování požadavku

1. Nechte aplikaci otevřenou a počkejte na dokončení načítání.
2. V Developer Tools → Network zapněte **Offline**.
3. Bez obnovení celé stránky klikněte na **Načíst znovu**.
4. Ověřte chybovou zprávu, ukončení spinneru a opětovné zpřístupnění tlačítka.
5. Nesmí se zobrazit zpráva o prázdném období.
6. Vraťte **No throttling** a načtení zopakujte.
7. Ověřte, že chyba zmizela a výběr svátků funguje.

Červený výpis v konzoli je při úmyslně vyvolané chybě očekávaný. V režimu Offline neobnovujte celou stránku: prohlížeč by nemusel načíst ani aplikaci z vývojového serveru.

## 15. Prázdná odpověď a okrajové případy

V `loadHolidays()` dočasně nahraďte pouze řádek s HTTP voláním:

```typescript
const holidays = await this.holidayApi.getHolidays('2026-02-02', '2026-02-02');
```

Pro tento den API vrací `[]`. V Network ověřte úspěšnou odpověď a v aplikaci prázdný stav bez chyby. Potom vraťte původní volání s `today` a koncem příštího roku. Případné dočasné varování editoru o nepoužitém `nextYear` tím zmizí.

Pro ověření validace dočasně změňte `apiUrl` ve službě na `https://openholidaysapi.org/Countries`. Vrácené objekty nejsou svátky: validace je musí odmítnout a stránka zobrazit chybu. Vraťte `/PublicHolidays`.

Zkontrolujte také výsledky funkce `daysUntil()` z kroku 6: stejný den = 0, přelom roku a změna letního času. V nabídce nezaměňujte dvě položky se stejným datem; `track holiday.id` je rozliší.

## 16. Produkční sestavení

Ukončete server pomocí `Ctrl+C` a spusťte:

```bash
npm run lint
ionic build
```

Oba příkazy musí skončit bez chyby. Build kontroluje také HTML vazby a typy. Dostupnost API ověřujete samostatně v prohlížeči.

Pokud máte v CounterApp nastavený Prettier a skripty z CV1, před kontrolami spusťte také `npm run format` a `npm run format:check`.

## 17. Uložení práce do Gitu

Zkontrolujte změny a vraťte dočasnou cestu `/Countries` i pevné testovací období:

```bash
git status
git diff
git add src/main.ts src/app
git commit -m "Pridej odpocet do svatku pres OpenHolidays API"
git status
git log --oneline -3
```

Příkazy jsou stejné na Windows i macOS. Pokud používáte `app.config.ts`, příkaz `git add src/app` zahrne i jeho změnu. Pracovní strom má být čistý a poslední commit má obsahovat CV5.

## 18. Kontrolní seznam CV5

- [ ] Pracuji ve větvi `cv5/rest-api` navazující na CV4.
- [ ] Mám model `PublicHoliday` a samostatnou `HolidayApiService`.
- [ ] Konfigurace aplikace obsahuje `provideHttpClient()`.
- [ ] Umím vysvětlit metodu GET a parametry země, jazyka a období.
- [ ] Používám `firstValueFrom()`, timeout a validaci odpovědi.
- [ ] Vím, proč je `name` pole a proč jazyk používá kód CS.
- [ ] Třetí záložka nabízí české svátky od dneška do konce příštího roku.
- [ ] Výběr jiného svátku přepočítá odpočet bez dalšího požadavku.
- [ ] Odpočet používá `computed()` a počítá kalendářní dny.
- [ ] Dnešní svátek zobrazí nulu; minulý se nenabízí.
- [ ] Svátky se stejným datem rozlišuji pomocí ID.
- [ ] Aplikace rozlišuje načítání, chybu, prázdný výsledek a data.
- [ ] Po obnovení připojení lze načtení zopakovat.
- [ ] Ověřil/a jsem požadavek v Network a vrátil/a testovací změny.
- [ ] Počítadlo a historie z CV4 stále fungují.
- [ ] Na stránce je mé jméno a zdroj dat OpenHolidays s odkazem na licenci.
- [ ] `npm run lint` a `ionic build` proběhnou bez chyby.
- [ ] Řešení je uložené v Git commitu.

## 19. Bonusové úkoly

Po dokončení povinné části můžete:

1. uložit ID a cílové datum vybraného svátku přes Preferences; po spuštění znovu vypočítat zbývající dny,
2. přidat výběr země ze zdroje `/Countries`,
3. zobrazit i školní prázdniny ze zdroje `/SchoolHolidays` a prozkoumat jejich regionální platnost,
4. uchovat poslední načtená data pro režim offline a jasně označit, že jde o uloženou kopii,
5. přepočítat odpočet automaticky po půlnoci bez zbytečného HTTP požadavku,
6. přidat vlastní uživatelské datum, například termín zkoušky.

Ukládejte cílové datum, ne pouze dnešní počet dnů. Číslo by zítra již nebylo aktuální. Při přidání zemí může být potřeba řešit regionální a vícedenní svátky; naše povinná část se soustředí na české veřejné svátky.

## 20. Nejčastější problémy

### `No provider for HttpClient`

Zkontrolujte `provideHttpClient()` v poskytovatelích skutečně spouštěné aplikace. Nepatří do `imports` stránky. Automatizované testy mají vlastní konfiguraci poskytovatelů.

### API vrací jiné názvy nebo název není obyčejný řetězec

Země je `CZ`, jazyk je `CS`. `name` je pole objektů s vlastnostmi `language` a `text`; použijte `holidayName(holiday)`.

### V Network nevidím požadavek

Otevřete třetí záložku, klikněte na **Načíst znovu** a zkontrolujte konzoli. Samotné vytvoření `http.get(...)` nestačí: odběr spouští `firstValueFrom()`.

### Chyba se stavem 0

Vypněte režim Offline a zkontrolujte URL. Stav 0 může znamenat problém s připojením, zablokovaný požadavek nebo CORS. Nepřidávejte `Access-Control-Allow-Origin` do požadavku klienta; tuto hlavičku nastavuje server. Nevypínejte zabezpečení prohlížeče.

### `ion-select` není známý element nebo chybí `date` pipe

Zkontrolujte TypeScript importy i pole `imports` stránky: musí obsahovat `IonSelect`, `IonSelectOption` a `DatePipe`, stejně jako ostatní použité komponenty.

### Odpočet je posunutý o den

Nepočítejte rozdíl mezi aktuálním časem a půlnocí svátku. Použijte obě kalendářní data podle kroku 6. Datum dneška musí vycházet z místního data zařízení. Po půlnoci obnovte odpočet tlačítkem.

### Spinner nezmizí nebo tlačítko zůstalo zakázané

Zkontrolujte blok `finally`, volání `this.loading.set(false)` a timeout ve službě. Signály v šabloně čtěte se závorkami.

### Po obnovení stránky je vybraný opět nejbližší svátek

V povinné části žije výběr jen v paměti stránky. Trvalé uložení volby je bonus. Lokální historie počítadel z CV4 se tím nemění.

## Oficiální dokumentace

- [OpenHolidays – úvod a parametry](https://www.openholidaysapi.org/en/)
- [OpenHolidays – interaktivní dokumentace](https://openholidaysapi.org/swagger/index.html)
- [OpenHolidays – původ dat a licence](https://www.openholidaysapi.org/en/faq/)
- [Angular – nastavení HttpClient](https://angular.dev/guide/http/setup)
- [Angular – HTTP požadavky](https://angular.dev/guide/http/making-requests)
- [Angular – signals a computed](https://angular.dev/guide/signals)
- [RxJS – firstValueFrom](https://rxjs.dev/api/index/function/firstValueFrom)
- [Ionic – ion-select](https://ionicframework.com/docs/api/select)
- [Ionic – životní cyklus stránky](https://ionicframework.com/docs/angular/lifecycle)
