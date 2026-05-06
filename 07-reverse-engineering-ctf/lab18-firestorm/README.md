# 🔥 Lab 18 — FireStorm CTF

### Rapport de Reverse Engineering Android — Firebase Challenge

---

## 📋 Informations générales

| Champ | Valeur |
|-------|--------|
| **Application** | FireStorm.apk |
| **Package** | com.pwnsec.firestorm |
| **Niveau** | Medium |
| **Outils** | JADX 1.5.5, Frida 16.0.19, Python 3, pyrebase4 |
| **Flag** | `PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}` |
| **Auteur** | Hiba Chagdaly |

---

## 🎯 Objectif

Une méthode `Password()` génère dynamiquement un mot de passe Firebase mais **n'est jamais appelée** dans le flux normal de l'app. On doit :
1. La trouver avec JADX
2. La forcer avec Frida
3. S'authentifier sur Firebase avec Python
4. Lire le flag dans la base de données

---

## 📸 Phase 1 — Préparation

![01_adb_install_launch](screenshots/01_adb_install_launch.png)

```bash
adb connect 169.254.135.101:5555
adb install FireStorm.apk
adb shell pm list packages | grep -i fire
# → package:com.pwnsec.firestorm
adb shell am start -n com.pwnsec.firestorm/.MainActivity
```

![02_app_launch](screenshots/02_app_launch.png)

---

## 🔍 Phase 2 — Analyse statique JADX

```bash
jadx-gui ~/lab18/FireStorm.apk &
```

### APK ouvert dans JADX
![03_jadx_opened](screenshots/03_jadx_opened.png)

### Méthode Password() — jamais appelée
![04_jadx_password_method](screenshots/04_jadx_password_method.png)

La méthode `Password()` construit le mot de passe en combinant des strings de `strings.xml` et une partie générée par `generateRandomString()`. Elle n'est appelée nulle part dans l'application.

### IDs R.string
![05_jadx_r_string_ids](screenshots/05_jadx_r_string_ids.png)

### Arbre MainActivity
![06_jadx_mainactivity_tree](screenshots/06_jadx_mainactivity_tree.png)

### strings.xml — Configuration Firebase
![07_strings_xml](screenshots/07_strings_xml.png)

| Clé | Valeur |
|-----|--------|
| `firebase_api_key` | `AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY` |
| `firebase_email` | `TK757567@pwnsec.xyz` |
| `firebase_database_url` | `https://firestorm-9d3db-default-rtdb.firebaseio.com` |
| `IDKMaybethepasswordpassowrd` | `v1n4of.5EY?%0z` *(faux — généré dynamiquement)* |

---

## 🪝 Phase 3 — Script Frida

### Création du script
![08_frida_script](screenshots/08_frida_script.png)

```javascript
Java.perform(function() {
    function getPassword() {
        Java.choose('com.pwnsec.firestorm.MainActivity', {
            onMatch: function(instance) {
                var pass = instance.Password();
                console.log("[+] Mot de passe Firebase : " + pass);
            },
            onComplete: function() {}
        });
    }
    setTimeout(getPassword, 3000);
});
```

```bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
```

### Résultat — Mot de passe obtenu
![09_frida_password_output](screenshots/09_frida_password_output.png)

> **Mot de passe : `C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC`**

---

## 🐍 Phase 4 — Script Python + Flag

```python
import pyrebase

config = {
    "apiKey": "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY",
    "authDomain": "firestorm-9d3db.firebaseapp.com",
    "databaseURL": "https://firestorm-9d3db-default-rtdb.firebaseio.com",
    "storageBucket": "firestorm-9d3db.appspot.com",
    "projectId": "firestorm-9d3db"
}

firebase = pyrebase.initialize_app(config)
auth = firebase.auth()

user = auth.sign_in_with_email_and_password(
    "TK757567@pwnsec.xyz",
    "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"
)

db = firebase.database()
flag_data = db.get(user['idToken'])
print("FLAG :", flag_data.val())
```

```bash
pip install pyrebase4 --break-system-packages
python3 get_flag.py
```

### Flag récupéré ✅
![10_python_flag](screenshots/10_python_flag.png)

---

## 🎉 Flag
