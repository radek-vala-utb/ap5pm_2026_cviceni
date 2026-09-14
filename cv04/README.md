# CV4 – Sdílená služba a lokální perzistence

## Cíl cvičení

V CV3 se uložená počítadla nacházela v poli komponenty `Tab1Page`. Po obnovení stránky se proto ztratila a záložka Historie byla stále prázdná. V tomto cvičení přesunete stav do služby `CounterService`, zpřístupníte jej více stránkám a uložíte jej pomocí pluginu Capacitor Preferences.

Po dokončení cvičení budete umět:

- vytvořit a používat Angular službu,
- získat sdílenou instanci služby pomocí dependency injection,
- vystavit stav služby jako signal pouze pro čtení,
- pracovat s asynchronním API a typem `Promise`,
- převést pole objektů do JSON a zpět,
- uložit lehká data pomocí Capacitor Preferences,
- zobrazit sdílená data na jiné záložce,
- odstranit jeden záznam nebo celou historii,
- rozlišit stav „načítám“, prázdnou historii a seznam dat.

Preferences jsou vhodné pro malé množství jednoduchých dat. Nejde o databázi; rozsáhlá data, časté zápisy a složité dotazy by patřily například do SQLite.

## 1. Výchozí stav

Navazujete na vlastní projekt `counter-app` dokončený v CV3. Před zahájením musí fungovat:

- samostatná komponenta `CounterComponent`,
- zadání názvu a změna hodnoty počítadla,
- uložení typovaného objektu `SavedCounter`,
- vykreslení více uložených počítadel pomocí `@for`,
- prázdný stav pomocí `@if` a `@else`,
- příkaz `ionic build`.

V CV3 se seznam zobrazuje pod počítadlem na první záložce a po obnovení stránky zmizí. To je správný výchozí stav.

## 2. Otevření projektu a vytvoření větve

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty\counter-app"
nvm use 26
git status
git switch -c cv4/preferences
code .
```

### macOS – Terminal

```bash
cd "$HOME/AP5PM-projekty/counter-app"
nvm use 26
git status
git switch -c cv4/preferences
code .
```

Pokud Node.js nepoužíváte přes `nvm`, příkaz `nvm use 26` vynechte. Pokud větev již existuje, použijte místo vytvoření:

```bash
git switch cv4/preferences
```

Před zahájením má `git status` hlásit čistý pracovní strom.

## 3. Instalace Capacitor Preferences

V kořenovém adresáři projektu spusťte:

### Windows – PowerShell

```powershell
npm install @capacitor/preferences@8
npx cap sync
```

### macOS – Terminal

```bash
npm install @capacitor/preferences@8
npx cap sync
```

Používáme hlavní verzi 8, protože projekt používá Capacitor 8. Příkaz `npx cap sync` aktualizuje nativní projekty, pokud již existují. Ve webovém prohlížeči Preferences používají `localStorage`, takže lze celé cvičení dokončit i před přidáním Androidu nebo iOS.

Ověřte instalaci:

```bash
npm ls @capacitor/preferences
```

## 4. Rozšíření datového modelu

Historie má zobrazovat také čas vytvoření záznamu. Otevřete:

```text
src/app/models/saved-counter.ts
```

A rozšiřte rozhraní:

```typescript
export interface SavedCounter {
  id: string;
  name: string;
  value: number;
  createdAt: string;
}
```

Čas ukládáme jako řetězec ve formátu ISO 8601. Taková hodnota je přenositelná, lze ji uložit do JSON a později převést zpět na `Date`.

## 5. Doplnění času při uložení

Otevřete:

```text
src/app/components/counter/counter.component.ts
```

V metodě `save()` doplňte do emitovaného objektu vlastnost `createdAt`:

```typescript
save(): void {
  const name = this.counterName.trim();

  if (!name) {
    return;
  }

  this.saved.emit({
    id: crypto.randomUUID(),
    name,
    value: this.count,
    createdAt: new Date().toISOString(),
  });

  this.counterName = '';
  this.count = 0;
}
```

TypeScript by po rozšíření rozhraní ohlásil chybu na původním objektu bez `createdAt`. Tím pomáhá udržet všechny objekty typu `SavedCounter` ve správném tvaru.

## 6. Vygenerování služby

V novém terminálu spusťte:

### Windows – PowerShell

```powershell
npx ng generate service services/counter --type=service
```

### macOS – Terminal

```bash
npx ng generate service services/counter --type=service
```

Generátor vytvoří soubor:

```text
src/app/services/counter.service.ts
```

Parametr `--type=service` zachová v Angularu 22 příponu `.service.ts`, kterou používáme v importech níže. Bez něj může současný generátor vytvořit pouze soubor `counter.ts`.

Služba bude jediným místem, které zná klíč použitý v Preferences a způsob převodu dat do JSON. Komponenty nebudou komunikovat s Preferences přímo.

## 7. Implementace CounterService

Nahraďte obsah `src/app/services/counter.service.ts`:

```typescript
import { Injectable, signal } from '@angular/core';
import { Preferences } from '@capacitor/preferences';
import { SavedCounter } from '../models/saved-counter';

