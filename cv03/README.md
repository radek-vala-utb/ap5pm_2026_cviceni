# CV3 – Znovupoužitelná komponenta a komunikace mezi komponentami

## Cíl cvičení

V CV2 byla veškerá logika počítadla součástí stránky `Tab1Page`. V tomto cvičení ji přesunete do samostatné komponenty `CounterComponent`. Stránka bude komponentu používat a přijímat od ní dokončená počítadla.

Po dokončení cvičení budete umět:

- vygenerovat novou Angular komponentu,
- rozdělit větší stránku na menší znovupoužitelné části,
- předat hodnotu z rodiče do potomka pomocí `input()`,
- poslat událost a data z potomka rodiči pomocí `output()`,
- používat typovaný datový model v rozhraní TypeScriptu,
- vykreslit kolekci pomocí moderního bloku `@for`,
- podmíněně zobrazit obsah pomocí `@if` a `@else`,
- pracovat na samostatné Git větvi.

Uložená počítadla budou zatím pouze v operační paměti. Po obnovení stránky zmizí. Perzistentní ukládání je tématem CV4.

## 1. Výchozí stav

Navazujete na vlastní projekt `counter-app` dokončený v CV2. Před zahájením musí fungovat:

- zadání názvu počítadla,
- zvýšení, snížení a reset hodnoty,
- tři navigační záložky,
- příkaz `ionic build`.

Pokud CV2 dokončené nemáte, nejprve se vraťte k jeho kontrolnímu seznamu. Nekopírujte řešení od jiného studenta.

## 2. Otevření projektu a kontrola prostředí

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty\counter-app"
nvm use 26
node --version
git status
code .
```

### macOS – Terminal

```bash
cd "$HOME/AP5PM-projekty/counter-app"
nvm use 26
node --version
git status
code .
```

Pokud Node.js nepoužíváte přes `nvm`, příkaz `nvm use 26` vynechte. `node --version` musí začínat `v26.`.

Výstup `git status` musí před zahájením hlásit čistý pracovní strom. Pokud obsahuje změny z CV2, nejprve je zkontrolujte a uložte do commitu.

## 3. Vytvoření pracovní větve

### Windows – PowerShell

```powershell
git switch -c cv3/reusable-counter
```

### macOS – Terminal

```bash
git switch -c cv3/reusable-counter
```

Příkaz je na obou platformách stejný. Větev odděluje rozpracované CV3 od funkčního výsledku CV2.

Pokud větev již existuje, pouze se na ni přepněte:

### Windows – PowerShell

```powershell
git switch cv3/reusable-counter
```

### macOS – Terminal

```bash
git switch cv3/reusable-counter
```

## 4. Spuštění výchozí aplikace

### Windows – PowerShell

```powershell
ionic serve
```

### macOS – Terminal

```bash
ionic serve
```

Ověřte aplikaci na `http://localhost:8100`. Vývojový server ponechte spuštěný; další příkazy zadávejte v novém terminálu otevřeném v adresáři projektu.

## 5. Vygenerování komponenty

Otevřete ve VS Code nový integrovaný terminál pomocí **Terminal → New Terminal**.

### Windows – PowerShell

```powershell
npx ng generate component components/counter
```

### macOS – Terminal

```bash
npx ng generate component components/counter
```

Zkrácený zápis stejného příkazu je `npx ng g c components/counter`. V materiálech používáme delší variantu, protože je srozumitelnější.

Generátor vytvoří:

```text
src/app/components/counter/
├── counter.component.ts
├── counter.component.html
├── counter.component.scss
└── counter.component.spec.ts
```

Soubor `.spec.ts` obsahuje automatizovaný test komponenty. V tomto cvičení jej nebudeme upravovat ani mazat.

## 6. Vytvoření datového modelu

Každé uložené počítadlo bude mít jednoznačný identifikátor, název a číselnou hodnotu.

Vytvořte adresář pro modely.

### Windows – PowerShell

```powershell
New-Item -ItemType Directory -Force -Path "src\app\models" | Out-Null
New-Item -ItemType File -Force -Path "src\app\models\saved-counter.ts" | Out-Null
```

### macOS – Terminal

