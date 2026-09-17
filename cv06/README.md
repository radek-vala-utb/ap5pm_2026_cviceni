# CV6 – První nativní sestavení pro Android

## Cíl cvičení

V CV5 jste CounterApp spouštěli jako webovou aplikaci v prohlížeči. V tomto cvičení přidáte do stejného projektu nativní platformu Android, sestavíte webovou část, přenesete ji pomocí Capacitoru do projektu pro Android a aplikaci spustíte v emulátoru nebo na fyzickém telefonu.

Po dokončení cvičení budete umět:

- vysvětlit roli Ionicu, Angularu, Capacitoru a Android Studia,
- nastavit identifikátor a zobrazovaný název aplikace,
- přidat balíček a nativní projekt `@capacitor/android`,
- rozlišit příkazy `ionic build`, `npx cap sync android`, `npx cap run android` a `npx cap open android`,
- vytvořit a spustit Android Virtual Device (AVD),
- nasadit debug sestavení do emulátoru nebo fyzického zařízení,
- ověřit síťový požadavek a trvalé uložení dat v nativní aplikaci,
- najít výpis aplikace v Logcatu a prohlédnout WebView pomocí Chrome DevTools,
- bezpečně spravovat vygenerovaný adresář `android/` v Gitu.

V tomto cvičení nevytváříte aplikaci znovu. Android je další cílová platforma existujícího projektu. Zdrojová Angular aplikace zůstává ve `src/` a stejný webový výstup běží v nativním kontejneru Capacitoru.

## 1. Výchozí stav

Navazujete na vlastní projekt `counter-app` dokončený v CV5. Před zahájením musí fungovat:

- ruční počítadlo a uložení záznamu,
- historie uložená pomocí Capacitor Preferences,
- odpočet do svátku načítaný z OpenHolidays API,
- chybový stav při nedostupné síti,
- příkazy `npm run lint` a `ionic build`.

Projekt má používat Capacitor 8.5 a v kořeni musí obsahovat `capacitor.config.ts`. Pokud CV5 dokončené nemáte, nejprve se vraťte k jeho kontrolnímu seznamu.

## 2. Co budete potřebovat

Na Windows, macOS i Linuxu potřebujete:

- Android Studio minimálně řady 2025.2.1,
- Android SDK Platform 36,
- Android SDK Build-Tools,
- Android SDK Platform-Tools,
- Android SDK Command-line Tools,
- Android Emulator a systémový obraz API 36, nebo fyzické zařízení s Androidem 7 či novějším,
- dostatek volného místa; systémový obraz a Gradle závislosti zabírají několik GB.

Samostatné JDK běžně neinstalujte. Android Studio obsahuje vhodný JetBrains Runtime a Gradle projekt jej umí použít. Pokud již máte vlastní `JAVA_HOME`, který ukazuje na staré JDK, může sestavení selhat; řešení najdete v části Nejčastější problémy.

Capacitor 8 podporuje Android od API 24. Pro cvičení používáme SDK a obraz API 36, aby měli všichni stejné prostředí.

### Kontrola komponent v Android Studiu

Na úvodní obrazovce Android Studia otevřete **More Actions → SDK Manager**. V otevřeném projektu je stejná nabídka v **Tools → SDK Manager**.

1. Na kartě **SDK Platforms** zaškrtněte **Android 16.0 (API 36)**.
2. Na kartě **SDK Tools** zapněte **Show Package Details** a ověřte aktuální verze položek:
   - Android SDK Build-Tools,
   - Android SDK Platform-Tools,
   - Android SDK Command-line Tools (latest),
   - Android Emulator.
3. Potvrďte změny tlačítkem **Apply** a přijměte licence SDK.

Na školním počítači mohou být všechny komponenty již připravené. Nic neodinstalovávejte a neměňte umístění SDK.

## 3. Otevření projektu a vytvoření větve

Novou větev vytvořte ze stavu dokončeného CV5.

### Windows – PowerShell

```powershell
Set-Location "$HOME\AP5PM-projekty\counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv6/android
code .
```

### macOS a Linux – Terminal

```bash
cd "$HOME/AP5PM-projekty/counter-app"
nvm use 26
git status
git log --oneline -3
git switch -c cv6/android
code .
```

Pokud na macOS příkaz `code .` není dostupný, použijte:

```bash
open -a "Visual Studio Code" .
```

