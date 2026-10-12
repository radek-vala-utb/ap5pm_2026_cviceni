# CV7 – PhotoGallery a nativní pluginy

## Cíl cvičení

V CV6 jste poprvé sestavili CounterApp pro Android. Nyní vytvoříte samostatnou aplikaci **PhotoGallery**, která pořídí fotografii fotoaparátem, uloží obrazová data do soukromého úložiště aplikace a po novém spuštění znovu sestaví galerii.

Po dokončení cvičení budete umět:

- vytvořit nový Ionic projekt se standalone komponentami,
- rozlišit webové API, Capacitor plugin a nativní implementaci pluginu,
- použít nové API `Camera.takePhoto()` z Capacitoru 8,
- uložit binární obsah fotografie pomocí `Filesystem`,
- uložit seznam metadat pomocí `Preferences`,
- načíst soubory po novém spuštění aplikace,
- odstranit fotografii i její metadata,
- vysvětlit rozdíly mezi během v prohlížeči a v Android WebView,
- ošetřit načítání a chybu nativní operace,
- sestavit, nainstalovat a najít debug APK.

Vytváříte nový projekt vedle CounterApp. CounterApp nemažte ani do něj PhotoGallery nevkládejte.

> **Poznámka k verzi API:** Starší návody používají `Camera.getPhoto()` a `CameraResultType`. Od Capacitoru 8.1 jsou tato rozhraní zastaralá. V tomto cvičení používáme nové `Camera.takePhoto()`, které vrací `MediaResult`.

## 1. Výchozí stav

Před zahájením musí fungovat prostředí z CV6:

- Node.js 26 a Ionic CLI,
- Android Studio a Android SDK Platform 36,
- alespoň jeden spustitelný emulátor API 36 nebo fyzický telefon,
- Git,
- příkazy `node --version`, `npm --version`, `ionic --version` a `git --version`.

V CV7 použijete Ionic 9, Angular 22 a Capacitor 8.5. Balíčky `core`, `cli` a `android` musí mít stejnou verzi. Jednotlivé pluginy mají vlastní číslování vydání, ale všechny musí patřit do kompatibilní hlavní řady 8.

## 2. Vytvoření projektu

Přejděte do adresáře, ve kterém máte své projekty, a vytvořte aplikaci ze šablony `tabs`.

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty"
nvm use 26
ionic start photo-gallery tabs --type=angular --capacitor
Set-Location photo-gallery
```

### macOS a Linux – Terminal

```bash
cd "$HOME/AP5PM-projekty"
nvm use 26
ionic start photo-gallery tabs --type=angular --capacitor
cd photo-gallery
```

V průvodci zvolte:

- framework **Angular**,
- způsob tvorby komponent **Standalone**,
- integraci s účtem Ionic odmítněte, pokud ji nepoužíváte.

Pokud Node.js nepoužíváte přes `nvm`, příkaz `nvm use 26` vynechte.

Spusťte výchozí projekt:

```bash
ionic serve
```

Ověřte, že fungují všechny tři záložky. Potom server ukončete pomocí `Ctrl+C`.

## 3. Git repozitář a první commit

Zkontrolujte, zda Ionic CLI vytvořilo Git repozitář:

```bash
git status
```

Pokud příkaz oznámí, že nejste v repozitáři, spusťte:

```bash
git init
git add .
git commit -m "Vytvořena aplikace PhotoGallery"
```

Pokud repozitář existuje, zkontrolujte jeho historii a případné výchozí změny uložte:

```bash
git log --oneline -3
git status
```

Pro práci v tomto cvičení vytvořte větev:

```bash
git switch -c cv7/photo-gallery
```

## 4. Nastavení Capacitoru

Otevřete `capacitor.config.ts` a nastavte jednoznačné ID a název aplikace:

```typescript
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'cz.utb.fai.ap5pm.photogallery',
  appName: 'PhotoGallery',
  webDir: 'www',
};

