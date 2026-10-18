# CV7 – Firebase a identita uživatele

## Cíl a plán na 90 minut

Pokračujte v `counter-app` po [CV6](../cv06/README.md). Dnes připojíte Firebase a zjistíte, jak aplikace pozná svého uživatele bez formuláře s heslem. Historie bude stále uložená lokálně pomocí Preferences. Cloudovou databázi použijete až v [CV8](../cv08/README.md).

| Čas | Náplň |
| --- | --- |
| 0–10 min | Výchozí aplikace, lokální data a identita |
| 10–30 min | Firebase projekt a nastavení služeb |
| 30–45 min | SDK a konfigurace |
| 45–65 min | Postupné sestavení přihlašovací služby |
| 65–80 min | Zobrazení identity a pokusy ve dvou oknech |
| 80–90 min | Kontrola, commit a rezerva |

**Výsledek:** aplikace zobrazí UID, po obnovení ho zachová a v soukromém okně získá jinou identitu. Nejde zatím o sdílení účtu mezi zařízeními.

Pracujte po jednotlivých krocích. U každého úryvku je uvedeno, kam patří. Nové metody vkládejte **dovnitř třídy**, importy na začátek souboru. Krátké nové soubory uvádíme celé; delší službu sestavíte postupně.

## 1. Příprava a otázka na začátek

Ověřte, že funguje uložení počítadla a lokální historie z CV4. Původní `CounterService` ponechte.

Otevřete terminál v kořeni `counter-app` a vytvořte větev:

```bash
git status
git switch -c cv7/firebase-auth
```

Pokud větev existuje, použijte `git switch cv7/firebase-auth`.

**Než začnete:** Když aplikaci otevřete na jiném počítači, objeví se tam historie z Preferences? Jakou informaci bude server potřebovat, aby odlišil vaše záznamy od cizích?

## 2. Vlastní Firebase projekt

