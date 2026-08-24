# CV1 – Vývojové prostředí a první Ionic aplikace

## Cíl cvičení

Po dokončení cvičení budete umět:

- zkontrolovat vývojové prostředí pro webovou a hybridní mobilní aplikaci,
- vytvořit nový projekt Ionic s frameworkem Angular a runtime Capacitor,
- orientovat se v základní struktuře projektu,
- spustit aplikaci ve webovém prohlížeči,
- provést jednoduchou změnu a ověřit automatické obnovení aplikace.

V tomto cvičení ještě nebudeme sestavovat nativní Android ani iOS aplikaci. Android Studio a Xcode budete potřebovat v dalších cvičeních.

## 1. Technologie používané v kurzu

V akademickém roce 2026 používáme tento základ:

| Nástroj | Verze |
| --- | --- |
| Node.js | řada 26 |
| Angular | řada 22 |
| Ionic Framework | řada 9 |
| Capacitor | řada 8, minimálně 8.5 |
| TypeScript | řada 6.0 |

V učebně je Node.js již nainstalovaný. Na vlastním počítači používejte Node.js 26 a během semestru neměňte hlavní verze frameworků v rozpracovaném projektu.

> **Důležité:** Nemíchejte návody pro Cordovu se současným postupem pro Capacitor. Příkazy začínající `ionic cordova` v tomto kurzu nepoužíváme.

## 2. Společné požadavky

Na všech operačních systémech budete potřebovat:

1. **Node.js 26** – běhové prostředí JavaScriptu a nástrojů Angular/Ionic.
2. **npm 11** – správce balíčků, instaluje se společně s Node.js.
3. **Git** – verzování a získávání výukových projektů.
4. **Visual Studio Code** nebo jiné IDE pro TypeScript a Angular.
5. Moderní webový prohlížeč, doporučen je Chrome.

Doporučená rozšíření VS Code:

- Angular Language Service,
- ESLint,
- Prettier – Code formatter.

Rozšíření editoru jsou pomocné nástroje. Pro spuštění projektu nejsou nutná.

### Kontrola instalace

Otevřete nový terminál a spusťte:

```bash
node --version
npm --version
git --version
```

Očekáváme:

- `node --version` začíná `v26.`,
- `npm --version` začíná `11.`,
- Git vypíše číslo své verze.

Pokud jste Node.js právě nainstalovali a příkaz není nalezen, zavřete všechna okna terminálu i VS Code a otevřete je znovu.

## 3. Instalace podle operačního systému

### Windows 10/11

Nainstalujte:

1. [Git for Windows](https://git-scm.com/download/win).
2. [Node.js](https://nodejs.org/) řady 26.
3. [Visual Studio Code](https://code.visualstudio.com/).

Pro správu více verzí Node.js lze použít `nvm-windows`. Není nutné jej instalovat na školních počítačích.

Pro pozdější Android vývoj budete potřebovat:

- [Android Studio](https://developer.android.com/studio) minimálně 2025.2.1,
- Android SDK Platform 36,
- Android SDK Build-Tools,
- Android SDK Platform-Tools,
- Android SDK Command-line Tools,
- emulátor se systémovým obrazem API 36 nebo fyzický telefon.

Samostatnou instalaci JDK běžně nepotřebujete. Android Studio obsahuje vhodné JDK. Pro emulátor musí být v BIOS/UEFI povolena hardwarová virtualizace a ve Windows příslušná virtualizační podpora.

### macOS

Nainstalujte:

1. [Git](https://git-scm.com/download/mac) – je dostupný také jako součást Xcode Command Line Tools.
2. [Node.js](https://nodejs.org/) řady 26, ideálně pomocí `nvm`.
3. [Visual Studio Code](https://code.visualstudio.com/).

Pro pozdější Android vývoj nainstalujte Android Studio a stejné balíčky SDK jako na Windows.

Pro pozdější iOS vývoj budete potřebovat:

- macOS,
- Xcode 26 nebo novější,
- Xcode Command Line Tools,
- iOS Simulator nebo fyzický iPhone/iPad.

Xcode Command Line Tools lze po instalaci Xcode doplnit příkazem:

```bash
xcode-select --install
```

Capacitor 8 používá u nových iOS projektů primárně Swift Package Manager. CocoaPods proto není obecně povinný.

### Linux

Nainstalujte:

1. Git z repozitáře své distribuce.
2. Node.js 26, doporučeně pomocí `nvm`.
3. Visual Studio Code nebo jiné IDE.

Pro pozdější Android vývoj nainstalujte Android Studio, SDK Platform 36 a ostatní Android SDK nástroje uvedené výše. Pro plynulý běh emulátoru musí být dostupná hardwarová virtualizace, na Linuxu obvykle KVM.

iOS aplikaci nelze lokálně sestavit na Windows ani Linuxu. Nativní iOS build vyžaduje macOS a Xcode.

## 4. Instalace Ionic CLI

Ionic CLI nainstalujte v terminálu:

```bash
npm install --global @ionic/cli@7.2.1
```

Ověřte instalaci:

```bash
ionic --version
```

Pokud na macOS nebo Linuxu skončí globální instalace chybou oprávnění, nepoužívejte `sudo npm install`. Nainstalujte Node.js pomocí `nvm` a příkaz opakujte v novém terminálu.

## 5. Vytvoření prvního projektu

V terminálu přejděte do adresáře, ve kterém chcete mít své školní projekty. Nevytvářejte projekt uvnitř jiného Ionic projektu.

Spusťte:

```bash
ionic start moje-prvni-aplikace blank --type=angular-standalone --capacitor
```

Parametr `--type=angular-standalone` výslovně určuje, že se má vytvořit projekt se Standalone komponentami. Pokud nástroj nabídne vytvoření účtu Ionic, můžete tuto možnost přeskočit.

Příkaz:

- vytvoří adresář `moje-prvni-aplikace`,
- připraví projekt Ionic a Angular,
- přidá konfiguraci Capacitoru,
- nainstaluje npm závislosti,
- vytvoří `package-lock.json`.

Přejděte do projektu:

```bash
cd moje-prvni-aplikace
```

Otevřete jej ve VS Code:

```bash
code .
```

Pokud příkaz `code` není dostupný, otevřete VS Code běžným způsobem a zvolte **File → Open Folder**.

## 6. Kontrola vytvořeného projektu

Spusťte:

```bash
npx ng version
npx cap --version
npm ls @angular/core @ionic/angular @capacitor/core
```

Zkontrolujte, že projekt obsahuje:

- Angular řady 22,
- Ionic řady 9,
- Capacitor řady 8.

Pokud se vytvořily jiné hlavní verze, nepokračujte v jejich ručním přepisování a informujte vyučujícího. Výstup příkazů přiložte ke svému dotazu.

Referenční kontrola návodu byla provedena s Angular 22.0.1, Ionic 9.0.0 a Capacitor 8.5.0. Menší rozdíl v opravé nebo vedlejší verzi je v pořádku; jiná hlavní verze není.

## 7. Spuštění aplikace v prohlížeči

V kořenovém adresáři projektu spusťte:

```bash
ionic serve
```

Ionic sestaví aplikaci, spustí vývojový server a obvykle otevře prohlížeč. Výchozí adresa je:

```text
http://localhost:8100
```

Terminál s běžícím serverem ponechte otevřený. Server ukončíte klávesovou zkratkou `Ctrl+C`.

### Pokud se prohlížeč neotevře

1. Zkontrolujte, že sestavení v terminálu neskončilo chybou.
2. Otevřete adresu `http://localhost:8100` ručně.
3. Pokud je port obsazený, Ionic nabídne jiný port nebo můžete použít:

```bash
ionic serve --port 8101
```

## 8. První změna aplikace

V editoru otevřete soubor:

```text
src/app/home/home.page.html
```

Najděte obsah domovské stránky a přidejte do elementu `ion-content` například:

```html
<h1>Moje první mobilní aplikace</h1>
<p>AP5PM 2026 – jméno a příjmení</p>
```

Soubor uložte. Běžící aplikace se má v prohlížeči automaticky aktualizovat. Pokud se změna neprojeví, zkontrolujte terminál a konzoli prohlížeče.

V nástrojích pro vývojáře prohlížeče zapněte simulaci mobilního zařízení a vyzkoušejte několik velikostí displeje. Jde pouze o simulaci rozměrů prohlížeče, nikoliv o plnohodnotný Android nebo iOS emulátor.

## 9. Co budeme později dělat pro Android

Následující postup dnes není povinným úkolem. Slouží jako přehled dalších kroků:

```bash
npm install --save-exact @capacitor/android@8.5.0
ionic build
npx cap add android
npx cap sync android
npx cap open android
```

Projekt se otevře v Android Studiu. Odtud jej lze spustit v emulátoru nebo fyzickém zařízení. Po každé změně webové části, kterou chcete přenést do nativního projektu, je potřeba znovu sestavit web a synchronizovat Capacitor:

```bash
ionic build
npx cap sync android
```

## 10. Co budeme později dělat pro iOS

Tento postup lze provést pouze na macOS s Xcode:

```bash
npm install --save-exact @capacitor/ios@8.5.0
ionic build
npx cap add ios
npx cap sync ios
npx cap open ios
```

Nativní projekt se otevře v Xcode, kde lze vybrat simulátor nebo připojené zařízení.

## 11. Kontrolní seznam CV1

Na konci cvičení musí platit:

- [ ] `node --version` vypisuje Node.js 26.
- [ ] `npm --version` vypisuje npm 11.
- [ ] Fungují příkazy `git` a `ionic`.
- [ ] Vytvořil/a jsem Angular projekt typu Standalone.
- [ ] Projekt používá Ionic a Capacitor.
- [ ] Aplikace běží na `http://localhost:8100`.
- [ ] Upravil/a jsem text domovské stránky a změna se zobrazila v prohlížeči.
- [ ] Umím vývojový server ukončit pomocí `Ctrl+C`.

## 12. Nejčastější problémy

### `ionic: command not found`

Zavřete a znovu otevřete terminál. Potom ověřte:

```bash
npm config get prefix
npm list --global --depth=0
```

### Chyba oprávnění při `npm install --global`

Na macOS/Linux nepoužívejte `sudo`. Použijte `nvm`, nainstalujte přes něj Node.js 26 a opakujte instalaci Ionic CLI.

### Projekt se po stažení nespustí

V již existujícím projektu instalujte přesně závislosti z lockfilu:

```bash
npm ci
ionic serve
```

Nemažte `package-lock.json`.

### `npm audit` hlásí zranitelnosti

Hlašení si poznamenejte a oznamte vyučujícímu. Bez jeho pokynu nespouštějte `npm audit fix --force`: tento příkaz může změnit hlavní verze závislostí a projekt rozbít. Pro splnění CV1 je rozhodující, že se referenční aplikace sestaví a spustí v prohlížeči.

### Bílá stránka nebo chyba v prohlížeči

Zkontrolujte:

1. chyby v terminálu s `ionic serve`,
2. konzoli v Developer Tools prohlížeče,
3. zda jste upravovali soubor uvnitř správného projektu,
4. zda je stále spuštěný vývojový server.

## Oficiální dokumentace

- [Ionic Angular – první aplikace](https://ionicframework.com/docs/angular/your-first-app)
- [Ionic CLI](https://ionicframework.com/docs/cli)
- [Capacitor – příprava prostředí](https://capacitorjs.com/docs/getting-started/environment-setup)
- [Capacitor pro Android](https://capacitorjs.com/docs/android)
- [Capacitor pro iOS](https://capacitorjs.com/docs/ios)
- [Angular – lokální prostředí](https://angular.dev/tools/cli/setup-local)