export default config;
```

`appId` můžete doplnit o vlastní identifikátor, ale po vytvoření Android projektu jej již neměňte. Nepřidávejte trvalou vlastnost `server.url`.

## 5. Instalace pluginů

PhotoGallery používá tři pluginy:

| Balíček | Úloha |
| --- | --- |
| `@capacitor/camera` | otevře fotoaparát a vrátí dočasný výsledek fotografie |
| `@capacitor/filesystem` | uloží a načte obrazová data v soukromém prostoru aplikace |
| `@capacitor/preferences` | uloží malý JSON seznam s názvy souborů a časy pořízení |

Nainstalujte verze odpovídající Capacitoru 8.5:

```bash
npm install @capacitor/core@8.5.0
npm install @capacitor/camera@8.2.4 @capacitor/filesystem@8.1.3 @capacitor/preferences@8.0.1
npm install @capacitor/android@8.5.0 @ionic/pwa-elements
npm install --save-dev @capacitor/cli@8.5.0
```

Ověřte verze:

```bash
npm ls @capacitor/core @capacitor/cli @capacitor/android
npm ls @capacitor/camera @capacitor/filesystem @capacitor/preferences
```

`core`, `cli` a `android` mají mít verzi `8.5.0`. Camera má verzi `8.2.4`, Filesystem `8.1.3` a Preferences `8.0.1`. Rozdílná vedlejší čísla pluginů jsou v pořádku; důležitá je společná hlavní řada 8.

### Proč instalujeme PWA Elements

Nativní Android používá systémový fotoaparát. Prohlížeč ale stejnou obrazovku nemá. Balíček `@ionic/pwa-elements` poskytuje webovou variantu rozhraní fotoaparátu; pokud není dostupná, Camera plugin může použít běžný výběr souboru.

Do `src/main.ts` přidejte import a registraci PWA elementů. Existující konfiguraci routeru zachovejte:

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { RouteReuseStrategy, provideRouter, withPreloading, PreloadAllModules } from '@angular/router';
import { IonicRouteStrategy, provideIonicAngular } from '@ionic/angular';
import { defineCustomElements } from '@ionic/pwa-elements/loader';

import { routes } from './app/app.routes';
import { AppComponent } from './app/app.component';

defineCustomElements(window);

bootstrapApplication(AppComponent, {
  providers: [
    { provide: RouteReuseStrategy, useClass: IonicRouteStrategy },
    provideIonicAngular(),
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
}).catch((error: unknown) => console.error(error));
```

Pokud vygenerovaný `main.ts` obsahuje další router funkce, například `withComponentInputBinding()`, nemažte je. Pouze přidejte import `defineCustomElements` a jeho volání před `bootstrapApplication()`.

## 6. Model uložené fotografie

Vytvořte adresář `src/app/models` a soubor `src/app/models/gallery-photo.ts`:

```typescript
export interface GalleryPhoto {
  filepath: string;
  webviewPath: string;
  createdAt: string;
}

export interface StoredPhoto {
  filepath: string;
  createdAt: string;
}
```

`GalleryPhoto` obsahuje také `webviewPath`, který lze použít jako `src` obrázku během aktuálního běhu aplikace. Do Preferences ale uložíme pouze `StoredPhoto`. Dočasná URL nebo base64 data by zbytečně zvětšovala seznam metadat.

## 7. Vygenerování PhotoService

V kořeni projektu spusťte:

```bash
npx ng generate service services/photo --type=service
```

Vzniknou soubory `src/app/services/photo.service.ts` a `photo.service.spec.ts`.

## 8. Implementace PhotoService

Nahraďte obsah `src/app/services/photo.service.ts`:

```typescript
import { Injectable, signal } from '@angular/core';
import { Camera, EncodingType } from '@capacitor/camera';
import type { MediaResult } from '@capacitor/camera';
import { Directory, Filesystem } from '@capacitor/filesystem';
import { Preferences } from '@capacitor/preferences';
import { GalleryPhoto, StoredPhoto } from '../models/gallery-photo';

const PHOTO_STORAGE = 'photo-gallery.photos';

function isStoredPhoto(value: unknown): value is StoredPhoto {
  return (
    typeof value === 'object' &&
    value !== null &&
    'filepath' in value &&
    typeof value.filepath === 'string' &&
    'createdAt' in value &&
    typeof value.createdAt === 'string'
  );
}

@Injectable({
  providedIn: 'root',
})
export class PhotoService {
  readonly photos = signal<GalleryPhoto[]>([]);
  private loaded = false;

  async loadSaved(): Promise<void> {
    if (this.loaded) {
      return;
    }

    const { value } = await Preferences.get({ key: PHOTO_STORAGE });
    let storedPhotos: StoredPhoto[] = [];

    try {
      const parsed: unknown = value ? JSON.parse(value) : [];
      storedPhotos = Array.isArray(parsed) ? parsed.filter(isStoredPhoto) : [];
    } catch {
      storedPhotos = [];
    }

    const loadedPhotos: GalleryPhoto[] = [];

    for (const photo of storedPhotos) {
      try {
        const { data } = await Filesystem.readFile({
          path: photo.filepath,
          directory: Directory.Data,
        });

        loadedPhotos.push({
          ...photo,
          webviewPath: await this.fileDataToUrl(data),
        });
      } catch (error) {
        console.warn(`Soubor ${photo.filepath} nelze načíst.`, error);
      }
    }

    this.photos.set(loadedPhotos);
    this.loaded = true;

    if (loadedPhotos.length !== storedPhotos.length) {
      await this.persistMetadata();
    }
  }

  async addNewToGallery(): Promise<void> {
    const capturedPhoto = await Camera.takePhoto({
      quality: 80,
      targetWidth: 1600,
      targetHeight: 1600,
      correctOrientation: true,
      encodingType: EncodingType.JPEG,
    });

    const savedPhoto = await this.savePicture(capturedPhoto);
    this.photos.update((photos) => [savedPhoto, ...photos]);
    await this.persistMetadata();
  }

  async deletePhoto(photo: GalleryPhoto): Promise<void> {
    await Filesystem.deleteFile({
      path: photo.filepath,
      directory: Directory.Data,
    });

    this.photos.update((photos) =>
      photos.filter((item) => item.filepath !== photo.filepath),
    );
    await this.persistMetadata();
  }

  private async savePicture(photo: MediaResult): Promise<GalleryPhoto> {
    const dataUrl = await this.dataUrlFromPath(photo.webPath);
    const filepath = `photo-${Date.now()}.jpeg`;

    await Filesystem.writeFile({
      path: filepath,
      data: dataUrl,
      directory: Directory.Data,
    });

    return {
      filepath,
      webviewPath: photo.webPath,
      createdAt: new Date().toISOString(),
    };
  }

  private async persistMetadata(): Promise<void> {
    const storedPhotos: StoredPhoto[] = this.photos().map(
      ({ filepath, createdAt }) => ({ filepath, createdAt }),
    );

    await Preferences.set({
      key: PHOTO_STORAGE,
      value: JSON.stringify(storedPhotos),
    });
  }

  private async dataUrlFromPath(path: string): Promise<string> {
    const response = await fetch(path);

    if (!response.ok) {
      throw new Error(`Fotografii nelze načíst: HTTP ${response.status}.`);
    }

    return this.blobToDataUrl(await response.blob());
  }

  private async fileDataToUrl(data: string | Blob): Promise<string> {
    if (data instanceof Blob) {
      return this.blobToDataUrl(data);
    }

    return data.startsWith('data:') ? data : `data:image/jpeg;base64,${data}`;
  }

  private blobToDataUrl(blob: Blob): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onerror = () => reject(new Error('Obrazová data nelze přečíst.'));
      reader.onload = () => {
        if (typeof reader.result === 'string') {
          resolve(reader.result);
        } else {
          reject(new Error('Obrazová data nemají očekávaný formát.'));
        }
      };
      reader.readAsDataURL(blob);
    });
  }
}
```

### Co služba dělá

| Metoda | Úloha |
| --- | --- |
| `Camera.takePhoto()` | otevře fotoaparát a vrátí `MediaResult` |
| `dataUrlFromPath()` | načte dočasný `webPath` a převede Blob na data URL |
| `Filesystem.writeFile()` | zapíše obraz do `Directory.Data` |
| `Preferences.set()` | uloží pouze seznam názvů souborů a časů |
| `Filesystem.readFile()` | po spuštění načte každý trvalý soubor |
| `Filesystem.deleteFile()` | odstraní skutečná obrazová data |

`Directory.Data` je soukromý adresář aplikace. Fotografie se proto automaticky neobjeví v systémové aplikaci Fotky nebo Galerie. To je v tomto cvičení záměr: obrázky patří PhotoGallery a odinstalování aplikace je odstraní.

`Filesystem` může na webu vrátit `Blob`, zatímco nativní platforma vrací base64 řetězec. Metoda `fileDataToUrl()` zpracuje obě varianty.

## 9. Logika záložky Fotografie

Galerii umístěte na druhou záložku. Nahraďte obsah `src/app/tab2/tab2.page.ts`:

```typescript
import { Component, inject, signal } from '@angular/core';
import { DatePipe } from '@angular/common';
import {
  IonButton,
  IonCol,
  IonContent,
  IonFab,
  IonFabButton,
  IonGrid,
  IonHeader,
  IonIcon,
  IonImg,
  IonNote,
  IonRow,
  IonSpinner,
  IonText,
  IonTitle,
  IonToolbar,
} from '@ionic/angular';
import { addIcons } from 'ionicons';
import { camera, trash } from 'ionicons/icons';
import { GalleryPhoto } from '../models/gallery-photo';
import { PhotoService } from '../services/photo.service';

@Component({
  selector: 'app-tab2',
  templateUrl: 'tab2.page.html',
  styleUrls: ['tab2.page.scss'],
  imports: [
    DatePipe,
    IonButton,
    IonCol,
    IonContent,
    IonFab,
    IonFabButton,
    IonGrid,
    IonHeader,
    IonIcon,
    IonImg,
    IonNote,
    IonRow,
    IonSpinner,
    IonText,
    IonTitle,
    IonToolbar,
  ],
})
export class Tab2Page {
  private readonly photoService = inject(PhotoService);

  readonly photos = this.photoService.photos;
  readonly busy = signal(false);
  readonly errorMessage = signal('');

  constructor() {
    addIcons({ camera, trash });
  }

  async ionViewWillEnter(): Promise<void> {
    await this.runOperation(() => this.photoService.loadSaved());
  }

  async addPhotoToGallery(): Promise<void> {
    await this.runOperation(() => this.photoService.addNewToGallery());
  }

  async removePhoto(photo: GalleryPhoto): Promise<void> {
    await this.runOperation(() => this.photoService.deletePhoto(photo));
  }

  private async runOperation(operation: () => Promise<void>): Promise<void> {
    if (this.busy()) {
      return;
    }

    this.busy.set(true);
    this.errorMessage.set('');

    try {
      await operation();
    } catch (error) {
      console.error(error);
      this.errorMessage.set('Operace byla zrušena nebo se nezdařila.');
    } finally {
      this.busy.set(false);
    }
  }
}
```

`runOperation()` zabraňuje spuštění dvou nativních operací současně a zaručuje vypnutí indikátoru i při chybě. Zrušení systémového fotoaparátu uživatelem může být vráceno jako odmítnutý Promise, proto jej stránka zachytí.

## 10. Šablona galerie

Nahraďte `src/app/tab2/tab2.page.html`:

```html
<ion-header [translucent]="true">
  <ion-toolbar>
    <ion-title>Fotografie</ion-title>
  </ion-toolbar>
</ion-header>

<ion-content [fullscreen]="true">
  <ion-header collapse="condense">
    <ion-toolbar>
      <ion-title size="large">Fotografie</ion-title>
    </ion-toolbar>
  </ion-header>

  @if (errorMessage()) {
    <ion-text color="danger">
      <p class="message">{{ errorMessage() }}</p>
    </ion-text>
  }

  @if (!busy() && photos().length === 0) {
    <ion-note class="empty-state">
      Galerie je prázdná. Pořiďte první fotografii tlačítkem fotoaparátu.
    </ion-note>
  }

  <ion-grid fixed>
    <ion-row>
      @for (photo of photos(); track photo.filepath) {
        <ion-col size="12" size-sm="6" size-lg="4">
          <article class="photo-card">
            <ion-img [src]="photo.webviewPath" [alt]="'Fotografie z ' + (photo.createdAt | date: 'medium')" />
            <ion-button
              class="delete-button"
              fill="solid"
              color="danger"
              size="small"
              aria-label="Odstranit fotografii"
              [disabled]="busy()"
              (click)="removePhoto(photo)"
            >
              <ion-icon slot="icon-only" name="trash" />
            </ion-button>
            <p>{{ photo.createdAt | date: 'medium' }}</p>
          </article>
        </ion-col>
      }
    </ion-row>
  </ion-grid>

  <ion-fab vertical="bottom" horizontal="center" slot="fixed">
    <ion-fab-button
      aria-label="Pořídit fotografii"
      [disabled]="busy()"
      (click)="addPhotoToGallery()"
    >
      @if (busy()) {
        <ion-spinner name="crescent" />
      } @else {
        <ion-icon name="camera" />
      }
    </ion-fab-button>
  </ion-fab>
</ion-content>
```