Ve [Firebase Console](https://console.firebase.google.com/) vytvořte vlastní projekt, například `ap5pm-counter-vase-jmeno`. Google Analytics pro cvičení nepotřebujete.

1. V přehledu projektu přidejte **webovou aplikaci** ikonou `</>`.
2. Pojmenujte ji `CounterApp`; Hosting nyní nezapínejte.
3. Ponechte si otevřený objekt `firebaseConfig`.
4. V **Authentication → Sign-in method** zapněte poskytovatele **Anonymous**. Při prvním vstupu může být potřeba tlačítko **Get started**.

Registrujeme webovou aplikaci i pro budoucí spuštění na Androidu: Ionic používá JavaScriptové SDK ve WebView. Nepotřebujete `google-services.json`.

Anonymní přihlášení vytvoří účet s identifikátorem **UID**. Uživatel nezadává heslo, ale server už dokáže rozpoznat jeho požadavky.

### Připravte databázi pro příště

V **Firestore Database → Create database** vytvořte databázi **Standard**, pokud konzole nabízí výběr edice. Zvolte evropské umístění a **Production mode**. Umístění později nelze snadno změnit.

Výchozí pravidla odmítají přístup aplikace. Dnes je ponechte takto; konkrétní přístup k vlastním záznamům povolíte v CV8. Nepovolujte vše pomocí `if true`.

**Kontrolní bod:** máte vlastní projekt, registrovanou webovou aplikaci, zapnutý Anonymous provider a vytvořený Firestore. Pokud se nastavení nedaří, řešte ho nyní s vyučujícím.

## 3. SDK a prostředí

V kořeni aplikace spusťte:

```bash
npm install firebase@12
npm ls firebase
```

Používáme modulární Firebase JavaScript SDK přímo, bez AngularFire. Konkrétní nainstalovanou verzi zaznamená `package-lock.json`, který uložíte do Gitu.

V `src/environments/environment.ts` zachovejte `production: false` a přidejte vlastnost `firebase`. Hodnoty převezměte z **Project settings → General → Your apps**:

```typescript
export const environment = {
  production: false,
  firebase: {
    apiKey: 'VLASTNI_HODNOTA',
    authDomain: 'VLASTNI_PROJEKT.firebaseapp.com',
    projectId: 'VLASTNI_PROJEKT',
    storageBucket: 'HODNOTA_Z_KONZOLE',
    messagingSenderId: 'HODNOTA_Z_KONZOLE',
    appId: 'HODNOTA_Z_KONZOLE',
  },
};
```

Stejnou vlastnost `firebase` vložte do `environment.prod.ts`; tam ponechte `production: true`. Pokud soubory prostředí dosud nemáte, vytvořte je v adresáři `src/environments`.

Webová konfigurace identifikuje projekt a smí být součástí klientské aplikace. Přístup k datům chrání přihlášení a pravidla databáze. Soukromý klíč servisního účtu do aplikace ani repozitáře nepatří.

**Vlastní vysvětlení:** Proč samotné utajení `projectId` nemůže nahradit pravidla databáze?

## 4. Jeden společný Firebase klient

Vytvořte krátký soubor `src/app/firebase.client.ts`:

```typescript
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';
import { environment } from '../environments/environment';

const app = initializeApp(environment.firebase);

export const firebaseAuth = getAuth(app);
export const firestore = getFirestore(app);
```

`initializeApp()` zavoláme na jediném místě. Ostatní soubory importují připravené instance. `firebaseAuth` slouží k přihlašování, `firestore` použijeme příště.

## 5. Přihlašovací služba po částech

Vygenerujte službu:

```bash
npx ng generate service services/firebase-auth --type=service
```

### 5.1 Kostra a stav

Do `src/app/services/firebase-auth.service.ts` nejprve vložte pouze tuto kostru:

```typescript
import { Injectable, signal } from '@angular/core';
import { signInAnonymously } from 'firebase/auth';
import { firebaseAuth } from '../firebase.client';

@Injectable({ providedIn: 'root' })
export class FirebaseAuthService {
  private readonly userIdState = signal<string | null>(null);
  readonly userId = this.userIdState.asReadonly();
}
```

`null` znamená, že identitu ještě neznáme. Zvenku lze signal číst, měnit ho bude pouze služba.

### 5.2 Obnovení relace a přihlášení

Do třídy přidejte metodu:

```typescript
private async signIn(): Promise<string> {
  await firebaseAuth.authStateReady();

  let user = firebaseAuth.currentUser;
  if (user === null) {
    const result = await signInAnonymously(firebaseAuth);
    user = result.user;
  }

  this.userIdState.set(user.uid);
  return user.uid;
}
```

Postup čtěte shora: nejprve počkáme na obnovení uložené relace. Existující účet použijeme znovu. Nový vytvoříme pouze tehdy, když žádný nemáme. Nakonec zveřejníme UID a také ho vrátíme volajícímu.

**Zastavte se:** Co by se mohlo stát, kdybychom četli `currentUser` ještě před `authStateReady()`? Proč je metoda `async`?

### 5.3 Jedno společné přihlášení

Dnes službu volá jedna stránka. Příště ji bude potřebovat i cloudová historie, proto mají všichni čekat na stejné přihlášení.

Mezi vlastnosti třídy přidejte:

```typescript
private signInPromise: Promise<string> | null = null;
```

A mezi metody přidejte veřejný vstup:

```typescript
initialize(): Promise<string> {
  if (this.signInPromise === null) {
    this.signInPromise = this.signIn();
  }

  return this.signInPromise;
}
```

Ukládáme rozběhnutý `Promise`, nejen výsledné UID. Druhé volání tak počká na první, i když přihlášení ještě neskončilo. Pro výukovou aplikaci při neúspěchu opravíme příčinu a obnovíme stránku; automatické opakování nyní nepřidáváme.

**Úkol ve dvojici:** Na papír napište, co vrátí dvě rychlá volání `initialize()`. Kolikrát se zavolá `signIn()`? Odpověď ověřte dočasným `console.log()` na začátku `signIn()` a poté výpis odstraňte.

## 6. Zobrazení identity vedle lokální historie

V `src/app/tab2/tab2.page.ts` ponechte původní `CounterService` i metody historie. Do existujících importů z `@angular/core` doplňte `signal` a přidejte import:

```typescript
import { FirebaseAuthService } from '../services/firebase-auth.service';
```

Do třídy `Tab2Page` přidejte dvě vlastnosti:

```typescript
readonly auth = inject(FirebaseAuthService);
readonly authError = signal('');
```

Dosavadní `ngOnInit()` **nahraďte** touto verzí. Historie se dál načítá lokálně a přihlášení má vlastní chybovou zprávu:

```typescript
async ngOnInit(): Promise<void> {
  await this.counterService.initialize();

  try {
    await this.auth.initialize();
  } catch (error) {
    console.error(error);
    this.authError.set('Přihlášení se nezdařilo. Zkontrolujte konfiguraci.');
  }
}
```

Do `tab2.page.html` vložte hned pod otevírací `ion-content`:

```html
@if (authError()) {
  <p class="state-message">{{ authError() }}</p>
} @else if (auth.userId(); as uid) {
  <p class="state-message">Anonymní uživatel: {{ uid }}</p>
} @else {
  <p class="state-message">Přihlašuji…</p>
}
```

Do `tab2.page.scss` přidejte, aby dlouhé UID nepřetékalo na telefonu:

```scss
.state-message {
  overflow-wrap: anywhere;
}
```

UID zobrazujeme kvůli pokusům v tomto a příštím cvičení.

## 7. Pokus: stejná aplikace, různé identity

Spusťte `ionic serve` a otevřete Historii.

**Nejprve odhadněte**, kdy se UID změní. Pak doplňte skutečný výsledek:

| Pokus | Odhad: stejné / jiné UID | Skutečnost |
| --- | --- | --- |
| Obnovení stránky | | |
| Druhá karta stejné adresy ve stejném profilu | | |
| Soukromé okno | | |

Ve **Firebase Console → Authentication → Users** najděte UID z aplikace. Soukromé okno má oddělené úložiště, a tedy vlastní účet. Pro tento pokus používejte stále stejnou adresu aplikace.

**Důležité:** Přihlášení samo nepřenáší historii z Preferences do cloudu. Smazání dat webu nebo odinstalování aplikace může odstranit anonymní relaci; nový účet už nebude mít přístup k datům původního UID.

## 8. Dokončení

```bash
npm run lint
ionic build
git diff --check
git add package.json package-lock.json src
git commit -m "CV7: pripojeni Firebase a anonymni identita"
```

Před commitem zkontrolujte vlastní `git diff`.

- [ ] Lokální historie stále funguje.
- [ ] UID odpovídá účtu ve Firebase Console.
- [ ] Obnovení stránky zachová UID; soukromé okno má jiné.
- [ ] Umím vysvětlit rozdíl mezi konfigurací projektu a oprávněním k datům.
- [ ] Umím vysvětlit, proč sdílíme `signInPromise`.
- [ ] Lint a sestavení prošly.

**Pokud zbývá čas:** V Network najděte požadavek přihlášení. Zkuste vypnout Anonymous provider, obnovit aplikaci v novém soukromém okně a přečíst chybu. Provider potom znovu zapněte.

## Když se nedaří

- **`auth/configuration-not-found` / `auth/operation-not-allowed`:** zkontrolujte Anonymous provider a `projectId`.
- **Chybný API klíč:** znovu porovnejte konfiguraci s vlastní webovou aplikací v Console.
- **Jiné UID po každém spuštění:** ověřte, že nemažete data webu, neměníte adresu a neotevíráte pokaždé nové soukromé okno.
- **Duplicitní inicializace aplikace:** `initializeApp()` má být pouze ve `firebase.client.ts`, nikoli ve stránkách nebo službách.

## K dalšímu čtení

- [Připojení Firebase k webové aplikaci](https://firebase.google.com/docs/web/setup)
- [Anonymní přihlášení](https://firebase.google.com/docs/auth/web/anonymous-auth)

Pokračujte v [CV8 – cloudová historie](../cv08/README.md).
