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

Pokud na macOS příkaz `code .` není dostupný, použijte:

```bash
open -a "Visual Studio Code" .
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

## 13. Samostatná práce – testy CounterService

`CounterService` obsahuje stav i pravidla pro jeho ukládání. Je proto vhodné testovat ji samostatně, bez spouštění stránek a bez zápisu do skutečného úložiště prohlížeče.

Vaším úkolem je doplnit testy do souboru vytvořeného generátorem:

```text
src/app/services/counter.service.spec.ts
```

### Co mají testy ověřit

Šablona v následující části již obsahuje test prvního spuštění bez dat a opakované inicializace. Tento test nemusíte vytvářet. Samostatně implementujte pouze následující scénáře:

1. **Načtení uložených dat** – JSON obsahující pole `SavedCounter` se po `initialize()` objeví v `counters()`.
2. **Přidání záznamu** – `add()` vloží nový záznam na začátek a zavolá `Preferences.set()` se správným klíčem a JSON hodnotou.
3. **Odstranění jednoho záznamu** – `remove(id)` odstraní pouze odpovídající objekt a nový stav uloží.
4. **Vymazání historie** – `clear()` vyprázdní signal a zavolá `Preferences.remove()` pouze pro klíč `saved-counters`.
5. **Poškozená uložená hodnota** – neplatný JSON nezpůsobí pád testu; služba nastaví prázdnou historii, dokončí inicializaci a zapíše chybu do konzole.

Netestujte přímo privátní metody `load()` a `persist()`. Ověřujte jejich výsledek přes veřejné metody služby a veřejné signály. Test pak zůstane platný, i když se později změní vnitřní implementace.

### Proč je potřeba mock

Jednotkový test nemá zapisovat do skutečného Preferences nebo `localStorage`. Modul `@capacitor/preferences` proto nahradíme mockem – malou řízenou náhradou, u které lze nastavit návratové hodnoty a kontrolovat volání.

Použijte následující základ testovacího souboru a doplňte jednotlivé bloky `it(...)` sami:

```typescript
import { TestBed } from '@angular/core/testing';
import { Preferences } from '@capacitor/preferences';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { SavedCounter } from '../models/saved-counter';
import { CounterService } from './counter.service';

vi.mock('@capacitor/preferences', () => ({
  Preferences: {
    get: vi.fn(),
    set: vi.fn(),
    remove: vi.fn(),
  },
}));

describe('CounterService', () => {
  let service: CounterService;

  const getMock = vi.mocked(Preferences.get);
  const setMock = vi.mocked(Preferences.set);
  const removeMock = vi.mocked(Preferences.remove);

  const first: SavedCounter = {
    id: 'first',
    name: 'První',
    value: 1,
    createdAt: '2026-09-17T08:00:00.000Z',
  };

  const second: SavedCounter = {
    id: 'second',
    name: 'Druhé',
    value: 2,
    createdAt: '2026-09-17T09:00:00.000Z',
  };

  beforeEach(() => {
    // Vynulujeme počty volání mocků z předchozího testu.
    vi.clearAllMocks();

    // Výchozí stav: v Preferences zatím není uložená historie.
    getMock.mockResolvedValue({ value: null });

    // Zápis i odstranění ve výchozím stavu úspěšně skončí.
    setMock.mockResolvedValue(undefined);
    removeMock.mockResolvedValue(undefined);

    // Pro každý test vytvoříme nové prostředí Angular dependency injection.
    TestBed.configureTestingModule({
      providers: [CounterService],
    });

    // Z testovacího injectoru získáme čerstvou instanci služby.
    service = TestBed.inject(CounterService);
  });

  it('should initialize only once', async () => {
    // Dvě volání bez čekání simulují souběžné požadavky na inicializaci.
    await Promise.all([service.initialize(), service.initialize()]);

    // Další volání proběhne až po dokončení první inicializace.
    await service.initialize();

    // Všechna volání musí sdílet jedinou operaci načtení Preferences.
    expect(getMock).toHaveBeenCalledTimes(1);

    // Služba dokončila inicializaci a při value: null má prázdnou historii.
    expect(service.initialized()).toBe(true);
    expect(service.counters()).toEqual([]);
  });

  // TODO: samostatně doplňte testy scénářů 1–5 ze zadání.
});
```

Připravený test ověřuje první spuštění bez uložených dat i opakovanou inicializaci. Dvě volání předaná do `Promise.all()` simulují souběžné požadavky. Třetí volání proběhne až po dokončení inicializace. Ve všech případech musí služba použít uložený Promise, takže `Preferences.get()` bude zavoláno pouze jednou.

### Užitečné vzory

Následující zápisy můžete v testech použít. Nejde o hotové řešení scénářů. První ukázka připraví data v Preferences a ověří jejich načtení:

```typescript
// Určíme, co má falešná metoda Preferences.get() vrátit.
getMock.mockResolvedValue({
  // Preferences ukládají pouze řetězce, proto pole převedeme na JSON.
  value: JSON.stringify([first, second]),
});