@Injectable({
  providedIn: 'root',
})
export class CounterService {
  private readonly storageKey = 'saved-counters';
  private readonly countersState = signal<SavedCounter[]>([]);
  private readonly initializedState = signal(false);
  private initializationPromise: Promise<void> | null = null;

  readonly counters = this.countersState.asReadonly();
  readonly initialized = this.initializedState.asReadonly();

  initialize(): Promise<void> {
    this.initializationPromise ??= this.load();
    return this.initializationPromise;
  }

  async add(counter: SavedCounter): Promise<void> {
    await this.initialize();
    this.countersState.update((counters) => [counter, ...counters]);
    await this.persist();
  }

  async remove(id: string): Promise<void> {
    await this.initialize();
    this.countersState.update((counters) =>
      counters.filter((counter) => counter.id !== id),
    );
    await this.persist();
  }

  async clear(): Promise<void> {
    await this.initialize();
    this.countersState.set([]);
    await Preferences.remove({ key: this.storageKey });
  }

  private async load(): Promise<void> {
    try {
      const { value } = await Preferences.get({ key: this.storageKey });

      if (value === null) {
        this.countersState.set([]);
        return;
      }

      const parsed: unknown = JSON.parse(value);

      if (!Array.isArray(parsed)) {
        throw new Error('Uložená historie nemá očekávaný formát pole.');
      }

      this.countersState.set(parsed as SavedCounter[]);
    } catch (error) {
      console.error('Historii počítadel se nepodařilo načíst.', error);
      this.countersState.set([]);
    } finally {
      this.initializedState.set(true);
    }
  }

  private async persist(): Promise<void> {
    await Preferences.set({
      key: this.storageKey,
      value: JSON.stringify(this.countersState()),
    });
  }
}
```

### Co služba dělá

| Část | Význam |
| --- | --- |
| `providedIn: 'root'` | Angular vytvoří jednu sdílenou instanci pro celou aplikaci. |
| `signal<SavedCounter[]>([])` | Udržuje reaktivní stav historie. |
| `asReadonly()` | Stránky mohou hodnotu číst, ale nemohou signal přímo přepsat. |
| `initialize()` | Spustí načtení nejvýše jednou a vrací stejný `Promise`. |
| `Preferences.get()` | Načte text uložený pod jedním klíčem. |
| `JSON.parse()` | Převede text zpět na pole objektů. |
| `JSON.stringify()` | Převede pole objektů na text pro Preferences. |
| `Preferences.remove()` | Odstraní pouze historii této aplikace. |

Nepoužívejte zde `Preferences.clear()`. Tato metoda by mohla odstranit také jiné hodnoty uložené aplikací. Pro vymazání historie známe přesný klíč, proto používáme `remove()`.

Proměnná `initializationPromise` zabraňuje tomu, aby dvě stránky spustily souběžně dvě načítání a později si navzájem přepsaly stav.

## 8. Zjednodušení první záložky

Historii nyní zobrazíme na druhé záložce. První záložka bude pouze přijímat událost z `CounterComponent` a předávat data službě.

Nahraďte obsah `src/app/tab1/tab1.page.ts`:

```typescript
import { Component, inject } from '@angular/core';
import { IonContent, IonHeader, IonTitle, IonToolbar } from '@ionic/angular';
import { CounterComponent } from '../components/counter/counter.component';
import { SavedCounter } from '../models/saved-counter';
import { CounterService } from '../services/counter.service';

@Component({
  selector: 'app-tab1',
  templateUrl: 'tab1.page.html',
  styleUrls: ['tab1.page.scss'],
  imports: [CounterComponent, IonContent, IonHeader, IonTitle, IonToolbar],
})
export class Tab1Page {
  private readonly counterService = inject(CounterService);

  async onSaved(counter: SavedCounter): Promise<void> {
    await this.counterService.add(counter);
  }
}
```

Nahraďte obsah `src/app/tab1/tab1.page.html`:

```html
<ion-header [translucent]="true">
  <ion-toolbar>
    <ion-title>CounterApp</ion-title>
  </ion-toolbar>
</ion-header>

<ion-content [fullscreen]="true" class="ion-padding">
  <app-counter
    heading="Nové počítadlo"
    (saved)="onSaved($event)"
  ></app-counter>