```bash
mkdir -p "src/app/models"
touch "src/app/models/saved-counter.ts"
```

Do `src/app/models/saved-counter.ts` vložte:

```typescript
export interface SavedCounter {
  id: string;
  name: string;
  value: number;
}
```

Rozhraní `interface` existuje pouze při vývoji a při překladu kontroluje tvar dat. Do výsledného JavaScriptu se nepřenáší.

## 7. Implementace CounterComponent

Otevřete:

```text
src/app/components/counter/counter.component.ts
```

Nahraďte celý obsah:

```typescript
import { Component, input, output } from '@angular/core';
import { FormsModule } from '@angular/forms';
import {
  IonButton,
  IonCard,
  IonCardContent,
  IonCardHeader,
  IonCardTitle,
  IonInput,
} from '@ionic/angular';
import { SavedCounter } from '../../models/saved-counter';

@Component({
  selector: 'app-counter',
  templateUrl: './counter.component.html',
  styleUrls: ['./counter.component.scss'],
  imports: [
    FormsModule,
    IonButton,
    IonCard,
    IonCardContent,
    IonCardHeader,
    IonCardTitle,
    IonInput,
  ],
})
export class CounterComponent {
  readonly heading = input('Nové počítadlo');
  readonly saved = output<SavedCounter>();

  counterName = '';
  count = 0;

  increment(): void {
    this.count++;
  }

  decrement(): void {
    if (this.count > 0) {
      this.count--;
    }
  }

  reset(): void {
    this.count = 0;
  }

  save(): void {
    const name = this.counterName.trim();

    if (!name) {
      return;
    }

    this.saved.emit({
      id: crypto.randomUUID(),
      name,
      value: this.count,
    });

    this.counterName = '';
    this.count = 0;
  }
}
```

Nové části:

| Zápis | Význam |
| --- | --- |
| `input('Nové počítadlo')` | Vstup, který může nastavit rodičovská komponenta |
| `heading()` | Přečtení aktuální hodnoty signal inputu |
| `output<SavedCounter>()` | Typovaný výstup komponenty |
| `saved.emit(data)` | Odeslání události a dat rodiči |
| `crypto.randomUUID()` | Vytvoření jednoznačného identifikátoru |

Angular 22 nabízí moderní funkce `input()` a `output()`. Ve starších návodech můžete potkat dekorátory `@Input()` a `@Output()`; v nových výukových příkladech používáme funkční API.

## 8. Šablona CounterComponent

Otevřete:

```text
src/app/components/counter/counter.component.html
```

Nahraďte obsah:

```html
<ion-card>
  <ion-card-header>
    <ion-card-title>{{ heading() }}</ion-card-title>
  </ion-card-header>

  <ion-card-content>
    <ion-input
      label="Název počítadla"
      label-placement="stacked"
      fill="outline"
      placeholder="Například návštěvníci"
      [(ngModel)]="counterName"
    ></ion-input>

    <p class="counter-value">{{ count }}</p>

    <div class="counter-actions">
      <ion-button
        color="danger"
        [disabled]="count === 0"
        (click)="decrement()"
      >
        −1
      </ion-button>

      <ion-button color="primary" (click)="increment()">
        +1
      </ion-button>

      <ion-button
        color="medium"
        fill="outline"
        [disabled]="count === 0"
        (click)="reset()"
      >
        Reset
      </ion-button>
    </div>

    <ion-button
      class="save-button"
      expand="block"
      color="success"
      [disabled]="!counterName.trim()"
      (click)="save()"
    >
      Uložit počítadlo
    </ion-button>
  </ion-card-content>
</ion-card>
```

Tlačítko Uložit je zakázané, dokud uživatel nezadá alespoň jeden neprázdný znak. Kontrola se opakuje také v metodě `save()`, protože samotné zakázání tlačítka není dostatečná ochrana aplikační logiky.

## 9. Styly CounterComponent

Do `src/app/components/counter/counter.component.scss` vložte:

```scss
ion-card {
  max-width: 32rem;
  margin: 1rem auto;
}

.counter-value {
  margin: 2rem 0;
  font-size: 4rem;
  font-weight: 700;
  line-height: 1;
  text-align: center;
}

.counter-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  justify-content: center;
}

.counter-actions ion-button {
  min-width: 6rem;
}

.save-button {
  margin-top: 1.5rem;
}
```

