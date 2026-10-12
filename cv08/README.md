# CV8 – CounterApp v cloudu pomocí Firebase

## Cíl cvičení

V CV4 jste ukládali historii CounterApp lokálně pomocí Capacitor Preferences. V CV8 se vrátíte ke CounterApp a vytvoříte druhou variantu služby, která ukládá záznamy do databáze **Cloud Firestore**. Každá instalace se nejprve anonymně přihlásí pomocí **Firebase Authentication** a bezpečnostní pravidla dovolí uživateli pracovat pouze s jeho vlastními daty.

Po dokončení cvičení budete umět:

- vytvořit a nakonfigurovat Firebase projekt,
- registrovat webovou aplikaci a připojit ji pomocí modulárního Firebase SDK,
- vysvětlit rozdíl mezi Firebase konfigurací a tajným přístupovým údajem,
- anonymně přihlásit uživatele,
- zapsat, načíst a odstranit dokumenty v Cloud Firestore,
- poslouchat změny databáze v reálném čase pomocí `onSnapshot()`,
- oddělit data uživatelů podle jejich `uid`,
- vytvořit bezpečnostní pravidla Firestore a ověřit jejich účinek,
- zobrazit stav připojování, prázdná data a chybu,
- ověřit cloudovou synchronizaci ve webové a Android aplikaci.

CV7 PhotoGallery je samostatný projekt a zůstává beze změny. V tomto cvičení pokračujete v projektu `counter-app` dokončeném v CV6.

> **Proč nepoužíváme AngularFire:** Aktuální AngularFire 20 deklaruje podporu Angularu 20, zatímco kurz používá Angular 22. Firebase JavaScript SDK je oficiální, modulární a nezávislé na verzi Angularu. Po vydání kompatibilního AngularFire lze stejnou architekturu převést na jeho Angular providery.

## 1. Výchozí stav

Navazujete na CounterApp po CV6. Musí fungovat:

- ruční počítadlo a model `SavedCounter`,
- historie uložená lokálně pomocí `CounterService` a Preferences,
- odpočet do svátku z CV5,
- Android projekt a emulátor z CV6,
- příkazy `npm run lint`, `ionic build` a `npx cap sync android`.

Model z CV4 má tuto podobu:

```typescript
export interface SavedCounter {
  id: string;
  name: string;
  value: number;
  createdAt: string;
}
```

Původní `CounterService` nemažte. Novou cloudovou službu vytvoříte vedle ní, abyste mohli porovnat lokální a vzdálenou perzistenci.