Pokud Node.js nepoužíváte přes `nvm`, příkaz `nvm use 26` vynechte. Pokud větev existuje, použijte `git switch cv6/android`.

Před zahájením má `git status` hlásit čistý pracovní strom. Adresář `android/` zatím v projektu být nemá.

## 4. Kontrola konfigurace Capacitoru

Otevřete `capacitor.config.ts`:

```typescript
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'cz.uhk.ap5pm.counterapp',
  appName: 'CounterApp',
  webDir: 'www',
};

export default config;
```

Význam vlastností:

| Vlastnost | Význam |
| --- | --- |
| `appId` | Jednoznačný identifikátor aplikace v obráceném doménovém zápisu. Nepoužívejte mezery, pomlčky ani diakritiku. |
| `appName` | Název, který se zobrazí pod ikonou aplikace. |
| `webDir` | Adresář s výsledkem webového buildu; v našem Ionic projektu je to `www`. |

Pokud již máte vlastní platný `appId`, můžete jej ponechat. Po přidání Android platformy jej už v tomto cvičení neměňte: promítá se do nativního `applicationId` a názvů balíčků.

> **Důležité:** Do konfigurace nepřidávejte trvalou vlastnost `server.url`. Ta by aplikaci místo zabalených souborů připojovala k vývojovému serveru a nepatří do odevzdávaného produkčního nastavení.

## 5. Instalace platformy Android

Verze balíčků `@capacitor/core`, `@capacitor/cli` a `@capacitor/android` mají patřit do stejné řady. Projekt z CV5 používá verzi 8.5.0, proto spusťte:

```bash
npm install @capacitor/android@8.5.0
```

Ověřte verze:

```bash
npm ls @capacitor/core @capacitor/cli @capacitor/android
```

Všechny tři položky mají mít verzi `8.5.0`. Potom vytvořte aktuální webový výstup a přidejte Android projekt:

```bash
ionic build
npx cap add android
```

Příkaz `npx cap add android` vytvoří adresář `android/`. Spouští se pro danou platformu pouze jednou. Při dalších změnách už nepoužívejte `add`, ale `sync`.

Zkontrolujte instalované platformy a pluginy:

```bash
npx cap ls
npx cap doctor android
```

`doctor` může upozornit na chybějící SDK, nepřijaté licence nebo nevhodné JDK. Než budete pokračovat, vyřešte hlášené chyby.

## 6. Co vzniklo v adresáři android

Prohlédněte si novou část projektu:

```text
android/
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── assets/
│       ├── java/
│       └── res/
├── build.gradle
├── gradle/
├── gradlew
├── gradlew.bat
├── settings.gradle
└── variables.gradle
```

Názvy některých Gradle souborů se mohou mezi menšími verzemi lišit. Jejich obsah v tomto cvičení ručně neměňte.

| Část | Úloha |
| --- | --- |
| `src/` | Zdrojová Angular/Ionic aplikace, kterou upravujete nejčastěji. |
| `www/` | Sestavené webové soubory vytvořené příkazem `ionic build`. |
| `android/` | Samostatný nativní Gradle projekt pro Android Studio. |
| `android/app/src/main/assets/public/` | Kopie webového buildu používaná nativní aplikací. |
| `AndroidManifest.xml` | Deklarace aplikace, aktivit, oprávnění a dalších vlastností Androidu. |

Needitujte ručně soubory v `android/app/src/main/assets/public/`. Další `sync` je přepíše. Změny uživatelského rozhraní vždy provádějte ve `src/`.

Adresář `android/` je součást zdrojového projektu a patří do Gitu. Naproti tomu `android/.gradle/`, `android/build/`, `android/app/build/` a `local.properties` jsou lokální nebo sestavené soubory a výchozí projekt je ignoruje.

## 7. Vývojový tok web → Android

Při běžné práci probíhají tři kroky:

```text
src/ ── ionic build ──> www/ ── cap sync android ──> android/ ── Gradle ──> aplikace
```

Příkazy mají rozdílné role:

| Příkaz | Co provede |
| --- | --- |
| `ionic build` | Přeloží Angular aplikaci ze `src/` do `www/`. |
| `npx cap sync android` | Zkopíruje hotový webový build do Android projektu a aktualizuje nativní pluginy. |
| `npx cap open android` | Otevře existující projekt v Android Studiu; nic nepřekládá ani nekopíruje. |
| `npx cap run android` | Provede sync, sestaví nativní debug aplikaci a nasadí ji do vybraného zařízení. |

