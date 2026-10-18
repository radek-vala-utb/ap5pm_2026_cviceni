# CV8 – Cloudová historie krok za krokem

## Cíl a plán na 90 minut

Navazujete na [CV7](../cv07/README.md): máte nastavený Firebase projekt, anonymní přihlášení a funkční lokální CounterApp. Dnes nahradíte úložiště historie službou pro Cloud Firestore. Model `SavedCounter` a komponenta počítadla se nemění.

| Čas | Náplň |
| --- | --- |
| 0–10 min | Identita, struktura dat a pravidla |
| 10–30 min | Kostra služby a jednorázové načtení |
| 30–50 min | Zápis a přepojení stránek |
| 50–60 min | Samostatné doplnění mazání |
| 60–75 min | Přechod na živý posluchač a dvě karty |
| 75–85 min | Oddělení uživatelů a odmítnutí cizího přístupu |
| 85–90 min | Kontrola a commit |

**Výsledek:** záznamy přežijí obnovení stránky, dvě karty stejného uživatele sdílejí změny a cizí uživatel k nim nemá přístup. Android a další rozšíření jsou bonus; nemusíte je stihnout během cvičení.

Postupujte po úryvcích. **Přidat metodu** znamená vložit ji dovnitř existující třídy, **nahradit metodu** znamená odstranit její předchozí verzi. Importy průběžně doplňujte podle pokynů. Nevkládejte druhou kopii celé třídy.

V kořeni `counter-app` po dokončeném CV7 vytvořte větev:

```bash
git status
git switch -c cv8/firestore
```

## 1. Kam patří záznam?

Použijeme cestu:

```text
users/{uid}/counters/{counterId}
```

`uid` určuje vlastníka, `counterId` je existující `SavedCounter.id`. Dokument obsahuje stejná pole jako model z CV4: `id`, `name`, `value`, `createdAt`. ISO čas v `createdAt` ponecháme beze změny.

**Nejprve sami:** Napište konkrétní cestu pro uživatele `anna` a záznam `pokus-1`. Která část cesty se změní, když Anna uloží druhé počítadlo?

### Pravidla na serveru

Ve **Firestore Database → Rules** publikujte následující pravidla. Stejný obsah uložte do nového souboru `firestore.rules` v kořeni projektu, aby byl součástí Gitu. Samotné uložení souboru pravidla nenasazuje.

```text
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/counters/{counterId} {
      allow read, write: if request.auth != null
                        && request.auth.uid == userId;
    }
  }
}
```

V této verzi řešíme **vlastnictví dat**: přihlášený uživatel smí pracovat pouze pod vlastním UID. Kontrola typů a povolených polí je bonus. Pravidla ostatní cesty nepovolují.

**Vysvětlete sousedovi:** Proč nestačí načíst všechny uživatele a v Angularu zobrazit jen vlastní záznamy? Pravidla nejsou filtr výsledků; náš dotaz musí směřovat rovnou do povolené kolekce.

## 2. Kostra cloudové služby

```bash
npx ng generate service services/firebase-counter --type=service
```

Do `src/app/services/firebase-counter.service.ts` vložte počáteční kostru:

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { collection, getDocs, orderBy, query } from 'firebase/firestore';
import { firestore } from '../firebase.client';
import { SavedCounter } from '../models/saved-counter';
import { FirebaseAuthService } from './firebase-auth.service';

@Injectable({ providedIn: 'root' })
export class FirebaseCounterService {
  private readonly auth = inject(FirebaseAuthService);
  private readonly countersState = signal<SavedCounter[]>([]);
  private readonly initializedState = signal(false);
  private readonly errorState = signal('');

