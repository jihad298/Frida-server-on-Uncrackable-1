# Frida-server-on-Uncrackable-1
# Etapes importantes – OWASP UnCrackable Level 1 (Frida / Windows)

## 1. Installation des outils (Windows)

- Installer `platform-tools` (ADB) depuis Android :  
  [https://developer.android.com/tools/releases/platform-tools](https://developer.android.com/tools/releases/platform-tools)  
- Ajouter `C:\platform-tools` au PATH (ou lancer `adb` depuis ce dossier).  
- Installer Python 3.8+ puis :

```bash
pip install --upgrade frida frida-tools
```

- Vérification :

```bash
frida --version
frida-ps --version
adb devices
```
<img width="1221" height="283" alt="image" src="https://github.com/user-attachments/assets/70037349-bce2-4d00-94b5-f5b04354338b" />


---

## 2. Préparation de l’appareil (Android / émulateur)

- Activer **Options développeur + Débogage USB** sur l’appareil.  
- Vérifier la connexion :

```bash
./adb devices
```
<img width="871" height="117" alt="image" src="https://github.com/user-attachments/assets/354e8cac-1616-4d98-9a4c-8242cdae9801" />

> Résultat attendu : l’appareil apparaît avec `device` (pas `unauthorized`).

---

## 3. Déploiement de frida‑server sur Android

- Identifier l’architecture CPU :

```bash
./adb shell getprop ro.product.cpu.abi
```

- Télécharger depuis [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases) le `frida-server-<version>-android-<arch>.xz` correspondant.  
- Décompresser avec 7‑Zip (Windows) pour obtenir `frida-server`.  
- Copier sur l’appareil :

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
```

- Lancer le serveur :

```bash
adb shell /data/local/tmp/frida-server -l 0.0.0.0
```

- Vérifier :

```bash
adb shell ps | grep frida
```

- Rediriger les ports :

```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

---

## 4. Vérification de la connexion Frida

Dans PowerShell :

```bash
frida-ps -U
frida-ps -Uai
```

> Résultat attendu : liste des applications Android.

---

## 5. Injection minimale sur UnCrackable1

- Installer `UnCrackable1.apk` sur l’appareil.  
- Identifier le package (souvent `owasp.mstg.uncrackable1`).
  <img width="690" height="55" alt="1" src="https://github.com/user-attachments/assets/2cbd7268-37d2-422d-8715-f72059e04991" />


Créer `hello.js` :

```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");
});
```

Puis lancer :

```bash
frida -U -f owasp.mstg.uncrackable1 -l hello.js
```
Erreur !
Root detected

l'App crashe avant que le script s’injecte correctement
<img width="782" height="188" alt="erreur root" src="https://github.com/user-attachments/assets/9c26b2f2-a8af-43e3-be00-515a2b16dfec" />
<img width="570" height="266" alt="root" src="https://github.com/user-attachments/assets/3cbf046d-b42a-4eb3-8963-e332e061b6fa" />
<img width="149" height="210" alt="w" src="https://github.com/user-attachments/assets/5cd73b38-fb5c-4b90-b34d-4f9a0995e101" />


Pour contourner ces protections, nous utilisons Frida afin d’injecter dynamiquement un script JavaScript (bypass.js) au moment de l’exécution.

```bypass.js
Java.perform(function(){
  let C0002c = Java.use("sg.vantagepoint.a.c");
  C0002c["a"].implementation = function () {
    console.log('a is called');
    let ret = this.a();
    console.log('a ret value is ' + ret);
    return false;
  };
});
```

<img width="1581" height="107" alt="image" src="https://github.com/user-attachments/assets/44ddb7bf-f0ba-4a47-8d28-370abbe118f2" />

<img width="1469" height="114" alt="image" src="https://github.com/user-attachments/assets/776adb36-d485-40e3-9ac4-05704cc46eaf" />

> Résultat attendu : message `[+] Frida Java.perform OK` + app qui démarre.

---

## 6. Test de hook natif (simple)

Créer `hello_native.js` :

```javascript
console.log("[+] Script chargé");

Interceptor.attach(Module.getExportByName(null, "recv"), {
  onEnter(args) {
    console.log("[+] recv appelée");
  }
});
```

Lancer :

```bash
frida -U -n "owasp.mstg.uncrackable1" -l hello_native.js
```

> Résultat attendu :  
- `[+] Script chargé` au chargement.  

---
## 7. Etape 6: Explorer la console interactive Frida (Analyse sécurité)

### 6.1 Vérifier l’architecture du processus

```js
Process.arch
```

Permet d’identifier l’architecture (`arm`, `arm64`, `x64`).
➡️ Important pour adapter les outils et comprendre l’environnement natif.

---

### 6.2 Identifier le module principal

```js
Process.mainModule
```

Retourne les informations sur le module principal.
➡️ Utile pour localiser le point d’entrée natif.

---

### 6.3 Inspecter une bibliothèque système critique

```js
Process.getModuleByName("libc.so")
```

Affiche les informations de `libc.so`.
➡️ Contient des fonctions essentielles (réseau, mémoire, fichiers).

---

### 6.4 Vérifier des fonctions sensibles

```js
Process.getModuleByName("libc.so").getExportByName("recv")
```

Fonctions :

* `connect`
* `send`
* `recv`
* `open`
* `read`

➡️ Intérêt sécurité :

* Surveillance réseau (`send`, `recv`, `connect`)
* Accès fichiers (`open`, `read`)

---

### 6.5 Lister les bibliothèques chargées

```js
Process.enumerateModules()
```

➡️ Permet de :

* repérer les librairies natives
* identifier crypto (`libssl.so`, `libcrypto.so`)
* détecter des librairies spécifiques à l’application

---

### 6.6 Lister les threads actifs

```js
Process.enumerateThreads()
```

### 6.7 Examiner la mémoire du processus

```js
Process.enumerateRanges('r-x')
```

### 6.8 Vérifier le runtime Java

```js
Java.available
```

### 6.9 Énumérer les classes Java

```js
Java.perform(function () {
  Java.enumerateLoadedClasses({
    onMatch: function (name) {
      if (name.indexOf("uncrackable") !== -1) console.log(name);
    },
    onComplete: function () {
      console.log("Fin de l'énumération");
    }
  });
});
```
✅ Classes identifiées :

sg.vantagepoint.uncrackable1.MainActivity
sg.vantagepoint.uncrackable1.MainActivity$1

---

### 6.10 Identifier les librairies crypto / TLS

```js
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1
)
```

✅ Résultat :

libcrypto.so → /apex/com.android.conscrypt/lib64/
libssl.so → /apex/com.android.conscrypt/lib64/
---

## Étape 7 — Instrumentation des fonctions natives

🔗 Hook des connexions réseau

```
const connectPtr = Process.getModuleByName("libc.so").getExportByName("connect");

Interceptor.attach(connectPtr, {
  onEnter(args) {
    console.log("[+] connect appelée — fd = " + args[0]);
  },
  onLeave(retval) {
    console.log("retour = " + retval.toInt32());
  }
});

```

📡 Surveillance send / recv
```
const sendPtr = Process.getModuleByName("libc.so").getExportByName("send");
const recvPtr = Process.getModuleByName("libc.so").getExportByName("recv");

Interceptor.attach(sendPtr, {
  onEnter(args) {
    console.log("[+] send — taille = " + args[2].toInt32());
  }
});

Interceptor.attach(recvPtr, {
  onEnter(args) {
    console.log("[+] recv demandé = " + args[2].toInt32());
  },
  onLeave(retval) {
    console.log("recv retourne = " + retval.toInt32());
  }
});

```
📁 Accès aux fichiers
```
const openPtr = Process.getModuleByName("libc.so").getExportByName("open");

Interceptor.attach(openPtr, {
  onEnter(args) {
    console.log("[+] open : " + args[0].readUtf8String());
  }
});
```
## ☕ Étape 8 — Hooks Java (analyse applicative)
📦 SharedPreferences
```
Java.perform(function () {
  var Impl = Java.use("android.app.SharedPreferencesImpl");

  Impl.getString.overload("java.lang.String", "java.lang.String")
    .implementation = function (key, defValue) {
      var result = this.getString(key, defValue);
      console.log("[Prefs] " + key + " => " + result);
      return result;
    };
});
```
```
🗄️ SQLite
Java.perform(function () {
  var SQLiteDatabase = Java.use("android.database.sqlite.SQLiteDatabase");

  SQLiteDatabase.rawQuery.overload("java.lang.String", "[Ljava.lang.String;")
    .implementation = function (sql, args) {
      console.log("[SQLite] " + sql);
      return this.rawQuery(sql, args);
    };
});
```

🐞 Détection du debugger
```
Java.perform(function () {
  var Debug = Java.use("android.os.Debug");

  Debug.isDebuggerConnected.implementation = function () {
    var result = this.isDebuggerConnected();
    console.log("[Debug] => " + result);
    return result;
  };
});
```

⚙️ Commandes système
```
Java.perform(function () {
  var Runtime = Java.use("java.lang.Runtime");

  Runtime.exec.overload("java.lang.String").implementation = function (cmd) {
    console.log("[Runtime.exec] " + cmd);
    return this.exec(cmd);
  };
});
```

📂 Manipulation de fichiers Java

```
Java.perform(function () {
  var File = Java.use("java.io.File");

  File.$init.overload("java.lang.String").implementation = function (path) {
    console.log("[File] chemin : " + path);
    return this.$init(path);
  };
});
```