## 2. Otevření projektu a vytvoření větve

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty\counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv8/firebase
code .
```

### macOS a Linux – Terminal

```bash
cd "$HOME/AP5PM-projekty/counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv8/firebase
code .
```

Pokud větev již existuje, použijte `git switch cv8/firebase`. Před zahájením má být pracovní strom čistý.

## 3. Vytvoření Firebase projektu

Otevřete [Firebase Console](https://console.firebase.google.com/) a zvolte **Create a project**.

1. Projekt pojmenujte například `ap5pm-counter-vase-jmeno`.
2. ID projektu musí být globálně jedinečné; navržené ID můžete upravit pouze při vytváření.
3. Google Analytics pro toto cvičení nepotřebujete a můžete je vypnout.
4. Počkejte na vytvoření projektu a otevřete jeho přehled.

Nepoužívejte společný projekt jiné osoby. Každý student má mít vlastní databázi, pravidla i kvóty.

### Registrace webové aplikace

V přehledu projektu klikněte na ikonu **Web** `</>`:

1. Přezdívku aplikace nastavte například na `CounterApp Web`.
2. Firebase Hosting nyní nezapínejte.
3. Dokončete registraci tlačítkem **Register app**.
4. Zobrazený objekt `firebaseConfig` si ponechte otevřený; použijete jej v další části.

Registrujete webovou aplikaci i pro Android build CounterApp, protože Ionic běží v Android WebView a používá JavaScriptové Firebase SDK. V tomto cvičení proto nepřidáváte soubor `google-services.json` ani nativní Firebase Android SDK.

## 4. Zapnutí anonymního přihlašování

Ve Firebase Console otevřete **Build → Authentication → Get started**.

1. Na kartě **Sign-in method** vyberte **Anonymous**.
2. Přepínač nastavte na **Enable**.
3. Změnu uložte.

Anonymní účet má stabilní `uid` uložené v daném profilu prohlížeče nebo v datech dané instalace aplikace. Uživatel nezadává jméno ani heslo. Vymazání dat aplikace, anonymního účtu nebo úložiště prohlížeče může vytvořit nové `uid`; původní data pak nový účet neuvidí.

## 5. Vytvoření Cloud Firestore

Otevřete **Build → Firestore Database → Create database**.

1. Zvolte databázi v **Production mode**.
2. Vyberte evropské umístění nejblíže uživatelům, například `eur3`, pokud je v nabídce.
3. Volbu umístění pečlivě zkontrolujte; po vytvoření ji nelze jednoduše změnit.
4. Dokončete vytvoření databáze.

Production mode na začátku odmítne všechny požadavky. Přístup povolíte až přesnými pravidly v části 8. Nepoužívejte otevřené testovací pravidlo `allow read, write: if true`.

## 6. Instalace Firebase SDK

V kořeni CounterApp nainstalujte ověřenou verzi modulárního SDK:

```bash
npm install firebase@12.19.0
```

Ověřte instalaci:

```bash
npm ls firebase
```

Nepotřebujete `firebase-tools` ani globální Firebase CLI. Databázi a pravidla v tomto cvičení nastavíte ve Firebase Console.

## 7. Konfigurace prostředí

Do `src/environments/environment.ts` doplňte konfiguraci z Firebase Console:

```typescript
export const environment = {
  production: false,
  firebase: {
    apiKey: 'DOPLNTE_VLASTNI_HODNOTU',
    authDomain: 'DOPLNTE.firebaseapp.com',
    projectId: 'DOPLNTE',
    storageBucket: 'DOPLNTE.firebasestorage.app',
    messagingSenderId: 'DOPLNTE',
    appId: 'DOPLNTE',
  },
};
```

Stejnou vlastnost `firebase` doplňte také do `src/environments/environment.prod.ts`:

```typescript
export const environment = {
  production: true,
  firebase: {
    apiKey: 'DOPLNTE_VLASTNI_HODNOTU',
    authDomain: 'DOPLNTE.firebaseapp.com',
    projectId: 'DOPLNTE',
    storageBucket: 'DOPLNTE.firebasestorage.app',
    messagingSenderId: 'DOPLNTE',
    appId: 'DOPLNTE',
  },
};
```

Hodnoty nekopírujte z ukázky ani od spolužáka. Vložte celý vlastní objekt z **Project settings → General → Your apps → SDK setup and configuration**.

Firebase webová konfigurace identifikuje projekt, ale není administrátorským tajemstvím. Bezpečnost dat musí zajišťovat Authentication a Firestore Security Rules. Do repozitáře naopak nikdy nepatří soukromý klíč servisního účtu, soubor s admin credentials ani heslo uživatele.

## 8. Bezpečnostní pravidla Firestore

Data uložíme do této struktury:

```text
users/{uid}/counters/{counterId}
```

Ve Firebase Console otevřete **Firestore Database → Rules**, nahraďte výchozí obsah a klikněte na **Publish**:

```text
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    function ownsUserData(userId) {
      return request.auth != null && request.auth.uid == userId;
    }

    function validCounter(counterId) {
      return request.resource.data.keys().hasOnly([
          'id', 'name', 'value', 'createdAt'
        ])
        && request.resource.data.id == counterId
        && request.resource.data.name is string
        && request.resource.data.name.size() > 0
        && request.resource.data.name.size() <= 60
        && request.resource.data.value is int
        && request.resource.data.createdAt is string;
    }

    match /users/{userId}/counters/{counterId} {
      allow read: if ownsUserData(userId);
      allow create, update: if ownsUserData(userId)
                            && validCounter(counterId);
      allow delete: if ownsUserData(userId);
    }
  }
}
```

Pravidla zajišťují:

- nepřihlášený klient nemůže číst ani zapisovat,
- přihlášený klient může pracovat pouze pod vlastním `uid`,
- ID dokumentu musí odpovídat poli `id`,
- dokument nesmí obsahovat neočekávaná pole,
- název má omezenou délku a hodnota musí být celé číslo.

Pravidla nejsou filtr. Aplikace musí dotazovat přímo kolekci aktuálního uživatele; nelze načíst všechny uživatele a očekávat, že Firestore nepovolené dokumenty dodatečně odfiltruje.

## 9. Inicializace Firebase klienta

Vytvořte `src/app/firebase.client.ts`:

```typescript
import { getApp, getApps, initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';
import { environment } from '../environments/environment';

const firebaseApp = getApps().length === 0
  ? initializeApp(environment.firebase)
  : getApp();

export const firebaseAuth = getAuth(firebaseApp);
export const firestore = getFirestore(firebaseApp);
```

Kontrola `getApps().length` brání opakované inicializaci při testech nebo vývojovém obnovování modulů. Ostatní části aplikace importují pouze připravené instance `firebaseAuth` a `firestore`.

## 10. Služba pro anonymní autentizaci

Vygenerujte službu:

```bash
npx ng generate service services/firebase-auth --type=service
```

Nahraďte obsah `src/app/services/firebase-auth.service.ts`:

```typescript
import { Injectable, signal } from '@angular/core';
import { signInAnonymously } from 'firebase/auth';
import { firebaseAuth } from '../firebase.client';

@Injectable({
  providedIn: 'root',
})
export class FirebaseAuthService {
  private readonly userIdState = signal<string | null>(null);
  private readonly readyState = signal(false);
  private initializationPromise: Promise<string> | null = null;

  readonly userId = this.userIdState.asReadonly();
  readonly ready = this.readyState.asReadonly();

  initialize(): Promise<string> {
    this.initializationPromise ??= this.signIn();
    return this.initializationPromise;
  }

  private async signIn(): Promise<string> {
    try {
      await firebaseAuth.authStateReady();

      const user = firebaseAuth.currentUser ??
        (await signInAnonymously(firebaseAuth)).user;

      this.userIdState.set(user.uid);
      return user.uid;
    } finally {
      this.readyState.set(true);
    }
  }
}
```

`authStateReady()` počká na načtení dříve uložené relace. Pokud účet existuje, služba jej znovu použije. Jinak `signInAnonymously()` vytvoří nový anonymní účet. `initializationPromise` zajišťuje, že souběžné stránky nespustí více přihlášení.

## 11. Vygenerování cloudové služby

Vytvořte druhou službu pro historii:

```bash
npx ng generate service services/firebase-counter --type=service
```

Lokální `counter.service.ts` ponechte v projektu. V dalších částech pouze přepojíte stránky na novou `FirebaseCounterService`.

## 12. Implementace FirebaseCounterService

Nahraďte obsah `src/app/services/firebase-counter.service.ts`:

```typescript
import { Injectable, inject, signal } from '@angular/core';
import {
  collection,
  deleteDoc,
  doc,
  onSnapshot,
  orderBy,
  query,
  setDoc,
  writeBatch,
} from 'firebase/firestore';
import type { CollectionReference, DocumentData } from 'firebase/firestore';
import { firestore } from '../firebase.client';
import { SavedCounter } from '../models/saved-counter';
import { FirebaseAuthService } from './firebase-auth.service';

function isSavedCounter(value: unknown): value is SavedCounter {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'string' &&
    'name' in value &&
    typeof value.name === 'string' &&
    'value' in value &&
    typeof value.value === 'number' &&
    Number.isInteger(value.value) &&
    'createdAt' in value &&
    typeof value.createdAt === 'string'
  );
}