`sync` sám nespouští Angular build. Bez nového `ionic build` by mohl zkopírovat starý obsah `www/`.

Pro ruční práci v Android Studiu proto používejte opakovatelnou posloupnost:

```bash
ionic build
npx cap sync android
npx cap open android
```

Při spuštění z příkazové řádky nejprve sestavte web a potom použijte:

```bash
ionic build
npx cap run android
```

## 8. Vytvoření virtuálního zařízení

Otevřete Android projekt:

```bash
npx cap open android
```

Při prvním otevření počkejte na dokončení **Gradle Sync** a případné stažení závislostí. První synchronizace může trvat několik minut a vyžaduje internet.

V Android Studiu otevřete **Tools → Device Manager** a zvolte **+ → Create Virtual Device**.

1. Vyberte kategorii **Phone** a běžný profil, například Pixel 8.
2. Pokračujte tlačítkem **Next**.
3. Vyberte stažený obraz **API 36**. Pokud chybí, použijte ikonu stažení.
4. Architekturu volte podle počítače:
   - na počítačích Apple Silicon použijte obraz ARM64,
   - na běžných počítačích s procesorem Intel/AMD použijte doporučený obraz pro danou platformu.
5. AVD pojmenujte například `Pixel_8_API_36` a dokončete průvodce.
6. V Device Manageru spusťte zařízení tlačítkem ▶.

Počkejte, až se zobrazí domovská obrazovka Androidu. První start je pomalejší; další starty mohou použít uložený snapshot.

Emulátor je samostatné zařízení s vlastním úložištěm. Data v Preferences se neukládají do úložiště prohlížeče na vašem počítači a jednotlivá AVD je mezi sebou nesdílejí.

## 9. První spuštění aplikace

### Varianta A – z příkazové řádky

S běžícím emulátorem spusťte v kořeni projektu:

```bash
ionic build
npx cap run android
```

Pokud je dostupných více zařízení, vyberte vytvořený emulátor. Seznam cílů lze předem zobrazit:

```bash
npx cap run android --list
```

Příkaz provede sync, sestaví debug APK, nainstaluje jej a spustí aplikaci.

### Varianta B – z Android Studia

1. V horní liště vyberte modul `app`.
2. Jako cíl vyberte `Pixel_8_API_36`.
3. Klikněte na zelené tlačítko **Run 'app'** ▶.
4. Počkejte na sestavení, instalaci a spuštění.

Pokud jste od posledního `sync` změnili soubory ve `src/`, vraťte se nejprve do terminálu a spusťte:

```bash
ionic build
npx cap sync android
```

Samotné tlačítko Run v Android Studiu webovou část znovu nesestaví.

## 10. Ověření CounterApp v emulátoru

V nainstalované aplikaci proveďte celý scénář:

1. Na záložce **Počítadlo** vytvořte záznam s názvem `Android`, hodnotou alespoň 2 a uložte jej.
2. Na záložce **Historie** ověřte, že se záznam zobrazil.
3. Aplikaci odeberte ze seznamu posledních aplikací a znovu ji spusťte přes ikonu.
4. Ověřte, že uložený záznam zůstal zachovaný.
5. Na záložce **Odpočet** načtěte české svátky a změňte vybranou položku.
6. Ověřte, že se zobrazuje český název, datum a vypočtený počet dnů.
7. Otočte emulátor na šířku a zpět. Obsah nesmí přetékat mimo obrazovku.
8. Stiskněte systémové tlačítko Zpět a zkontrolujte chování navigace.

Tím současně ověřujete:

- Angular a Ionic UI uvnitř Android WebView,
- plugin Capacitor Preferences na nativní platformě,
- přístup k internetu deklarovaný v Android projektu,
- životní cyklus aplikace po jejím zavření a opětovném otevření.

Ukončení aplikace nemaže její data. Odinstalování aplikace nebo použití **Settings → Apps → CounterApp → Storage & cache → Clear storage** je odstraní.

## 11. Experiment s buildem a synchronizací

Ověřte, že rozumíte cestě zdrojového kódu do aplikace.

1. Ve `src/app/tab1/tab1.page.html` dočasně změňte titulek `Počítadlo` na `Počítadlo – změna`.
2. V Android Studiu pouze znovu stiskněte Run.
3. V aplikaci se změna ještě neprojeví, protože Android projekt obsahuje předchozí webový build.
4. V terminálu spusťte:

   ```bash
   ionic build
   npx cap sync android
   ```