`track photo.filepath` dává každé položce stabilní identitu. Alternativní text tlačítek a obrázků zpřístupňuje galerii asistivním technologiím.

## 11. Styly galerie

Nahraďte `src/app/tab2/tab2.page.scss`:

```scss
.message,
.empty-state {
  display: block;
  margin: 1rem;
  text-align: center;
}

.photo-card {
  position: relative;
  overflow: hidden;
  margin: 0;
  border-radius: 0.75rem;
  background: var(--ion-color-light);
  box-shadow: 0 0.25rem 1rem rgb(0 0 0 / 12%);
}

.photo-card ion-img {
  aspect-ratio: 4 / 3;
  background: var(--ion-color-step-100);
}

.photo-card ion-img::part(image) {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.photo-card p {
  margin: 0;
  padding: 0.75rem;
  color: var(--ion-color-medium);
  font-size: 0.875rem;
}

.delete-button {
  position: absolute;
  top: 0.35rem;
  right: 0.35rem;
  margin: 0;
}

ion-grid {
  padding-bottom: 6rem;
}
```

Spodní odsazení mřížky zabraňuje tomu, aby plovoucí tlačítko zakrylo poslední fotografii.

## 12. Úprava navigace a informací o autorovi

V `src/app/tabs/tabs.page.html` změňte druhou záložku na:

```html
<ion-tab-button tab="tab2" href="/tabs/tab2">
  <ion-icon aria-hidden="true" name="images"></ion-icon>
  <ion-label>Fotografie</ion-label>
</ion-tab-button>
```

Ověřte, že je ikona `images` registrovaná v `src/app/tabs/tabs.page.ts`. Ve standalone komponentě musí být každá ikona používaná názvem předaná do `addIcons()`.

Na třetí záložce změňte titulek na **O aplikaci** a zobrazte:

- název aplikace PhotoGallery,
- své jméno a příjmení,
- text „AP5PM 2026 – CV7“.

První záložku můžete přejmenovat na **Úvod** a stručně na ní popsat, že fotografie zůstávají v soukromém úložišti aplikace.

## 13. První test v prohlížeči

Spusťte:

```bash
ionic serve
```

Na záložce Fotografie proveďte:

1. Klikněte na tlačítko fotoaparátu.
2. Povolte prohlížeči přístup ke kameře, nebo vyberte obrazový soubor.
3. Pořiďte alespoň dvě fotografie.
4. Obnovte stránku pomocí `Ctrl+R` nebo `Cmd+R`.
5. Ověřte, že se fotografie znovu načetly.
6. Jednu fotografii odstraňte a stránku znovu obnovte.
7. Ověřte, že smazaná položka se nevrátila.

V Chrome DevTools otevřete kartu **Application**:

- v IndexedDB najděte obsah webové implementace Filesystemu,
- v Local Storage najděte data pluginu Preferences,
- porovnejte velikost obrazového souboru s malým JSON seznamem metadat.

Webová implementace simuluje mobilní úložiště pomocí úložišť prohlížeče. Fotografie se nezapisují jako běžné soubory do složky Obrázky na počítači.

## 14. Přidání platformy Android

Ukončete vývojový server a vytvořte produkční webový build:

```bash
ionic build
npx cap add android
npx cap sync android
```

`npx cap add android` spusťte pouze jednou. Při dalších změnách používejte:

```bash
ionic build
npx cap sync android
```

Zkontrolujte platformu a pluginy:

```bash
npx cap ls
npx cap doctor android
```

V seznamu mají být `Camera`, `Filesystem` a `Preferences`.

## 15. Oprávnění a soukromé úložiště

Camera plugin na Androidu pro běžné pořízení fotografie nevyžaduje, abyste do `AndroidManifest.xml` ručně přidávali oprávnění `CAMERA`. Otevře systémovou aplikaci fotoaparátu a převezme její výsledek. Dialog s oprávněním aplikace PhotoGallery proto neočekávejte.

V tomto cvičení navíc nepoužíváme `saveToGallery: true` ani veřejné adresáře Filesystemu. Ručně tedy nepřidávejte `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE` ani široké oprávnění ke všem souborům zařízení. Data ukládá PhotoGallery do vlastního `Directory.Data`.

Pokud systémový fotoaparát není dostupný, uživatel pořízení zruší nebo nativní operace selže, aplikace nesmí spadnout a má zobrazit chybovou zprávu z `runOperation()`.

## 16. Spuštění v emulátoru