</ion-content>
```

Soubor `src/app/tab1/tab1.page.scss` může zůstat prázdný. Styly samotného počítadla již od CV3 patří komponentě `CounterComponent`.

## 9. Implementace záložky Historie

Nahraďte obsah `src/app/tab2/tab2.page.ts`:

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { DatePipe } from '@angular/common';
import {
  IonButton,
  IonContent,
  IonHeader,
  IonItem,
  IonLabel,
  IonList,
  IonNote,
  IonSpinner,
  IonTitle,
  IonToolbar,
} from '@ionic/angular';
import { CounterService } from '../services/counter.service';

@Component({
  selector: 'app-tab2',
  templateUrl: 'tab2.page.html',
  styleUrls: ['tab2.page.scss'],
  imports: [
    DatePipe,
    IonButton,
    IonContent,
    IonHeader,
    IonItem,
    IonLabel,
    IonList,
    IonNote,
    IonSpinner,
    IonTitle,
    IonToolbar,
  ],
})
export class Tab2Page implements OnInit {
  readonly counterService = inject(CounterService);

  async ngOnInit(): Promise<void> {
    await this.counterService.initialize();
  }

  async remove(id: string): Promise<void> {
    await this.counterService.remove(id);
  }

  async clear(): Promise<void> {
    await this.counterService.clear();
  }
}
```

`DatePipe` používáme pro čitelné zobrazení ISO řetězce. Služba je zde veřejná, protože její signály čte HTML šablona.

Nahraďte obsah `src/app/tab2/tab2.page.html`:

```html
<ion-header [translucent]="true">
  <ion-toolbar>
    <ion-title>Historie</ion-title>
  </ion-toolbar>
</ion-header>

<ion-content [fullscreen]="true" class="ion-padding">
  @if (!counterService.initialized()) {
    <div class="state-message">
      <ion-spinner></ion-spinner>
      <p>Načítám historii…</p>
    </div>
  } @else if (counterService.counters().length === 0) {
    <p class="state-message">Historie je zatím prázdná.</p>
  } @else {
    <div class="history-header">
      <p>Počet záznamů: {{ counterService.counters().length }}</p>
      <ion-button color="danger" fill="outline" size="small" (click)="clear()">
        Vymazat vše
      </ion-button>
    </div>

    <ion-list>
      @for (counter of counterService.counters(); track counter.id) {
        <ion-item>
          <ion-label>
            <h2>{{ counter.name }}</h2>
            <p>{{ counter.createdAt | date: 'd. M. y, HH:mm' }}</p>
          </ion-label>

          <ion-note slot="end">{{ counter.value }}</ion-note>

          <ion-button
            slot="end"
            color="danger"
            fill="clear"
            aria-label="Odstranit záznam"
            (click)="remove(counter.id)"
          >
            Odstranit
          </ion-button>
        </ion-item>
      }
    </ion-list>
  }
</ion-content>
```

Do `src/app/tab2/tab2.page.scss` vložte:

```scss
.history-header,
ion-list,
.state-message {
  max-width: 40rem;
  margin: 1rem auto;
}

.history-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.state-message {
  color: var(--ion-color-medium);
  text-align: center;
}

ion-note {
  font-size: 1.25rem;
  font-weight: 700;
}
```

## 10. Tok dat v aplikaci

Po úpravě probíhá uložení následovně:

```text
CounterComponent
  │  saved(counter)
  ▼
Tab1Page
  │  CounterService.add(counter)
  ▼
CounterService signal
  ├──► Tab2Page – okamžitě překreslí historii
  └──► Preferences – uloží JSON pro další spuštění
```

Komponenta počítadla neví, kde se data ukládají. `Tab1Page` pouze předává událost službě. `Tab2Page` pouze čte stav a vyvolává operace služby. Zodpovědnost za data a perzistenci má `CounterService`.

## 11. Spuštění a ověření ve webovém prohlížeči

Spusťte:

```bash
ionic serve
```

Postupně ověřte:

1. záložka Historie se po prvním otevření zobrazí jako prázdná,
2. na záložce Počítadlo uložte alespoň tři různé záznamy,
3. v Historii jsou záznamy seřazené od nejnovějšího,
4. každý záznam obsahuje název, hodnotu a čas,
5. po obnovení stránky záznamy zůstanou zachované,
6. odstranění jednoho záznamu neovlivní ostatní,
7. odstraněný záznam se nevrátí ani po dalším obnovení,
8. tlačítko Vymazat vše odstraní celou historii,
9. po vymazání se zobrazí prázdný stav,
10. konzole prohlížeče neobsahuje červené chyby.

V Developer Tools můžete v části **Application → Local Storage → http://localhost:8100** najít uložený klíč. Přesný technický název klíče může Capacitor ve webovém prostředí doplnit vlastním prefixem.

## 12. Kontrola odolnosti uložených dat