5. V Android Studiu aplikaci znovu spusťte. Nový titulek se nyní zobrazí.
6. Vraťte původní text `Počítadlo` a znovu proveďte build a sync.

Před commitem ověřte pomocí `git diff`, že dočasná změna nezůstala ve zdrojovém kódu.

## 12. Kontrola síťové chyby

Odpočet z CV5 musí správně rozlišit chybu sítě také v nativní aplikaci.

1. Nechte aplikaci běžet a počkejte na úspěšné načtení svátků.
2. V emulátoru otevřete rychlá nastavení a zapněte **Airplane mode**; případně vypněte Wi-Fi i mobilní data.
3. Vraťte se do CounterApp a stiskněte **Načíst znovu**.
4. Ověřte chybovou zprávu, ukončení spinneru a opětovné zpřístupnění tlačítka.
5. Historie z Preferences musí být stále dostupná.
6. Obnovte připojení a načtení zopakujte.

Emulátor používá síť hostitelského počítače, ale z pohledu aplikace jde o jiné zařízení. Adresa `localhost` uvnitř emulátoru neoznačuje váš počítač. Pro OpenHolidays používáte veřejnou HTTPS adresu, takže není potřeba nic měnit.

## 13. Logcat a diagnostika nativní aplikace

V Android Studiu otevřete **View → Tool Windows → Logcat**.

1. Vyberte běžící emulátor.
2. Vyberte proces aplikace s ID `cz.uhk.ap5pm.counterapp` nebo vlastním `appId`.
3. Nastavte úroveň alespoň na **Info**.
4. Přepínejte záložky a zopakujte načtení svátků.
5. Při testu bez sítě najděte zprávy související s WebView nebo HTTP chybou.

Logcat obsahuje výpis celého zařízení, proto vždy filtrujte podle procesu aplikace. Červený záznam při úmyslně vypnuté síti je očekávaný; pády, `FATAL EXCEPTION` nebo opakované chyby při běžném spuštění očekávané nejsou.

## 14. Kontrola WebView v Chrome DevTools

Capacitor zobrazuje webovou část v Android WebView. Běžící debug aplikaci lze prohlížet podobně jako webovou stránku.

1. Nechte emulátor i CounterApp spuštěné.
2. V desktopovém Chromu otevřete `chrome://inspect/#devices`.
3. V části zařízení najděte WebView s názvem aplikace.
4. Klikněte na **inspect**.
5. Na kartě **Console** ověřte, že při běžném použití nejsou neošetřené chyby.
6. Na kartě **Network** stiskněte v aplikaci **Načíst znovu** a najděte požadavek `PublicHolidays`.
7. Ověřte metodu `GET`, stav `200` a parametry `CZ`, `CS`, `validFrom` a `validTo`.

Pokud WebView v seznamu nevidíte, zkontrolujte, že je aplikace právě otevřená, emulátor je připojený a používáte debug sestavení.

## 15. Volitelně: spuštění na fyzickém telefonu

Fyzické zařízení můžete použít místo emulátoru. Jde o volitelnou variantu; ve školní učebně postačí AVD.

1. V telefonu otevřete **Nastavení → Informace o telefonu**.
2. Sedmkrát klepněte na **Číslo sestavení**, dokud se nepovolí možnosti pro vývojáře.
3. V **Možnostech pro vývojáře** zapněte **Ladění USB**.
4. Připojte telefon datovým USB kabelem.
5. Odemkněte telefon a potvrďte důvěru v počítač a otisk RSA klíče.
6. Ověřte zařízení příkazem:

   ```bash
   npx cap run android --list
   ```

7. Spusťte `npx cap run android` a vyberte telefon, nebo jej vyberte v Android Studiu.

Na Windows může výrobce telefonu vyžadovat USB ovladač. Školní nebo firemní telefon může instalaci debug aplikace blokovat. Nikdy nevypínejte bezpečnostní ochrany zařízení kvůli tomuto cvičení.

Po dokončení můžete ladění USB opět vypnout a v možnostech pro vývojáře odvolat autorizace ladění.

## 16. Volitelně: Live Reload

Při častých úpravách lze spustit vývojový server a načítat změny v zařízení bez opakovaného balení webových souborů:

```bash
npx cap run android --live-reload
```

CLI spustí nebo použije vývojový server, nastaví dočasnou adresu pro běžící aplikaci a pokusí se propojit potřebný port. Konkrétní nabídka cíle a síťové chování závisí na zařízení a verzi nástrojů.