// Počkáme, až služba hodnotu načte, převede z JSON a aktualizuje signal.
await service.initialize();

// Signal musí po inicializaci obsahovat stejné dva objekty.
expect(service.counters()).toEqual([first, second]);

// Současně ověříme, že služba četla správný klíč Preferences.
expect(getMock).toHaveBeenCalledWith({ key: 'saved-counters' });
```

`mockResolvedValue()` nastavuje budoucí výsledek asynchronní metody. Objekt se nevrátí okamžitě jako běžná návratová hodnota, ale jako úspěšně dokončený `Promise`. Hodnota uvnitř musí být řetězec, protože Preferences neumějí přímo ukládat pole objektů.

Zápis do Preferences je jiný scénář. `initialize()` pouze čte, takže sama `Preferences.set()` nezavolá. Kontrolu `setMock` proveďte až po operaci, která stav skutečně mění, například po `add()`:

```typescript
// Metoda add() nejprve zajistí inicializaci, přidá objekt a potom stav uloží.
await service.add(first);

// Ověříme přesný objekt předaný do Preferences.set().
expect(setMock).toHaveBeenCalledWith({
  // Služba musí pro čtení i zápis používat stejný klíč.
  key: 'saved-counters',

  // Pole ve stavu služby musí být před uložením převedeno na JSON řetězec.
  value: JSON.stringify([first]),
});
```

Asynchronní test označíme klíčovým slovem `async` a před voláním asynchronní metody použijeme `await`:

```typescript
// async dovoluje uvnitř testu používat await.
it('popis scénáře', async () => {
  // Test se zde pozastaví, dokud initialize() svůj Promise nedokončí.
  await service.initialize();

  // Očekávání zapisujeme až potom, kdy služba stihla načíst a zpracovat data.
  expect(service.initialized()).toBe(true);
});
```

Bez `await` by test pokračoval ihned. Očekávání by se mohlo vyhodnotit ještě před dokončením `Preferences.get()` a test by mohl selhávat podle rychlosti provedení, nikoliv podle správnosti služby.

### Rozšiřující scénář: poškozená uložená data

Služba může v Preferences najít dva různé druhy chybných dat:

- **syntakticky neplatný JSON**, například `'this is not a valid JSON'`; chyba vznikne přímo při volání `JSON.parse()`,
- **platný JSON nesprávného tvaru**, například `'{"name":"test"}'`; převod proběhne, ale výsledkem je objekt místo očekávaného pole a služba vyvolá vlastní chybu `Uložená historie nemá očekávaný formát pole.`

V obou případech blok `catch` chybu zachytí, zapíše ji pomocí `console.error()`, nastaví prázdnou historii a blok `finally` dokončí inicializaci. Test má ověřit především to, že služba nespadne a vrátí se do bezpečného stavu.

Postup testu je následující:

1. nastavte chybnou hodnotu vrácenou z Preferences,
2. vytvořte spy na `console.error`,
3. teprve potom zavolejte a očekejte `service.initialize()`,
4. zkontrolujte stav služby a zachycený výpis,
5. spy nakonec obnovte pomocí `mockRestore()`.

> **Na pořadí záleží:** Spy musí vzniknout **před** `service.initialize()`, protože `console.error()` se zavolá právě během inicializace. Spy vytvořený až po `await service.initialize()` už proběhlé volání nezachytí.

Jednoduchá kostra testu syntakticky neplatného JSON:

```typescript
it('should recover from invalid JSON', async () => {
  // 1. Preferences vrátí text, který nelze převést pomocí JSON.parse().
  getMock.mockResolvedValue({
    value: 'this is not a valid JSON',
  });

  // 2. Sledování musíme zapnout ještě před spuštěním initialize().
  // mockImplementation zároveň zabrání vypsání očekávané chyby do testovacího výstupu.
  const errorSpy = vi
    .spyOn(console, 'error')
    .mockImplementation(() => undefined);

  // 3. Chyba vznikne a bude zachycena uvnitř této inicializace.
  await service.initialize();

  // 4. Služba nespadla a přešla do bezpečného prázdného stavu.
  expect(service.counters()).toEqual([]);
  expect(service.initialized()).toBe(true);

  // CounterService volá console.error se zprávou a zachyceným objektem Error.
  expect(errorSpy).toHaveBeenCalledWith(
    'Historii počítadel se nepodařilo načíst.',
    expect.any(Error),
  );

  // 5. Vrátíme console.error do původního stavu pro ostatní testy.
  errorSpy.mockRestore();
});
```

Pro druhou variantu změňte pouze připravenou hodnotu:

```typescript
getMock.mockResolvedValue({
  value: JSON.stringify({ name: 'test' }),
});
```

Jde o platný JSON, takže `JSON.parse()` uspěje. Následná kontrola `Array.isArray(parsed)` ale zjistí, že výsledkem není pole. Očekávání prázdné historie, dokončené inicializace a volání `console.error()` zůstávají stejná.

V `toHaveBeenCalledWith()` je prvním argumentem přesný text zprávy služby a druhým argumentem objekt chyby, který jednoduše ověříme pomocí `expect.any(Error)`.

### Spuštění a odevzdání

Nejprve spusťte pouze test služby:

```bash
npm test -- --watch=false --include=src/app/services/counter.service.spec.ts
```

Potom ověřte celou testovací sadu:

```bash
npm test -- --watch=false
```

Odevzdaný testovací soubor musí obsahovat připravený inicializační test a samostatně implementované scénáře 1–4. Scénář 5 s poškozenými daty je rozšiřující úloha. Testy musí být navzájem nezávislé – žádný test nesmí spoléhat na stav vytvořený předchozím testem.

## 14. Produkční sestavení

Ukončete vývojový server pomocí `Ctrl+C` a spusťte:

```bash
npm test -- --watch=false
npm run lint
ionic build
```

Po instalaci pluginu musí být změněné soubory `package.json` a `package-lock.json`. Nevytvářejte ani neupravujte nativní adresáře ručně.

## 15. Uložení práce do Gitu

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

## 16. Kontrolní seznam CV4

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
- [ ] `counter.service.spec.ts` obsahuje připravený inicializační test a povinné scénáře 1–4.
- [ ] Testy používají mock Preferences a nezapisují do skutečného úložiště.
- [ ] `npm test -- --watch=false` skončí bez chyby.
- [ ] `npm run lint` a `ionic build` skončí bez chyby.
- [ ] Výsledek je uložený v Git commitu.

## 17. Bonusové úkoly

Po dokončení povinné části můžete:

1. zobrazit součet hodnot všech uložených počítadel,
2. přidat řazení historie podle názvu nebo hodnoty,
3. přidat vyhledávání podle názvu,
4. před smazáním celé historie zobrazit `ion-alert`,
5. po úspěšném uložení nebo odstranění zobrazit `ion-toast`,

Potvrzovací dialog a toast budou podrobněji využity v některém z následujících cvičení. Bonusové řešení proto držte oddělené od služby pro ukládání dat.

## 18. Nejčastější problémy

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
- [Angular – Testing services](https://angular.dev/guide/testing/services)
- [Angular – DatePipe](https://angular.dev/api/common/DatePipe)
- [Ionic UI Components](https://ionicframework.com/docs/components)