Styly počítadla jsme přesunuli ze stránky přímo ke komponentě. Odstraňte proto původní obsah `src/app/tab1/tab1.page.scss`; v dalším kroku jej nahradíme styly seznamu.

## 10. Zapojení komponenty do Tab1Page

Stránka bude rodičem a `CounterComponent` jejím potomkem.

```text
Tab1Page  ── heading ──>  CounterComponent
Tab1Page  <─ saved(data) ─ CounterComponent
```

Otevřete `src/app/tab1/tab1.page.ts` a nahraďte celý obsah:

```typescript
import { Component } from '@angular/core';
import {
  IonContent,
  IonHeader,
  IonItem,
  IonLabel,
  IonList,
  IonListHeader,
  IonNote,
  IonTitle,
  IonToolbar,
} from '@ionic/angular';
import { CounterComponent } from '../components/counter/counter.component';
import { SavedCounter } from '../models/saved-counter';

@Component({
  selector: 'app-tab1',
  templateUrl: 'tab1.page.html',
  styleUrls: ['tab1.page.scss'],
  imports: [
    CounterComponent,
    IonContent,
    IonHeader,
    IonItem,
    IonLabel,
    IonList,
    IonListHeader,
    IonNote,
    IonTitle,
    IonToolbar,
  ],
})
export class Tab1Page {
  savedCounters: SavedCounter[] = [];

  onSaved(counter: SavedCounter): void {
    this.savedCounters.unshift(counter);
  }
}
```

Metoda `unshift()` vloží nový prvek na začátek pole, takže nejnovější počítadlo bude v seznamu první.

## 11. Šablona Tab1Page

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

  @if (savedCounters.length > 0) {
    <ion-list>
      <ion-list-header>
        <ion-label>Uložená počítadla</ion-label>
      </ion-list-header>

      @for (counter of savedCounters; track counter.id) {
        <ion-item>
          <ion-label>
            <h2>{{ counter.name }}</h2>
            <p>{{ counter.id }}</p>
          </ion-label>
          <ion-note slot="end">{{ counter.value }}</ion-note>
        </ion-item>
      }
    </ion-list>
  } @else {
    <p class="empty-state">
      Zatím nebylo uloženo žádné počítadlo.
    </p>
  }
</ion-content>
```

Komunikace probíhá ve dvou směrech:

1. `heading="Nové počítadlo"` předává text z rodiče do inputu potomka.
2. `(saved)="onSaved($event)"` zachytí událost potomka; `$event` obsahuje objekt `SavedCounter`.

Blok `@for` potřebuje výraz `track counter.id`. Angular podle identifikátoru pozná, který řádek přibyl nebo se změnil, a nemusí znovu vytvářet celý seznam.

## 12. Styly seznamu

Do `src/app/tab1/tab1.page.scss` vložte:

```scss
ion-list {
  max-width: 32rem;
  margin: 1rem auto;
}

ion-note {
  font-size: 1.5rem;
  font-weight: 700;
}