Spusťte AVD z CV6 a potom použijte:

```bash
ionic build
npx cap run android
```

Alternativně otevřete projekt v Android Studiu:

```bash
npx cap open android
```

Fotoaparát emulátoru může zobrazovat virtuální scénu místo skutečného obrazu. To je v pořádku. Proveďte celý scénář:

1. pořiďte dvě fotografie,
2. ověřte jejich zobrazení v mřížce,
3. aplikaci odeberte ze seznamu posledních aplikací,
4. spusťte ji znovu přes ikonu,
5. ověřte zachování obou fotografií,
6. jednu fotografii odstraňte,
7. aplikaci znovu spusťte a ověřte trvalé odstranění,
8. změňte orientaci a ověřte responzivní rozvržení,
9. zrušte jednou systémový fotoaparát a ověřte, že se lze o pořízení pokusit znovu.

Data v prohlížeči a data v emulátoru jsou dvě oddělená úložiště. Fotografie pořízené při `ionic serve` se v Android aplikaci neobjeví.

## 17. Tok dat

```text
Camera.takePhoto()
        │
        ▼
dočasný MediaResult.webPath
        │
        ├──> okamžité zobrazení v <ion-img>
        │
        ▼
Filesystem.writeFile(Directory.Data)
        │
        ├──> trvalý soubor photo-....jpeg
        │
        ▼
Preferences.set()
        │
        └──> JSON: filepath + createdAt
```

Při dalším spuštění jde tok opačně: Preferences poskytne seznam názvů, Filesystem načte odpovídající soubory a služba z nich vytvoří `webviewPath` pro šablonu.

Preferences nejsou databáze obrázků. Jsou vhodné pro malé hodnoty a metadata. Vložení celých base64 fotografií do Preferences by bylo pomalé, paměťově náročné a špatně spravovatelné.

## 18. Sestavení debug APK

Nejprve proveďte finální build a synchronizaci:

```bash
npm run lint
ionic build
npx cap sync android
```

### Windows – PowerShell

```powershell
Set-Location android
.\gradlew.bat assembleDebug
Set-Location ..
```

### macOS a Linux – Terminal

```bash
cd android
./gradlew assembleDebug
cd ..
```

Výsledný soubor najdete zde:

```text
android/app/build/outputs/apk/debug/app-debug.apk
```

Debug APK je podepsané vývojovým klíčem a je určené pro testování nebo odevzdání v rámci cvičení. Není určené k publikování do Google Play.

Před odevzdáním APK nainstalujte nebo spusťte v emulátoru a ověřte, že nejde o starší build.

## 19. Kontrola a Git

V kořeni projektu spusťte:

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

Zkontrolujte, že:

- `capacitor.config.ts` obsahuje správné `appId` a neobsahuje `server.url`,
- `core`, `cli` a `android` mají verzi 8.5.0 a všechny pluginy patří do hlavní řady 8,
- adresář `android/` je sledovaný Gitem,
- `android/local.properties`, `android/.gradle/` ani adresáře `build/` nejsou připravené ke commitu,
- v aplikaci je vaše jméno,
- APK vzniklo až po posledním `ionic build` a `npx cap sync android`.

Uložte práci:

```bash
git add .
git commit -m "Dokončena aplikace PhotoGallery"
git status
```

## 20. Kontrolní seznam CV7

- [ ] Projekt PhotoGallery je samostatný Git repozitář.
- [ ] Projekt používá standalone komponenty.
- [ ] `appId` je platný a jedinečný.
- [ ] Core, CLI a Android mají verzi 8.5.0; Camera, Filesystem a Preferences patří do hlavní řady 8.
- [ ] Aplikace používá `Camera.takePhoto()`, nikoliv zastaralé `getPhoto()`.
- [ ] Lze pořídit a zobrazit více fotografií.
- [ ] Fotografie zůstávají dostupné po novém spuštění aplikace.
- [ ] Lze odstranit obrazový soubor i jeho metadata.
- [ ] Prázdná galerie, probíhající operace a chyba mají vlastní stav UI.
- [ ] Aplikace funguje v prohlížeči i na Androidu.
- [ ] Na třetí záložce je jméno autora.
- [ ] `npm run lint` a `ionic build` proběhnou bez chyby.
- [ ] Finální debug APK lze nainstalovat a spustit.
- [ ] Projekt je uložený v Git commitu a pracovní strom je čistý.

## 21. Bonusové úkoly