Live Reload používejte pouze při vývoji:

- aplikace závisí na běžícím serveru,
- počítač a fyzický telefon musí být síťově dosažitelné,
- nejde o ověření skutečně zabaleného buildu,
- před commitem nesmí v `capacitor.config.ts` zůstat `server.url`.

Povinnou kontrolu cvičení vždy dokončete statickým buildem:

```bash
ionic build
npx cap sync android
npx cap run android
```

## 17. Produkční kontrola a Git

Zavřete dočasné ladicí nástroje a v kořeni projektu spusťte:

```bash
npm run lint
ionic build
npx cap sync android
npx cap doctor android
git status
git diff --check
```

Pokud máte nastavený Prettier, spusťte také `npm run format` a `npm run format:check`.

Před commitem zkontrolujte:

- `capacitor.config.ts` neobsahuje `server.url`,
- dočasný titulek z experimentu je vrácený,
- `package.json` i `package-lock.json` obsahují `@capacitor/android`,
- adresář `android/` je sledovaný Gitem,
- nejsou přidané adresáře `build/`, `.gradle/` ani soubor `local.properties`.

Práci uložte:

```bash
git add capacitor.config.ts package.json package-lock.json android
git commit -m "Pridej nativni platformu Android"
git status
git log --oneline -3
```

Výsledný pracovní strom má být čistý. Nativní projekt commitujeme, protože může obsahovat záměrné změny manifestu, zdrojů, Gradle konfigurace a pluginů, které musí dostat i ostatní vývojáři.

## 18. Kontrolní seznam CV6

- [ ] Pracuji ve větvi `cv6/android` navazující na CV5.
- [ ] Mám nainstalované Android SDK Platform 36, potřebné SDK Tools a emulátor nebo fyzické zařízení.
- [ ] `capacitor.config.ts` obsahuje platný `appId`, název `CounterApp` a `webDir: 'www'`.
- [ ] `@capacitor/core`, `@capacitor/cli` a `@capacitor/android` používají verzi 8.5.0.
- [ ] Rozumím rozdílu mezi `ionic build`, `cap sync`, `cap run` a `cap open`.
- [ ] Adresář `android/` vznikl pomocí `npx cap add android` a vím, že `add` neopakuji.
- [ ] Aplikace se sestaví, nainstaluje a spustí na Androidu.
- [ ] Ruční počítadlo i navigace fungují na dotykové obrazovce.
- [ ] Záznam v historii zůstane po zavření a novém spuštění aplikace.
- [ ] OpenHolidays API vrátí data při dostupné síti.
- [ ] Při vypnuté síti se zobrazí chyba a po obnovení lze načtení zopakovat.
- [ ] Ověřil/a jsem aplikaci na výšku i na šířku.
- [ ] Umím najít proces aplikace v Logcatu.
- [ ] Umím otevřít běžící WebView přes `chrome://inspect` a najít HTTP požadavek.
- [ ] V konfiguraci nezůstalo `server.url` ani jiná dočasná změna.
- [ ] `npm run lint`, `ionic build`, `npx cap sync android` a `npx cap doctor android` skončí bez chyby.
- [ ] Nativní projekt je uložený v Git commitu a pracovní strom je čistý.

## 19. Bonusové úkoly

Po dokončení povinné části můžete:

1. změnit orientaci, velikost písma a světlý/tmavý režim zařízení a opravit nalezené problémy s rozvržením,
2. vytvořit druhé AVD s menším displejem a porovnat chování aplikace,
3. zjistit v Logcatu čas studeného startu aplikace,
4. najít debug APK vytvořený Gradlem a nainstalovat jej příkazem `adb install`,
5. upravit ikonu a splash screen pomocí nástroje `@capacitor/assets`,
6. přidat zobrazení informace, zda aplikace běží na webu nebo na nativní platformě, pomocí `Capacitor.getPlatform()`,
7. ověřit chování historie po **Clear storage** a po odinstalování aplikace,
8. spustit aplikaci na fyzickém telefonu a porovnat ji s emulátorem.

Release podepisování a publikování do Google Play nejsou součástí CV6. Debug APK není určené k distribuci koncovým uživatelům.

## 20. Nejčastější problémy

### `Web asset directory (./www) must contain an index.html file`

Neexistuje aktuální webový build. V kořeni projektu spusťte:

```bash
ionic build
npx cap sync android
```

Zkontrolujte také, že `webDir` v `capacitor.config.ts` odpovídá hodnotě `www`.

### `android platform has already been added`

Příkaz `npx cap add android` se spouští jen jednou. Existující platformu aktualizujte pomocí:

```bash
npx cap sync android
```

Nemažte celý adresář `android/`; mohli byste přijít o nativní konfiguraci.

### Aplikace nezobrazuje poslední změny

Android Studio používá kopii webových souborů z posledního syncu. Spusťte:

```bash
ionic build
npx cap sync android
```

Potom aplikaci znovu nasaďte. Neopravujte kopii v `android/app/src/main/assets/public/`.

### Android Studio hlásí chybu Gradle nebo JDK

V Android Studiu otevřete **Settings → Build, Execution, Deployment → Build Tools → Gradle** a pro **Gradle JDK** zvolte přiložený JetBrains Runtime. Potom spusťte **Sync Project with Gradle Files**.

Pokud terminál používá staré `JAVA_HOME`, spusťte sestavení přes Android Studio nebo nastavte cestu k podporovanému JDK podle instalace Android Studia. Nestahujte náhodnou verzi Gradlu ani ručně neměňte Gradle wrapper.

### SDK location not found

Otevřete Android projekt přes Android Studio a zkontrolujte **SDK Manager**. Android Studio vytvoří lokální soubor `android/local.properties` s cestou k SDK. Tento soubor necommitujte, protože cesta je specifická pro váš počítač.

### Nejsou přijaty licence Android SDK

V SDK Manageru nainstalujte požadované balíčky a potvrďte jejich licence. Pokud používáte nástroje z příkazové řádky a znáte cestu k SDK, lze licence potvrdit také nástrojem `sdkmanager --licenses`.

### Emulátor se nespustí nebo je velmi pomalý

Zkontrolujte volné místo, hardwarovou virtualizaci a doporučený systémový obraz pro architekturu počítače. Ve Windows má být dostupný Windows Hypervisor Platform; na Linuxu KVM; na macOS použijte doporučený ARM64 obraz pro Apple Silicon. Pokud grafika způsobuje černou obrazovku, v nastavení AVD zkuste softwarové vykreslování.

### `npx cap run android` nenabízí žádné zařízení

Spusťte AVD a počkejte na domovskou obrazovku. U fyzického telefonu odemkněte obrazovku a potvrďte ladění USB. Potom ověřte:

```bash
npx cap run android --list
```

### OpenHolidays v emulátoru nefunguje

Nejprve v prohlížeči emulátoru otevřete libovolnou HTTPS stránku. Pokud nefunguje ani ta, obnovte síť emulátoru. Pokud internet funguje, prohlédněte chybu v Logcatu a WebView DevTools. Neměňte URL na `http` a nevypínejte zabezpečení sítě.

### Historie po novém spuštění zmizela

Zkontrolujte, zda jste aplikaci pouze zavřeli, nebo odinstalovali či vymazali její data. Přeinstalování stejného debug buildu přes Run obvykle data zachová; odinstalace a **Clear storage** je odstraní. Také ověřte, že aplikace používá stejné `appId` jako při předchozím spuštění.

### WebView není v `chrome://inspect`

Používejte debug sestavení, ne release variantu. Aplikace musí být otevřená v popředí a zařízení připojené. U fyzického telefonu musí být potvrzeno ladění USB. Zavřete a znovu otevřete stránku `chrome://inspect/#devices`.

## Oficiální dokumentace

- [Capacitor – Android](https://capacitorjs.com/docs/android)
- [Capacitor – vývojový postup](https://capacitorjs.com/docs/basics/workflow)
- [Capacitor CLI – sync](https://capacitorjs.com/docs/cli/commands/sync)
- [Capacitor CLI – run](https://capacitorjs.com/docs/cli/commands/run)
- [Capacitor – Live Reload](https://capacitorjs.com/docs/guides/live-reload)
- [Android Developers – vytvoření a správa AVD](https://developer.android.com/studio/run/managing-avds)
- [Android Developers – spuštění aplikace v emulátoru](https://developer.android.com/studio/run/emulator)
- [Android Developers – spuštění na fyzickém zařízení](https://developer.android.com/studio/run/device)
- [Chrome DevTools – vzdálené ladění Android zařízení](https://developer.chrome.com/docs/devtools/remote-debugging/)