.empty-state {
  margin: 2rem auto;
  color: var(--ion-color-medium);
  text-align: center;
}
```

## 13. Ověření funkčnosti

V prohlížeči postupně ověřte:

1. komponenta zobrazuje nadpis předaný z `Tab1Page`,
2. bez názvu nelze počítadlo uložit,
3. po zadání názvu lze tlačítko Uložit použít,
4. uložené počítadlo se objeví v seznamu,
5. nejnovější položka se objeví nahoře,
6. každá položka má odlišné UUID,
7. po uložení se formulář i hodnota resetují,
8. lze uložit několik různých počítadel,
9. po obnovení stránky seznam zmizí – v CV3 je to očekávané.

Otevřete Developer Tools a zkontrolujte, že konzole neobsahuje červené chyby.

## 14. Produkční sestavení

Ukončete vývojový server pomocí `Ctrl+C` a spusťte kontrolní build.

### Windows – PowerShell

```powershell
ionic build
```

### macOS – Terminal

```bash
ionic build
```

Build musí skončit bez chyby. Varování není totéž co chyba; pokud si nejste významem výpisu jistí, přiložte jej k dotazu vyučujícímu.

## 15. Kontrola změn a Git commit

### Windows – PowerShell

```powershell
git status
git diff
git add src/app
git commit -m "Refaktor pocitadla do samostatne komponenty"
```

### macOS – Terminal

```bash
git status
git diff
git add src/app
git commit -m "Refaktor pocitadla do samostatne komponenty"
```

Nakonec ověřte:

### Windows – PowerShell

```powershell
git status
git log --oneline -3
```

### macOS – Terminal

```bash
git status
git log --oneline -3
```

Pracovní strom má být čistý a nejnovější commit má obsahovat řešení CV3.

## 16. Kontrolní seznam CV3

- [ ] Pracuji ve větvi `cv3/reusable-counter`.
- [ ] `CounterComponent` byla vytvořena Angular generátorem.
- [ ] Datový model `SavedCounter` je v samostatném souboru.
- [ ] Počítadlo používá moderní `input()` a `output()`.
- [ ] `Tab1Page` neobsahuje logiku zvyšování a snižování hodnoty.
- [ ] Rodič předává potomkovi nadpis přes input.
- [ ] Potomek odesílá rodiči typovaný objekt přes output.
- [ ] Seznam používá `@if`, `@for` a `track counter.id`.
- [ ] Lze uložit více počítadel a nejnovější je první.
- [ ] Příkaz `ionic build` skončí bez chyby.
- [ ] Výsledek je uložený v Git commitu.

## 17. Bonusové úkoly

Po dokončení povinné části můžete:

1. přidat tlačítko pro odstranění jedné uložené položky,
2. zobrazit počet uložených položek v nadpisu seznamu,
3. vypočítat součet hodnot všech uložených počítadel,
4. vložit na stránku dvě nezávislé instance `<app-counter>`,
5. přidat druhý input komponenty pro výchozí hodnotu počítadla.

Dvě instance komponenty musí mít navzájem nezávislý stav. Tím ověříte, že stav skutečně patří jednotlivé instanci `CounterComponent`.

## 18. Nejčastější problémy

### `app-counter is not a known element`

V `tab1.page.ts` zkontrolujte:

```typescript
import { CounterComponent } from '../components/counter/counter.component';
```

`CounterComponent` musí být také uvedena v poli `imports` dekorátoru `@Component`.

### `Property 'saved' does not exist`

Zkontrolujte, že `CounterComponent` obsahuje:

```typescript
readonly saved = output<SavedCounter>();
```

Současně musí být `output` importován z `@angular/core`.

### Nadpis zobrazuje chybu nebo `[object Object]`

Signal input se v TypeScriptu a šabloně čte jako funkce:

```html
{{ heading() }}
```

Nesprávně je pouze `{{ heading }}`.

### Seznam se nevykresluje

Zkontrolujte:

- zda metoda `save()` volá `this.saved.emit(...)`,
- zda šablona rodiče obsahuje `(saved)="onSaved($event)"`,
- zda `onSaved()` přidává přijatý objekt do `savedCounters`,
- chyby v konzoli prohlížeče.

### `crypto.randomUUID is not a function`

Použijte aktuální Chrome nebo Edge a aplikaci otevírejte přes `http://localhost:8100`. Neotevírejte vygenerované HTML přímo jako soubor. Pokud chyba zůstane, oznamte verzi prohlížeče vyučujícímu.

### Data po obnovení stránky zmizí

Jde o očekávané chování CV3. Pole `savedCounters` existuje pouze v paměti stránky. V CV4 jej přesuneme do služby a uložíme pomocí Capacitor Preferences.

## Oficiální dokumentace

- [Angular – Accepting data with input properties](https://angular.dev/guide/components/inputs)
- [Angular – Custom events with outputs](https://angular.dev/guide/components/outputs)
- [Angular – Control flow](https://angular.dev/guide/templates/control-flow)
- [Ionic UI Components](https://ionicframework.com/docs/components)
- [MDN – crypto.randomUUID](https://developer.mozilla.org/docs/Web/API/Crypto/randomUUID)