Po dokončení povinné části můžete:

1. přidat vlastní potvrzovací dialog před odstraněním fotografie,
2. použít `Camera.chooseFromGallery()` a umožnit import existující fotografie,
3. doplnit popisek fotografie a uložit jej společně s metadaty,
4. zobrazit detail fotografie na samostatné routě,
5. zobrazit počet fotografií v badge záložky,
6. doplnit tlačítko pro změnu pořadí od nejnovější nebo nejstarší,
7. přidat test `PhotoService` s mocky Camera, Filesystem a Preferences,
8. ošetřit návrat výsledku, pokud Android ukončí aplikaci během otevřeného fotoaparátu,
9. vyzkoušet aplikaci na fyzickém telefonu a porovnat chování s emulátorem,
10. vytvořit vlastní ikonu a splash screen pomocí `@capacitor/assets`.

Použití `saveToGallery: true` je rozšiřující úloha. Vyžaduje pochopení systémové galerie a oprávnění pro různé verze Androidu; nepřidávejte široká oprávnění ke všem souborům zařízení.

## 22. Nejčastější problémy

### `Camera.takePhoto is not a function`

Projekt používá starší nebo nesourodou verzi pluginu. Ověřte:

```bash
npm ls @capacitor/core @capacitor/camera
npx cap sync android
```

`@capacitor/core` má mít verzi 8.5.0 a Camera musí být alespoň ve verzi 8.1, která nové API zavedla. Pro toto cvičení použijte Camera 8.2.4. Po instalaci pluginu je nutné synchronizovat Android projekt.

### Web otevře pouze výběr souboru

Zkontrolujte instalaci `@ionic/pwa-elements`, import `defineCustomElements` a jeho volání před `bootstrapApplication()`. Prohlížeč bez použitelné kamery může správně přejít na výběr obrazového souboru.

### Po obnovení stránky fotografie zmizí

Ověřte, že po `Filesystem.writeFile()` voláte `Preferences.set()` a při vstupu na záložku `loadSaved()`. V DevTools zkontrolujte IndexedDB a Local Storage.

### Po spuštění zůstane prázdný rámeček

Do Preferences se nemá ukládat starý dočasný `webPath`. Po spuštění musí aplikace soubor znovu načíst přes `Filesystem.readFile()` a vytvořit nový zobrazitelný data URL.

### Fotografie není v systémové Galerii

To je očekávané. `Directory.Data` je soukromý adresář aplikace. Tato PhotoGallery spravuje vlastní kolekci a nepoužívá `saveToGallery: true`.

### Android nezobrazí žádost o oprávnění fotoaparátu

To je při této implementaci očekávané. Plugin otevře systémovou aplikaci fotoaparátu a PhotoGallery sama oprávnění `CAMERA` nedeklaruje. Oprávnění ke sdílenému úložišti by bylo potřeba řešit až u rozšíření, které zapisuje do veřejné galerie.

### V emulátoru je místo kamery virtuální místnost

Jde o simulovaný vstup fotoaparátu. Lze jej ovládat rozšířenými nastaveními emulátoru. Pro reálnou fotografii použijte fyzický telefon.

### Aplikace po změně kódu ukazuje starou verzi

Před spuštěním v Android Studiu proveďte:

```bash
ionic build
npx cap sync android
```

Android Studio samo znovu nevytváří Angular build.

### `File does not exist`

Metadata odkazují na chybějící soubor, například po ručním zásahu do úložiště. Implementace v tomto cvičení chybnou položku přeskočí a seznam metadat opraví. Zkontrolujte varování v konzoli nebo Logcatu.

### APK obsahuje starou aplikaci

Gradle sestavuje obsah posledního syncu. Zopakujte `ionic build`, `npx cap sync android` a teprve potom `assembleDebug`. Zkontrolujte čas změny výsledného APK.

## Oficiální dokumentace

- [Ionic – Photo Gallery tutorial](https://ionicframework.com/docs/angular/your-first-app)
- [Capacitor – Camera](https://capacitorjs.com/docs/apis/camera)
- [Capacitor – Filesystem](https://capacitorjs.com/docs/apis/filesystem)
- [Capacitor – Preferences](https://capacitorjs.com/docs/apis/preferences)
- [Capacitor – Android](https://capacitorjs.com/docs/android)
- [Android Developers – úložiště specifické pro aplikaci](https://developer.android.com/training/data-storage/app-specific)