@Injectable({
  providedIn: 'root',
})
export class FirebaseCounterService {
  private readonly auth = inject(FirebaseAuthService);
  private readonly countersState = signal<SavedCounter[]>([]);
  private readonly initializedState = signal(false);
  private readonly errorState = signal('');
  private initializationPromise: Promise<void> | null = null;

  readonly counters = this.countersState.asReadonly();
  readonly initialized = this.initializedState.asReadonly();
  readonly errorMessage = this.errorState.asReadonly();
  readonly userId = this.auth.userId;

  initialize(): Promise<void> {
    this.initializationPromise ??= this.connect();
    return this.initializationPromise;
  }

  async add(counter: SavedCounter): Promise<void> {
    await this.initialize();

    try {
      await setDoc(doc(this.userCounters(), counter.id), counter);
      this.errorState.set('');
    } catch (error) {
      this.reportError('Záznam se nepodařilo uložit.', error);
    }
  }

  async remove(id: string): Promise<void> {
    await this.initialize();

    try {
      await deleteDoc(doc(this.userCounters(), id));
      this.errorState.set('');
    } catch (error) {
      this.reportError('Záznam se nepodařilo odstranit.', error);
    }
  }

  async clear(): Promise<void> {
    await this.initialize();

    try {
      const batch = writeBatch(firestore);

      for (const counter of this.countersState()) {
        batch.delete(doc(this.userCounters(), counter.id));
      }

      await batch.commit();
      this.errorState.set('');
    } catch (error) {
      this.reportError('Historii se nepodařilo vymazat.', error);
    }
  }