  readonly counters = this.countersState.asReadonly();
  readonly initialized = this.initializedState.asReadonly();
  readonly errorMessage = this.errorState.asReadonly();
}
```

Rozhraní `counters()` a `initialized()` odpovídá lokální službě z CV4. Stránky díky tomu nebudeme celé přepisovat.

### 2.1 Kolekce přihlášeného uživatele

Do třídy přidejte:

```typescript
private async userCounters() {
  const uid = await this.auth.initialize();
  return collection(firestore, 'users', uid, 'counters');
}
```

Metoda nejdříve získá UID, teprve potom sestaví odkaz na kolekci. Vytvoření odkazu samo data nenačítá. Návratový typ zde TypeScript odvodí; `async` znamená, že volající použije `await`.

### 2.2 Jednorázové čtení

Přidejte veřejnou metodu:

```typescript
async load(): Promise<void> {
  this.errorState.set('');

  try {
    const counters = await this.userCounters();
    const snapshot = await getDocs(query(counters, orderBy('createdAt', 'desc')));
    const items = snapshot.docs.map((item) => item.data() as SavedCounter);
    this.countersState.set(items);
  } catch (error) {
    console.error(error);
    this.errorState.set('Historii se nepodařilo načíst.');
  } finally {
    this.initializedState.set(true);
  }
}
```

Čtení má čtyři kroky: odkaz na kolekci → seřazený dotaz → odpověď serveru → pole pro Angular signal. `finally` ukončí indikátor načítání i při chybě.

`as SavedCounter` je informace pro TypeScript, **ne kontrola dat za běhu**. Pro cvičení předpokládáme dokumenty vytvořené naší aplikací. V produkci bychom doplnili validaci pravidly i při čtení.

**Úkol:** Než budete pokračovat, změňte směr řazení na vzestupný a vysvětlete, co se změní. Pak vraťte `'desc'`.

### 2.3 Vstup pro stránky

Přidejte do třídy metodu, kterou už stránky znají z lokální služby:

```typescript
async initialize(): Promise<void> {
  await this.load();
}
```

Zatím jde o jednorázové načtení. Později stejný vstup použijeme ke spuštění jednoho posluchače.

## 3. Uložení prvního dokumentu

Do existujícího importu z `firebase/firestore` doplňte `doc` a `setDoc`. Do třídy přidejte:

```typescript
async add(counter: SavedCounter): Promise<void> {
  this.errorState.set('');

  try {
    const counters = await this.userCounters();
    await setDoc(doc(counters, counter.id), counter);
    await this.load();
  } catch (error) {
    console.error(error);
    this.errorState.set('Záznam se nepodařilo uložit.');
  }
}
```

`doc()` sestaví odkaz na konkrétní dokument, `setDoc()` do něj zapíše objekt. Po zápisu zatím historii znovu načteme.

**Předpověď:** Co udělá druhé uložení objektu se stejným `id`? Vznikne druhý dokument, nebo se přepíše první? Ověřte to později v Console.

## 4. Přepojení existujících stránek

### První záložka

V `src/app/tab1/tab1.page.ts` nahraďte import lokální služby:

```typescript
import { FirebaseCounterService } from '../services/firebase-counter.service';
```

Změňte pouze injekci:

```typescript
private readonly counterService = inject(FirebaseCounterService);
```

`onSaved()` už z CV4 volá `await this.counterService.add(counter)`, proto ho ponechte. HTML a `CounterComponent` neměňte.

### Historie

Stejný import a změnu injekce proveďte v `src/app/tab2/tab2.page.ts`; vlastnost zde musí zůstat veřejná pro šablonu:

```typescript
readonly counterService = inject(FirebaseCounterService);
```

Ponechte `auth`, protože v HTML dál zobrazuje UID. Odstraňte `authError` a import `signal`, pokud už ho stránka nepoužívá. Celou metodu `ngOnInit()` nahraďte:

```typescript
async ngOnInit(): Promise<void> {
  await this.counterService.initialize();
}
```

Cloudová služba nyní zajišťuje i potřebné přihlášení a zachytává chyby.

V `tab2.page.html` nahraďte blok identity z CV7 těmito dvěma nezávislými bloky:

```html
@if (counterService.errorMessage()) {
  <p class="state-message">{{ counterService.errorMessage() }}</p>
}

@if (auth.userId(); as uid) {
  <p class="state-message">Anonymní uživatel: {{ uid }}</p>
}
```

Zbytek šablony včetně stavů načítání a prázdné historie ponechte.

**Dočasně odstraňte** z `Tab2Page` metody `remove()` a `clear()` a z HTML obě odpovídající tlačítka (`Odstranit` a `Vymazat vše`). Služba zatím mazat neumí. Tlačítka i metody vrátíte v následujícím kroku. Nyní má být aplikace sestavitelná.

### Kontrolní bod: první cloudová historie

Spusťte `ionic serve`, uložte dva různé záznamy a otevřete Historii. Ve **Firestore Database → Data** najděte své `users/{uid}/counters`. Ověřte pole dokumentu a zachování záznamů po obnovení stránky.

Původní lokální data se nepřenesla. Prázdná cloudová historie při prvním spuštění je v pořádku.

## 5. Mazání doplňte sami

Pro odstranění jednoho dokumentu slouží `deleteDoc(odkazNaDokument)`. Doplňte `deleteDoc` do importu z `firebase/firestore`.

**Samostatně, asi 5 minut:** Napište do služby metodu `async remove(id: string): Promise<void>` podle metody `add()`:

1. vymažte starou chybovou zprávu,
2. získejte kolekci přes `await this.userCounters()`,
3. sestavte odkaz `doc(counters, id)` a předejte ho do `deleteDoc()`,
4. počkejte na odstranění a znovu načtěte seznam,
5. při chybě nastavte zprávu o neúspěšném odstranění.

<details>
<summary>Kontrola řešení až po vlastním pokusu</summary>

```typescript
async remove(id: string): Promise<void> {
  this.errorState.set('');

  try {
    const counters = await this.userCounters();
    await deleteDoc(doc(counters, id));
    await this.load();
  } catch (error) {
    console.error(error);
    this.errorState.set('Záznam se nepodařilo odstranit.');
  }
}
```

</details>

Do `Tab2Page` vraťte tuto metodu a do položky seznamu v HTML tlačítko:

```typescript
async remove(id: string): Promise<void> {
  await this.counterService.remove(id);
}
```

```html
<ion-button slot="end" fill="clear" color="danger" (click)="remove(counter.id)">
  Odstranit