Preferences vrací `string | null` a uložený text nemusí vždy obsahovat očekávaný JSON. Proto služba:

- samostatně řeší chybějící hodnotu `null`,
- obaluje `JSON.parse()` blokem `try/catch`,
- ověřuje, že výsledkem je pole,
- při chybě zobrazí prázdnou historii a zapíše problém do konzole.

Kontrola `Array.isArray()` zatím neověřuje jednotlivé vlastnosti každého objektu. Úplná validace cizích dat bude důležitější při komunikaci se vzdáleným API.

## 13. Produkční sestavení

Ukončete vývojový server pomocí `Ctrl+C` a spusťte:

```bash
npm run lint
ionic build
```

Po instalaci pluginu musí být změněné soubory `package.json` a `package-lock.json`. Nevytvářejte ani neupravujte nativní adresáře ručně.

## 14. Uložení práce do Gitu

Nejprve zkontrolujte změny:

```bash
git status
git diff
```

Potom řešení uložte:

```bash
git add package.json package-lock.json src/app
git commit -m "Uloz historii pomoci Capacitor Preferences"
```

Nakonec ověřte:

```bash
git status
git log --oneline -3
```

Pracovní strom má být čistý a nejnovější commit má obsahovat řešení CV4.

## 15. Kontrolní seznam CV4

- [ ] Pracuji ve větvi `cv4/preferences`.
- [ ] Projekt obsahuje `@capacitor/preferences` hlavní verze 8.
- [ ] `SavedCounter` obsahuje čas vytvoření `createdAt`.
- [ ] `CounterService` je dostupná přes `providedIn: 'root'`.
- [ ] Stav historie je uložený v signal a ven je vystavený pouze pro čtení.
- [ ] Komponenty nevolají Preferences přímo.
- [ ] Inicializace služby proběhne nejvýše jednou.
- [ ] Záložka Počítadlo ukládá záznam prostřednictvím služby.
- [ ] Záložka Historie zobrazuje všechny uložené záznamy.
- [ ] Lze odstranit jeden záznam i celou historii.
- [ ] Data přežijí obnovení stránky.
- [ ] `npm run lint` a `ionic build` skončí bez chyby.
- [ ] Výsledek je uložený v Git commitu.

## 16. Bonusové úkoly

Po dokončení povinné části můžete:

1. zobrazit součet hodnot všech uložených počítadel,
2. přidat řazení historie podle názvu nebo hodnoty,
3. přidat vyhledávání podle názvu,
4. před smazáním celé historie zobrazit `ion-alert`,
5. po úspěšném uložení nebo odstranění zobrazit `ion-toast`,

Potvrzovací dialog a toast budou podrobněji využity v některém z následujících cvičení. Bonusové řešení proto držte oddělené od služby pro ukládání dat.

## 17. Nejčastější problémy

### `Cannot find module '@capacitor/preferences'`

Plugin není nainstalovaný nebo editor ještě nenačetl nové závislosti. V kořeni projektu spusťte:

```bash
npm install @capacitor/preferences@8
npx cap sync
```

Potom restartujte TypeScript server nebo VS Code.

### Historie po obnovení zmizí

Zkontrolujte:

- zda metoda `add()` volá `await this.persist()`,
- zda `persist()` používá stejný `storageKey` jako `load()`,
- zda je stránka otevřená přes `http://localhost:8100`,
- chyby v konzoli prohlížeče.

### Historie se zobrazí až po přepnutí záložky

V šabloně čtěte signal jako funkci:

```html
counterService.counters()
```

Nesprávně je `counterService.counters` bez závorek.

### `No pipe found with name 'date'`

Do `tab2.page.ts` importujte `DatePipe` z `@angular/common` a přidejte jej do pole `imports` komponenty.

### Tlačítko odstraní více záznamů

Každý objekt musí mít vlastní `id` a cyklus musí používat:

```html
@for (counter of counterService.counters(); track counter.id)
```

Metoda `remove()` porovnává právě tuto hodnotu `id`.

### Poškozený JSON vyvolá chybu v konzoli

To je očekávané. Služba chybu zachytí, vypíše její příčinu pro vývojáře a aplikace pokračuje s prázdnou historií. Tlačítkem Vymazat vše lze poškozenou hodnotu odstranit.

## Oficiální dokumentace

- [Capacitor Preferences](https://capacitorjs.com/docs/apis/preferences)
- [Angular – Creating and using services](https://angular.dev/guide/di/creating-and-using-services)
- [Angular – Signals](https://angular.dev/guide/signals)
- [Angular – Dependency injection](https://angular.dev/guide/di)
- [Angular – DatePipe](https://angular.dev/api/common/DatePipe)
- [Ionic UI Components](https://ionicframework.com/docs/components)