  private async connect(): Promise<void> {
    try {
      await this.auth.initialize();
      const countersQuery = query(
        this.userCounters(),
        orderBy('createdAt', 'desc'),
      );

      await new Promise<void>((resolve) => {
        onSnapshot(
          countersQuery,
          (snapshot) => {
            const counters = snapshot.docs
              .map((item) => item.data())
              .filter(isSavedCounter);

            this.countersState.set(counters);
            this.errorState.set('');
            this.initializedState.set(true);
            resolve();
          },
          (error) => {
            this.reportError('Cloudovou historii se nepodařilo načíst.', error);
            this.initializedState.set(true);
            resolve();
          },
        );
      });
    } catch (error) {
      this.reportError('Přihlášení k Firebase se nezdařilo.', error);
      this.initializedState.set(true);
    }
  }

  private userCounters(): CollectionReference<DocumentData> {
    const userId = this.auth.userId();

    if (userId === null) {
      throw new Error('Firebase uživatel není přihlášený.');
    }

    return collection(firestore, 'users', userId, 'counters');
  }

  private reportError(message: string, error: unknown): void {
    console.error(message, error);
    this.errorState.set(message);
  }
}
```

### Co služba dělá

| Část | Význam |
| --- | --- |
| `auth.initialize()` | získá stabilní anonymní `uid` |
| `collection(..., uid, ...)` | omezuje dotaz na data aktuálního uživatele |
| `onSnapshot()` | pošle první stav a potom každou další změnu v reálném čase |
| `orderBy('createdAt', 'desc')` | řadí ISO časy od nejnovějších |
| `setDoc()` | uloží záznam pod jeho existujícím UUID |
| `deleteDoc()` | odstraní jeden dokument |
| `writeBatch()` | odstraní více dokumentů jedním atomickým požadavkem |
| `isSavedCounter()` | ověří data přijatá ze vzdálené databáze za běhu |

`onSnapshot()` vrací funkci pro odhlášení posluchače. Služba je singleton po celou dobu běhu aplikace, proto posluchač zůstává aktivní. U služby s kratší životností byste jej museli odhlásit při zničení.

Batch ve Firestore má omezený počet operací. Pro malé cvičné seznamy je jedno volání dostačující; produkční aplikace s velkou kolekcí by mazala po stránkách nebo použila serverovou funkci.

## 13. Přepojení první záložky

V `src/app/tab1/tab1.page.ts` změňte import a injektovanou službu. Výsledný soubor:

```typescript
import { Component, inject } from '@angular/core';
import { IonContent, IonHeader, IonTitle, IonToolbar } from '@ionic/angular';
import { CounterComponent } from '../components/counter/counter.component';
import { SavedCounter } from '../models/saved-counter';
import { FirebaseCounterService } from '../services/firebase-counter.service';

