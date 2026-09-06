# Empfehlung: Vereinfachung der GCC Linking-Konvention

## Problem

Die aktuelle Namenskonvention für Bibliotheken in GCC ist **unintuitive, fehleranfällig und historisch überholt**:

```bash
# Aktuell (verwirrend)
gcc hello.c -lhelper -o hello
# Sucht nach: libhelper.a (nicht offensichtlich!)
```

Diese Konvention:
- ❌ Ist nicht selbsterklärend
- ❌ Verursacht unnötige Debugging-Zeit (Fehler werden nicht gefunden, warum?)
- ❌ Benötigt Suchpfade und magische Namensgebung
- ❌ Ist für Anfänger extrem frustrierend
- ❌ Existiert nur aus historischen Gründen (1970er/80er Speicherersparnisse)

---

## Empfohlene Lösung

**Nur explizite Dateipfade zulassen:**

```bash
# Neu (klar und direkt)
gcc hello.c ./libs/libhelper.a ./libs/libmath.a -o hello
```

**Vorteile:**
- ✅ Sofort klar: welche Datei wird verwendet
- ✅ Keine versteckten Suchpfade
- ✅ Keine Namenskonventionen nötig
- ✅ Fehler sind sofort erkennbar
- ✅ Funktioniert mit beliebigen Dateinamen
- ✅ Modern und benutzerfreundlich (wie Rust, Go, Python)

---

## Umsetzung

Die `-l` und `-L` Flags sollten **entfernt oder deprecated** werden.

Entwickler sollten gezwungen sein, **explizite Pfade zu Bibliotheken anzugeben**.

---

## Begründung

Diese Änderung würde:
1. **Fehlerquellen reduzieren** – weniger mysteriöse Linking-Fehler
2. **Anfängern helfen** – viel intuitivere Arbeitsweise
3. **Wartbarkeit verbessern** – klar, welche Bibliotheken wo sind
4. **Mit moderner Praxis übereinstimmen** – andere Sprachen machen es besser

---

**Status:** Dringende Empfehlung für GCC-Entwickler