</ion-button>
```

Ověřte odstranění konkrétního dokumentu i v Console. Hromadné mazání `clear()` necháme jako bonus; tlačítko `Vymazat vše` zatím nevracejte.

## 6. Od jednorázového čtení k živým datům

Otevřete dvě běžné karty **stejné adresy ve stejném profilu**, v obou Historii. UID musí být stejné. V první kartě přidejte záznam.

**Pozorujte:** První karta si po zápisu zavolá `load()`. Druhá karta o změně neví, dokud stránku neobnovíte. Co musí být jinak, aby server druhou kartu upozornil sám?

### 6.1 Nahraďte načtení posluchačem

V importu z `firebase/firestore` odstraňte `getDocs` a přidejte `onSnapshot`. Celou metodu `load()` nahraďte soukromou metodou:

```typescript
private async connect(): Promise<void> {
  try {
    const counters = await this.userCounters();
    const countersQuery = query(counters, orderBy('createdAt', 'desc'));

    onSnapshot(
      countersQuery,
      (snapshot) => {
        const items = snapshot.docs.map((item) => item.data() as SavedCounter);
        this.countersState.set(items);
        this.initializedState.set(true);
      },
      (error) => {
        console.error(error);
        this.errorState.set('Živou historii se nepodařilo načíst.');
        this.initializedState.set(true);
      },
    );
  } catch (error) {
    console.error(error);
    this.errorState.set('Připojení k historii se nezdařilo.');
    this.initializedState.set(true);
  }
}
```

`onSnapshot()` doručí první stav i další změny. Callback aktualizuje signal a Angular překreslí seznam. `connect()` končí po registraci posluchače; **nečeká na první data**. Proto `initializedState` nastavujeme až v callbacku nebo při chybě.

### 6.2 Spusťte pouze jeden posluchač

Mezi vlastnosti služby přidejte:

```typescript
private connectionPromise: Promise<void> | null = null;
```

Dosavadní `initialize()` nahraďte:

```typescript
initialize(): Promise<void> {
  if (this.connectionPromise === null) {
    this.connectionPromise = this.connect();
  }

  return this.connectionPromise;
}
```

Jde o stejný princip jako u přihlašování v CV7. Návrat na záložku nemá založit další posluchač. Tato služba žije po celou dobu aplikace. `onSnapshot()` vrací také funkci pro odpojení; u krátce žijící služby nebo při odhlášení uživatele bychom ji museli zavolat. Odhlášení v tomto cvičení nemáme.

### 6.3 Zápis už nenačítá seznam

Z metod `add()` a `remove()` **odstraňte řádek**:

```typescript
await this.load();
```

Jiné řádky těchto metod neměňte. Aktualizaci seznamu nyní zajišťuje posluchač. Nepřidávejte záznam ještě ručně do signalu, jinak by mohl vzniknout dvakrát.

Výsledný import Firestore má obsahovat právě používané funkce:

```typescript
import {
  collection,
  deleteDoc,
  doc,
  onSnapshot,
  orderBy,
  query,
  setDoc,
} from 'firebase/firestore';
```

**Kontrolní bod:** Obnovte obě karty, v obou otevřete Historii. Přidejte a odstraňte záznam. Změny se mají projevit v obou kartách bez obnovení. Vysvětlete vlastními slovy, kdo teď mění `countersState`.

## 7. Vlastní data a cizí data

### Dva různí uživatelé

V soukromém okně otevřete aplikaci a Historii. UID bude jiné a seznam prázdný. Přidejte záznam; v běžném okně se objevit nesmí.

Tím jste ověřili oddělené cesty. Ještě ale potřebujete zjistit, zda server opravdu odmítne cizí přístup.

### Zkuste zakázaný přístup

V běžném okně si poznamenejte UID ze soukromého okna. V metodě `userCounters()` dočasně nahraďte pouze návratový řádek:

```typescript
return collection(firestore, 'users', 'UID_ZE_SOUKROMEHO_OKNA', 'counters');
```

Místo zástupného textu vložte skutečné cizí UID. Obnovte běžné okno a otevřete Historii. Očekáváte `permission-denied`, chybovou zprávu a žádná cizí data. Pokud lint hlásí nepoužité `uid`, jde o důsledek dočasné změny; po testu ji ihned vraťte:

```typescript
return collection(firestore, 'users', uid, 'counters');
```

Aplikaci znovu obnovte a ověřte vlastní historii. Pravidla neuvolňujte.

**Otázka:** Který z pokusů prokázal bezpečnost pravidel a který pouze správnou cestu v klientském kódu?

## 8. Dokončení a odevzdání

```bash
npm run lint
ionic build
git diff --check
git diff
```

- [ ] Přidání a odstranění funguje i po obnovení stránky.
- [ ] Dvě karty se stejným UID sdílejí změny bez obnovení.
- [ ] Soukromé okno má jiné UID a vlastní data.
- [ ] Přístup k cizímu UID server odmítl; testovací změna je vrácená.
- [ ] V metodách nezůstalo volání odstraněného `load()` ani import `getDocs`.
- [ ] Lokální `CounterService` zůstal v projektu pro porovnání.
- [ ] Soubor `firestore.rules` odpovídá publikovaným pravidlům.
- [ ] Lint a sestavení prošly.

```bash
git add src firestore.rules
git commit -m "CV8: cloudova historie a zive zmeny"
```

Před odchodem vysvětlete spolužákovi bez čtení kódu cestu: **kliknutí na Uložit → dokument ve Firestore → callback → signal → seznam**.

## Bonusy podle zbývajícího času

Povinnou část máte dokončenou. Vyberte si jeden úkol:

1. **Vymazání celé historie:** napište `clear()` pomocí `writeBatch()`, `batch.delete()` pro každý aktuální záznam a `batch.commit()`. Vraťte tlačítko i metodu stránky. Pracujte jen s malou cvičnou historií.
2. **Validace dat:** rozšiřte pravidlo zápisu o kontrolu polí a typů; při mazání není `request.resource.data` dostupné, proto oddělte `create, update` od `delete`. Zkuste zapsat chybný typ hodnoty.
3. **Android:** spusťte `ionic build`, `npx cap sync android` a `npx cap run android`. Uložte záznam a aplikaci znovu spusťte. WebView má vlastní anonymní účet, a proto jiná data než desktop.
4. **Řazení a omezení:** pomocí `limit(10)` zobrazte deset nejnovějších záznamů. Co nyní znamená počet položek v UI?
5. **Síť:** po načtení historie vypněte síť a zkuste zápis. Posluchač může ukázat lokální změnu před potvrzením serveru; `setDoc()` může zůstat čekat. Po obnovení sítě zkontrolujte dokument v Console. Nezaměňujte lokální zobrazení za potvrzený zápis.

## Když se nedaří

- **`permission-denied`:** porovnejte UID, cestu kolekce a publikovaná pravidla. Zkontrolujte, zda nezůstalo cizí testovací UID.
- **Historie je prázdná:** cloudová historie nepřebírá staré Preferences. Zkontrolujte správný Firebase projekt a účet.
- **Druhá karta se neaktualizuje:** ověřte shodnou adresu a UID, nahrazení `load()` posluchačem a otevření Historie v obou kartách.
- **Nelze najít `load`, `clear` nebo `remove`:** některý z mezikroků není dokončen. `load()` ve finální službě není; `clear()` a jeho tlačítko patří do bonusu.
- **Načítání nekončí:** zkontrolujte Console a oba callbacky posluchače. Úspěch i chyba musí nastavit `initializedState`.
- **Záznam se objeví dvakrát:** signal historie má měnit pouze callback `onSnapshot()`.

## Oficiální dokumentace

- [Zápis dokumentů](https://firebase.google.com/docs/firestore/manage-data/add-data)
- [Jednorázové čtení](https://firebase.google.com/docs/firestore/query-data/get-data)
- [Poslech změn](https://firebase.google.com/docs/firestore/query-data/listen)
- [Podmínky bezpečnostních pravidel](https://firebase.google.com/docs/firestore/security/rules-conditions)