@Component({
  selector: 'app-tab1',
  templateUrl: 'tab1.page.html',
  styleUrls: ['tab1.page.scss'],
  imports: [CounterComponent, IonContent, IonHeader, IonTitle, IonToolbar],
})
export class Tab1Page {
  private readonly counterService = inject(FirebaseCounterService);

  async onSaved(counter: SavedCounter): Promise<void> {
    await this.counterService.add(counter);
  }
}
```

HTML první záložky není třeba měnit. `CounterComponent` stále vytváří stejný objekt; neví, zda jej rodič uloží lokálně nebo do cloudu.

## 14. Přepojení záložky Historie

V `src/app/tab2/tab2.page.ts` nahraďte import `CounterService` importem `FirebaseCounterService` a upravte typ injekce:

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
  IonText,
  IonTitle,
  IonToolbar,
} from '@ionic/angular';
import { FirebaseCounterService } from '../services/firebase-counter.service';

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
    IonText,
    IonTitle,
    IonToolbar,
  ],
})
export class Tab2Page implements OnInit {
  readonly counterService = inject(FirebaseCounterService);

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

Do `src/app/tab2/tab2.page.html` vložte hned pod otevírací značku `ion-content` zobrazení chyby a anonymního ID:

```html
@if (counterService.errorMessage()) {
  <ion-text color="danger">
    <p class="state-message">{{ counterService.errorMessage() }}</p>
  </ion-text>
}

@if (counterService.userId(); as userId) {
  <ion-note class="user-id">Anonymní uživatel: {{ userId }}</ion-note>
}
```

Zbytek šablony historie z CV4 ponechte. Stavy `initialized()` a `counters()` mají stejné rozhraní jako původní služba.

Do `src/app/tab2/tab2.page.scss` doplňte:

```scss
.user-id {
  display: block;
  max-width: 40rem;
  margin: 1rem auto;
  overflow-wrap: anywhere;
  font-family: monospace;
  text-align: center;
}
```

UID zobrazujeme pro výuku a ověřování izolace. V běžné aplikaci by technický identifikátor uživatele nemusel být součástí hlavního UI.

## 15. První spuštění a kontrola Console

Spusťte:

```bash
ionic serve
```

Na záložce Historie počkejte na zobrazení anonymního UID. Potom:

1. vytvořte tři záznamy,
2. ověřte jejich pořadí od nejnovějšího,
3. obnovte stránku,
4. ověřte, že UID i záznamy zůstaly stejné,
5. odstraňte jeden záznam,
6. zkontrolujte, že zmizel bez ručního obnovení.

Ve Firebase Console otevřete:

- **Authentication → Users** a najděte anonymní účet se stejným UID,
- **Firestore Database → Data** a projděte cestu `users/{uid}/counters`,
- jeden dokument a jeho pole `id`, `name`, `value`, `createdAt`.

Dokumenty v Console ručně neupravujte během běžného scénáře. Pokud změnu provedete, otevřená aplikace ji díky `onSnapshot()` obdrží v reálném čase.

## 16. Ověření synchronizace v reálném čase

Otevřete CounterApp ve dvou běžných kartách stejného profilu prohlížeče.

1. V obou kartách otevřete Historii.
2. Ověřte, že zobrazují stejné UID.
3. V první kartě přidejte záznam.
4. Záznam se má bez obnovení zobrazit také ve druhé kartě.
5. Ve druhé kartě záznam odstraňte.
6. Odstranění se má projevit v první kartě.

Obě karty sdílejí stejnou uloženou relaci Firebase Auth. `onSnapshot()` udržuje posluchač nad dotazem a aktualizuje Angular signal při každém novém snapshotu.

## 17. Ověření oddělení uživatelů

Otevřete aplikaci v anonymním nebo soukromém okně prohlížeče.

1. Historie má být prázdná.
2. Zobrazené UID se musí lišit od běžného okna.
3. Vytvořte jeden záznam v anonymním okně.
4. Tento záznam se nesmí objevit v běžném okně.
5. Ve Firebase Console najděte dvě oddělené cesty pod dvěma UID.

Tím ověřujete kombinaci tří opatření:

- klient zapisuje do cesty obsahující aktuální UID,
- dotaz načítá pouze tuto cestu,
- bezpečnostní pravidla odmítnou přístup k cizímu UID.

Pouhé filtrování v Angularu by nebylo zabezpečení. Nedůvěryhodný klient lze změnit; rozhodující kontrola musí probíhat na serveru ve Firestore Rules.

## 18. Negativní test bezpečnostních pravidel

Dočasně v `firebase-counter.service.ts` změňte pouze pro tento test metodu `userCounters()` tak, aby místo vlastního UID použila UID z druhého okna:

```typescript
return collection(firestore, 'users', 'CIZI_UID', 'counters');
```

Po obnovení aplikace musí Firestore požadavek odmítnout chybou `permission-denied` a UI zobrazit chybu načtení. V konzoli nesmí být vypsána cizí data.

Ihned potom vraťte původní bezpečnou implementaci:

```typescript
return collection(firestore, 'users', userId, 'counters');
```

Před commitem zkontrolujte `git diff`, aby v kódu nezůstalo cizí UID ani testovací změna.

## 19. Rozdíl mezi lokální a cloudovou službou

| Vlastnost | Preferences | Cloud Firestore |
| --- | --- | --- |
| Umístění dat | konkrétní prohlížeč nebo instalace | vzdálený Firebase projekt |
| Přihlášení | není potřeba | anonymní Firebase účet |
| Reálný čas | pouze vlastní signal | `onSnapshot()` mezi klienty stejného účtu |
| Bezpečnost | izolace úložiště zařízení | Authentication + Security Rules |
| Provoz bez sítě | lokální data ihned dostupná | SDK může ukázat lokální změnu, potvrzení zápisu ale čeká na síť |
| Náklady a kvóty | úložiště zařízení | kvóty čtení, zápisů a uložených dat |
| Odinstalování | data se smažou | cloudová data zůstanou, ale nové anonymní UID k nim nemá přístup |

Cloud není automaticky lepší pro každý typ dat. Preferences jsou vhodné pro malé lokální nastavení. Firestore dává smysl pro synchronizaci, více klientů a data spravovaná serverem.

## 20. Spuštění na Androidu

Vytvořte aktuální webový build a synchronizujte Android projekt:

```bash
npm run lint
ionic build
npx cap sync android
npx cap run android
```

V emulátoru proveďte:

1. otevřete Historii a počkejte na anonymní UID,
2. vytvořte alespoň dva záznamy,
3. ukončete aplikaci a znovu ji spusťte,
4. ověřte stejné UID a zachovaná data,
5. zapněte Airplane mode a zkuste přidat další záznam,
6. ověřte, že aplikace nespadne; lokální posluchač může změnu zobrazit, ale Promise zápisu čeká na potvrzení serveru,
7. síť obnovte, počkejte na synchronizaci a aplikaci spusťte znovu.

Android WebView má vlastní úložiště a vytvoří jiné anonymní UID než desktopový prohlížeč. Proto neuvidí jeho data. Přenos identity mezi zařízeními vyžaduje skutečný přihlašovací mechanismus, kterému se budete věnovat v CV9.

Firebase používá veřejné HTTPS adresy. Není potřeba měnit `localhost`, Android Network Security Config ani povolovat nezabezpečené HTTP.

Firestore používá takzvanou kompenzaci latence: lokální snapshot může obsahovat dosud nepotvrzený zápis. `setDoc()` se dokončí až po potvrzení backendem, takže při vypnuté síti může jeho Promise zůstat čekat. Produkční UI by stav neodeslaných změn rozlišovalo pomocí metadat snapshotu `hasPendingWrites`.

## 21. Kontrola síťových požadavků

Při běhu v prohlížeči otevřete Chrome DevTools:

1. Na kartě **Network** filtrujte `firestore` nebo `googleapis`.
2. Vytvořte a odstraňte záznam.
3. Sledujte aktivní spojení a požadavky Firebase SDK.
4. Na kartě **Console** nesmí být při normálním použití `permission-denied` ani chyba konfigurace.

Ve Firebase Console na kartě **Usage** lze sledovat počet čtení, zápisů a odstranění. Posluchač snapshotů spotřebovává čtení při prvním načtení i při změnách výsledků. Nevytvářejte zbytečně nové posluchače při každém vstupu na záložku.

## 22. Produkční kontrola a Git

Spusťte:

```bash
npm run lint
ionic build
npx cap sync android
npx cap doctor android
git status
git diff --check
```

Pokud máte nastavený Prettier, spusťte také:

```bash
npm run format
npm run format:check
```

Před commitem ověřte:

- oba soubory `environment` obsahují konfiguraci vašeho Firebase projektu,
- v repozitáři není servisní účet, soukromý klíč ani heslo,
- Firestore Rules nejsou otevřené pomocí `if true`,
- negativní test s cizím UID je vrácený,
- aplikace používá cestu `users/{uid}/counters`,
- původní lokální `CounterService` zůstal zachovaný,
- `package.json` a `package-lock.json` obsahují Firebase SDK 12.19.0,
- `capacitor.config.ts` neobsahuje `server.url`.

Uložte práci:

```bash
git add package.json package-lock.json src
git commit -m "Přidána cloudová historie pomocí Firebase"
git status
```

Pravidla vytvořená pouze ve Firebase Console nejsou součástí tohoto commitu. V bonusovém úkolu je můžete přesunout do `firestore.rules` a spravovat verzovaně pomocí Firebase CLI.

## 23. Kontrolní seznam CV8

- [ ] Firebase projekt patří studentovi a obsahuje registrovanou webovou aplikaci.
- [ ] Je zapnutý Anonymous provider ve Firebase Authentication.
- [ ] Cloud Firestore je vytvořený v Production mode.
- [ ] Bezpečnostní pravidla oddělují data podle `request.auth.uid`.
- [ ] Projekt používá modulární Firebase SDK 12.19.0.
- [ ] Konfigurace je v obou souborech prostředí.
- [ ] V repozitáři není soukromý klíč ani servisní účet.
- [ ] Aplikace používá `signInAnonymously()` a zobrazuje UID.
- [ ] Historie se načítá pomocí jednoho posluchače `onSnapshot()`.
- [ ] Přidání, odstranění a vymazání historie funguje.
- [ ] Dvě karty se stejným UID se synchronizují v reálném čase.
- [ ] Dva různí anonymní uživatelé navzájem nevidí svá data.
- [ ] Negativní test cizího UID skončil chybou `permission-denied`.
- [ ] Aplikace funguje v prohlížeči i na Androidu.
- [ ] Chyba sítě ani Firebase nezpůsobí pád aplikace.
- [ ] `npm run lint` a `ionic build` proběhnou bez chyby.
- [ ] Projekt je uložený v Git commitu a pracovní strom je čistý.

## 24. Bonusové úkoly

Po dokončení povinné části můžete:

1. přidat přepínač mezi lokálním `CounterService` a `FirebaseCounterService`,
2. zobrazit stav online/offline pomocí Capacitor Network,
3. zapnout a otestovat lokální Firestore cache pro web,
4. omezit dotaz pomocí `limit()` na posledních 50 záznamů,
5. přidat editaci názvu dokumentu pomocí `updateDoc()`,
6. nahradit ISO řetězec serverovým časem `serverTimestamp()`,
7. otestovat pravidla pomocí Firebase Emulator Suite,
8. uložit pravidla do souboru `firestore.rules` a nasazovat je Firebase CLI,
9. zapnout Firebase App Check a ověřit požadavky aplikace,
10. propojit anonymní účet s trvalým účtem bez ztráty dat.

Nezapínejte placený tarif ani službu vyžadující fakturační účet pouze kvůli bonusu. Sledujte kvóty vlastního projektu.

## 25. Nejčastější problémy

### `auth/configuration-not-found`

Ve Firebase Console není zapnutý Anonymous provider nebo aplikace používá konfiguraci jiného projektu. Zkontrolujte **Authentication → Sign-in method** a `projectId` v prostředí.

### `Missing or insufficient permissions`

Firestore Rules požadavek odmítly. Ověřte:

- že jste pravidla publikovali,
- že `request.auth` není `null`,
- že cesta obsahuje stejné UID jako přihlášený uživatel,
- že dokument obsahuje pouze povolená pole správných typů,
- že pole `id` odpovídá ID dokumentu.

Neřešte chybu otevřením pravidel pomocí `allow read, write: if true`.

### `Firebase App named '[DEFAULT]' already exists`

Firebase byl inicializován vícekrát. Použijte připravenou kontrolu `getApps().length === 0 ? initializeApp(...) : getApp()` a inicializaci ponechte pouze v `firebase.client.ts`.

### Historie se načítá bez konce

Zkontrolujte Network a Console. `onSnapshot()` musí v úspěšné i chybové callback větvi nastavit `initializedState` a dokončit Promise. Častou příčinou je nepublikované pravidlo nebo chybný `projectId`.

### Po obnovení vzniklo jiné UID

Firebase Auth musí mít povolené úložiště webu. Zkontrolujte, zda prohlížeč neblokuje webová data, nepoužíváte pokaždé nové anonymní okno a ručně nemažete úložiště webu. Android ztratí UID po vymazání dat aplikace nebo odinstalování.

### Desktop a Android zobrazují jiná data

To je u anonymního přihlášení očekávané: každá instalace má vlastní účet a UID. Sdílení dat mezi zařízeními vyžaduje přihlášení stejného uživatele trvalou metodou.

### Změna v druhé kartě se neprojeví

Ověřte, že obě karty zobrazují stejné UID a služba používá `onSnapshot()`, nikoliv jednorázové `getDocs()`. Anonymní okno má jiné úložiště a jiný účet.

### `The query requires an index`

Dotaz na jednu kolekci s jediným `orderBy('createdAt')` běžně používá automatický index. Pokud jste přidali další filtry, Firestore může vyžadovat složený index. Použijte odkaz z chybové zprávy pouze tehdy, když rozumíte změněnému dotazu.

### Aplikace funguje přes `ionic serve`, ale ne na Androidu

Spusťte:

```bash
ionic build
npx cap sync android
```

Potom zkontrolujte Logcat a WebView DevTools. Ověřte, že produkční `environment.prod.ts` obsahuje stejnou Firebase konfiguraci jako vývojové prostředí.

### Záznam se po kliknutí objeví dvakrát

Nepřidávejte záznam ručně do signalu po `setDoc()`. Jediným zdrojem cloudového stavu je `onSnapshot()`, který po úspěšném zápisu pošle aktualizovaný seznam.

## Oficiální dokumentace

- [Firebase – přidání SDK do webové aplikace](https://firebase.google.com/docs/web/setup)
- [Firebase Authentication – anonymní přihlášení](https://firebase.google.com/docs/auth/web/anonymous-auth)
- [Cloud Firestore – přidávání dat](https://firebase.google.com/docs/firestore/manage-data/add-data)
- [Cloud Firestore – posluchače v reálném čase](https://firebase.google.com/docs/firestore/query-data/listen)
- [Cloud Firestore – řazení a omezení dotazu](https://firebase.google.com/docs/firestore/query-data/order-limit-data)
- [Cloud Firestore Security Rules – podmínky](https://firebase.google.com/docs/firestore/security/rules-conditions)
- [Cloud Firestore Security Rules – bezpečné dotazy](https://firebase.google.com/docs/firestore/security/rules-query)
- [Firebase – kvóty Firestore](https://firebase.google.com/docs/firestore/quotas)
